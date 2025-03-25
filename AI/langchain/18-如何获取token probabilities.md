- [地址](https://python.langchain.com/docs/how_to/logprobs/)

<mark style="background: #FFB86CA6;">code:</mark>
```python
from langchain.chat_models import init_chat_model

# 初始化模型
llm = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai"
)
# 配置logprobs
llm = llm.bind(logprobs=True)
# init_chat_model(model="gpt-4o-mini", model_provider="openai").bind(logprobs=True)

msg = llm.invoke("hello")
print(msg.response_metadata["logprobs"]["content"][:5])
"""
[{'token': 'Hello',
  'bytes': [72, 101, 108, 108, 111],
  'logprob': -4.5491004129871726e-05,
  'top_logprobs': []},
 {'token': '!', 'bytes': [33], 'logprob': 0.0, 'top_logprobs': []},
 {'token': ' How',
  'bytes': [32, 72, 111, 119],
  'logprob': -9.088346359931165e-07,
  'top_logprobs': []},
 {'token': ' can',
  'bytes': [32, 99, 97, 110],
  'logprob': -1.8624639324116288e-06,
  'top_logprobs': []},
 {'token': ' I', 'bytes': [32, 73], 'logprob': 0.0, 'top_logprobs': []}]
"""
```
<mark style="background: #FFB86CA6;">streaming:</mark>
```python
ct = 0
full = None

# AIMessageChunk可以使用简单的+进行拼接
for chunk in llm.stream("hello"):
    if ct < 5:
        full = chunk if full is None else full + chunk
        if "logprobs" in full.response_metadata:
            print(full.response_metadata["logprobs"]["content"])
    else:
        break

    ct += 1

"""
[]
[{'token': 'Hello', ..., 'logprob': -2.165027217415627e-05, ...}]
[{'token': 'Hello', ..., 'logprob': -2.165027217415627e-05,...}, {'token': '!', ..., 'logprob': 0.0, ...}]
[{'token': 'Hello', ..., 'logprob': -2.165027217415627e-05, ...}, {'token': '!', ..., 'logprob': 0.0, ...}, {'token': ' How', ..., 'logprob': -1.0280383548888494e-06, ...}]
[{'token': 'Hello', ..., 'logprob': -2.165027217415627e-05, ...}, {'token': '!', ..., 'logprob': 0.0, ...}, {'token': ' How', ..., 'logprob': -1.0280383548888494e-06, ...}, {'token': ' can', ..., 'logprob': -1.2664456789934775e-06, ...}]
"""
```