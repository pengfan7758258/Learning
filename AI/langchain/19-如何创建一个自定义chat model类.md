- [地址](https://python.langchain.com/docs/how_to/custom_chat_model/)
- 使用标准 `BaseChatModel` 接口包装您的 LLM，允许您以最少的代码修改在现有的 LangChain 程序中使用您的 LLM！
- 实现 `BaseChatModel`的三个好处：
	- 自动成为LangChain Runnable
	- batch via a threadpool
	- async support
	- astream_events API
- langchain内置message类型
![[19-如何创建一个自定义chat model类.png]]

<mark style="background: #FFB8EBA6;">实现一个聊天模型，该模型回显prompt中最后一条message的前n个字符</mark>
<mark style="background: #FFB86CA6;">导包：</mark>
```python
# 导包
from typing import Any, Dict, Iterator, List, Optional

from langchain_core.callbacks import CallbackManagerForLLMRun
from langchain_core.language_models import BaseChatModel
from langchain_core.messages import (
    AIMessage,
    AIMessageChunk,
    BaseMessage,
    HumanMessage
)
from langchain_core.messages.ai import UsageMetadata
from langchain_core.outputs import ChatGeneration, ChatGenerationChunk, ChatResult
from pydantic import Field
```
<mark style="background: #FFB86CA6;">自定义chat model：</mark>
- \_generate重写：invoke、batch等方法调用时所处逻辑
- \_stream重写：stream方法流式输出所处逻辑
```python
class ChatParrotLink(BaseChatModel):
    """自定义一个chat model, 与输入的`parrot_buffer_length`字符相关

    在为LangChain贡献实现时, 请仔细记录该模型包括初始化参数，包括
    如何初始化模型并包含任何相关内容的示例链接到基础模型文档或API.

    Example:

        .. code-block:: python
            model = ChatParrotLink(parrot_buffer_length=2, model="bird-brain-001")
            result = model.invoke([HumanMessage(content="hello")])
            # AIMessage(content='hel', ...)
            result = model.batch([[HumanMessage(content="hello")],
                                 [HumanMessage(content="world")]])

    """
    # 模型名
    model_name: str = Field(alias="model")
    # 要回显的prompt最后一条message中的字符数
    parrot_buffer_length: int
    temperature: Optional[float] = None
    max_tokens: Optional[int] = None
    timeout: Optional[int] = None
    stop: Optional[List[str]] = None
    max_retries: int = 2

    def _generate(
        self,
        messages: List[BaseMessage],
        stop: Optional[List[str]] = None,
        run_manager: Optional[CallbackManagerForLLMRun] = None,
        **kwargs: Any,
    ) -> ChatResult:
        """重写 _generate 方法实现chat model逻辑.

        这可以是对API的调用、对本地模型的调用或任何其他生成对输入prompt响应的实现.

        Args:
            messages: message list组成的prompt.
            stop: 模型应停止生成的字符串列表。如果生成因停止token而停止,则停止token本身应作为输出的一部分.
                  目前这一规则并未在所有模型中被强制执行，但遵循它是一个很好的实践，
                  因为它可以大大简化模型输出的解析过程，并更容易理解生成过程为何停止
            run_manager: 带有LLM callback的run manager.
        """
        # 用实际逻辑替换它，以从消息列表生成响应
        last_message = messages[-1]
        tokens = last_message.content[: self.parrot_buffer_length]
        ct_input_tokens = sum(len(message.content) for message in messages) # 输入token个数
        ct_output_tokens = len(tokens) # 输出token个数
        message = AIMessage(
            content=tokens,
            additional_kwargs={},  # 添加message额外参数
            response_metadata={  # 使用response metadata
                "time_in_seconds": 3,
            },
            usage_metadata={
                "input_tokens": ct_input_tokens,
                "output_tokens": ct_output_tokens,
                "total_tokens": ct_input_tokens + ct_output_tokens,
            },
        )

        generation = ChatGeneration(message=message)
        return ChatResult(generations=[generation])

    @property
    def _llm_type(self) -> str:
        """获取此聊天模型使用的language model类型"""
        return "echoing-chat-model-advanced"

    def _stream(
        self,
        messages: List[BaseMessage],
        stop: Optional[List[str]] = None,
        run_manager: Optional[CallbackManagerForLLMRun] = None,
        **kwargs: Any,
    ) -> Iterator[ChatGenerationChunk]:
        """model Stream output.

        如果模型可以生成流式输出，则应实现此方法以流媒体的方式。如果模型不支持流式传输，
        不要实现它。在这种情况下, 流式传输请求将自动执行由_generate方法处理

        Args:
            messages: message list组成的prompt.
            stop: 模型应停止生成的字符串列表。如果生成因停止token而停止,则停止token本身应作为输出的一部分.
                  目前这一规则并未在所有模型中被强制执行，但遵循它是一个很好的实践，
                  因为它可以大大简化模型输出的解析过程，并更容易理解生成过程为何停止
            run_manager: 带有LLM callback的run manager.
        """
        last_message = messages[-1]
        tokens = last_message.content[: self.parrot_buffer_length]
        ct_input_tokens = sum(len(message.content) for message in messages)

        # 流式输出逻辑
        for token in tokens:
            usage_metadata = UsageMetadata(
                {
                    "input_tokens": ct_input_tokens,
                    "output_tokens": 1,
                    "total_tokens": ct_input_tokens + 1,
                }
            )
            ct_input_tokens = 0
            chunk = ChatGenerationChunk(
                message=AIMessageChunk(content=token, usage_metadata=usage_metadata)
            )

            if run_manager:
                # 在LangChain的较新版本中，这是可选的。on_lim_new_token将被自动调用
                run_manager.on_llm_new_token(token, chunk=chunk)
            yield chunk

        # add other information (e.g., response metadata)
        chunk = ChatGenerationChunk(
            message=AIMessageChunk(content="", response_metadata={"time_in_sec": 3})
        )

        if run_manager:
            # 在LangChain的较新版本中，这是可选的。on_lim_new_token将被自动调用
            run_manager.on_llm_new_token(token, chunk=chunk)
        yield chunk

    @property
    def _identifying_params(self) -> Dict[str, Any]:
        """返回标识参数的字典.
        LangChain回调系统使用此信息用于跟踪目的, 使监控LLM成为可能。
        """
        return {
            # The model name allows users to specify custom token counting
            # rules in LLM monitoring applications (e.g., in LangSmith users
            # can provide per token pricing for their model and monitor
            # costs for the given LLM.)
            "model_name": self.model_name,
        }
```
<mark style="background: #FFB86CA6;">测试：</mark>
```python
model = ChatParrotLink(parrot_buffer_length=3, model="my_custom_model")

model.invoke(
    [
        HumanMessage(content="hello!"),
        AIMessage(content="Hi there human!"),
        HumanMessage(content="Meow!"),
    ]
)
"""
------------------ output -------------------
AIMessage(content='Meo', additional_kwargs={}, response_metadata={'time_in_seconds': 3}, id='run-0ec17852-30d4-4ef1-b74f-59836e44f025-0', usage_metadata={'input_tokens': 26, 'output_tokens': 3, 'total_tokens': 29})
"""

model.invoke("hello")
"""
------------------ output -------------------
AIMessage(content='hel', additional_kwargs={}, response_metadata={'time_in_seconds': 3}, id='run-156cd7c4-c432-4b22-9267-f6cbe4f786b4-0', usage_metadata={'input_tokens': 5, 'output_tokens': 3, 'total_tokens': 8})
"""

result = await model.ainvoke("hello")
result
"""
------------------ output -------------------
AIMessage(content='hel', additional_kwargs={}, response_metadata={'time_in_seconds': 3}, id='run-873c7c04-a100-4091-ba9d-2041e2873177-0', usage_metadata={'input_tokens': 5, 'output_tokens': 3, 'total_tokens': 8})
"""

model.batch(["hello", "goodbye"])
"""
------------------ output -------------------
[AIMessage(content='hel', additional_kwargs={}, response_metadata={'time_in_seconds': 3}, id='run-96739ad7-370e-4337-9a6c-4b673c9889cf-0', usage_metadata={'input_tokens': 5, 'output_tokens': 3, 'total_tokens': 8}),
 AIMessage(content='goo', additional_kwargs={}, response_metadata={'time_in_seconds': 3}, id='run-49d2cc0a-9105-4f62-93e2-e2c9c92ae2dd-0', usage_metadata={'input_tokens': 7, 'output_tokens': 3, 'total_tokens': 10})]
"""

for chunk in model.stream("cat"):
    print(chunk.content, end="|")
# output: c|a|t||

async for chunk in model.astream("cat"):
    print(chunk.content, end="|")
# output: c|a|t||
```