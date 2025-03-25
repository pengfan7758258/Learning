- [地址](https://python.langchain.com/docs/how_to/trim_messages/)
- 所有模型都有context window限制，可以接受的input token数量是有限的。如果你有非常长的消息或一个累积了长消息历史的chain/agent，你需要管理传递给模型的输入消息长度
	1. 修剪为max token
	2. 修剪为num message
- pip install langchain_core\==0.3.47

## trim message参数及其功能
1.一般性：
	1. 聊天历史记录应以`HumanMessage`开头或`SystemMessage` 后跟 `HumanMessage` 开头
	2. 聊天记录以 `HumanMessage` 或 `ToolMessage` 结束
	3. `ToolMessage` 只能在涉及工具调用的 `AIMessage` 之后出现
2. 上面这点通过设置 `start_on="human"` 和 `ends_on=("human", "tool")` 来实现
3. 为了保留最新消息并且丢弃聊天记录中的历史消息。可以通过设置`strategy="last"` 来实现
	1. 这个参数的策略是，从最后一条message开始保留
4. 一般来说System包含了一些特殊instruction（角色、few-shot等），所以需要将其保留，使用`include_system=True` 来实现


## 裁剪message根据token数量
案例需求：根据token数裁剪聊天历史，裁剪后的聊天历史将生成一个有效的聊天历史，其中包含 `SystemMessage` 
1. strategy="last"：保留最新消息
2. include_system=True：保留`SystemMessage` 
3. token_counter：传入具体的model实例（会根据传入的具体model实例去计算token）或一个函数（根据函数逻辑计算token）
4. max_tokens：与`token_counter`一起限制token数
<mark style="background: #FFB86CA6;">code：</mark>
```python
from langchain_core.messages import (
    AIMessage,
    HumanMessage,
    SystemMessage,
    ToolMessage,
    trim_messages,
)
from langchain_core.messages.utils import count_tokens_approximately

messages = [
    SystemMessage("你是一个助手, 你总会以开玩笑的方式回复."),
    HumanMessage("我想知道为什么它叫langchain"),
    AIMessage(
        '好吧，我想他们认为“WordRope”和“SentenceString”只是听起来不一样！'
    ),
    HumanMessage("哈里森到底在追谁"),
    AIMessage(
        "嗯，让我想想。\n\n为什么，他可能在追逐办公室里的最后一杯咖啡！"
    ),
    HumanMessage("你管不会说话的鹦鹉叫什么"),
]


input_messages = trim_messages(
    messages,
    # Keep the last <= n_count tokens of the messages.
    strategy="last",
    # 根据model或者自定义token_counter函数计算token
    token_counter=count_tokens_approximately, # count_tokens_approximately一种近似计算token的方式，最好还是根据所选择的model来计算token数量
    # 根据所需的对话长度进行调整
    max_tokens=20,
    # 大多数聊天模式都希望聊天历史记录以以下任一方式开头:
    # (1) a HumanMessage
    # (2) `SystemMessage` 后跟 `HumanMessage` 开头
    start_on="human",
    # 大多数聊天模式都希望聊天历史记录以以下任一方式结束:
    # (1) a HumanMessage or
    # (2) a ToolMessage
    end_on=("human", "tool"),
    # 通常,我们需要保留SystemMessage
    include_system=True,
    # allow_partial是否部分保存，False代表否，如果是True可能出现不是一句完整的话语
    allow_partial=False,
)
input_messages
"""
--------------- output -----------------
[SystemMessage(content='你是一个助手, 你总会以开玩笑的方式回复.', additional_kwargs={}, response_metadata={}),
 HumanMessage(content='你管不会说话的鹦鹉叫什么', additional_kwargs={}, response_metadata={})]
"""
```

## 裁剪message根据message数量
1. 设置 `token_counter=len`，在这种情况下，每条message计为一个单独的标记，并配合 `max_tokens` 将控制最大消息数
<mark style="background: #FFB86CA6;">code：</mark>
```python
input_messages = trim_messages(
    messages,
    strategy="last",
    token_counter=len, # 设置为len
    max_tokens=3,
    start_on="human",
    end_on=("human", "tool"),
    include_system=True,
)
input_messages
# 这里可以看到我们设置max_tokens=3但是只保留了两条message，那是因为start_on、end_on、strategy等参数共同作用后的结果
"""
--------------- output -----------------
[SystemMessage(content='你是一个助手, 你总会以开玩笑的方式回复.', additional_kwargs={}, response_metadata={}),
 HumanMessage(content='你管不会说话的鹦鹉叫什么', additional_kwargs={}, response_metadata={})]
"""
```

## 进阶用法
1. allow_partial=True：允许拆分消息的内容
```python
input_messages = trim_messages(
    messages,
    max_tokens=56,
    strategy="last",
    token_counter=count_tokens_approximately,
    include_system=True,
    allow_partial=True,
)
input_messages
"""
这里因为使用的是中文的message，count_tokens_approximately好像对于中文token不好使
可以去官方案例查看这部分的英文输出能力

官方案例：
原始message：
messages = [  
	SystemMessage("you're a good assistant, you always respond with a joke."),  
	HumanMessage("i wonder why it's called langchain"),  
	AIMessage(  
	'Well, I guess they thought "WordRope" and "SentenceString" just didn\'t have the same ring to it!'  
	),  
	HumanMessage("and who is harrison chasing anyways"),  
	AIMessage(  
	"Hmmm let me think.\n\nWhy, he's probably chasing after the last cup of coffee in the office!"  
	),  
	HumanMessage("what do you call a speechless parrot"),  
]

使用trim_messages后：
[SystemMessage(content="you're a good assistant, you always respond with a joke.", additional_kwargs={}, response_metadata={}),
 AIMessage(content="\nWhy, he's probably chasing after the last cup of coffee in the office!", additional_kwargs={}, response_metadata={}),
 HumanMessage(content='what do you call a speechless parrot', additional_kwargs={}, response_metadata={})]

allow_partial=True作用：
可以看到是从最后开始往前数token，除去SystemMessage的token数后，
保留到了倒数第二个AIMessage的部分token
"""
```
2. 可以通过指定 `strategy="first"` 来执行获取第一个 `max_tokens` 的翻转操作
```python
input_messages = trim_messages(
    messages,
    max_tokens=45,
    strategy="first",
    token_counter=count_tokens_approximately,
)
input_messages
"""
官网英文案例输出：
[SystemMessage(content="you're a good assistant, you always respond with a joke.", additional_kwargs={}, response_metadata={}),
 HumanMessage(content="i wonder why it's called langchain", additional_kwargs={}, response_metadata={})]

strategy="first"：
有了这个策略便不需要include_system=False,因为是从开头开始保留
"""

```

## 使用 `ChatModel` 作为token_counter
```python
from langchain.chat_models import init_chat_model

model = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai"
)
input_messages = trim_messages(
    messages,
    max_tokens=45,
    strategy="first",
    token_counter=model,
)
```

## 编写自定义token_counter函数
编写一个自定义令牌计数器函数，该函数接收一个message list并返回一个整数
```python
from typing import List

import tiktoken
from langchain_core.messages import BaseMessage, ToolMessage

# token编码
def str_token_counter(text: str) -> int:
    enc = tiktoken.get_encoding("o200k_base")
    return len(enc.encode(text))

# token_counter函数
def tiktoken_counter(messages: List[BaseMessage]) -> int:
    """近似实现： https://github.com/openai/openai-cookbook/blob/main/examples/How_to_count_tokens_with_tiktoken.ipynb

    为了简单起见仅支持 str Message.contents.
    """
    num_tokens = 3  # 每个回复都包含书主要结构： <|start|>assistant<|message|>
    tokens_per_message = 3
    tokens_per_name = 1
    for msg in messages:
        if isinstance(msg, HumanMessage):
            role = "user"
        elif isinstance(msg, AIMessage):
            role = "assistant"
        elif isinstance(msg, ToolMessage):
            role = "tool"
        elif isinstance(msg, SystemMessage):
            role = "system"
        else:
            raise ValueError(f"Unsupported messages type {msg.__class__}")
        num_tokens += (
            tokens_per_message
            + str_token_counter(role)
            + str_token_counter(msg.content)
        )
        if msg.name:
            num_tokens += tokens_per_name + str_token_counter(msg.name)
    return num_tokens


trim_messages(
    messages,
    token_counter=tiktoken_counter, # 使用我们自定义的token_counter函数
    strategy="last",
    max_tokens=45,
    start_on="human",
    end_on=("human", "tool"),
    include_system=True,
)
```

## Chain实现trim message
```python
llm = ChatOpenAI(model="gpt-4o")

trimmer = trim_messages(
    token_counter=llm,
    strategy="last",
    max_tokens=45,
    start_on="human",
    end_on=("human", "tool"),
    include_system=True
)

chain = trimmer | llm
chain.invoke(messages) # messges -> trimmer message -> llm
```