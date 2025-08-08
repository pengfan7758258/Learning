- 安装：`pip install langchain-mcp-adapters`

# 自定义MCP Server
-  `math_server.py`
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Math")

@mcp.tool()
def add(a: int, b: int) -> int:
	"""Add two numbers"""
	return a + b
	
@mcp.tool()
def multiply(a: int, b: int) -> int:
	"""Multiply two numbers"""
	return a * b

if __name__ == "__main__":
	# stdio表示本地
	mcp.run(transport="stdio")
```
- `weather_server.py`
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather")

@mcp.tool()
async def get_weather(location: str) -> str:
	"""Get weather for location."""
	return "It's always sunny in New York"

if __name__ == "__main__":
	# streamable-http表示remote server
	mcp.run(transport="streamable-http")
```

# MCP Client
- `in_a_agent.py`
```python
import asyncio

from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent

client = MultiServerMCPClient(
	{
	"math": {
		"command": "python",
		# 替换你的py绝对路径 math_server.py file
		"args": ["/home/.../math_server.py"],
		"transport": "stdio",
	},
	"weather": {
		# 确保你启动了本地的 weather server on port 8000
		"url": "http://localhost:8000/mcp",
		"transport": "streamable_http",
		}
	}
)

# 定义 `tools` and 每一个response that use `await` under the async `main()`
async def main():
	# 获取所有工具
	tools = await client.get_tools()
	print("tools:", tools)
	
	agent = create_react_agent( # 需要"OPENAI_API_KEY"环境变量
		"openai:gpt-4o",
		tools # llm绑定工具
	)
	
	math_response = await agent.ainvoke({"messages": "what's (3 + 5) x 12?"})
	print(math_response)
	weather_response = await agent.ainvoke({"messages": "what is the weather in nyc?"})
	print(weather_response)

if __name__ == "__main__":
	asyncio.run(main())
```
- `in_a_workflow.py`
```python
import asyncio

from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.prebuilt import ToolNode, tools_condition
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-4.1")

client = MultiServerMCPClient(
	{
	"math": {
		"command": "python",
		# 替换你的py绝对路径 math_server.py file
		"args": ["/home/.../math_server.py"],
		"transport": "stdio",
	},
	"weather": {
		# 确保你启动了本地的 weather server on port 8000
		"url": "http://localhost:8000/mcp",
		"transport": "streamable_http",
		}
	}
)

async def main():
	# 获取所有工具
	tools = await client.get_tools()
	llm_with_tools = model.bind_tools(tools)
	print("tools:", tools)
	
	def call_model(state: MessagesState):
		print("\n"+"-"*30+"\n")
		print(state["messages"])
		print("-"*30+"\n\n")
		response = llm_with_tools.invoke(state["messages"])
		return {"messages": [response]}
	
	builder = StateGraph(MessagesState)
	builder.add_node(call_model)
	builder.add_node(ToolNode(tools))
	builder.add_edge(START, "call_model")
	builder.add_conditional_edges(
		"call_model",
		tools_condition,
	)
	builder.add_edge("tools", "call_model")
	graph = builder.compile()
	# graph.get_graph().draw_png("graph.png") # graph workflow png保存本地
	
	math_response = await graph.ainvoke({"messages": "what's (3 + 5) x 12?"})
	print(math_response)
	weather_response = await graph.ainvoke({"messages": "what is the weather in nyc?"})
	print(weather_response)

  
if __name__ == "__main__":
	asyncio.run(main())
```