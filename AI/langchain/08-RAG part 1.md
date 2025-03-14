- 基本构建前置：[[05-构建简单LLM应用-semantic search engine]]
- 在构建好的search engine上再加上`prompt填充相关context->llm输出结果`就是一个RAG的经典流程

<mark style="background: #FFB8EBA6;">流程图: Load->Split->Embed->Store</mark>
![[08-RAG-01.png]]
<mark style="background: #FFB8EBA6;">流程图: Retrieve->Generate</mark>
![[08-RAG-01 1.png]]


<mark style="background: #FFB86CA6;">网页loader：</mark>
```python
import bs4
from langchain_community.document_loaders import WebBaseLoader

# 仅读取class是post-title、post-header、post-content 在整个HTML中.
bs4_strainer = bs4.SoupStrainer(class_=("post-title", "post-header", "post-content"))
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs={"parse_only": bs4_strainer},
)
docs = loader.load()

assert len(docs) == 1
print(f"Total characters: {len(docs[0].page_content)}")
# output：Total characters: 43130
# web_paths只有一个url，所以读取后只有一个doc
```
- [SoupStrainer](https://beautiful-soup-4.readthedocs.io/en/latest/index.html?highlight=soupstrainer)：解析html文档时，加载整个html再解析很费时间，如果知道目标的标签的id、class、或者只知道是某个a标签，那么排除其它不是这类的标签后再解析会更快，这个的作用就是如此
<mark style="background: #FFB86CA6;">分割chunk：</mark>
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,  # chunk size (characters)
    chunk_overlap=200,  # chunk overlap (characters)
    add_start_index=True,  # 追踪 index in original document
)
all_splits = text_splitter.split_documents(docs)

print(f"Split blog post into {len(all_splits)} sub-documents.")
# output：Split blog post into 66 sub-documents.
```
<mark style="background: #FFB86CA6;">向量存储：这次用faiss</mark>
```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
  
from langchain_community.vectorstores import FAISS
vector_store = FAISS.from_documents(all_splits, embedding=embeddings)
vector_store
# output: <langchain_community.vectorstores.faiss.FAISS at 0x20f807f84c0>
```
<mark style="background: #FFB86CA6;">prompt:</mark>
```python
from langchain import hub

# pull rag-prompt模版
prompt = hub.pull("rlm/rag-prompt")
# ChatPromptTemplate对象

# 模版填充
example_messages = prompt.invoke(
    {"context": "(context goes here)", "question": "(question goes here)"}
).to_messages()

assert len(example_messages) == 1
print(example_messages[0].content)
"""
You are an assistant for question-answering tasks. Use the following pieces of retrieved context to answer the question. If you don't know the answer, just say that you don't know. Use three sentences maximum and keep the answer concise. Question: (question goes here) Context: (context goes here) Answer:
"""
```
<mark style="background: #FFB86CA6;">自定义prompt：</mark>
```python
from langchain_core.prompts import PromptTemplate

template = """Use the following pieces of context to answer the question at the end.
If you don't know the answer, just say that you don't know, don't try to make up an answer.
Use three sentences maximum and keep the answer as concise as possible.
Always say "thanks for asking!" at the end of the answer.

{context}

Question: {question}

Helpful Answer:"""
custom_rag_prompt = PromptTemplate.from_template(template)
```
<mark style="background: #FFB86CA6;">Langgraph定义workflow</mark>
1. State定义
2. nodes定义(application steps)
3. control flow（能可视化流程生成图片，很有用）
```python
# 1.State定义
from langchain.chat_models import init_chat_model
from langchain_core.documents import Document
from typing_extensions import List, TypedDict

# 囊括可能出现的所有状态值
class State(TypedDict):
    question: str
    context: List[Document]
    answer: str

# 初始化模型
llm = init_chat_model(
    "gpt-4o-mini", # 模型名
    model_provider="openai" # 模型提供者

)

# 2.Nodes (application steps)
# two steps: retrieval and generation
def retrieve(state: State):
    retrieved_docs = vector_store.similarity_search(state["question"])
    return {"context": retrieved_docs}

def generate(state: State):
    docs_content = "\n\n".join(doc.page_content for doc in state["context"])
    messages = prompt.invoke({"question": state["question"], "context": docs_content})
    response = llm.invoke(messages)
    return {"answer": response.content}

# 3.Control flow
from langgraph.graph import START, StateGraph

# 定义graph,并添加两个节点，和一条边
graph_builder = StateGraph(state_schema=State).add_sequence([retrieve, generate])
graph_builder.add_edge(START, "retrieve") # 添加开始节点
graph = graph_builder.compile() # 编译

# 可视化 control flow
from IPython.display import Image, display
display(Image(graph.get_graph().draw_mermaid_png()))
```
![[08-RAG-01 2.png]]
<mark style="background: #FFB86CA6;">测试：</mark>
```python
# invoke
result = graph.invoke({"question": "What is Task Decomposition?"})

# vector_store.similarity_search() 默认的topk是4
# len(result["context"]) == 4
print(f'Context: {result["context"]}\n\n')
# output: Context: [Document(id='9f7369fa-62...
print(f'Answer: {result["answer"]}')
# output: Answer: Task Decomposition is a process...

# # Stream steps
for step in graph.stream(
    {"question": "What is Task Decomposition?"}, stream_mode="updates" # stream_mode="updates"一个node结束后就输出这个node结果
):
    print(f"{step}\n----------------\n")
"""
{'retrieve': {'context': [Document(...),Document(...),Document(...),Document(...)]}}
----------------
{'generate': {'answer': 'Task decomposition is ....'}}
----------------
"""

# Stream tokens
for message, metadata in graph.stream(
    {"question": "What is Task Decomposition?"}, stream_mode="messages" # messages逐字输出
):
    print(message.content, end="|", flush=True)

# 两种异步调用方式
"""
result = await graph.ainvoke(...)
async for step in graph.astream(...):
"""
```
<mark style="background: #D2B3FFA6;">graph.stream的stream_model参数：</mark>
![[08-RAG-01 4.png]]
