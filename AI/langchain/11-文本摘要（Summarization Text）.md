- [地址](https://python.langchain.com/docs/tutorials/summarization/)
- LLM有输入长度限制，总结、蒸馏文档
![[11-文本摘要（Summarization Text）.png]]

三种方式：
- 一个prompt总结所有文档：stuff
- 单独汇总每个文档，然后将摘要 “reduce” 为最终摘要：Map-reduce
- 当前子文档依赖于前面的文档，迭代优化：iterative refinement
![[11-文本摘要（Summarization Text） 1.png]]

<mark style="background: #FFB86CA6;">方式一：Stuff</mark>
- 只需将所有文档“塞”到一个提示中即可。这是最简单的方法
```python
# web网页文档读取
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader("https://lilianweng.github.io/posts/2023-06-23-agent/")
docs = loader.load()

# 模型加载
from langchain.chat_models import init_chat_model  
llm = init_chat_model("gpt-4o-mini", model_provider="openai")
"""
如果文档很长，可以使用更大的context window model
128k token OpenAI `gpt-4o`
200k token Anthropic `claude-3-5-sonnet-20240620`
"""

from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain.chains.llm import LLMChain
from langchain_core.prompts import ChatPromptTemplate

# 定义prompt
prompt = ChatPromptTemplate.from_messages(
    [("system", "Write a concise summary of the following:\\n\\n{context}")]
)

# 实例化 chain
chain = create_stuff_documents_chain(llm, prompt)

# 调用 chain
result = chain.invoke({"context": docs})
print(result)

# steaming输出
for token in chain.stream({"context": docs}):  
	print(token, end="|")
```
<mark style="background: #FFB86CA6;">Map-Reduce：通过并行化总结长文本</mark>
```python
"""
省略，代码太长，中间有些逻辑还没明白，不过大致流程已经知晓,如果有类似业务可以再精进研究一番
"""
```