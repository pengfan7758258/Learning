![[03-langgraph之Essay Writer(论文撰写器).png]]

```python
from dotenv import load_dotenv
load_dotenv()
from langgraph.graph import StateGraph, END
from typing import TypedDict, List
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.messages import SystemMessage, HumanMessage
from langchain_openai import ChatOpenAI

memory = InMemorySaver()

class AgentState(TypedDict):
	task: str
	plan: str
	draft: str
	critique: str
	content: List[str]
	revision_number: int
	max_revisions: int

model = ChatOpenAI(model="gpt-3.5-turbo", temperature=0)

# 你是一位专家写作者，任务是写一篇高水平的论文大纲。 为用户提供的主题写一个这样的大纲。给出论文的提纲以及任何相关的注释或章节说明。
PLAN_PROMPT = """You are an expert writer tasked with writing a high level outline of an essay. \
Write such an outline for the user provided topic. Give an outline of the essay along with any relevant notes \
or instructions for the sections."""

# 你是一名论文助理，负责撰写优秀的5段式论文根据用户的要求和初步大纲，尽可能地生成最佳论文。如果用户提出修改建议，
# 则根据这些建议修改你先前写的版本。根据需要使用以下所有信息：
# ------
# {content}
WRITER_PROMPT = """You are an essay assistant tasked with writing excellent 5-paragraph essays.\
Generate the best essay possible for the user's request and the initial outline. \
If the user provides critique, respond with a revised version of your previous attempts. \
Utilize all the information below as needed:

------

{content}"""

# 你是一名老师，正在批改一篇论文。为用户提交的内容生成评论和建议。提供详细的建议，包括长度、深度、风格等要求。
REFLECTION_PROMPT = """You are a teacher grading an essay submission. \
Generate critique and recommendations for the user's submission. \
Provide detailed recommendations, including requests for length, depth, style, etc."""

# 你是一位研究人员，负责提供可用于撰写下列论文的信息。请生成一组搜索关键词，以便获取相关资料。关键词最多不超过三个
RESEARCH_PLAN_PROMPT = """You are a researcher charged with providing information that can \
be used when writing the following essay. Generate a list of search queries that will gather \
any relevant information. Only generate 3 queries max."""

# 您是一名研究人员，负责提供以下信息：在进行任何要求的修订时使用（如下所述）。生成一个搜索查询列表，以收集任何相关信息。最多只生成3个查询。
RESEARCH_CRITIQUE_PROMPT = """You are a researcher charged with providing information that can \
be used when making any requested revisions (as outlined below). \
Generate a list of search queries that will gather any relevant information. Only generate 3 queries max."""

from pydantic import BaseModel
class Queries(BaseModel):
	queries: List[str]

# pip install tavily-python
from tavily import TavilyClient
import os
tavily = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

def plan_node(state: AgentState):
	messages = [
		SystemMessage(content=PLAN_PROMPT),
		HumanMessage(content=state['task'])
	]
	response = model.invoke(messages)
	return {"plan": response.content}

def research_plan_node(state: AgentState):
	queries = model.with_structured_output(Queries).invoke([
		SystemMessage(content=RESEARCH_PLAN_PROMPT),
		HumanMessage(content=state['task'])
	])
	
	content = state.get('content', [])
	for q in queries.queries:
		response = tavily.search(query=q, max_results=2)
		for r in response['results']:
			content.append(r['content'])
	return {"content": content}

def generation_node(state: AgentState):
	content = "\n\n".join(state['content'] or []) # 根据task生成queries，并根据queries在tavily获取内容
	user_message = HumanMessage(content=f"{state['task']}\n\nHere is my plan:\n\n{state['plan']}") # plan是根据task生成的论文大纲
	messages = [
		SystemMessage(content=WRITER_PROMPT.format(content=content)), # 开始生成论文具体内容-草稿版本
		user_message
	]
	response = model.invoke(messages)
	
	return {
		"draft": response.content,
		"revision_number": state.get("revision_number", 1) + 1
	}

def reflection_node(state: AgentState):
	messages = [
		SystemMessage(content=REFLECTION_PROMPT), # 开始批改论文草稿
		HumanMessage(content=state['draft'])
	]
	
	response = model.invoke(messages)
	return {"critique": response.content}

def research_critique_node(state: AgentState):
	queries = model.with_structured_output(Queries).invoke([
		SystemMessage(content=RESEARCH_CRITIQUE_PROMPT), # 根据批改意见生成新的queries
		HumanMessage(content=state['critique'])
	
	])
	
	content = state['content'] or []
	for q in queries.queries:
		response = tavily.search(query=q, max_results=2)
		for r in response['results']:
			content.append(r['content'])
	
	return {"content": content}

def should_continue(state):
	if state["revision_number"] > state["max_revisions"]:
		return END
	return "reflect"

builder = StateGraph(AgentState)
builder.add_node("planner", plan_node)
builder.add_node("generate", generation_node)
builder.add_node("reflect", reflection_node)
builder.add_node("research_plan", research_plan_node)
builder.add_node("research_critique", research_critique_node)
builder.set_entry_point("planner")
builder.add_conditional_edges(
	"generate",
	should_continue,
	{END: END, "reflect": "reflect"}
)
builder.add_edge("planner", "research_plan")
builder.add_edge("research_plan", "generate")
builder.add_edge("reflect", "research_critique")
builder.add_edge("research_critique", "generate")
graph = builder.compile(checkpointer=memory)

thread = {"configurable": {"thread_id": "1"}}
for s in graph.stream({
	'task': "what is the difference between langchain and langsmith",
	"max_revisions": 2,
	"revision_number": 1,
	}, thread):
	print(s)
```

