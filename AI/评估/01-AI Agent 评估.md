# 地址：
- https://www.bilibili.com/video/BV1HTAHebEuj/?spm_id_from=333.788.player.switch&vd_source=e4234f5ddbe45b813cf4296e06e14b9b&p=5
- 标题：`吴恩达《评估Agent|Evaluating AI Agent》`

# 规范化的评估驱动流程，帮助决策以下问题
- 想知道是否应更新特定步骤的prompt吗？
- 想知道是否该调整工作流逻辑吗？
- 想知道是否该更换使用的LLM吗？


# 如何构建评估体系以持续改进agent的输出质量及执行路径

# 添加observability在代码中，一款成熟的observablity的工具Arize Phoenix
持续监控，定期评估
安装：`pip install arize-phoenix`
运行：`python -m phoenix.server.main serve`
地址: `http://localhost:6006`

# 有一招提升LLM生成效果
任务拆分更简单的任务，将本身一步复杂生成的回复，拆分两步甚至多步达到高质量回复

# 可能会出现什么问题？
![[01-AI Agent 评估.png]]
可以使用人工监控、LLM作为判别器两种方式监控输出质量

需要构建一个测试用例集，这样才知道自己在进行更改prompt或code或LLM是导致性能提升还是性能降低。测试用例挑选一些具有代表性的、能反映关键信息的。建立覆盖多用户场景的标准化测试集，确保每次代码变更后自动运行。

# agent主要components
![[01-AI Agent 评估 1.png]]
- 一个router
	- plan、决定使用什么skill or  function call
	- 可以是一个LLM、一个NLP分类器、甚至是一个基于规则的code。按道理来说这块设计的越简单系统越稳定，但是功能范围也会因此受限制
- skills技能模块：包裹追踪、查询商品、qa问答
	- 每个skill是一个完整的逻辑链，可以是API调用、LLM calls、RAG、数据库查询等
- memory和state


在多数agent系统中，执行完skill模块后还会返回到router中继续判断要不要执行其它skill模块还是返回结果

