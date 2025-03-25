- [地址](https://python.langchain.com/docs/how_to/chat_token_usage_tracking/)

<mark style="background: #FFB86CA6;">加载模型：</mark>
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai"
)
```

<mark style="background: #FFB86CA6;">AIMessage.usage_metadata:</mark>
```python
openai_response = llm.invoke("hello")  
openai_response.usage_metadata
"""
----------- output ------------
{'input_tokens': 8, 'output_tokens': 9, 'total_tokens': 17}
"""
```

AIMessage.response_metadata：
```python
print(f'OpenAI: {openai_response.response_metadata["token_usage"]}\n')
"""
----------- output ------------
OpenAI: {'completion_tokens': 9, 'prompt_tokens': 8, 'total_tokens': 17}
"""
```

<mark style="background: #FFB86CA6;">openai在langchain-community有人实现了context managers跟踪token的花了多少钱：</mark>
```python
# 在上下文管理器中的token花费和生成情况都会被输出
with get_openai_callback() as cb:  
	result = llm.invoke("Tell me a joke")  
	print(cb)
"""
-------------- output -------------
Tokens Used: 27
	Prompt Tokens: 11
	Completion Tokens: 16
Successful Requests: 1
Total Cost (USD): $2.95e-05
"""
```