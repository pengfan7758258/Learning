- [地址](https://python.langchain.com/docs/how_to/streaming/)
- Streaming对于使app基于LLMs对最终用户的响应速度至关重要

<mark style="background: #FFB86CA6;">sync stream:</mark>
```python
from langchain.chat_models import init_chat_model  
  
model = init_chat_model("gpt-4o-mini", model_provider="openai")

for chunk in model.stream("what color is the sky?"):  
	print(chunk.content, end="|", flush=True)
# output：The| sky| appears| blue| during| the| day|.|
```
<mark style="background: #FFB86CA6;">async stream：</mark>
- `AIMessageChunk`：这个 chunk 表示 `AIMessage` 的一部分
```python
chunks = []
async for chunk in model.astream("what color is the sky?"):  
	chunks.append(chunk)
	print(chunk.content, end="|", flush=True)
# output：The| sky| appears| blue| during| the| day|.|

print(chunks[0])
# AIMessageChunk(content='', additional_kwargs={}, response_metadata={}, id='run-a97b717b-5f86-42b3-98eb-c330bd30aeb8')

#  AIMessageChunk在设计上是可以累加的
chunks[0] + chunks[1] + chunks[2] + chunks[3] + chunks[4]
# AIMessageChunk(content='The sky appears blue during', additional_kwargs={}, response_metadata={}, id='run-a97b717b-5f86-42b3-98eb-c330bd30aeb8')
```
<mark style="background: #FFB86CA6;">Chains LECL：</mark>
- 某些可运行对象（如提示模板和聊天模型 ）无法处理单个块，而是聚合所有前面的步骤。此类 runnable 可能会中断流式处理过程
- <mark style="background: #BBFABBA6;">StrOutputParser解析输出</mark>
```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template("告诉我一个笑话关于{topic}")

# StrOutputParser 来解析模型的输出,它从AIMessageChunk中提取content字段
parser = StrOutputParser()
chain = prompt | model | parser

async for chunk in chain.astream({"topic": "美女"}):
    print(chunk, end="|", flush=True)
"""
|有|一个|美女|走|进|咖|啡|店|，|点|了一|杯|咖|啡|。|店|员|看|着|她|，|惊|讶|地|问|：“|请|问|，你|是|要|加|糖|还是|要|减|肥|？”

|美女|微|笑|着|回答|：“|我|当然|要|加|糖|，|减|肥|的|事情|留|给|那些|喝|无|糖|咖|啡|的人|吧|！”

|店|员|点|点|头|：“|那|我|给|您的|咖|啡|加|两|勺|糖|，|顺|便|给|我|一|勺|自|信|！”||
"""
```
- <mark style="background: #BBFABBA6;">JsonOutputParser解析输出为json</mark>
```python
from langchain_core.output_parsers import JsonOutputParser
# 因为Langchain旧版本, JsonOutputParser对于一些model不支持
# 必须在prompt告诉模型我们的输出格式为json
chain = model | JsonOutputParser()
async for text in chain.astream(
    "以json格式输出内容。"
    "讲一讲掩耳盗铃的故事"
    "关键字为`content`,只有这一个关键字"
):
    print(text, flush=True)
"""
{} 
{'content': ''} 
{'content': '掩'} 
{'content': '掩耳'} 
{'content': '掩耳盗'} 
{'content': '掩耳盗铃'} 
{'content': '掩耳盗铃是'} 
{'content': '掩耳盗铃是一个'} 
......
{'content': '掩耳盗铃是一个成语，字面意思是捂住自己的耳朵去盗铃。这个成语来源于一个古代的故事：有一个小贼想要偷一个悬挂在门口的铃铛，但知道铃铛一响会引起别人的注意。他想了一个办法，用手捂住自己的耳朵，认为这样就听不到铃声，结果在盗铃时铃声响起，最终被人抓住。这个故事告诫人们，逃避现实并不能解决问题，反而可能会导致更大的失败。'}
"""


```
- <mark style="background: #BBFABBA6;">加入自己的extract逻辑函数</mark>
```python
# 一个对最终输入而不是中断流进行操作的函数
def _extract_content(inputs):
    """一个对最终输入而不是中断流进行操作的函数。"""

    if not isinstance(inputs, dict) or "content" not in inputs:
        return ""

    content = inputs["content"]

    return content

chain = model | JsonOutputParser() | _extract_content

async for text in chain.astream(
    "以json格式输出内容。"
    "讲一讲掩耳盗铃的故事"
    "关键字为`content`,只有这一个关键字"
):
    print(text, end="|", flush=True)
# output: 掩耳盗铃是一个源自中国的成语，故事讲述了一位小偷想要偷铃铛，但害怕铃声被人听到，于是他用手捂住自己的耳朵，以为这样就不会听到铃声。结果铃铛被偷走，声响依然传出。这个故事寓意着自欺欺人，无法逃避现实，强调了掩盖问题并不能解决问题的道理。|
```
- <mark style="background: #BBFABBA6;">Generator Functions</mark>
```python
# Generator Functions
from langchain_core.output_parsers import JsonOutputParser

  
async def _extract_content_stream(input_stream):
    """一个对流进行操作的函数。"""
    async for input in input_stream:
        if not isinstance(input, dict) or "content" not in input:
            continue
        print(input)
        content = input["content"]
        yield content

chain = model | JsonOutputParser() | _extract_content_stream

async for text in chain.astream(
    "以json格式输出内容。"
    "讲一讲掩耳盗铃的故事"
    "关键字为`content`,只有这一个关键字"
):
    pass
"""
{'content': ''} 
{'content': '掩'} 
{'content': '掩耳'} 
{'content': '掩耳盗'} 
{'content': '掩耳盗铃'} 
{'content': '掩耳盗铃是'} 
{'content': '掩耳盗铃是一个'} 
......
{'content': '掩耳盗铃是一个成语，字面意思是捂住自己的耳朵去盗铃。这个成语来源于一个古代的故事：有一个小贼想要偷一个悬挂在门口的铃铛，但知道铃铛一响会引起别人的注意。他想了一个办法，用手捂住自己的耳朵，认为这样就听不到铃声，结果在盗铃时铃声响起，最终被人抓住。这个故事告诫人们，逃避现实并不能解决问题，反而可能会导致更大的失败。'}
"""
```
<mark style="background: #FFB86CA6;">Non-streaming components：</mark>
- Retrievers
```python
from langchain_community.vectorstores import FAISS
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import OpenAIEmbeddings

template = """Answer the question based only on the following context:
{context}

Question: {question}
"""
prompt = ChatPromptTemplate.from_template(template)

vectorstore = FAISS.from_texts(
    ["harrison worked at kensho", "harrison likes spicy food"],
    embedding=OpenAIEmbeddings(),
)
retriever = vectorstore.as_retriever()

chunks = [chunk for chunk in retriever.stream("where did harrison work?")]
chunks
"""
--------------- output ------------
[[Document(id='dab51192-81b9-4819-9237-b12c7cfacbb6', metadata={}, page_content='harrison worked at kensho'),
  Document(id='d720ada6-494e-4e6c-bc38-e8fbc240792a', metadata={}, page_content='harrison likes spicy food')]]
"""
# Stream 只是从该组件生成最终结果,并没有真正的流式输出

# 流式输出只要在最后一个非流式输出步骤之后开始即可
from langchain_core.runnables import RunnablePassthrough

retrieval_chain = (
    {
        "context": retriever.with_config(run_name="Docs"), # with_config(run_name="Docs") 是为这个步骤设置一个名称（"Docs"），方便调试和日志记录。
        "question": RunnablePassthrough(), # 将输入string作为question
    }
    | prompt # 将上一步的dict作为placeholder填充prompt
    | model
    | StrOutputParser()
)

for chunk in retrieval_chain.stream(
    "Where did harrison work? Write 3 made up sentences about this place."
):
    print(chunk, end="|", flush=True)
"""
|H|arrison| worked| at| Kens|ho|.| At| Kens|ho|,| the| team| is| known| for| their| innovative| approach| to| data| analysis|,| allowing| clients| to| gain| valuable| insights| quickly|.| The| office| ambiance| is| lively|,| adorned| with| colorful| art| and| equipped| with| state|-of|-the|-art| technology| that| inspires| creativity| among| the| employees|.| Every| Friday|,| the| staff| gathers| for| a| fun| brainstorming| session|,| where| they| share| ideas| over| delicious| snacks|,| often| featuring| spicy| dishes| that| match| Harrison|'s| taste|.||
"""
```

<mark style="background: #FFB86CA6;">asteam_events：</mark>
```python
events = []
async for event in model.astream_events("hello"):
    events.append(event)
	print(event)
"""
----------- output ------------
{'event': 'on_chat_model_start', 'data': {'input': 'hello'}, 'name': 'ChatOpenAI', 'tags': [], 'run_id': '12d0132e-8a87-49b0-b85a-a81426ab04f3', 'metadata': {'ls_provider': 'openai', 'ls_model_name': 'gpt-4o-mini', 'ls_model_type': 'chat', 'ls_temperature': None}, 'parent_ids': []}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content='', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content='Hello', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content='!', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content=' How', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content=' can', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content=' I', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content=' assist', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content=' you', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content=' today', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content='?', ..}, ..}
{'event': 'on_chat_model_stream', .. AIMessageChunk(content='', ..}, ..}
{'event': 'on_chat_model_end', 'data': {'output': AIMessageChunk(content='Hello! How can I assist you today?', additional_kwargs={}, response_metadata={'finish_reason': 'stop', 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_b8bc95a0ac'}, id='run-12d0132e-8a87-49b0-b85a-a81426ab04f3')}, 'run_id': '12d0132e-8a87-49b0-b85a-a81426ab04f3', 'name': 'ChatOpenAI', 'tags': [], 'metadata': {'ls_provider': 'openai', 'ls_model_name': 'gpt-4o-mini', 'ls_model_type': 'chat', 'ls_temperature': None}, 'parent_ids': []}
"""
```
-  <mark style="background: #BBFABBA6;">Chain</mark>
```python
chain = (
    model | JsonOutputParser()
)  # 因为Langchain旧版本bug, JsonOutputParser对于一些model不支持stream results

num_events = 0

async for event in chain.astream_events(
    "以json格式输出内容。"
    "讲一讲掩耳盗铃的故事"
    "关键字为`content`,只有这一个关键字"
):
    e = event["event"]
    print(e) # 查看当前event名称
    num_events += 1
    if num_events > 30:
        print("...")
        break
```
- <mark style="background: #BBFABBA6;">Filtering Events</mark>
```python
# By Name
chain = model.with_config({"run_name": "model"}) | JsonOutputParser().with_config(
    {"run_name": "my_parser"}
)

max_events = 0
async for event in chain.astream_events(
    "以json格式输出内容。"
    "讲一讲掩耳盗铃的故事"
    "关键字为`content`,只有这一个关键字"
    include_names=["my_parser"], # 只保留输出该名字的event
):
    print(event)
    max_events += 1
    if max_events > 10:
        print("...")
        break

# By Type TODO 在官网查看
# By Tags TODO 在官网查看
```
<mark style="background: #FFB86CA6;">Propagating Callbacks：</mark>
- 如果您在工具内部使用调用runnable对象，则需要将callback传播到runnable对象；否则，不会生成任何流事件
- 当使用 `RunnableLambdas` or 装饰器`@chain` , callbacks 会自动执行
```python
from langchain_core.runnables import RunnableLambda
from langchain_core.tools import tool

def reverse_word(word: str):
    return word[::-1]

reverse_word = RunnableLambda(reverse_word)

# 错误方式
@tool
def bad_tool(word: str):
    """自定义tool 无法回调 callbacks."""
    return reverse_word.invoke(word)

async for event in bad_tool.astream_events("hello"):
    print(event)


# 正确方式
@tool
def correct_tool(word: str, callbacks):
    """自定义工具 正确callbacks."""
    return reverse_word.invoke(word, {"callbacks": callbacks})

async for event in correct_tool.astream_events("hello"):
    print(event)
```