# Agent Example And Phoenix 监控
![[01-AI Agent 评估 2.png]]
架构：
![[01-AI Agent 评估 3.png]]
![[01-AI Agent 评估 4.png]]
```python
"""
pip install openinference-instrumentation-openai
pip install duckdb
pip install arize-phoenix-otel
"""

from openai import OpenAI
import pandas as pd
import json
import duckdb
from pydantic import BaseModel, Field
from IPython.display import Markdown

import phoenix as px
import os
os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "http://localhost:6006"
from phoenix.otel import register
from openinference.instrumentation.openai import OpenAIInstrumentor
from openinference.semconv.trace import SpanAttributes
from opentelemetry.trace import Status, StatusCode
from openinference.instrumentation import TracerProvider

from dotenv import load_dotenv
load_dotenv()

openai_api_key = os.getenv("OPENAI_API_KEY")
if not openai_api_key:
	raise ValueError("OPENAI_API_KEY environment variable is not set.")

phoenix_collector_endpoint = os.getenv("PHOENIX_COLLECTOR_ENDPOINT", "http://localhost:6006/v1/traces")

client = OpenAI(api_key=openai_api_key)
MODEL = "gpt-4o-mini"

PROJECT_NAME="tracing-agent"
tracer_provider = register(
	project_name=PROJECT_NAME,
	endpoint=phoenix_collector_endpoint
)
# 配置OPENAI的自动化，之后有openai相应的请求会自动添加到phoenix监控中
OpenAIInstrumentor().instrument(tracer_provider=tracer_provider)

tracer = tracer_provider.get_tracer(__name__) # 获取tracer

# 配置本地数据，后续使用duckdb读取充当数据库
# 数据集下载地址：https://www.kaggle.com/datasets/banupriya11/store-sales-price-elasticity-promotions-data
TRANSACTION_DATA_FILE_PATH = 'data/Store_Sales_Price_Elasticity_Promotions_Data.parquet'

### Tool 1: Database Lookup
SQL_GENERATION_PROMPT = """
Generate an SQL query based on a prompt. Do not reply with anything besides the SQL query.
The prompt is: {prompt}

The available columns are: {columns}
The table name is: {table_name}
"""
def generate_sql_query(prompt: str, columns: list, table_name: str) -> str:
	"""Generate an SQL query based on a prompt"""
	formatted_prompt = SQL_GENERATION_PROMPT.format(prompt=prompt,
		columns=columns,
		table_name=table_name)
	
	response = client.chat.completions.create(
		model=MODEL,
		messages=[{"role": "user", "content": formatted_prompt}],
	)
	return response.choices[0].message.content

@tracer.tool() # 装饰器配置后会自动化将函数的输入和输出记录到phoenix监控中
def lookup_sales_data(prompt: str) -> str:
	"""Implementation of sales data lookup from parquet file using SQL"""
	try:
		# define the table name
		table_name = "sales"
		# step 1: read the parquet file into a DuckDB table
		df = pd.read_parquet(TRANSACTION_DATA_FILE_PATH)	
		duckdb.sql(f"CREATE TABLE IF NOT EXISTS {table_name} AS SELECT * FROM df")
		# step 2: generate the SQL code
		sql_query = generate_sql_query(prompt, df.columns, table_name)
		# clean the response to make sure it only includes the SQL code
		sql_query = sql_query.strip()
		sql_query = sql_query.replace("```sql", "").replace("```", "")
		with tracer.start_as_current_span("execute_sql_query", openinference_span_kind="chain") as span: # 另外一种手动配置phoenix
			span.set_input(sql_query)
			# step 3: execute the SQL query
			result = duckdb.sql(sql_query).df()
			span.set_output(value=str(result))
			span.set_status(Status(StatusCode.OK, "SQL query executed successfully"))
		return result.to_string()
	except Exception as e:
		return f"Error accessing data: {str(e)}"


### Tool 2: Data Analysis
DATA_ANALYSIS_PROMPT = """
Analyze the following data: {data}
Your job is to answer the following question: {prompt}
"""

@tracer.tool() # 装饰器配置后会自动化将函数的输入和输出记录到phoenix监控中
def analyze_sales_data(prompt: str, data: str) -> str:
	"""Implementation of AI-powered sales data analysis"""
	formatted_prompt = DATA_ANALYSIS_PROMPT.format(data=data, prompt=prompt)
	
	response = client.chat.completions.create(
		model=MODEL,
		messages=[{"role": "user", "content": formatted_prompt}],
	)
	analysis = response.choices[0].message.content
	return analysis if analysis else "No analysis could be generated"


### Tool 3: Data Visualization
CHART_CONFIGURATION_PROMPT = """
Generate a chart configuration based on this data: {data}
The goal is to show: {visualization_goal}
"""

# 配置LLM的structed output BaseModel
class VisualizationConfig(BaseModel):	
	chart_type: str = Field(..., description="Type of chart to generate")
	x_axis: str = Field(..., description="Name of the x-axis column")
	y_axis: str = Field(..., description="Name of the y-axis column")
	title: str = Field(..., description="Title of the chart")

@tracer.chain()
def extract_chart_config(data: str, visualization_goal: str) -> dict:
	"""Generate chart visualization configuration
	
	Args:
		data: String containing the data to visualize
		visualization_goal: Description of what the visualization should show
		
	Returns:
		Dictionary containing line chart configuration
	"""

	formatted_prompt = CHART_CONFIGURATION_PROMPT.format(data=data,
		visualization_goal=visualization_goal)
	
	response = client.beta.chat.completions.parse(
	
	model=MODEL,
	
	messages=[{"role": "user", "content": formatted_prompt}],
		response_format=VisualizationConfig,
	)
	
	try:
		# Extract axis and title info from response
		content = response.choices[0].message.content
		# Return structured chart config
		return {
			"chart_type": content.chart_type,
			"x_axis": content.x_axis,			
			"y_axis": content.y_axis,			
			"title": content.title,			
			"data": data		
		}
	
	except Exception:
		return {	
			"chart_type": "line",
			"x_axis": "date",
			"y_axis": "value",
			"title": visualization_goal,
			"data": data
		}

CREATE_CHART_PROMPT = """
Write python code to create a chart based on the following configuration.
Only return the code, no other text.
config: {config}
"""

@tracer.chain()
def create_chart(config: dict) -> str:
	"""Create a chart based on the configuration"""
	formatted_prompt = CREATE_CHART_PROMPT.format(config=config)
	response = client.chat.completions.create(
		model=MODEL,
		messages=[{"role": "user", "content": formatted_prompt}],
	)
	code = response.choices[0].message.content
	code = code.replace("```python", "").replace("```", "")
	code = code.strip()
	
	return code

@tracer.tool()
def generate_visualization(data: str, visualization_goal: str) -> str:
	"""Generate a visualization based on the data and goal"""
	config = extract_chart_config(data, visualization_goal)
	code = create_chart(config)
	return code


### Tool schema
tools = [
	{
		"type": "function",
		"function": {
			"name": "lookup_sales_data",
			"description": "Look up data from Store Sales Price Elasticity Promotions dataset",
			"parameters": {
				"type": "object",
				"properties": {
					"prompt": {"type": "string", "description": "The unchanged prompt that the user provided."}
				},
				"required": ["prompt"]
			}
		}
	},
	{
		"type": "function",
		"function": {
			"name": "analyze_sales_data",
			"description": "Analyze sales data to extract insights",
			"parameters": {
				"type": "object",
				"properties": {
					"data": {"type": "string", "description": "The lookup_sales_data tool's output."},
					"prompt": {"type": "string", "description": "The unchanged prompt that the user provided."}
				},
				"required": ["data", "prompt"]
			}	
		}
	},
	{
		"type": "function",
		"function": {
			"name": "generate_visualization",
			"description": "Generate Python code to create data visualizations",
			"parameters": {
				"type": "object",
				"properties": {
					"data": {"type": "string", "description": "The lookup_sales_data tool's output."},
					"visualization_goal": {"type": "string", "description": "The goal of the visualization."}
				},			
				"required": ["data", "visualization_goal"]		
			}
		}
	}
]

tool_implementations = {
	"lookup_sales_data": lookup_sales_data,
	"analyze_sales_data": analyze_sales_data,
	"generate_visualization": generate_visualization
}


### Router Logic

@tracer.chain()
def handle_tool_calls(tool_calls, messages):
	for tool_call in tool_calls:
		function = tool_implementations[tool_call.function.name]
		function_args = json.loads(tool_call.function.arguments)
		result = function(**function_args)
		messages.append({"role": "tool", "content": result, "tool_call_id": tool_call.id})
		
	return messages

SYSTEM_PROMPT = """
You are a helpful assistant that can answer questions about the Store Sales Price Elasticity Promotions dataset.
"""

def run_agent(messages):
	print("Running agent with messages:", messages)

	if isinstance(messages, str):
		messages = [{"role": "user", "content": messages}]
	
	# Check and add system prompt if needed
	if not any(
		isinstance(message, dict) and message.get("role") == "system" for message in messages):
		system_prompt = {"role": "system", "content": SYSTEM_PROMPT}
		messages.insert(0, system_prompt) # Ensure system prompt is first
	
	while True:
		# Router Span
		print("Starting router call span")
		with tracer.start_as_current_span(
			"router_call", openinference_span_kind="chain",
		) as span:
			span.set_input(value=messages)
			response = client.chat.completions.create(
				model=MODEL,		
				messages=messages,		
				tools=tools,
			)
			messages.append(response.choices[0].message.model_dump())
			tool_calls = response.choices[0].message.tool_calls
			print("Received response with tool calls:", bool(tool_calls))
			span.set_status(StatusCode.OK)
		if tool_calls:
			print("Starting tool calls span")
			messages = handle_tool_calls(tool_calls, messages)
			span.set_output(value=tool_calls)
		else:
			print("No tool calls, returning final response")
			span.set_output(value=response.choices[0].message.content)
			return response.choices[0].message.content

def start_main_span(messages):
	print("Starting main span with messages:", messages)
	with tracer.start_as_current_span(
		"AgentRun", openinference_span_kind="agent"
	) as span:
		span.set_input(value=messages)
		ret = run_agent(messages)
		print("Main span completed with return value:", ret)
		span.set_output(value=ret)
		span.set_status(StatusCode.OK)
		return ret

# 运行agent，带有phoenix监控，程序结束后可在phoenix监控的endpoint监控页面查看
result = start_main_span([{"role": "user",
"content": "Which stores did the best in 2021?"}])
```


