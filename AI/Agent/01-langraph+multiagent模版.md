![[01-langraph+multiagent模版.png]]

```python
from operator import add
from typing import TypedDict, Annotated, Literal

from dotenv import load_dotenv
from langchain.chat_models import init_chat_model
from langchain_core.messages import AnyMessage, AIMessage, HumanMessage
from langgraph.graph import StateGraph
from langgraph.config import get_stream_writer
from langgraph.constants import START, END

load_dotenv()

llm = init_chat_model("openai:gpt-4o")

node_types = ["joke", "other"]

# Define langgraph state
class State(TypedDict):
	messages: Annotated[list[AnyMessage], add]
	node_type: str

def supervisor_node(state: State) -> State:
	print(">>> supervisor_node")
	writer = get_stream_writer()
	writer({"node": ">>> supervisor_node"})
	
	if state.get("node_type"):
		return {"node_type": "END"}

	# 根据用户问题，对问题进行分类，分类结果保存到type中	
	system_prompt = "你是一个专业的助手，负责对用户的问题进行分类，并将任务分给其他Agent执行。" \
	"用户问题是关于讲笑话的，分类结果是 joke。" \
	"用户问题是关于其他的，分类结果是 other。" \
	"只返回分类结果，不要返回其他内容。"
	
	user_prompt = state["messages"][0].content
	messages = [
		("system", system_prompt),
		("user", user_prompt)
	]
	
	# Invoke the LLM to classify the user question
	response = llm.invoke(messages)
	node_type = response.content.strip()
	writer({"node_type": node_type})
	if node_type not in node_types:
		node_type = "other_node" # Default to other_node if classification fails
	
	return {"node_type": node_type}

  

def joke_node(state: State) -> State:
	print(">>> joke_node")
	writer = get_stream_writer()
	writer({"node": ">>> joke_node"})
	
	system_prompt = "你是一个专业的笑话讲述者，不超过100字。"
	user_prompt = state["messages"][0].content
	messages = [
		("system", system_prompt),
		("user", user_prompt)
	]
	
	# Invoke the LLM to generate a joke
	response = llm.invoke(messages)
	content = response.content
	return {"messages": [AIMessage(content=content)]}

  
def other_node(state: State) -> State:
	print(">>> other_node")
	writer = get_stream_writer()
	writer({"node": ">>> other_node"})
	
	return {"messages": [AIMessage(content="无法回答.")]}

  
  
def routing_func(state: State) -> Literal["joke_node", "other_node", "END"]:
	print(">>> routing_func")	
	writer = get_stream_writer()
	writer({"node": ">>> routing_func"})
	
	if state["node_type"] == "joke":
		return "joke_node"
	elif state["node_type"] == "other":
		return "other_node"
	else:
		return "END"

  

# builder graph
builder = StateGraph(State)

# add nodes
builder.add_node("supervisor_node", supervisor_node)
builder.add_node("joke_node", joke_node)
builder.add_node("other_node", other_node)

# add edges
builder.add_edge(START, "supervisor_node")
builder.add_conditional_edges(
	"supervisor_node",
	routing_func,
	{
		"joke_node": "joke_node",
		"other_node": "other_node",
		"END": END
	}
)

for node in ["joke_node", "other_node"]:
	builder.add_edge(node, "supervisor_node")

  
graph = builder.compile()
# download graph workflow png
# graph.get_graph().draw_png("langraph_multiagent_example.png")


for chunk in graph.stream({
	"messages": [
		HumanMessage(content="今天天气怎么样？")
	
	]},
	stream_mode="updates"):
	print(chunk)
```

