用于在多 Agent 架构中让某个 Agent 将控制权转交给另一个 Agent（通过返回一个 `Command` 命令）。这个模式常用于多智能体（multi-agent）系统或工作流系统中。

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import InjectedState
from langgraph.graph import StateGraph, START, MessagesState
from langgraph.types import Command


def create_handoff_tool(*, agent_name: str, description: str | None = None):
	name = f"transfer_to_{agent_name}"
	description = description or f"Ask {agent_name} for help."

	@tool(name, description=description)
	def handoff_tool(
		# `InjectedState` 是一个“注入型参数”的类型标记，意味着这个参数不是由调用者手动传入，而是由框架自动在运行时注入的。拿到graph会话状态（`state`）
		state: Annotated[MessagesState, InjectedState],
		# `InjectedToolCallId` 是另一个自动注入参数，代表这个工具被调用时的唯一「调用 ID」
		tool_call_id: Annotated[str, InjectedToolCallId],
	) -> Command:

		tool_message = {
			"role": "tool",
			"content": f"Successfully transferred to {agent_name}",
			"name": name,
			"tool_call_id": tool_call_id,
		}
		
		return Command(
			goto=agent_name, # graph注册的node name
			update={**state, "messages": state["messages"]+[tool_message]},
			graph=Command.PARENT # 当前agent所在graph的父graph,这个案例中因为supervisor已经是最上层的graph，所以还是自身
		)

	return handoff_tool

  

# Handoffs
assign_to_research_agent = create_handoff_tool(
	agent_name="research_agent",
	description="Assign task to a researcher agent."
)

assign_to_math_agent = create_handoff_tool(
	agent_name="math_agent",
	description="Assign task to a math agent."
)
```