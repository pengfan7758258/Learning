```python
import tiktoken
from math import ceil
from PIL import Image

model_name="gpt-4o"
ENCODING = tiktoken.encoding_for_model(model_name)

def get_tokens(text: str, encoding: tiktoken.Encoding):
    """计算text消耗token数"""
    return len(encoding.encode(text))

def calc_image_tokens(image_path: str):
    """
    计算一张图像消耗的token数量
    """
    with open(image_path, "rb") as f:
        width, height = Image.open(f).size

    #如果图像任一维度超过 1024 像素，则按比例缩放至不超过 1024；
    #这是因为多模态模型对图像大小是有限制的，例如 OpenAI 的 GPT-4o 支持的图像最大边为 1024。
    if width > 1024 or height > 1024:
        if width > height:
            height = int(height * 1024 / width)
            width = 1024
        else:
            width = int(width * 1024 / height)
            height = 1024

    # 模型内部将图像以 512x512 为单位进行切分
    h = ceil(height / 512)
    w = ceil(width / 512)
    # 基础token85 每个512x512消耗170个token 有h*w个512x512块
    tokens = 85 + 170 * h * w
    return tokens
    
print(get_tokens("你无敌了，你可以飞上天", ENCODING))
# output: 9
```