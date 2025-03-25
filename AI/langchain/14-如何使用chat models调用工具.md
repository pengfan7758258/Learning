- [地址](https://python.langchain.com/docs/how_to/tool_calling/)
- `tool calling`使模型仅生成工具的参数，实际运行（或不运行）取决于用户
![[14-如何使用chat models调用工具.png]]
- 并不是所有model都支持`tool calling`,官网有提供支持工具调用的[模型地址](https://python.langchain.com/docs/integrations/chat/)
- langchain实现了标注接口定义工具，将工具传递给LLMs和工具调用

## 定义Tool
- 为了使模型能够调用工具，我们需要传入描述工具功能及其参数的tool schema
- 支持`tool calling`的chat model实现了`.bind_tool()`，用于将tool传递给LLM
- tool schema的定义方式
	- Python function
		- 注意：如果后续使用agent一起结合，这种方式会报错
		- 可以使用from langchain_core.tools import tool装饰器装饰python函数
```python
# 函数名、类型提示、docstring都是传递给模型的工具模式的一部分。定义良好的描述性模式是快速工程的扩展，是使模型性能良好的重要组成部分。

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

```
	- Pydantic model
		- 请注意，除非提供默认值，否则所有字段都`是必需的`
```python
from pydantic import BaseModel, Field

class add(BaseModel):
    """Add two integers."""
    a: int = Field(..., description="First integer")
    b: int = Field(..., description="Second integer")

class multiply(BaseModel):
    """Multiply two integers."""
    a: int = Field(..., description="First integer")
    b: int = Field(..., description="Second integer")
```
	- TypedDict class
```python
from typing_extensions import Annotated, TypedDict

class add(TypedDict):
    """Add two integers."""
    # Annotations必须具有type，并且可以选择包含默认值和描述（按此顺序）。
    a: Annotated[int, ..., "First integer"]
    b: Annotated[int, ..., "Second integer"]

class multiply(TypedDict):
    """Multiply two integers."""
    a: Annotated[int, ..., "First integer"]
    b: Annotated[int, ..., "Second integer"]

tools = [add, multiply]
```
	- LangChain `Tool objects`
		- 有内置的一些Tools工具可以调用
		- 还可以使用`from langchain_core.tools import tool`使用@tool的方式装饰一个工具


<mark style="background: #FFB86CA6;">LLM绑定tool并运行：</mark>
```python
# 初始化模型
from langchain.chat_models import init_chat_model  
  
llm = init_chat_model("gpt-4o-mini", model_provider="openai")

# 绑定工具并运行
llm_with_tools = llm.bind_tools(tools)

query = "What is 3 * 12?"

llm_with_tools.invoke(query)
# output：AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_iXj4DiW1p7WLjTAQMRO0jxMs', 'function': {'arguments': '{"a":3,"b":12}', 'name': 'multiply'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 17, 'prompt_tokens': 80, 'total_tokens': 97}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_483d39d857', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-0b620986-3f62-4df7-9ba3-4595089f9ad4-0', tool_calls=[{'name': 'multiply', 'args': {'a': 3, 'b': 12}, 'id': 'call_iXj4DiW1p7WLjTAQMRO0jxMs', 'type': 'tool_call'}], usage_metadata={'input_tokens': 80, 'output_tokens': 17, 'total_tokens': 97})
# 可以看到生成的是一个tool_call的AIMessage

# 多工具调用
query = "What is 3 * 12? Also, what is 11 + 49?"

llm_with_tools.invoke(query)
# AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_chVsrnjW8n5HiO0LFN8f5zUg', 'function': {'arguments': '{"a": 3, "b": 12}', 'name': 'multiply'}, 'type': 'function'}, {'id': 'call_VHM7aCfaL5LbQ0hwDZ4hxkFL', 'function': {'arguments': '{"a": 11, "b": 49}', 'name': 'add'}, 'type': 'function'}], 'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 51, 'prompt_tokens': 109, 'total_tokens': 160, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_b8bc95a0ac', 'finish_reason': 'tool_calls', 'logprobs': None}, id='run-0d0be08d-7254-4bf9-83f8-6f63a844aa40-0', tool_calls=[{'name': 'multiply', 'args': {'a': 3, 'b': 12}, 'id': 'call_chVsrnjW8n5HiO0LFN8f5zUg', 'type': 'tool_call'}, {'name': 'add', 'args': {'a': 11, 'b': 49}, 'id': 'call_VHM7aCfaL5LbQ0hwDZ4hxkFL', 'type': 'tool_call'}], usage_metadata={'input_tokens': 109, 'output_tokens': 51, 'total_tokens': 160, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})

# `.tool_calls` 包含有效的工具调用和其参数
llm_with_tools.invoke(query).tool_calls
"""
------------ output -----------------
[{'name': 'multiply',
  'args': {'a': 3, 'b': 12},
  'id': 'call_1fyhJAbJHuKQe6n0PacubGsL',
  'type': 'tool_call'},
 {'name': 'add',
  'args': {'a': 11, 'b': 49},
  'id': 'call_fc2jVkKzwuPWyU7kS9qn1hyG',
  'type': 'tool_call'}]
"""
```
<mark style="background: #FFB86CA6;">输出使用工具解析器：</mark>
- 注意：只有pydandit类定义的工具才能使用此PydanticToolsParser
```python
from langchain_core.output_parsers import PydanticToolsParser
from pydantic import BaseModel, Field


class add(BaseModel):
    """Add two integers."""
    a: int = Field(..., description="First integer")
    b: int = Field(..., description="Second integer")


class multiply(BaseModel):
    """Multiply two integers."""
    a: int = Field(..., description="First integer")
    b: int = Field(..., description="Second integer")

chain = llm_with_tools | PydanticToolsParser(tools=[add, multiply])
chain.invoke(query)
# output： [multiply(a=3, b=12), add(a=11, b=49)]
```