- [地址](https://python.langchain.com/docs/tutorials/classification/)

![[06-构建简单LLM应用-classification.png]]

- input：用户输入
- schema：定义对document打什么样的tag
- function：告诉模型使用什么函数来对document打tag，schema会填充在这里面

<mark style="background: #ADCCFFA6;">代码：</mark>
```python
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

  
# 定义prompt
tagging_prompt = ChatPromptTemplate.from_template(
"""
Extract the desired information from the following passage.
Only extract the properties mentioned in the 'Classification' function.

Passage:
{input}
"""

)

# pydantic定义输出结构
class Classification(BaseModel):
    sentiment: str = Field(description="The sentiment of the text, only positive or negative")
    aggressiveness: int = Field(
        description="How aggressive the text is on a scale from 1 to 10"
    )
    language: str = Field(description="The language the text is written in")

  
# 给LLM加上结构化输出 - pydantic类来约束
llm = llm.with_structured_output(
    Classification
)

input = "这个房间的厕所好像有点堵了，希望下次来不会有这种事情了"
prompt = tagging_prompt.invoke({"input": input})
response = llm.invoke(prompt)
response
# output：Classification(sentiment='negative', aggressiveness=2, language='Chinese')

# convert dict，有点像pydantic类的功能
response.model_dump()
# output：{'sentiment': 'negative', 'aggressiveness': 2, 'language': 'Chinese'}
```
<mark style="background: #BBFABBA6;">输出约束主要是在pydantic类：</mark>
```python
class Classification(BaseModel):
    sentiment: str = Field(..., enum=["happy", "neutral", "sad"])
    aggressiveness: int = Field(
        ...,
        description="describes how aggressive the statement is, the higher the number the more aggressive",
        enum=[1, 2, 3, 4, 5],
    )
    language: str = Field(
        ..., enum=["spanish", "english", "french", "german", "italian"]
    )
"""
enum：约束输出的值范围
description：描述以确保model理解该属性
typing：str、int等来约束输出的类型
"""
```