```python
from dotenv import load_dotenv
load_dotenv()

from llama_index.core import SimpleDirectoryReader
from llama_index.core.node_parser import SentenceSplitter
from llama_index.core import Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.core import SummaryIndex, VectorStoreIndex
from llama_index.core.tools import QueryEngineTool
from llama_index.core.query_engine.router_query_engine import RouterQueryEngine
from llama_index.core.selectors import LLMSingleSelector

def get_router_query_engine(file_path):
	# `SimpleDirectoryReader` 是用于从本地文件（或目录）中读取文本/文档
	# `.load_data()`：解析文件为 `Document` 对象的列表，里面包含文档内容和元数据（metadata）
	documents = SimpleDirectoryReader(input_files=[file_path]).load_data()
	
	splitter = SentenceSplitter(chunk_size=1024)
	# `get_nodes_from_documents(documents)`：将 `Document` 对象切分为 `Node` 对象，这些 `Node` 是 LlamaIndex 的最小信息单元。
	nodes = splitter.get_nodes_from_documents(documents)
	
	# `Settings`：LlamaIndex 全局配置对象
	Settings.llm = OpenAI(model="gpt-3.5-turbo")
	Settings.embed_model = OpenAIEmbedding(model="text-embedding-ada-002")
	
	# `SummaryIndex(nodes)`：创建一个**摘要索引**，用于对整篇文档做高层次总结。它默认不会调用embedding模型
	summary_index = SummaryIndex(nodes)
	# `VectorStoreIndex(nodes)`：创建一个**向量索引**，支持语义检索（找相关内容）。在初始化时就会向量化传入的nodes
	vector_index = VectorStoreIndex(nodes)
	
	# `.as_query_engine()`：从索引中生成可问答的“引擎”。
	summary_query_engine = summary_index.as_query_engine(
	    response_mode="tree_summarize", # 表示采用“树形递归总结”的方式逐层汇总信息（用于摘要）
	    use_async=True, # 表示采用“树形递归总结”的方式逐层汇总信息（用于摘要）
	)
	vector_query_engine = vector_index.as_query_engine()
	
	# `QueryEngineTool` 是一个封装查询引擎的工具类，可用于构建 Agent 或 Router
	summary_tool = QueryEngineTool.from_defaults(
	    query_engine=summary_query_engine, # 描述该工具适用于哪类问题（用于自动选择合适工具）
	    description=( # 描述该工具适用于哪类问题（用于自动选择合适工具）
	        "Useful for summarization questions related to MetaGPT"
	    ),
	)
	vector_tool = QueryEngineTool.from_defaults(
	    query_engine=vector_query_engine,
	    description=(
	        "Useful for retrieving specific context from the MetaGPT paper."
	    ),
	)
	
	# `RouterQueryEngine`：一个能根据用户问题选择合适工具的“智能路由器”
	query_engine = RouterQueryEngine(
	    selector=LLMSingleSelector.from_defaults(), # `RouterQueryEngine`：一个能根据用户问题选择合适工具的“智能路由器”
	    query_engine_tools=[
	        summary_tool,
	        vector_tool,
	    ],
	    verbose=True # 启用详细输出，能看到选择路径和逻辑
	)
	
	# response = query_engine.query("What is the summary of the document?")
	# print(response.response)
	# response = query_engine.query("How do agents share information with other agents?")
	# print(response.response)
	return query_engine


query_engine = get_router_query_engine("metagpt.pdf")

response = query_engine.query("Tell me about the ablation study results?")
print(response.response)
```