```python
"""
功能：支持image和pdf的上传，用ocr提取内容。ocr用的是paddleocr-vl
配置：4090 24G显存
----------------------------------------------------------------
csdn参考：https://blog.csdn.net/fufan_LLM/article/details/153832353

安装：
conda create -n ocr_rag python==3.11
conda activate ocr_rag
python -m pip install paddlepaddle-gpu==3.2.0 -i https://www.paddlepaddle.org.cn/packages/stable/cu126/

验证：
python
import paddle
paddle.utils.run_check()
# 出现"PaddlePaddle is installed successfully!"就是成功。
python -m pip install https://paddle-whl.bj.bcebos.com/nightly/cu126/safetensors/safetensors-0.6.2.dev0-cp38-abi3-linux_x86_64.whl
# 安装PaddleOCR所有功能依赖
pip install "paddleocr[all]"
pip install gradio gradio-pdf

模型下载：
pip install modelscope
编辑vim download_paddleocr_vl.py：
	from modelscope import snapshot_download
	# 下载完整模型（包含 PaddleOCR-VL-0.9B 和 PP-DocLayoutV2）
	model_dir = snapshot_download('PaddlePaddle/PaddleOCR-VL', local_dir='./PaddleOCR-VL-0.9B')
执行下载：python download_paddleocr_vl.py
下载成功后检查文件：PP-DocLayoutV2应该是直接在当前目录下，但是其它文件散乱，我们`mkdir PaddleOCR-VL-0.9B`，将其余文件放入PaddleOCR-VL-0.9B中


------------------------------------
b站：https://www.bilibili.com/video/BV1XpyqBMEE6/?spm_id_from=333.337.search-card.all.click&vd_source=e4234f5ddbe45b813cf4296e06e14b9b

参考魔搭社区的说明文档就能安装
我是用一张4090显卡，pdf扫描件900多页，不用迭代器时显存直接打满，看不到输出，分不清是否在正常工作
如果要一次性导出很长的pdf扫描件，一定要使用迭代器，可以随时监控进度
使用迭代器后大约占显存20G左右，能稳定输出，隔一会儿就一次性输出几页

from paddleocr import PaddleOCRVL
pipeline = PaddleOCRVL()
output = pipeline.predict_iter(".xxx.pdf")
for res in output:
	res.print()
	res.save_to_json(save_path="output-json")
	res.save_to_markdown(save_path="output-markdown")

  
从导出结果来看，识别率高于pdfplumber
能正确解析图片、表格
能过滤掉页眉页脚

但是生成的markdown标题层次与实际目录树不符，不能依此构建目录树
生成JSON中也没有足够的支持
---------------------------------------
cmd命令：
paddleocr doc_parser \
	--input /root/autodl-tmp/page_8.png \
	--save_path ./output \
	--vl_rec_model_dir /root/autodl-tmp/PaddleOCR-VL-0.9B \
	--layout_detection_model_dir /root/autodl-tmp/PP-DocLayoutV2
----------------------------------------------------
"""

import re
from paddleocr import PaddleOCRVL
import gradio as gr
from gradio_pdf import PDF
import mimetypes
import tempfile
import os

pipeline = PaddleOCRVL(
	layout_detection_model_dir="/root/autodl-tmp/PP-DocLayoutV2",
	vl_rec_model_dir="/root/autodl-tmp/PaddleOCR-VL-0.9B"
)

def paddle_ocr(file_path):
	output = pipeline.predict_iter(file_path)
	for idx,res in enumerate(output):
		if idx > 12: # 只用前12页，快速测试
			break
		# 剔除html div标签
		cleaned = re.sub(r"<div.*?</div>", "", res.markdown["markdown_texts"], flags=re.DOTALL)
		yield cleaned

def show_preview(file):
	if not file:
		return gr.update(visible=False), gr.update(visible=False)
	
	file_path = file.name if hasattr(file, "name") else file
	mime_type, _ = mimetypes.guess_type(file_path)
	
	if mime_type and mime_type.startswith("image/"):
		return gr.update(value=file_path, visible=True), gr.update(visible=False)
	elif mime_type == "application/pdf":
		# PDF 组件需要传文件路径
		return gr.update(visible=False), gr.update(value=file_path, visible=True)
	else:
		return gr.update(visible=False), gr.update(visible=False)

# Parse 按钮回调（流式输出）
def parse_file(file, all_pages):
	if not file:
		yield gr.update(value="请先上传图片或PDF！"), gr.update(visible=False), gr.update(visible=False)
		return
	file_path = file.name if hasattr(file, "name") else file
	
	page_results = []
	for idx, page_text in enumerate(paddle_ocr(file_path), start=1):
		page_results.append(page_text)
		# 每次 yield：更新当前页 + slider + state
		yield (
			gr.update(value=page_text), # 当前页结果
			gr.update(visible=True, maximum=idx, value=idx), # 更新页码 slider
			page_results.copy() # 注意，这里直接返回 Python 列表
		)
	
	# 所有页都处理完毕后再 yield 一次最终状态
	if len(page_results) == 1:
		yield (
		gr.update(value=page_results[0]),
		gr.update(visible=False),
		gr.update(visible=False)
	)

# 翻页回调
def show_page(page_num, all_texts):
	# print(f"翻到第 {page_num} 页，共 {len(all_texts)} 页") # 调试信息
	if not all_texts:
		return "暂无内容"
	index = int(page_num) - 1
	return all_texts[index] if 0 <= index < len(all_texts) else "越界啦"

# 🆕 新增：导出 Markdown
def export_markdown(all_texts):
	if not all_texts:
		return None # 没内容时返回空
	
	md_content = "\n\n-------------------------------------\n\n".join(all_texts)
	# 生成临时 markdown 文件
	tmp_path = os.path.join(tempfile.gettempdir(), "ocr_result.md")
	with open(tmp_path, "w", encoding="utf-8") as f:
		f.write(md_content)
	return tmp_path # 返回路径给 Gradio 自动提供下载

  

with gr.Blocks() as demo:
	with gr.Row():
	# 左侧：文件上传 & 预览
		with gr.Column(scale=1):
			file_input = gr.File(label="上传图片或PDF", file_types=["image", ".pdf"])
			image_preview = gr.Image(label="图片预览", visible=False)
			pdf_preview = PDF(label="PDF 预览", interactive=True, visible=False)
			parse_button = gr.Button("Parse") # OCR 触发按钮
		
		# 右侧：OCR 结果展示
		with gr.Column(scale=2):
			result_box = gr.Textbox(
				label="OCR 结果",
				interactive=True,
				lines=20,
				placeholder="这里显示OCR识别结果..."
			)
			
			page_slider = gr.Slider(
				minimum=1,
				maximum=1,
				step=1,
				label="页码",
				visible=False
			)
			all_pages_state = gr.State([]) # 存储所有页的识别结果
			export_button = gr.Button("导出 Markdown")
			export_file = gr.File(label="下载识别结果", visible=False)

	# 文件上传时显示预览
	file_input.change(show_preview, inputs=file_input, outputs=[image_preview, pdf_preview])
	
	# 点击 Parse 按钮执行 OCR
	# 注意：开启 streaming（Gradio 会检测到 yield 自动流式刷新）
	parse_button.click(
		parse_file,
		inputs=[file_input, all_pages_state],
		outputs=[result_box, page_slider, all_pages_state]
	)
	
	# 拖动滑块切换页码
	page_slider.change(show_page, inputs=[page_slider, all_pages_state], outputs=result_box)
	
	# 🆕 新增：导出按钮点击事件
	export_button.click(
		export_markdown,
		inputs=all_pages_state,
		outputs=export_file,
	)

  
demo.launch()

# # ocr功能测试
# file_path = "extracted_images/page_9.png"
# paddle_ocr(file_path)
```