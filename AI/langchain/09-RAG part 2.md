[地址](https://python.langchain.com/docs/tutorials/qa_chat_history/)

多步骤检索，长期记忆

<mark style="background: #FFB86CA6;">chain方式实现</mark>
```python
# 模型初始化
from langchain.chat_models import init_chat_model
llm = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai"
)

# embedding模型，这个非常危险，会暴露openai_key
from langchain.embeddings import OpenAIEmbeddings
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")

# web documents
import bs4
from langchain import hub
from langchain_community.document_loaders import WebBaseLoader
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter
from typing_extensions import List, TypedDict

  
# 读取网页博客并chunk
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            class_=("post-content", "post-title", "post-header")
        )
    ),
)

docs = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)
all_splits = text_splitter.split_documents(docs)

# vector store
from langchain_community.vectorstores import FAISS
vector_store = FAISS.from_documents(documents=all_splits, embedding=embeddings)

# langgraph创建graph_builder,使用MessagesState管理State
from langgraph.graph import MessagesState, StateGraph
graph_builder = StateGraph(MessagesState)

# 将检索步骤 转换为 工具tool
from langchain_core.tools import tool

@tool(response_format="content_and_artifact")
def retrieve(query: str):
    """检索和query相关的信息."""
    retrieved_docs = vector_store.similarity_search(query, k=2)
    serialized = "\n\n".join(
        (f"Source: {doc.metadata}\n" f"Content: {doc.page_content}")
        for doc in retrieved_docs
    )
    return serialized, retrieved_docs

# 构建graph
from langchain_core.messages import SystemMessage
from langgraph.prebuilt import ToolNode
# 1.生成AImessage并带有检索工具调用
def query_or_respond(state: MessagesState):
    llm_with_tools = llm.bind_tools([retrieve])
    response = llm_with_tools.invoke(state["messages"])
    # MessagesState将消息附加到状态，而不是覆盖
    return {"messages": [response]}

# 2.retrieval的工具Node
tools = ToolNode([retrieve])

# 3.使用检索到的内容生成响应
def generate(state: MessagesState):
    """生成答案."""
    # Get generated ToolMessages
    recent_tool_messages = []
    for message in reversed(state["messages"]): # 找寻最近的的tool message
        if message.type == "tool":
            recent_tool_messages.append(message)
        else:
            break
    tool_messages = recent_tool_messages[::-1]

    # Format prompt
    docs_content = "\n\n".join(doc.content for doc in tool_messages)
    system_message_content = (
        "You are an assistant for question-answering tasks. "
        "Use the following pieces of retrieved context to answer "
        "the question. If you don't know the answer, say that you "
        "don't know. Use three sentences maximum and keep the "
        "answer concise."
        "\n\n"
        f"{docs_content}"
    )
    
    conversation_messages = [
        message
        for message in state["messages"]
        if message.type in ("human", "system")
        or (message.type == "ai" and not message.tool_calls)
    ]

    prompt = [SystemMessage(system_message_content)] + conversation_messages

    # Run
    response = llm.invoke(prompt)
    return {"messages": [response]}

# 编译graph
from langgraph.graph import END
from langgraph.prebuilt import tools_condition

graph_builder.add_node(query_or_respond)
graph_builder.add_node(tools)
graph_builder.add_node(generate)

graph_builder.set_entry_point("query_or_respond") # 与add_edge(START,query_or_respond)作用一样

# 允许query_or_respond函数short-circuit短路：
# 如果它没有生成工具调用，则直接响应用户，使我们的应用程序能够支持对话体验 -- 例如，响应可能不需要检索步骤的通用问候语“hi”
graph_builder.add_conditional_edges( # 添加条件边，根据条件函数的结果决定下一步的走向
    "query_or_respond", # 这是源节点，表示条件边从该节点出发
    tools_condition, # 这是条件函数，用于决定下一步的走向
    {END: END, "tools": "tools"}, # 这是一个映射字典，表示条件函数的返回值对应的目标节点 - 如果 tools_condition 返回 END，则图将结束; 如果 tools_condition 返回 "tools"，则图将跳转到 tools 节点

)
graph_builder.add_edge("tools", "generate")
graph_builder.add_edge("generate", END)
graph = graph_builder.compile()

# 可视化 control flow
from IPython.display import Image, display
display(Image(graph.get_graph().draw_mermaid_png()))
```
![[09-RAG part 2.png]]
```python
# Test
input_message = "Hello"

for step in graph.stream(
    {"messages": [{"role": "user", "content": input_message}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()
"""
output:
========[1m Human Message [0m================
Hello
==========[1m Ai Message [0m================
Hello! How can I assist you today?
"""

input_message = "What is Task Decomposition?"

for step in graph.stream(
    {"messages": [{"role": "user", "content": input_message}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()
"""
output:
=========== Human Message ================
What is Task Decomposition?
============ Ai Message =========
Tool Calls: 
retrieve (call_UEMguofyc7ktfXuyGGsf2E38) 
Call ID: call_UEMguofyc7ktfXuyGGsf2E38 
Args: 
query: Task Decomposition
============== Tool Message ================
Name: retrieve
......
============== Ai Message ===================
Task Decomposition is the process of breaking down a ...
"""

# Memory persistence
from langgraph.checkpoint.memory import MemorySaver
graph = graph_builder.compile(checkpointer=MemorySaver())
# 为当前线程指定一个id
config = {"configurable": {"thread_id": "abc123"}}
# 后面有部分没复制，就是把config参数给graph.stream当参数就能记住历史聊天
```

<mark style="background: #FFB86CA6;">agent方式实现</mark>
```python
from langgraph.prebuilt import create_react_agent

agent_executor = create_react_agent(llm, [retrieve], checkpointer=MemorySaver())

display(Image(agent_executor.get_graph().draw_mermaid_png()))
"""
这里的工具调用不是结束运行的最终生成步骤而是循环回到原始LLM调用。
然后,模型可以使用检索到的上下文回答问题或生成另一个工具调用以获取更多信息
"""
```
![[09-RAG part 2 1.png]]
```python
config = {"configurable": {"thread_id": "def237"}}

  
input_message = (
    "What is the standard method for Task Decomposition?\n\n"
    "Once you get the answer, look up common extensions of that method."
)

# 找到一个问题答案后再次调用tools去检索相关信息
for event in agent_executor.stream(
    {"messages": [{"role": "user", "content": input_message}]},
    stream_mode="values",
    config=config,
):
    event["messages"][-1].pretty_print()
"""
============== Human Message ================
What is the standard method for Task Decomposition? Once you get the answer, look up common extensions of that method.
============== Ai Message ================
Tool Calls: 
retrieve (call_ESh3TDYMa50F9rPl3oUNbREi) 
Call ID: call_ESh3TDYMa50F9rPl3oUNbREi 
Args: 
query: standard method for Task Decomposition
============== Tool Message ================
Name: retrieve
....
============== Ai Message ================
Tool Calls: 
retrieve (call_X1kbJbHnswkj8TV8YjDEy3Hc) 
Call ID: call_X1kbJbHnswkj8TV8YjDEy3Hc 
Args: 
query: extensions of Task Decomposition method 
============== Tool Message ================
Name: retrieve
...
============== Ai Message ================
### Standard Method for Task Decomposition
...
### Common Extensions of Task Decomposition
...
"""
```
