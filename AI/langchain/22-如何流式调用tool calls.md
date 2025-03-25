- [地址](https://python.langchain.com/docs/how_to/tool_streaming/)

```python
from langchain.chat_models import init_chat_model

# 加载模型
llm = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai",
    temperature=0
)

# 定义工具
def add(a: int, b: int) -> int:  
	"""Add two integers.  
	  
	Args:  
		a: First integer  
		b: Second integer  
	"""  
	return a + b  
  
  
def multiply(a: int, b: int) -> int:  
	"""Multiply two integers.  
	  
	Args:  
		a: First integer  
		b: Second integer  
	"""  
	return a * b

  
tools = [add, multiply]

# LLM绑定工具
llm_with_tools = llm.bind_tools(tools)

# 流式输出工具调用
query = "What is 3 * 12? Also, what is 11 + 49?"

  
async for chunk in llm_with_tools.astream(query):
    print(chunk.tool_call_chunks, type(chunk))
"""
-------------- output -----------------
[] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': 'multiply', 'args': '', 'id': 'call_lv1ZrF7KCMKIghGBca0aZZOB', 'index': 0, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': None, 'args': '{"a"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': None, 'args': ': 3, ', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': None, 'args': '"b": 1', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': None, 'args': '2}', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': 'add', 'args': '', 'id': 'call_glFRYXiqDp0P7u53Rq2rgJBM', 'index': 1, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': None, 'args': '{"a"', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': None, 'args': ': 11,', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': None, 'args': ' "b": ', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[{'name': None, 'args': '49}', 'id': None, 'index': 1, 'type': 'tool_call_chunk'}] <class 'langchain_core.messages.ai.AIMessageChunk'>
[] <class 'langchain_core.messages.ai.AIMessageChunk'>
"""

# AIMessageChunk类型可以使用add进行拼接
first = True
async for chunk in llm_with_tools.astream(query):
    if first:
        gathered = chunk
        first = False
    else:
        gathered = gathered + chunk

    print(gathered.tool_call_chunks) # 输出的args是str

"""
[]
[{'name': 'multiply', 'args': '', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a"', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, ', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, "b": 1', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, "b": 12}', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, "b": 12}', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}, {'name': 'add', 'args': '', 'id': 'call_LaD7wdiERrtvrpFHFJYk9tRX', 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, "b": 12}', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}, {'name': 'add', 'args': '{"a"', 'id': 'call_LaD7wdiERrtvrpFHFJYk9tRX', 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, "b": 12}', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}, {'name': 'add', 'args': '{"a": 11,', 'id': 'call_LaD7wdiERrtvrpFHFJYk9tRX', 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, "b": 12}', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}, {'name': 'add', 'args': '{"a": 11, "b": ', 'id': 'call_LaD7wdiERrtvrpFHFJYk9tRX', 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, "b": 12}', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}, {'name': 'add', 'args': '{"a": 11, "b": 49}', 'id': 'call_LaD7wdiERrtvrpFHFJYk9tRX', 'index': 1, 'type': 'tool_call_chunk'}]
[{'name': 'multiply', 'args': '{"a": 3, "b": 12}', 'id': 'call_teCWWOn5TEwxCghguGe9tr6T', 'index': 0, 'type': 'tool_call_chunk'}, {'name': 'add', 'args': '{"a": 11, "b": 49}', 'id': 'call_LaD7wdiERrtvrpFHFJYk9tRX', 'index': 1, 'type': 'tool_call_chunk'}]
"""

first = True
async for chunk in llm_with_tools.astream(query):
    if first:
        gathered = chunk
        first = False
    else:
        gathered = gathered + chunk

    print(gathered.tool_calls) # 输出的args参数是一个dict
```

