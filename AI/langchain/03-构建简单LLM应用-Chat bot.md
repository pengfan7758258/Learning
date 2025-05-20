[地址](https://python.langchain.com/docs/tutorials/chatbot/)


<mark style="background: #ADCCFFA6;">langgraph构建单节点graph，自动持久化message list构建多轮回话：</mark>
```python
from typing import Sequence

from langchain_core.messages import (AIMessage, 
	HumanMessage, 
	SystemMessage, 
	trim_messages
	BaseMessage)
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph.message import add_messages
from langgraph.graph import START, MessagesState, StateGraph
from typing_extensions import Annotated, TypedDict
  
# 定义一个graph
workflow = StateGraph(state_schema=MessagesState)  

# 定义一个model的函数调用
def call_model(state: MessagesState):  
	response = model.invoke(state["messages"])  
	return {"messages": response}  
  
  
# 定义一个单节点的graph
workflow.add_edge(START, "model")  # 创建边
workflow.add_node("model", call_model)  # 创建节点
  
# 添加memory, workflow.compile实现了Runnable interface
app = workflow.compile(checkpointer=MemorySaver())

# 定义config并设置`thread_id`来追踪、定位
# 有多个用户时你的app能同时处理多个对话线程，由不一样的thread_id控制
config = {"configurable": {"thread_id": "abc123"}}

query = "Hi! I'm Bob."
input_messages = [HumanMessage(query)]

output = app.invoke(
    {"messages": input_messages},
    config # 配置项的thread_id不变, 会添加到当前thread_id的message list中
)
# output["messages"]保存了所有的历史对话

# 优化输出
print(output["messages"][-1].pretty_print())
"""
================================== Ai Message ================================== Hello, Bob! What would you like to talk about today?
"""
```
<mark style="background: #ADCCFFA6;">异步方式调用:</mark>
```python
# 异步函数 for node:
async def call_model(state: MessagesState):
    response = await model.ainvoke(state["messages"]) # 异步调用
    return {"messages": response}


# 定义graph:
workflow = StateGraph(state_schema=MessagesState)
workflow.add_edge(START, "model")
workflow.add_node("model", call_model)
app = workflow.compile(checkpointer=MemorySaver())

# 异步调用await:
output = await app.ainvoke({"messages": input_messages}, config)
```
prompt templates+managing conversation history
```python
# 有language和messages两个占位参数
prompt_template = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are a helpful assistant. Answer all questions to the best of your ability in {language}.",
        ),
        MessagesPlaceholder(variable_name="messages"), # variable_name="messages" 会自动去变量里找messages填充
    ]
)

# 修剪器,管理对话历史的一种策略
trimmer = trim_messages(
    max_tokens=65, # 最大保留token数量
    strategy="last", # 策略，last代表从最后开始保留
    token_counter=model, # token计数方式，model代表按照当前model的token方式计数
    include_system=True, # 是否包含system prompt
    allow_partial=False, # 是否部分保存，False代表否，如果是True可能出现不是一句完整的话语
    start_on="human", # 从HumanMessage开始保留
)

# graph中的state不止有messages后,StateGraph的state_schema参数、call_model函数的声明类要重新定义
class State(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    language: str

workflow = StateGraph(state_schema=State) # workflow的state状态属性就是定义的State类

def call_model(state: State):
	trimmed_messages = trimmer.invoke(state["messages"]) # message修剪器修剪message
    prompt = prompt_template.invoke(
	    {"messages": trimmed_messages, "language": state["language"]}
    ) # 模版填充
    response = model.invoke(prompt) # 模型调用
    return {"messages": response}

workflow.add_edge(START, "model")
workflow.add_node("model", call_model)

app = workflow.compile(checkpointer=MemorySaver())

config = {"configurable": {"thread_id": "abc567"}}
query = "What is my name?"
language = "English"

input_messages = [HumanMessage(query)]
output = app.invoke(
    {"messages": input_messages, "language": language},
    config,
)

print(output["messages"][-1].pretty_print())
"""
================================== Ai Message ================================== I don’t know your name. What would you like me to call you?
"""
```
<mark style="background: #ADCCFFA6;">Streaming流式:</mark>
```python
"""
这个stream的输出方式不像是真实的流式，因为它是走的call_model函数
而这个函数我们定义的时候就是内部直接调用model.invoke
所以其实只是LLM整体return后又包装了一层
"""
# thread_id不一样，开启一个新的会话
config = {"configurable": {"thread_id": "abc999"}}
query = "Hi I'm Todd, please tell me a joke."
language = "English"

input_messages = [HumanMessage(query)]

for chunk, metadata in app.stream( # stream函数替代invoke
    {"messages": input_messages, "language": language},
    config,
    stream_mode="messages", # 关键参数
):
    print(chunk.content, end="|", flush=True)
```