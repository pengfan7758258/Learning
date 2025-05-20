- [地址](https://python.langchain.com/docs/tutorials/extraction/)
- 从非结构化的data中提取结构化data
- 使用few shot prompt提高性能

<mark style="background: #FFB86CA6;">schema：pydantic定义提取字段和字段信息</mark>
```python
from typing import Optional
from pydantic import BaseModel, Field

class Person(BaseModel):
    """Information about a person."""
    # 重点:
    # 1. 每个field设置为`optional`、`default=None` -- 允许模型不提取，防止模型自己造数据
    # 2. 每个field有`description` -- 这个描述会给LLM.
    # 好的description可以提高extraction results.
    name: Optional[str] = Field(default=None, description="The name of the person")
    hair_color: Optional[str] = Field(
        default=None, description="The color of the person's hair if known"
    )
    height_in_meters: Optional[str] = Field(
        default=None, description="Height measured in meters"
    )
```
<mark style="background: #FFB86CA6;">extractor：提取器</mark>
```python
from langchain.chat_models import init_chat_model
from langchain_core.prompts import ChatPromptTemplate
# 1.prompt构建
prompt_template = ChatPromptTemplate.from_messages(  
	[  
		(  
			"system",  
			"You are an expert extraction algorithm. "  
			"Only extract relevant information from the text. "  
			"If you do not know the value of an attribute asked to extract, "  
			"return null for the attribute's value.",  
		),   
			("human", "{text}"),  
	]  
)

# 2.初始化模型并添加输出结构约束
llm = init_chat_model("gpt-4o-mini", model_provider="openai").with_structured_output(schema=Person)

# 3.测试
text = "Alan Smith is 6 feet tall and has blond hair."
prompt = prompt_template.invoke({"text": text})
llm.invoke(prompt)
# output:Person(name='Alan Smith', hair_color='blond', height_in_meters='1.83')
# 这里很酷的可以看见尽管没给出具体身高，但是llm经过推理能得出height_in_meters

# 测试一个中文的
text = "今天天气真好，艾伦走在路上，一头红发真的亮眼，两米多的身高一眼就能看到他"
prompt = prompt_template.invoke({"text": text})
llm.invoke(prompt)
# output：Person(name='艾伦', hair_color='红色', height_in_meters='2.0')

# 测试身高字段没有在内容中出现
text = "今天天气真好，艾伦走在路上，一头红发真的亮眼，一眼就能看到他"
prompt = prompt_template.invoke({"text": text})
llm.invoke(prompt)
# output：Person(name='艾伦', hair_color='红色', height_in_meters=None)
```
<mark style="background: #FFB86CA6;">多实体提取：能提取实体嵌套的情况</mark>
	- 这里可能并不是最好的提取demo，在官方文档中有更多改进提取质量的方式
```python
from typing import List, Optional
from pydantic import BaseModel, Field

class Person(BaseModel):
    """Information about a person."""
    name: Optional[str] = Field(default=None, description="The name of the person")
    hair_color: Optional[str] = Field(
        default=None, description="The color of the person's hair if known"
    )
    height_in_meters: Optional[str] = Field(
        default=None, description="Height measured in meters"
    )

# 多实体提取pydantic - 关键点
class Data(BaseModel):
    """Extracted data about people."""
    people: List[Person]

structured_llm = init_chat_model("gpt-4o-mini", model_provider="openai").with_structured_output(schema=Data)
text = "My name is Jeff, my hair is black and i am 6 feet tall. Anna has the same color hair as me."
prompt = prompt_template.invoke({"text": text})
results = structured_llm.invoke(prompt)
# output: Data(people=[Person(name='Jeff', hair_color='black', height_in_meters='1.83'), Person(name='Anna', hair_color='black', height_in_meters=None)])
results.model_dump()["people"]
"""
output:
[{'name': 'Jeff', 'hair_color': 'black', 'height_in_meters': '1.83'},
 {'name': 'Anna', 'hair_color': 'black', 'height_in_meters': None}]
"""

# 测试如果没有people会如何
text = "实验室一个人都没有啊，大家都去哪里了"
prompt = prompt_template.invoke({"text": text})
results = structured_llm.invoke(prompt)
results
# output: Data(people=[])
# 结果是空列表，很好
```
<mark style="background: #FFB86CA6;">tool_example_to_messages:</mark>
	- 将pydantic声明类转换成messages list
```python
from langchain_core.utils.function_calling import tool_example_to_messages

# few examples 少案例
examples = [
    (
        "The ocean is vast and blue. It's more than 20,000 feet deep.",
        Data(people=[]),
    ),
    (
        "Fiona traveled far from France to Spain.",
        Data(people=[Person(name="Fiona", height_in_meters=None, hair_color=None)]),
    ),
]


messages = []

for txt, tool_call in examples:
    if tool_call.people:
        # This final message is optional for some providers
        ai_response = "Detected people."
    else:
        ai_response = "Detected no people."
    messages.extend(tool_example_to_messages(txt, [tool_call], ai_response=ai_response))

print(messages)
"""
[
HumanMessage(content="The ocean is vast and blue. It's more than 20,000 feet deep.", additional_kwargs={}, response_metadata={}), 

AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'b5c60c77-ee9c-4a92-befd-056ba2f6cd04', 'type': 'function', 'function': {'name': 'Data', 'arguments': '{"people":[]}'}}]}, response_metadata={}, tool_calls=[{'name': 'Data', 'args': {'people': []}, 'id': 'b5c60c77-ee9c-4a92-befd-056ba2f6cd04', 'type': 'tool_call'}]), 

ToolMessage(content='You have correctly called this tool.', tool_call_id='b5c60c77-ee9c-4a92-befd-056ba2f6cd04'), 

AIMessage(content='Detected no people.', additional_kwargs={}, response_metadata={}), 

HumanMessage(content='Fiona traveled far from France to Spain.', additional_kwargs={}, response_metadata={}), 

AIMessage(content='', additional_kwargs={'tool_calls': [{'id': '61c6e367-834f-4b19-af4e-12b1b8c63beb', 'type': 'function', 'function': {'name': 'Data', 'arguments': '{"people":[{"name":"Fiona","hair_color":null,"height_in_meters":null}]}'}}]}, response_metadata={}, tool_calls=[{'name': 'Data', 'args': {'people': [{'name': 'Fiona', 'hair_color': None, 'height_in_meters': None}]}, 'id': '61c6e367-834f-4b19-af4e-12b1b8c63beb', 'type': 'tool_call'}]),

ToolMessage(content='You have correctly called this tool.', tool_call_id='61c6e367-834f-4b19-af4e-12b1b8c63beb'), 

AIMessage(content='Detected people.', additional_kwargs={}, response_metadata={})
]
"""
```
<mark style="background: #FFB86CA6;">无messages事例 vs 有messages事例</mark>
```python
# 没有message 事例
message_no_extraction = {
    "role": "user",
    "content": "The solar system is large, but earth has only 1 moon.",
}

structured_llm = init_chat_model("gpt-4o-mini", model_provider="openai").with_structured_output(schema=Data)
structured_llm.invoke([message_no_extraction])
# output: Data(people=[])

# 有message 事例
structured_llm.invoke(messages + [message_no_extraction])
# output: Data(people=[])
# 效果一样，那就是这个案例不够复杂，可能再复杂一点就会不一样
# 无语，官网事例是不一样的
```