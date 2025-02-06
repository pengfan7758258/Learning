### token压缩率

自己训练的tokenizer可能token转化压缩率不够好
压缩率的意义：
- 压缩率是衡量分词器性能的一个重要指标。其核心思想是：  **分词器生成的 token 数量越少，压缩率越高，表示分词器能够高效地表示原始文本信息。**

![[02-tokenizer训练.png]]
![[02-tokenizer训练-4.png]]
![[02-tokenizer训练-1.png]]

对于中文，原始字符是天然的分隔单位（每个汉字独立），因此理想情况下，token 数不会超过原始字符数,但由于分词器的编码规则、特殊符号、子词拆分等机制，某些情况下 token 数可能会超过原始字符数。所以在设计分词器时，合理平衡词表大小和覆盖率，避免过度拆分，才能在压缩率和灵活性之间取得良好的折中。
![[02-tokenizer训练-2.png]]


### vocab size 词表大小的选择

因为LLM体积非常小，为了避免模型头重脚轻（词嵌入embedding层参数占整个LLM比太高），所以词表长度需要选择比较小。

强大的开源模型例如01万物、千问、chatglm、mistral、Llama3等，它们的tokenizer词表长度如下：
![[02-tokenizer训练-5.png]]


### 训练方法
#### 一.tokenizers库
1. pip install tokenizers\==0.19.1
2. 准备数据集：tokenizer_train.jsonl，一个字段text，纯文本，每行是一个独立的JSON对象
![[01-数据集-5.png]]
3. 关键代码
**train:**
```python
# 1.模块导入部分
from tokenizers import (
    decoders,        # 解码器模块，用于将token IDs转换回文本
    models,          # 包含不同分词模型（如BPE/WordPiece）
    pre_tokenizers,  # 预分词规则（如按空格/标点分割）
    trainers,        # 不同模型的训练器
    Tokenizer,       # 主分词器类
)

# 2.初始化分词器
tokenizer = Tokenizer(models.BPE())  # 创建基于BPE算法的分词器

# 3.预分词设置
"""
- `ByteLevel`：将文本视为UTF-8字节序列，处理所有Unicode字符，解码时会将字节转换回UTF-8字符
- `add_prefix_space`：设为False时，"hello"直接处理；设为True时会变成" hello"（这里有空格前缀）
"""
tokenizer.pre_tokenizer = pre_tokenizers.ByteLevel(
    add_prefix_space=False  # 不在文本开头添加空格（影响##前缀处理）
)

# 4.特殊标记定义
"""
- `<unk>`：未知token（必须存在）
- `<s>`：句子开始（如用于文本生成）
- `</s>`：句子结束
"""
special_tokens = ["<unk>", "<s>", "</s>"]  # 必须包含的三个特殊token

# 5.训练器配置
"""
- `initial_alphabet`：包含ByteLevel预定义的所有256个字节字符，确保基础字符被优先包含
"""
trainer = trainers.BpeTrainer(
    vocab_size=6400,               # 目标词汇表大小（包含特殊token <unk>|<s>|s</s>）
    special_tokens=special_tokens, # 强制特殊token加入词汇表
    show_progress=True,            # 显示训练进度条
    initial_alphabet=pre_tokenizers.ByteLevel.alphabet()  # 初始字符集
)

# 6.定义读取JSONL文件的生成器函数
def read_texts_from_jsonl(file_path):
    with open(file_path, 'r', encoding='utf-8') as f:  # 打开文件，指定UTF-8编码
        for line in f:  # 逐行读取文件
            data = json.loads(line)  # 将每行JSON字符串解析为Python字典
            yield data['text']  # 生成器返回字典中的'text'字段

# 7.调用生成器函数读取数据
texts = read_texts_from_jsonl(data_path)

# 8.使用迭代器训练分词器
tokenizer.train_from_iterator(texts, trainer=trainer)

# 9.设置解码器
"""
- **作用**：指定将token IDs解码回文本时使用`ByteLevel`解码器（与之前的`ByteLevel`预分词器匹配）。
- **必要性**：必须设置，否则解码时无法正确处理字节级编码的token（如`##`前缀）。
"""
tokenizer.decoder = decoders.ByteLevel()

# 10.检查特殊token索引
assert tokenizer.token_to_id("<unk>") == 0
assert tokenizer.token_to_id("<s>") == 1
assert tokenizer.token_to_id("</s>") == 2

# 11.保存分词器
"""
保存分词器的主配置和模型文件
"""
tokenizer_dir = "./model/minimind_tokenizer"
os.makedirs(tokenizer_dir, exist_ok=True)
tokenizer.save(os.path.join(tokenizer_dir, "tokenizer.json"))
tokenizer.model.save("./model/minimind_tokenizer")

# 12.如果需要与Hugging Face Transformers库集成（如使用`AutoTokenizer`加载）
from transformers import PreTrainedTokenizerFast

# 转换为Transformers兼容的tokenizer
hf_tokenizer = PreTrainedTokenizerFast(
    tokenizer_object=tokenizer,
    bos_token="<s>",
    eos_token="</s>",
    unk_token="<unk>"
)

# 添加模板（例如使用Llama2的对话格式）
hf_tokenizer.chat_template = "{% if messages[0]['role'] == 'system' %}{{ messages[0]['content'] }}{% endif %}{% for message in messages %}{% if message['role'] == 'user' %}<s>user\n{{ message['content'] }}</s>\n<s>assistant\n{% else %}{{ message['content'] }}</s>\n{% endif %}{% endfor %}"

# 自动生成完整配置
hf_tokenizer.save_pretrained(tokenizer_dir)

```
**eval:**
```python
# 1.加载自定义分词器
tokenizer = AutoTokenizer.from_pretrained("./model/minimind_tokenizer")

# 2.定义聊天消息
messages = [
    {"role": "system", "content": "你是一个优秀的聊天机器人，总是给我正确的回应！"},
    {"role": "user", "content": '你来自哪里？'},
    {"role": "assistant", "content": '我来自地球'}
]

# 3.#### 应用聊天模板
new_prompt = tokenizer.apply_chat_template(
    messages,
    tokenize=False  # 不进行实际分词
)
print(new_prompt)

# 4.打印实际词汇表大小
actual_vocab_size = len(tokenizer)
print('tokenizer实际词表长度：', actual_vocab_size)  # 输出：6400 - 训练指定的

# 5.编码与解码验证
model_inputs = tokenizer(new_prompt)  # 编码
input_ids = model_inputs['input_ids']
response = tokenizer.decode(input_ids)
print('decoder和原始文本是否一致：', response == new_prompt) # 如果为False，找到原因
"""
False可能多原因:
1.add_prefix_space训练为False，token_config.json为True
2.在解码时关闭清理空格:s
response = tokenizer.decode(input_ids, clean_up_tokenization_spaces=False)
"""
```
