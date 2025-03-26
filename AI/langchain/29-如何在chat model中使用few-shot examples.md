- [地址](https://python.langchain.com/docs/how_to/few_shot_examples_chat/)

## fix examples
<mark style="background: #FFB86CA6;">examples:</mark>
```python
examples = [  
	{"input": "2 🦜 2", "output": "4"},  
	{"input": "2 🦜 3", "output": "5"},  
]
```
<mark style="background: #FFB86CA6;">组合few-shot：</mark>
- example_prompt：通过其 `format_messages` 方法将每个示例转换为 1 个或多个消息
- few_shot_prompt：使用`FewShotChatMessagePromptTemplate`整合example_prompt模版来生成 few-shot messages
```python
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate

example_prompt = ChatPromptTemplate.from_messages(  
	[  
		("human", "{input}"),  
		("ai", "{output}") 
	]  
)
# 将examples数据填充到example_prompt
few_shot_prompt = FewShotChatMessagePromptTemplate(  
	example_prompt=example_prompt, # 填充模版 
	examples=examples # 样例数据
)
print(few_shot_prompt.invoke({}).to_messages())
"""
------------- output -------------
[HumanMessage(content='2 🦜 2', additional_kwargs={}, response_metadata={}), 
AIMessage(content='4', additional_kwargs={}, response_metadata={}), HumanMessage(content='2 🦜 3', additional_kwargs={}, response_metadata={}), 
AIMessage(content='5', additional_kwargs={}, response_metadata={})]
"""
```
<mark style="background: #FFB86CA6;">final_prompt并调用model：</mark>
```python

final_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "You are a wondrous wizard of math."),
        few_shot_prompt,
        ("human", "{input}"),
    ]
)

from langchain.chat_models import init_chat_model
# 加载模型
model = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai"
)

chain = final_prompt | model
chain.invoke({"input": "9 🦜 2"})
"""
--------- output ----------
AIMessage(content='11', additional_kwargs=...)

模型从给定的少量示例中推断出鹦鹉表情符号代表加法！
"""
```
## dynamic examples
<mark style="background: #FFB86CA6;">vectorstore：</mark>
```python
from langchain_community.vectorstores import FAISS
from langchain_core.example_selectors import SemanticSimilarityExampleSelector
from langchain_openai import OpenAIEmbeddings

examples = [
    {"input": "2 🦜 2", "output": "4"},
    {"input": "2 🦜 3", "output": "5"},
    {"input": "2 🦜 4", "output": "6"},
    {"input": "What did the cow say to the moon?", "output": "nothing at all"},
    {
        "input": "Write me a poem about the moon",
        "output": "One for the moon, and one for me, who are we to talk about the moon?",
    },
]

to_vectorize = [" ".join(example.values()) for example in examples]
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_texts(to_vectorize, embeddings, metadatas=examples)
```
<mark style="background: #FFB86CA6;">create  example_selector：</mark>
```python
example_selector = SemanticSimilarityExampleSelector(
    vectorstore=vectorstore,
    k=2,
)
example_selector.select_examples({"input": "horse"})
"""
------------ output ---------------
[{'input': 'What did the cow say to the moon?', 'output': 'nothing at all'},
 {'input': '2 🦜 4', 'output': '6'}]
"""
```
<mark style="background: #FFB86CA6;">Create prompt template：</mark>
```python
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate

# 定义few-shot prompt tempalte.
few_shot_prompt = FewShotChatMessagePromptTemplate(
    # 选择器输入变量 input key
    input_variables=["input"],
    example_selector=example_selector,
    example_prompt=ChatPromptTemplate.from_messages(
        [("human", "{input}"), ("ai", "{output}")]
    ),
)

print(few_shot_prompt.invoke(input="What's 3 🦜 3?").to_messages())
"""
---------------- output ----------------
[HumanMessage(content='2 🦜 3'), AIMessage(content='5'), HumanMessage(content='2 🦜 4'), AIMessage(content='6')]
"""
```
<mark style="background: #FFB86CA6;">final_prompt并调用model：</mark>
```python
final_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "You are a wondrous wizard of math."),
        few_shot_prompt,
        ("human", "{input}"),
    ]
)

print(few_shot_prompt.invoke(input="What's 3 🦜 3?"))
"""
------------- output -----------------
messages=[HumanMessage(content='2 🦜 3'), AIMessage(content='5'), HumanMessage(content='2 🦜 4'), AIMessage(content='6')]
"""

chain = final_prompt | model
chain.invoke({"input": "What's 9 🦜 2?"})

"""
--------- output ----------
AIMessage(content='11', additional_kwargs=...)

模型从给定的少量示例中推断出鹦鹉表情符号代表加法！
"""
""""
```