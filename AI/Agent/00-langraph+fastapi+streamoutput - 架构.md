# agent定义
```python
from langchain_core.messages import SystemMessage
from langgraph.graph import START, MessagesState, StateGraph
from langgraph.prebuilt import ToolNode, tools_condition

from datetime import date

def get_agent(llm):
	# Define tools/llm
	tools = ["工具列表"]

	llm_with_tools = llm.bind_tools(tools)

	# System message
	sys_msg = SystemMessage(
		content="You are a helpful assistant tasked with finding and explaining relevant information about internal contracts. "
"Always explain results you get from the tools in a concise manner to not overwhelm the user but also don't be too technical. "
"Answer questions as if you are answering to non-technical management level. "
"Important: Be confident and accurate in your tool choice! Avoid asking follow-up questions if possible. "
f"Today is {date.today()}"
	)

	# Node
	def assistant(state: MessagesState):
		return {"messages": [llm_with_tools.invoke([sys_msg] + state["messages"])]}
	
	# Graph
	builder = StateGraph(MessagesState)
	
	# Define nodes: these do the work
	builder.add_node("assistant", assistant)
	builder.add_node("tools", ToolNode(tools))
	
	# Define edges: these determine how the control flow moves
	builder.add_edge(START, "assistant")
	builder.add_conditional_edges(
		"assistant",
		# If the latest message (result) from assistant is a tool call -> tools_condition routes to tools
		# If the latest message (result) from assistant is a not a tool call -> tools_condition routes to END
		tools_condition,
	)
	
	builder.add_edge("tools", "assistant")
	return builder.compile()
```
![[00-langraph+fastapi+streamoutput - 架构 -1.png]]


# AgentManager
- **管理不同model provider**
```python
import os
from langchain_openai import ChatOpenAI
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_anthropic import ChatAnthropic
from langchain_mistralai import ChatMistralAI

from backend.agent import get_agent

from typing import Any

class AgentManager:
	agents = {}

	def __init__(self):
		self.init_agents()

	def init_agents(self):
		if os.getenv("OPENAI_API_KEY"):
			self.agents["gpt-4o"] = get_agent(ChatOpenAI(model="gpt-4o", temperature=0))
		
		if os.getenv("GOOGLE_API_KEY"):
			self.agents["gemini-1.5-pro"] = get_agent(
				ChatGoogleGenerativeAI(
					model="gemini-1.5-pro",
					temperature=0,
				)
			)
		
			self.agents["gemini-2.0-flash"] = get_agent(
				ChatGoogleGenerativeAI(
					model="gemini-2.0-flash",
					temperature=0,
				)
			)
		
		if os.getenv("ANTHROPIC_API_KEY"):
			self.agents["sonnet-3.5"] = get_agent(
				ChatAnthropic(
					model="claude-3-5-sonnet-latest",
					temperature=0,
				)
			)
		
		if os.getenv("MISTRAL_API_KEY"):
			self.agents["mistral-large"] = get_agent(
				ChatMistralAI(
					model="mistral-large-latest",
				)
			)
		
		print(f"Loaded {len(self.agents)} llms.")

	def get_model_by_name(self, name: str):
		try:
			return self.agents[name]
		except KeyError:
			raise ValueError(f"The model {name} wasn't initiated")
```

# API接口
```python
import json

from dotenv import load_dotenv
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from langchain_core.messages import HumanMessage, ToolMessage, AIMessage, AIMessageChunk

from backend.agent_manager import AgentManager

load_dotenv()

agent_manager = AgentManager()

app = FastAPI()

app.add_middleware(
	CORSMiddleware,
	allow_credentials=True,
	allow_origins=["*"], # Allow all origins
	allow_methods=["*"], # Allow all methods
	allow_headers=["*"], # Allow all headers
)

@app.get("/")
async def root():
	return {"status": "OK"}

class RunPayload(BaseModel):
	model: str
	prompt: str
	history: str

  
def rebuild_history(history):
	"""构建history message list"""

async def runner(model: str, prompt: str, history: str):
	if history != "[]":
		previous_messages = rebuild_history(history)
	else:
		previous_messages = []
	
	prompt_message = HumanMessage(content=prompt)
	input_messages = [*previous_messages, prompt_message]
	messages = agent_manager.get_model_by_name(model).astream(
		input={"messages": input_messages}, stream_mode=["messages", "updates"]
	)
	
	# 仅用于演示:
	# - 我们将内存上下文作为历史传递，因为我们不想在这个演示中处理数据库,一般这块也可以是后端传递过来history data
	# - "updates" stream_mode 返回 AIMessage 而不是 AIMessageChunk
	# - 我们不会将上下文存储在服务器上，以免耗尽所有内存-在生产环境中，您会将其存储在db中
	context = json.loads(history)
	context.append(prompt_message.model_dump_json())
	
	async for message in messages:
		if message[0] == "messages":
			chunk = message[1]
		# output tool call section type
		if hasattr(chunk[0], "tool_calls") and len(chunk[0].tool_calls) > 0:
			for tool in chunk[0].tool_calls:
				if tool.get('name'):
					tool_calls_content = json.dumps(tool)
					yield f"data: {json.dumps({'content': tool_calls_content, 'type': 'tool_call'})}\n\n"
	
			if isinstance(chunk[0], ToolMessage):
				yield f"data: {json.dumps({'content': chunk[0].content, 'type': 'tool_message'})}\n\n"
			if isinstance(chunk[0], AIMessageChunk):
				yield f"data: {json.dumps({'content': chunk[0].content, 'type': 'ai_message'})}\n\n"
			if isinstance(chunk[0], HumanMessage):
				yield f"data: {json.dumps({'content': chunk[0].content, 'type': 'user_message'})}\n\n"
	
		if message[0] == "updates":
			# use pydantic BaseClass method model_dump_json to dump message model to be stringified into history	
			if "assistant" in message[1]:
				for history_message in message[1]["assistant"]["messages"]:
					context.append(history_message.model_dump_json())
				elif "tools" in message[1]:
					for tool_message in message[1]["tools"]["messages"]:
						context.append(tool_message.model_dump_json())
	
	yield f"data: {json.dumps({'content': context, 'type': 'history'})}\n\n"
	yield f"data: {json.dumps({'content': '', 'type': 'end'})}\n\n"

  
@app.post("/run/")
async def run(payload: RunPayload):
	return StreamingResponse(
		runner(model=payload.model, prompt=payload.prompt, history=payload.history),
		media_type="text/event-stream"
	)
```