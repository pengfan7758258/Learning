- [地址](https://python.langchain.com/docs/how_to/filter_messages/)
- `filter_messages` 工具根据type、ID 或name过滤消息
- include包含、exclude排除

```python
from langchain_core.messages import (
    AIMessage,
    HumanMessage,
    SystemMessage,
    filter_messages,
)

messages = [
    SystemMessage("you are a good assistant", id="1"),
    HumanMessage("example input", id="2", name="example_user"),
    AIMessage("example output", id="3", name="example_assistant"),
    HumanMessage("real input", id="4", name="bob"),
    AIMessage("real output", id="5", name="alice"),
]

filter_messages(messages, include_types="human")
"""
---------- output ------------
[HumanMessage(content='example input', name='example_user', id='2'),
 HumanMessage(content='real input', name='bob', id='4')]
"""

filter_messages(messages, exclude_names=["example_user", "example_assistant"])
"""
---------- output ------------
[SystemMessage(content='you are a good assistant', id='1'),
 HumanMessage(content='real input', name='bob', id='4'),
 AIMessage(content='real output', name='alice', id='5')]
"""

filter_messages(messages, include_types=[HumanMessage, AIMessage], exclude_ids=["3"])
"""
---------- output ------------
[HumanMessage(content='example input', name='example_user', id='2'),
 HumanMessage(content='real input', name='bob', id='4'),
 AIMessage(content='real output', name='alice', id='5')]
"""
```