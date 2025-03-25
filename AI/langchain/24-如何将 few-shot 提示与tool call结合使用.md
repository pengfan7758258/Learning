- [地址](https://python.langchain.com/docs/how_to/tools_few_shot/)
- 对于更复杂的工具使用，在提示符中添加`few-shot`非常有用

```python
from langchain.chat_models import init_chat_model
from langchain_core.tools import tool

# 加载模型
llm = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai",
)

# 定义工具
@tool
def add(a: int, b: int) -> int:  
    """Add two integers.  
    Args:  
        a: First integer  
        b: Second integer  
    """  
    return a + b  

@tool
def multiply(a: int, b: int) -> int:  
    """Multiply two integers.  
    Args:  
        a: First integer  
        b: Second integer  
    """  
    return a * b

tools = [add, multiply]

# LLM绑定工具
llm = llm.bind_tools(tools)

# 此处只是生成了工具调用的AIMessage，真实调用还需要使用agent去调用
llm.invoke(
    "110+9x7等于多少"
).tool_calls
"""
------------- output ----------------
[{'name': 'add',
  'args': {'a': 110, 'b': 63},
  'id': 'call_fzolstGbCMR8nlheYHqUyEwL',
  'type': 'tool_call'},
 {'name': 'multiply',
  'args': {'a': 9, 'b': 7},
  'id': 'call_UFXpHIXVn9HJXXmjr9Zce5U6',
  'type': 'tool_call'}]
"""

# 添加带有tool call的few-shot
from langchain_core.messages import AIMessage, HumanMessage, ToolMessage
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough

examples = [
    HumanMessage(
        "317253和128472相乘后加4等于多少", name="example_user"
    ),
    AIMessage(
        "",
        name="example_assistant",
        tool_calls=[
            {"name": "multiply", "args": {"x": 317253, "y": 128472}, "id": "1"}
        ],
    ),
    ToolMessage("16505054784", tool_call_id="1"),
    AIMessage(
        "",
        name="example_assistant",
        tool_calls=[{"name": "add", "args": {"x": 16505054784, "y": 4}, "id": "2"}],
    ),
    ToolMessage("16505054788", tool_call_id="2"),
    AIMessage(
        "317253和128472相乘后加4等于16505054788",
        name="example_assistant",
    ),
]

system = """你是一个数学天才"""
few_shot_prompt = ChatPromptTemplate.from_messages(
    [
        ("system", system),
        *examples,
        ("human", "{input}"),
    ]
)
chain = {"input": RunnablePassthrough()} | few_shot_prompt | llm
chain.invoke("119x8-20x(30-11)等于多少？").tool_calls
"""
------------ output --------------
[{'name': 'multiply',
  'args': {'a': 119, 'b': 8},
  'id': 'call_4dH6o0Anq46HYeTPsr1nh0Q2',
  'type': 'tool_call'},
 {'name': 'multiply',
  'args': {'a': 20, 'b': 19},
  'id': 'call_80QeD8E51PzbA72t6XkyHCLH',
  'type': 'tool_call'}]

----------- 另一种调用方式 -----------
chain = few_shot_prompt | llm
chain.invoke(
    {"input": "119x8-20x(30-11)等于多少？"}
).tool_calls
"""
```

<mark style="background: #FFB8EBA6;">真实工具agent调用跳转：</mark>[[16-如何 debug 你的 LLM apps]]