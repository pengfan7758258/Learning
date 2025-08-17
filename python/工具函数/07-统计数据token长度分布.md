```python
from transformers import AutoTokenizer
import json
from pathlib import Path
import numpy as np
from tqdm import tqdm

# 你的模型路径
model_path = "models/Qwen2.5-0.5B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_path, use_fast=True)

# 假设你有一个 JSON 格式的数据集，形如 [{"instruction": "...", "output": "..."}]
# 修改成你自己数据集的路径
data_path = Path("LLaMA-Factory/data/alpaca_zh_demo.json")

with open(data_path, "r", encoding="utf-8") as f:
	dataset = json.load(f)

lengths = []
for item in tqdm(dataset, desc="Tokenizing"):
	text = item.get("instruction", "") + "\n" + item.get("output", "")
	tokens = tokenizer.encode(text, add_special_tokens=True)
	lengths.append(len(tokens))

lengths = np.array(lengths)

print(f"样本总数: {len(lengths)}")
print(f"平均长度: {int(lengths.mean())}")
print(f"中位数: {int(np.median(lengths))}")
print(f"最小长度: {lengths.min()}")
print(f"最大长度: {lengths.max()}")
print(f"95% 分位数: {int(np.percentile(lengths, 95))}") # 找到lengths list中95%的值小于等于该值
print(f"99% 分位数: {int(np.percentile(lengths, 99))}") # 找到lengths list中99%的值小于等于该值

  
## 这个图看起来并不明朗
# import matplotlib.pyplot as plt
# # 画图
# plt.figure(figsize=(10, 6))
# plt.hist(lengths, bins=50, color="skyblue", edgecolor="black", alpha=0.7)
# plt.axvline(int(np.percentile(lengths, 95)), color="red", linestyle="--", label="95% percentile")
# plt.axvline(int(np.percentile(lengths, 99)), color="orange", linestyle="--", label="99% percentile")
# plt.axvline(int(lengths.mean()), color="green", linestyle="-.", label="mean")
# plt.title("datasets Token Length distribution", fontsize=16)
# plt.xlabel("Token length", fontsize=14)
# plt.ylabel("Sample number", fontsize=14)
# plt.legend()
# plt.grid(alpha=0.3)
# plt.tight_layout()
# plt.show()
```