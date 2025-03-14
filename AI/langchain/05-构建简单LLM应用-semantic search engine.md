- [地址](https://python.langchain.com/docs/tutorials/retrievers/)
- 建立于PDF文档上，根据input检索段落
- 大致流程：文档读取->文档分割chunk->向量化->存储向量->query输入检索

<mark style="background: #FFB86CA6;">文档读取：</mark>
- langchain集成了百种通用资源loaders(e.g.,csv,pdf等)，非常方便去整合数据
```python
from langchain_community.document_loaders import PyPDFLoader

file_path = "nke-10k-2023.pdf" # 官网的nike给的nike事例文档
# PyPDFLoader为pdf的每一页加载一个Document对象.
loader = PyPDFLoader(file_path)
docs = loader.load()
print(len(docs))
```

<mark style="background: #FFB86CA6;">文档分割：</mark>
- text splitter，可以控制chunk大小，overlap大小，切割字符等参数
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000, chunk_overlap=200, add_start_index=True
)
# all_splits是一个list，里面都是Document对象
# Document对象有page_content、matadata属性
# metadata有一个start_index代表这个Document在原始文档中的索引位置从哪里开始
all_splits = text_splitter.split_documents(docs)

len(all_splits)
```

<mark style="background: #FFB86CA6;">向量化：</mark>
- langchain支持很多向量化的方法，有开源的（ollama）也有商业的（openai），[地址](https://python.langchain.com/docs/integrations/text_embedding/)
```python
from langchain_openai import OpenAIEmbeddings

# openai的text-embedding-3-large维度是3072
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")

# 测试向量化
vector_1 = embeddings.embed_query(all_splits[0].page_content)
print(f"Generated vectors of length {len(vector_1)}\n") 
# output：Generated vectors of length 3072
print(vector_1[:10])
# output：[0.009320048615336418, -0.016064738854765892, ...]
```

<mark style="background: #FFB86CA6;">vector store/向量存储：</mark>
- lanchain集成的[向量库](https://python.langchain.com/docs/integrations/vectorstores/)(e.g,faiss,milvus(需要搭建)等)
- 向量数据库可能支持的功能：
	- 同步和异步查询
	- 通过str查询、通过vector查询
	- 返回当前查询方式下match的score
	- By similarity and [maximum marginal relevance](https://python.langchain.com/api_reference/core/vectorstores/langchain_core.vectorstores.base.VectorStore.html#langchain_core.vectorstores.base.VectorStore.max_marginal_relevance_search) 
		- 相似性和最大边际相关性
		- similarity：匹配结果和query高度相似
		- maximum marginal relevance（mmr）：匹配的结果之间差异性大，匹配的内容之间不要一样
```python
from langchain_core.vectorstores import InMemoryVectorStore

# InMemoryVectorStore存储在了内存,方便测试,还有很多其它的能写，例如milvus
vector_store = InMemoryVectorStore(embeddings)
"""
from langchain_milvus import Milvus  # milvus向量数据库要单独搭建
vector_store = Milvus(embedding_function=embeddings)
"""
# 向量化所有文档数据
ids = vector_store.add_documents(documents=all_splits)

results = vector_store.similarity_search(
    "How many distribution centers does Nike have in the US?"
) # 底层做的是先把query向量化后，使用向量来进行相似度搜索

print(results[0])
"""
page_content='direct to consumer operations sell products through the following number of retail stores in the United States:
U.S. RETAIL STORES NUMBER
NIKE Brand factory stores 213 
NIKE Brand in-line stores (including employee-only stores) 74 
Converse stores (including factory stores) 82 
TOTAL 369 
In the United States, NIKE has eight significant distribution centers. Refer to Item 2. Properties for further information.
2023 FORM 10-K 2' metadata={'producer': 'EDGRpdf Service w/ EO.Pdf 22.0.40.0', 'creator': 'EDGAR Filing HTML Converter', 'creationdate': '2023-07-20T16:22:00-04:00', 'title': '0000320187-23-000039', 'author': 'EDGAR Online, a division of Donnelley Financial Solutions', 'subject': 'Form 10-K filed on 2023-07-20 for the period ending 2023-05-31', 'keywords': '0000320187-23-000039; ; 10-K', 'moddate': '2023-07-20T16:22:08-04:00', 'source': 'nke-10k-2023.pdf', 'total_pages': 107, 'page': 4, 'page_label': '5', 'start_index': 3125}
"""
# 异步search, 结果是一样的
"""
results = await vector_store.asimilarity_search("How many distribution centers does Nike have in the US?")
"""
# 结果中包含score

results = vector_store.similarity_search_with_score("How many distribution centers does Nike have in the US?")
doc, score = results[0]

print(f"Score: {score}\n") # output: Score: 0.7148648981089245

# 向量化->匹配相似向量
embedding = embeddings.embed_query("How were Nike's margins impacted in 2023?")
results = vector_store.similarity_search_by_vector(embedding)
```
<mark style="background: #BBFABBA6;">Retrievers</mark>
- 构建检索器，支持batch检索
```python
# 方式一
from typing import List
from langchain_core.documents import Document
from langchain_core.runnables import chain

@chain
def retriever(query: str) -> List[Document]:
    return vector_store.similarity_search(query, k=1)

  
retriever.batch(
    [
        "How many distribution centers does Nike have in the US?",
        "When was Nike incorporated?",
    ],
)
"""
output:
[[Document(id='d40a6007-0b5b-4a25-a86e-c1eaff0af7c0', metadata=...],
 [Document(id='1d5870c9-557b-4986-a610-80e6ddd27a83', metadata=...]]
"""

# 方式二

retriever = vector_store.as_retriever(
    search_type="similarity", # 还有mmr、similarity_score_threshold类型支持
    search_kwargs={"k": 1}, # 一些超参数设定
)

  
retriever.batch(
    [
        "How many distribution centers does Nike have in the US?",
        "When was Nike incorporated?",
    ],

)
"""
和方式一的输出结果保持一致，因为设置的超参数是一样的
output:
[[Document(id='d40a6007-0b5b-4a25-a86e-c1eaff0af7c0', metadata=...],
 [Document(id='1d5870c9-557b-4986-a610-80e6ddd27a83', metadata=...]]
"""
```