# Three Evaluators
- code-based evaluators 基于代码的评估器：比较预期输出和代码输出
- LLM-as-a-judge LLM评委
![[01-AI Agent 评估 5.png]]
- human annotation 人工标注

## Agent之Router 评估
- Function calling choice：router是否选择了正确的function调用?
- Parameter extraction：function参数是否被正确提取？


# Phoenix 评估各个模块并集成到监控中展示
```python
# 继续 `Agent Example And Phoenix 监控`下的代码
import phoenix as px
import os
import json
from tqdm import tqdm

from phoenix.evals import (
	TOOL_CALLING_PROMPT_TEMPLATE,
	llm_classify,
	OpenAIModel
)
from phoenix.trace import SpanEvaluations
from phoenix.trace.dsl import SpanQuery
from openinference.instrumentation import suppress_tracing

import nest_asyncio
nest_asyncio.apply()

# 设置一些基础测试问题
agent_questions = [
	"What was the most popular product SKU?",
	"What was the total revenue across all stores?",
	"Which store had the highest sales volume?",
	"Create a bar chart showing total sales by store",
	"What percentage of items were sold on promotion?",
	"What was the average transaction value?"
]

# 循环调用
for question in tqdm(agent_questions, desc="Processing questions"):
	try:
		ret = start_main_span([{"role": "user", "content": question}])
	except Exception as e:
		print(f"Error processing question: {question}")
		print(e)
		continue

### 判断LLM作为Router是否选择正确了tool

# 从phoenix导出数据
# 1.设置span query
query = SpanQuery().where(
	# Filter for the `LLM` span kind.
	# The filter condition is a string of valid Python boolean expression.
	"span_kind == 'LLM'", # LLM模块
).select(
	question="input.value",
	tool_call="llm.tools"
)

# 2.获取数据
tool_calls_df = px.Client(endpoint="http://localhost:6006").query_spans(query,
	project_name=PROJECT_NAME,
	timeout=None)

# 只要有tool工具调用的模块
tool_calls_df = tool_calls_df.dropna(subset=["tool_call"])
tool_calls_df.head()

# gpt-4o来评判是否正确调用了tool
with suppress_tracing(): # 这个上下文的内容不会放入phoenix
	tool_call_eval = llm_classify(
		dataframe = tool_calls_df,
		template = TOOL_CALLING_PROMPT_TEMPLATE.template[0].template.replace("{tool_definitions}",
		json.dumps(tools).replace("{", '"').replace("}", '"')),
		rails = ['correct', 'incorrect'],
		model=OpenAIModel(model="gpt-4o"),	
		provide_explanation=True
	)

tool_call_eval['score'] = tool_call_eval.apply(lambda x: 1 if x['label']=='correct' else 0, axis=1)
tool_call_eval.head()

# 将eval的数据回传到phoenix
px.Client(endpoint="http://localhost:6006").log_evaluations(
	SpanEvaluations(eval_name="Tool Calling Eval", dataframe=tool_call_eval),
)


### 判断生成的code代码是否能用

query = SpanQuery().where(
	"name =='generate_visualization'" # phoenix的UI界面中的name
).select(
	generated_code="output.value"
)

code_gen_df = px.Client(endpoint="http://localhost:6006").query_spans(query,
	project_name=PROJECT_NAME,
	timeout=None)
code_gen_df.head()

def code_is_runnable(output: str) -> bool:
	"""Check if the code is runnable"""
	output = output.strip()
	output = output.replace("```python", "").replace("```", "")
	
	try:
		exec(output) # 执行成功
		return True
	except Exception as e:
		return False

code_gen_df["label"] = code_gen_df["generated_code"].apply(code_is_runnable).map({True: "runnable", False: "not_runnable"})
code_gen_df["score"] = code_gen_df["label"].map({"runnable": 1, "not_runnable": 0})

px.Client(endpoint="http://localhost:6006").log_evaluations(
	SpanEvaluations(eval_name="Runnable Code Eval", dataframe=code_gen_df),
)



### LLM判断agent answer能否回答用户的question

CLARITY_LLM_JUDGE_PROMPT = """
In this task, you will be presented with a query and an answer. Your objective is to evaluate the clarity of the answer in addressing the query. A clear response is one that is precise, coherent, and directly addresses the query without introducing unnecessary complexity or ambiguity. An unclear response is one that is vague, disorganized, or difficult to understand, even if it may be factually correct.

Your response should be a single word: either "clear" or "unclear," and it should not include any other text or characters. "clear" indicates that the answer is well-structured, easy to understand, and appropriately addresses the query. "unclear" indicates that some part of the response could be better
structured or worded.
Please carefully consider the query and answer before determining your response.

After analyzing the query and the answer, you must write a detailed explanation of your reasoning to justify why you chose either "clear" or "unclear." Avoid stating the final label at the beginning of your explanation. Your reasoning should include specific points about how the answer does or does not meet the
criteria for clarity.

[BEGIN DATA]
Query: {query}
Answer: {response}
[END DATA]
Please analyze the data carefully and provide an explanation followed by your response.

EXPLANATION: Provide your reasoning step by step, evaluating the clarity of the answer based on the query.
LABEL: "clear" or "unclear"
"""

query = SpanQuery().where(
	"span_kind=='AGENT'"
).select(
	response="output.value",
	query="input.value"
)

# The Phoenix Client can take this query and return the dataframe.
clarity_df = px.Client(endpoint="http://localhost:6006").query_spans(query,
	project_name=PROJECT_NAME,
	timeout=None)
clarity_df.head()

with suppress_tracing(): # 这个上下文的内容不会放入phoenix
	clarity_eval = llm_classify(
		dataframe = clarity_df,
		template = CLARITY_LLM_JUDGE_PROMPT,
		rails = ['clear', 'unclear'],
		model=OpenAIModel(model="gpt-4o"),
		provide_explanation=True
	)

clarity_eval['score'] = clarity_eval.apply(lambda x: 1 if x['label']=='clear' else 0, axis=1)
clarity_eval.head()

px.Client(endpoint="http://localhost:6006").log_evaluations(
	SpanEvaluations(eval_name="Response Clarity", dataframe=clarity_eval),
)


### LLM判断生成的sql和用户question能否匹配上

SQL_EVAL_GEN_PROMPT = """
SQL Evaluation Prompt:
-----------------------
You are tasked with determining if the SQL generated appropiately answers a given instruction taking into account its generated query and response.

Data:
-----
- [Instruction]: {question}
This section contains the specific task or problem that the sql query is intended to solve.

- [Reference Query]: {query_gen}
This is the sql query submitted for evaluation. Analyze it in the context of the provided instruction.

Evaluation:
-----------
Your response should be a single word: either "correct" or "incorrect".
You must assume that the db exists and that columns are appropiately named.
You must take into account the response as additional information to determine the correctness.

- "correct" indicates that the sql query correctly solves the instruction.
- "incorrect" indicates that the sql query correctly does not solve the instruction correctly.

Note: Your response should contain only the word "correct" or "incorrect" with no additional text or characters.
"""

query = SpanQuery().where(
	"span_kind=='LLM'"
).select(
	query_gen="llm.output_messages",
	question="input.value",
)

# The Phoenix Client can take this query and return the dataframe.
sql_df = px.Client(endpoint="http://localhost:6006").query_spans(query,
	project_name=PROJECT_NAME,
	timeout=None)

sql_df = sql_df[sql_df["question"].str.contains("Generate an SQL query based on a prompt.", na=False)]
sql_df.head()

with suppress_tracing():
	sql_gen_eval = llm_classify(
		dataframe = sql_df,
		template = SQL_EVAL_GEN_PROMPT,
		rails = ['correct', 'incorrect'],
		model=OpenAIModel(model="gpt-4o"),
		provide_explanation=True
	)

sql_gen_eval['score'] = sql_gen_eval.apply(lambda x: 1 if x['label']=='correct' else 0, axis=1)
sql_gen_eval.head()

px.Client(endpoint="http://localhost:6006").log_evaluations(
	SpanEvaluations(eval_name="SQL Gen Eval", dataframe=sql_gen_eval),
)
```
phoenix evalute评分显示：
![[01-AI Agent 评估 6.png]]

