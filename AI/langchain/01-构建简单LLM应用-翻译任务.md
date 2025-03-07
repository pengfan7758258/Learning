[地址](https://python.langchain.com/docs/tutorials/llm_chain/)

<mark style="background: #BBFABBA6;">安装：</mark>
- pip install -qU "langchain[openai]"

<mark style="background: #BBFABBA6;">下面是一个英文翻译意大利的任务：</mark>
```python
import os
os.environ["OPENAI_API_KEY"] = "your openai_api_key"

# 1.初始化模型
from langchain.chat_models import init_chat_model

model = init_chat_model(
	"gpt-4o-mini", # 指定模型名字
	model_provider="openai" # 模型提供者
)

# 2.创建messages
from langchain_core.messages import HumanMessage, SystemMessage

messages = [
    SystemMessage("Translate the following from English into Italian"),
    HumanMessage("hi!"),
]

# 3.传入messages到model
model.invoke(messages)
# AIMessage(content='Ciao!', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 4, 'prompt_tokens': 20, 'total_tokens': 24, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_06737a9306', 'finish_reason': 'stop', 'logprobs': None}, id='run-ab7af498-589a-45db-b394-cf5e4541c4e0-0', usage_metadata={'input_tokens': 20, 'output_tokens': 4, 'total_tokens': 24, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})
"""
AIMessage属性：
content：输出结果
id：本次运行线程id名
usage_metadata：含有输入token数、输出token数、输入和输出总共token数
"""
```
<mark style="background: #BBFABBA6;">三种模型调用的等效方式：</mark>
```python
# 第一种
model.invoke("Hello")
# 第二种
model.invoke([{"role": "user", "content": "Hello"}])
# 第三种
model.invoke([HumanMessage("Hello")])
```
<mark style="background: #BBFABBA6;">异步、流式输出：</mark>
```python
for token in model.stream(messages):
    print(token.content, end="|")
"""
输出：|C|iao|!||
"""
```
<mark style="background: #BBFABBA6;">prompt templates：和字符串占位符一个原理</mark>
```python
from langchain_core.prompts import ChatPromptTemplate
# 定义system template
system_template = "Translate the following from English into {language}"
# 定义整个prompt template
prompt_template = ChatPromptTemplate.from_messages(
    [("system", system_template), ("user", "{text}")] # 有两个变量：language和text
)

# 动态填充prompt
prompt = prompt_template.invoke({"language": "Italian", "text": "hi!"})
# 和开始的messages列表一样，就是SystemMessage和HumanMessage
# ChatPromptValue(messages=[SystemMessage(content='Translate the following from English into Italian', additional_kwargs={}, response_metadata={}), HumanMessage(content='hi!', additional_kwargs={}, response_metadata={})])

prompt.to_messages()
# [SystemMessage(content='Translate the following from English into Italian', additional_kwargs={}, response_metadata={}), HumanMessage(content='hi!', additional_kwargs={}, response_metadata={})]

response = model.invoke(prompt)
print(response.content) # 输出：Ciao!
```