# Agent轨迹推理评估
若能用6个步骤而非11个步骤解答用户问题，使用更少的LLM调用，更短的响应时间，岂不妙哉
![[01-AI Agent 评估 8.png]]
convergence：核心思想是选择足够多相似的问题，使agent在处理每个问题时采取相同的路径
![[01-AI Agent 评估 9.png]]


# 实验表格参考
![[01-AI Agent 评估 11.png]]

# 完整案例
```python
# 继续之前的代码
import phoenix as px
from phoenix.evals import OpenAIModel, llm_classify, TOOL_CALLING_PROMPT_TEMPLATE
from phoenix.experiments import run_experiment, evaluate_experiment
from phoenix.experiments.types import Example
from phoenix.experiments.evaluators import create_evaluator
from phoenix.otel import register
import pandas as pd

from datetime import datetime
import json
import os

import nest_asyncio
nest_asyncio.apply()

# 重构agent输出,方便评估
def process_messages(messages):
	tool_calls = []
	tool_responses = []
	final_output = None
	
	for i, message in enumerate(messages):
	# Extract tool calls
		if 'tool_calls' in message and message['tool_calls']:
			for tool_call in message['tool_calls']:
				tool_name = tool_call['function']['name']
				tool_input = tool_call['function']['arguments']	
				tool_calls.append(tool_name)
				
				# Prepare tool response structure with tool name and input
				tool_responses.append({				
					"tool_name": tool_name,		
					"tool_input": tool_input,			
					"tool_response": None	
				})
	
	# Extract tool responses
	if message['role'] == 'tool' and 'tool_call_id' in message:
		for tool_response in tool_responses:
			if message['tool_call_id'] in message.values():
				tool_response["tool_response"] = message['content']
	
	# Extract final output
	if message['role'] == 'assistant' and not message.get('tool_calls') and not message.get('function_call'):
		final_output = message['content']
	
	result = {
		"tool_calls": tool_calls,
		"tool_responses": tool_responses,
		"final_output": final_output,
		"unchanged_messages": messages,
		"path_length": len(messages)
	}
	return result
	
eval_model=OpenAIModel(model="gpt-4o")
px_client = px.Client()

overall_experiment_questions = [
	{'question': 'What was the most popular product SKU?',
	'sql_result': ' SKU_Coded Total_Qty_Sold 0 6200700 52262.0',
	'sql_generated': '```sql\nSELECT SKU_Coded, SUM(Qty_Sold) AS Total_Qty_Sold\nFROM sales\nGROUP BY SKU_Coded\nORDER BY Total_Qty_Sold DESC\nLIMIT 1;\n```'},
	{'question': 'What was the total revenue across all stores?',
	'sql_result': ' Total_Revenue 0 1.327264e+07',
	'sql_generated': '```sql\nSELECT SUM(Total_Sale_Value) AS Total_Revenue\nFROM sales;\n```'},
	{'question': 'Which store had the highest sales volume?',
	'sql_result': ' Store_Number Total_Sales_Volume 0 2970 59322.0',
	'sql_generated': '```sql\nSELECT Store_Number, SUM(Total_Sale_Value) AS Total_Sales_Volume\nFROM sales\nGROUP BY Store_Number\nORDER BY Total_Sales_Volume DESC\nLIMIT 1;\n```'},
	{'question': 'Create a bar chart showing total sales by store',
	'sql_result': ' Store_Number Total_Sales 0 880 420302.088397 1 1650 580443.007953 2 4180 272208.118542 3 550 229727.498752 4 1100 497509.528013 5 3300 619660.167018 6 3190 335035.018792 7 2970 836341.327191 8 3740 359729.808228 9 2530 324046.518720 10 4400 95745.620250 11 1210 508393.767785 12 330 370503.687331 13 2750 453664.808068 14 1980 242290.828499 15 1760 350747.617798 16 3410 410567.848126 17 990 378433.018639 18 4730 239711.708869 19 4070 322307.968330 20 3080 495458.238811 21 2090 309996.247965 22 1320 592832.067579 23 2640 308990.318559 24 1540 427777.427815 25 4840 389056.668316 26 2860 132320.519487 27 2420 406715.767402 28 770 292968.918642 29 3520 145701.079372 30 660 343594.978075 31 3630 405034.547846 32 2310 412579.388504 33 2200 361173.288199 34 1870 401070.997685',
	'sql_generated': '```sql\nSELECT Store_Number, SUM(Total_Sale_Value) AS Total_Sales\nFROM sales\nGROUP BY Store_Number;\n```'},
	{'question': 'What percentage of items were sold on promotion?',
	'sql_result': ' Promotion_Percentage 0 0.625596',
	'sql_generated': "```sql\nSELECT \n (SUM(CASE WHEN On_Promo = 'Yes' THEN 1 ELSE 0 END) * 100.0) / COUNT(*) AS Promotion_Percentage\nFROM \n sales;\n```"},
	{'question': 'What was the average transaction value?',
	'sql_result': ' Average_Transaction_Value 0 19.018132',
	'sql_generated': '```sql\nSELECT AVG(Total_Sale_Value) AS Average_Transaction_Value\nFROM sales;\n```'},
	{'question': 'Create a line chart showing sales in 2021',
	'sql_result': ' sale_month total_quantity_sold total_sales_value 0 2021-11-01 43056.0 499984.428193 1 2021-12-01 75724.0 910982.118423',
	'sql_generated': '```sql\nSELECT MONTH(Sold_Date) AS Month, SUM(Total_Sale_Value) AS Total_Sales\nFROM sales\nWHERE YEAR(Sold_Date) = 2021\nGROUP BY MONTH(Sold_Date)\nORDER BY MONTH(Sold_Date);\n```'}
]

overall_experiment_df = pd.DataFrame(overall_experiment_questions)

now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

# create a dataset consisting of input questions and expected outputs
dataset = px_client.upload_dataset(dataframe=overall_experiment_df,
	dataset_name=f"overall_experiment_inputs-{now}",
	input_keys=["question"],
	output_keys=["sql_result", "sql_generated"])

CLARITY_LLM_JUDGE_PROMPT = """
In this task, you will be presented with a query and an answer. Your objective is to evaluate the clarity of the answer in addressing the query. A clear response is one that is precise, coherent, and directly addresses the query without introducing unnecessary complexity or ambiguity. An unclear response is one that is vague, disorganized, or difficult to understand, even if it may be factually correct.

Your response should be a single word: either "clear" or "unclear," and it should not include any other text or characters. "clear" indicates that the answer is well-structured, easy to understand, and appropriately addresses the query. "unclear" indicates that the answer is ambiguous, poorly organized, or
not effectively communicated. Please carefully consider the query and answer before determining your response.

After analyzing the query and the answer, you must write a detailed explanation of your reasoning to justify why you chose either "clear" or "unclear." Avoid stating the final label at the beginning of your explanation. Your reasoning should include specific points about how the answer does or does not meet the
criteria for clarity.

[BEGIN DATA]
Query: {query}
Answer: {response}
[END DATA]
Please analyze the data carefully and provide an explanation followed by your response.

EXPLANATION: Provide your reasoning step by step, evaluating the clarity of the answer based on the query.
LABEL: "clear" or "unclear"
"""

ENTITY_CORRECTNESS_LLM_JUDGE_PROMPT = """
In this task, you will be presented with a query and an answer. Your objective is to determine whether all the entities mentioned in the answer are correctly identified and accurately match those in the query. An entity refers to any specific person, place, organization, date, or other proper noun. Your evaluation
should focus on whether the entities in the answer are correctly named and appropriately associated with the context in the query.

Your response should be a single word: either "correct" or "incorrect," and it should not include any other text or characters. "correct" indicates that all entities mentioned in the answer match those in the query and are properly identified. "incorrect" indicates that the answer contains errors or mismatches in the entities referenced compared to the query.

After analyzing the query and the answer, you must write a detailed explanation of your reasoning to justify why you chose either "correct" or "incorrect." Avoid stating the final label at the beginning of your explanation. Your reasoning should include specific points about how the entities in the answer do or do not match the entities in the query.

[BEGIN DATA]
Query: {query}
Answer: {response}
[END DATA]
Please analyze the data carefully and provide an explanation followed by your response.

EXPLANATION: Provide your reasoning step by step, evaluating whether the entities in the answer are correct and consistent with the query.
LABEL: "correct" or "incorrect"
"""

### 定义各个评估函数

# evaluate router
def function_calling_eval(input: str, output: str) -> float:
	if output is None:
		return 0

	function_calls = output.get("tool_calls")
	if function_calls:
	
		eval_df = pd.DataFrame({
			"question": [input.get("question")] * len(function_calls),
			"tool_call": function_calls
		})
		
		tool_call_eval = llm_classify(
			data = eval_df,
			template = TOOL_CALLING_PROMPT_TEMPLATE.template[0].template.replace("{tool_definitions}",
			json.dumps(tools).replace("{", '"').replace("}", '"')),
			rails = ['correct', 'incorrect'],
			model=eval_model,
			provide_explanation=True	
		)
		
		tool_call_eval['score'] = tool_call_eval.apply(lambda x: 1 if x['label']=='correct' else 0, axis=1)
		return tool_call_eval['score'].mean()
	
	else:
		return 0

# evaluate tool1: database lookup
def evaluate_sql_result(output, expected) -> bool:
	if output is None:
		return False
	
	sql_result = output.get("tool_responses")
	if not sql_result:
		return True
	
	# Find first lookup_sales_data response
	sql_result = next((r for r in sql_result if r.get("tool_name") == "lookup_sales_data"), None)
	if not sql_result:
		return True
	
	# Get the first response
	sql_result = sql_result.get("tool_response", "")
	
	# Extract just the numbers from both strings
	result_nums = ''.join(filter(str.isdigit, sql_result))
	expected_nums = ''.join(filter(str.isdigit, expected.get("sql_result")))
	return result_nums == expected_nums

# evaluator for tool 2: data analysis
def evaluate_clarity(output: str, input: str) -> bool:
	if output is None:
		return False
	
	df = pd.DataFrame({"query": [input.get("question")],
		"response": [output.get("final_output")]})
		response = llm_classify(
			data=df,
			template=CLARITY_LLM_JUDGE_PROMPT,
			rails=["clear", "unclear"],
			model=eval_model,
			provide_explanation=True
		)
	
	return response['label'] == 'clear'

# evaluator for tool 2: data analysis
def evaluate_entity_correctness(output: str, input: str) -> bool:
	if output is None:
		return False
	
	df = pd.DataFrame({"query": [input.get("question")],
	"response": [output.get("final_output")]})
	
	response = llm_classify(
		data=df,
		template=ENTITY_CORRECTNESS_LLM_JUDGE_PROMPT,
		rails=["correct", "incorrect"],
		model=eval_model,
		provide_explanation=True
	)
	return response['label'] == 'correct'

# evaluator for tool 3: data visualization
def code_is_runnable(output: str) -> bool:
	"""Check if the code is runnable"""
	if output is None:
		return False

	generated_code = output.get("tool_responses")	
	if not generated_code:
		return True
	
	# Find first lookup_sales_data response
	generated_code = next((r for r in generated_code if r.get("tool_name") == "generate_visualization"), None)
	if not generated_code:
		return True
	
	# Get the first response
	generated_code = generated_code.get("tool_response", "")
	generated_code = generated_code.strip()
	generated_code = generated_code.replace("```python", "").replace("```", "")
	try:
		exec(generated_code)
		return True
	except Exception as e:
		return False

def run_agent_task(example: Example) -> str:
	print("Starting agent with messages:", example.input.get("question"))
	messages = [{"role": "user", "content": example.input.get("question")}]
	ret = run_agent(messages)
	return process_messages(ret)

experiment = run_experiment(dataset,
					run_agent_task,
					evaluators=[function_calling_eval,
								evaluate_sql_result,
								evaluate_clarity,
								evaluate_entity_correctness,
								code_is_runnable],
					experiment_name="Overall Experiment",
					experiment_description="Evaluating the overall experiment")
```