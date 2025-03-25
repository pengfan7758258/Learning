[地址](https://python.langchain.com/docs/tutorials/sql_qa/)
[事例数据下载地址](https://database.guide/2-sample-databases-sqlite/)

<mark style="background: #ADCCFFA6;">执行流程图：</mark>
1. 模型将用户输入转换为 SQL 查询
2. 执行查询
3. 模型使用查询结果响应用户输入
![[10-Build a QA system over SQL data.png]]

<mark style="background: #FFB86CA6;">sqlite测试数据库连接：</mark>
```python
# sqlite3样本数据连接测试
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri("sqlite:///Chinook.db")

print(db.dialect) # 数据库类型：sqlite
print(db.get_usable_table_names()) # 所有表名：['Album', 'Artist', 'Customer', 'Employee', 'Genre', 'Invoice', 'InvoiceLine', 'MediaType', 'Playlist', 'PlaylistTrack', 'Track']

db.run("SELECT * FROM Artist LIMIT 10;") # sql语句运行
db.get_table_info() # 所有表结构信息
```
<mark style="background: #FFB86CA6;">chains：</mark>
```python
# State声明
from typing import Optional
from pydantic import BaseModel, Field

class State(BaseModel):
    question: Optional[str] # 输入问题
    query: Optional[str] # 生成的查询
    result: Optional[str] # 查询结果
    answer: Optional[str] # 生成的答案


# 转换question 到 SQL语句
# 为了获取可靠的sql语句，使用langchain的structed output限制输出格式

# 初始化模型
from langchain.chat_models import init_chat_model
llm = init_chat_model("gpt-4o-mini", model_provider="openai")
  
# prompt获取
from langchain import hub
query_prompt_template = hub.pull("langchain-ai/sql-query-system-prompt")

assert len(query_prompt_template.messages) == 1
query_prompt_template.messages[0].pretty_print()
"""
=================[1m System Message [0m===============
Given an input question, create a syntactically correct {dialect} query to run to help find the answer. Unless the user specifies in his question a specific number of examples they wish to obtain, always limit your query to at most {top_k} results. You can order the results by a relevant column to return the most interesting examples in the database.

Never query for all the columns from a specific table, only ask for a the few relevant columns given the question.

Pay attention to use only the column names that you can see in the schema description. Be careful to not query for columns that do not exist. Also, pay attention to which column is in which table.

Only use the following tables:
{table_info}

Question: {input}

中文翻译：
给定一个输入问题，创建一个语法正确的{dialect}查询来运行，以帮助找到答案。除非用户在问题中指定了他们希望获取的特定数量的示例，否则请始终将查询限制在最多{top_k}个结果。您可以按相关列对结果进行排序，以返回数据库中最有趣的示例。
永远不要查询特定表中的所有列，只询问给定问题的少数相关列。
请注意仅使用在模式描述中可见的列名。注意不要查询不存在的列。此外，请注意哪个列在哪个表中。
仅使用下表：
{table_info}
问题：{输入}
"""

# write_sql函数声明
from pydantic import BaseModel, Field
class QueryOutput(BaseModel):
    """SQL语句输出格式限制"""
    query: str = Field(..., description="语法有效的SQL语句.")

  
def write_query(state: State):
    """从获得信息中生成sql语句"""
    prompt = query_prompt_template.invoke(
        {
            "dialect": db.dialect, # 数据库类型
            "top_k": 10,
            "table_info": db.get_table_info(), # 表结构信息
            "input": state.question, # 用户question
        }
    )

    structured_llm = llm.with_structured_output(QueryOutput) # 输出格式限制（结构化输出）
    result = structured_llm.invoke(prompt)
    return {"query": result.query}

# 自动执行操控数据库（后续介绍加入人工确认步骤）
from langchain_community.tools.sql_database.tool import QuerySQLDatabaseTool

def execute_query(state: State):
    """执行SQL语句."""
    execute_query_tool = QuerySQLDatabaseTool(db=db)
    return {"result": execute_query_tool.invoke(state.query)}

# 答案生成节点函数
def generate_answer(state: State):
    """使用数据库检索到的信息辅助生成答案"""
    prompt = (
        "Given the following user question, corresponding SQL query, "
        "and SQL result, answer the user question.\n\n"
        f'Question: {state.question}\n'
        f'SQL Query: {state.query}\n'
        f'SQL Result: {state.result}'
    )

    response = llm.invoke(prompt)
    return {"answer": response.content}

# langgraph生成work flow
from langgraph.graph import StateGraph

graph_builder = StateGraph(State).add_sequence(
    [write_query, execute_query, generate_answer]
)

graph_builder.set_entry_point("write_query")
graph = graph_builder.compile()

# 可视化
from IPython.display import Image, display
display(Image(graph.get_graph().draw_mermaid_png()))
```
![[10-Build a QA system over SQL data 1.png]]
```python
# 测试
input_dict = {"question": "How many employees are there?"}

# 动态创建 State 对象
initial_state = State(
    question=input_dict.get("question", ""),
    query=input_dict.get("query", ""),
    result=input_dict.get("result", ""),
    answer=input_dict.get("answer", "")
)

  
for step in graph.stream(
    initial_state, stream_mode="updates"
):
    print(step)
"""
{'write_query': {'query': 'SELECT COUNT(*) AS EmployeeCount FROM Employee;'}}
{'execute_query': {'result': '[(8,)]'}}
{'generate_answer': {'answer': 'There are 8 employees.'}}
"""
```
<mark style="background: #FFB86CA6;">人机协同：</mark>
```python
## 可以在敏感步骤终端app，以便人工审核
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory, interrupt_before=["execute_query"])

# 持久化, 需要指定thread ID
config = {"configurable": {"thread_id": "2"}}
display(Image(graph.get_graph().draw_mermaid_png()))
```
![[10-Build a QA system over SQL data 2.png]]
```python
# 添加一个简单的 yes/no 人工干预
for step in graph.stream(
    initial_state,
    config,
    stream_mode="updates",
):
    print(step)

# 运行到断点的函数之前会停止work flow

# 加入判断逻辑
try:
    user_approval = input("Do you want to go to execute query? (yes/no): ")
except Exception:
    user_approval = "no"

if user_approval.lower() == "yes":
    # 继续graph execution
    for step in graph.stream(None, config, stream_mode="updates"):
        print(step)
else:
    print("Operation cancelled by user.")

"""
{'write_query': {'query': 'SELECT COUNT(*) AS EmployeeCount FROM Employee;'}}
{'__interrupt__': ()} # 这里输出了断点的记录
{'execute_query': {'result': '[(8,)]'}}
{'generate_answer': {'answer': 'There are 8 employees.'}}
"""
```
<mark style="background: #FFB86CA6;">agents：</mark>
```python
# SQLDatabaseToolkit
# 创建和执行查询 -  检查查询语法 - 检索表描述
from langchain_community.agent_toolkits import SQLDatabaseToolkit

toolkit = SQLDatabaseToolkit(db=db, llm=llm)
tools = toolkit.get_tools() # 四个数据库相关tool
tools
"""
[QuerySQLDatabaseTool(description="Input to t...>),
 InfoSQLDatabaseTool(description='Input to this ...),
 ListSQLDatabaseTool(db=<langchain_community.utilities...),
 QuerySQLCheckerTool(description='Use this tool to ...)]
"""

# System Prompt
from langchain import hub

prompt_template = hub.pull("langchain-ai/sql-agent-system-prompt")
assert len(prompt_template.messages) == 1
prompt_template.messages[0].pretty_print()
"""
================[1m System Message [0m============

You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run, then look at the results of the query and return the answer.
Unless the user specifies a specific number of examples they wish to obtain, always limit your query to at most {top_k} results.
You can order the results by a relevant column to return the most interesting examples in the database.
Never query for all the columns from a specific table, only ask for the relevant columns given the question.
You have access to tools for interacting with the database.
Only use the below tools. Only use the information returned by the below tools to construct your final answer.
You MUST double check your query before executing it. If you get an error while executing a query, rewrite the query and try again.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the database.

To start you should ALWAYS look at the tables in the database to see what you can query.
Do NOT skip this step.
Then you should query the schema of the most relevant tables.

中文翻译：
您是一个设计用于与SQL数据库交互的代理。
给定一个输入问题，创建一个语法正确的{dialect}查询来运行，然后查看查询结果并返回答案。
除非用户指定了他们希望获取的特定数量的示例，否则始终将查询限制在最多{top_k}个结果。
您可以按相关列对结果进行排序，以返回数据库中最有趣的示例。
永远不要查询特定表中的所有列，只询问给定问题的相关列。
您可以访问与数据库交互的工具。
仅使用以下工具。仅使用以下工具返回的信息来构建您的最终答案。
在执行查询之前，您必须仔细检查查询。如果在执行查询时出错，请重写查询并重试。
不要对数据库执行任何DML语句（INSERT、UPDATE、DELETE、DROP等）。
首先，您应该始终查看数据库中的表，看看可以查询什么。
请勿跳过此步骤。
然后，您应该查询最相关表的模式。
"""

# 填充system prompt
system_message = prompt_template.format(dialect="SQLite", top_k=5)

# 初始化agent
from langchain_core.messages import HumanMessage
from langgraph.prebuilt import create_react_agent

agent_executor = create_react_agent(llm, tools, prompt=system_message)

display(Image(agent_executor.get_graph().draw_mermaid_png()))
```
![[10-Build a QA system over SQL data 3.png]]
```python
question = "Which country's customers spent the most?"

for step in agent_executor.stream(
    {"messages": [{"role": "user", "content": question}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()

"""
================================ Human Message ================================= Which country's customers spent the most? 
================================== Ai Message ================================== Tool Calls: 
sql_db_list_tables (call_ASmcvjmcHZpYkWMPOsPuD9fT) 
Call ID: call_ASmcvjmcHZpYkWMPOsPuD9fT 
Args: 
================================= Tool Message ================================= Name: sql_db_list_tables Album, Artist, Customer, Employee, Genre, Invoice, InvoiceLine, MediaType, Playlist, PlaylistTrack, Track ================================== Ai Message ================================== Tool Calls: 
sql_db_schema (call_VbQWaxteRMygKbVaWYgB0dTK)
Call ID: call_VbQWaxteRMygKbVaWYgB0dTK 
Args: 
table_names: Customer sql_db_schema (call_hmtUNITDtpeuoZdUDcFNjMRq) 
Call ID: call_hmtUNITDtpeuoZdUDcFNjMRq 
Args: 
table_names: Invoice sql_db_schema (call_0TsMGAQtmXDTU5q53BkVKlcZ) 
Call ID: call_0TsMGAQtmXDTU5q53BkVKlcZ 
Args: 
table_names: InvoiceLine
================================= Tool Message ================================= Name: sql_db_schema CREATE TABLE "InvoiceLine" ( "InvoiceLineId" INTEGER NOT NULL, "InvoiceId" INTEGER NOT NULL, "TrackId" INTEGER NOT NULL, "UnitPrice" NUMERIC(10, 2) NOT NULL, "Quantity" INTEGER NOT NULL, PRIMARY KEY ("InvoiceLineId"), FOREIGN KEY("TrackId") REFERENCES "Track" ("TrackId"), FOREIGN KEY("InvoiceId") REFERENCES "Invoice" ("InvoiceId") ) /* 3 rows from InvoiceLine table: InvoiceLineId InvoiceId TrackId UnitPrice Quantity 1 1 2 0.99 1 2 1 4 0.99 1 3 2 6 0.99 1 */ 
================================== Ai Message ================================== Tool Calls: 
sql_db_query_checker (call_7BrfMTqCyOrB0Ub7qZesQfy2) 
Call ID: call_7BrfMTqCyOrB0Ub7qZesQfy2 
Args: 
query: 
SELECT c.Country, SUM(i.Total) AS TotalSpent FROM Customer c JOIN Invoice i ON c.CustomerId = i.CustomerId GROUP BY c.Country ORDER BY TotalSpent DESC LIMIT 5; ================================= Tool Message ================================= Name: sql_db_query_checker 
```sql SELECT c.Country, SUM(i.Total) AS TotalSpent FROM Customer c JOIN Invoice i ON c.CustomerId = i.CustomerId GROUP BY c.Country ORDER BY TotalSpent DESC LIMIT 5; ``` 
================================== Ai Message ================================== Tool Calls: sql_db_query (call_gfKOxcVUH2NZRyZZ3VIBUiPl) 
Call ID: call_gfKOxcVUH2NZRyZZ3VIBUiPl 
Args: 
query: SELECT c.Country, SUM(i.Total) AS TotalSpent FROM Customer c JOIN Invoice i ON c.CustomerId = i.CustomerId GROUP BY c.Country ORDER BY TotalSpent DESC LIMIT 5; 
================================= Tool Message ================================= Name: sql_db_query [('USA', 523.06), ('Canada', 303.96), ('France', 195.1), ('Brazil', 190.1), ('Germany', 156.48)] 
================================== Ai Message ================================== The countries whose customers spent the most are as follows: 
1. **USA** - Total spent: $523.06 
2. **Canada** - Total spent: $303.96 
3. **France** - Total spent: $195.10 
4. **Brazil** - Total spent: $190.10 
5. **Germany** - Total spent: $156.48
"""

"""
从输出我们能得到的信息：
1.四个关于数据库工具的调用
列出可用的表->检查与question相关的三个表结构->join联合查询多个表->检查sql语句->执行查询
"""
```
<mark style="background: #FFB86CA6;">处理高基数列: vector store充当tool</mark>
- `专有名词`建造vector store - 里面有`专有名词`的各种描述，来作为外部知识
```python
import ast
import re

def query_as_list(db, query):
    res = db.run(query)
    res = [el for sub in ast.literal_eval(res) for el in sub if el]
    res = [re.sub(r"\b\d+\b", "", string).strip() for string in res] # \b\d+\b匹配完全数字的单词 譬如：123 how are you 983f -> how are you 983f
    return list(set(res))

  
artists = query_as_list(db, "SELECT Name FROM Artist")[:10]
albums = query_as_list(db, "SELECT Title FROM Album")[:10]
print(len(artists), len(albums))
print(artists + albums)

from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
# embedding模型、向量数据库
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
vector_store = FAISS.from_texts(artists + albums, embedding=embeddings)

# 构建一个检索工具，可以在数据库中搜索相关的专有名词
from langchain.agents.agent_toolkits import create_retriever_tool

retriever = vector_store.as_retriever(search_kwargs={"k": 5})
description = (
    "Use to look up values to filter on. Input is an approximate spelling "
    "of the proper noun, output is valid proper nouns. Use the noun most "
    "similar to the search."
)
"""
中文翻译：
用于查找要过滤的值。输入是专有名词的近似拼写，输出是有效的专有名词。使用与搜索最相似的名词
"""

retriever_tool = create_retriever_tool(
    retriever,
    name="search_proper_nouns",
    description=description,
)

# 在system message中添加new tool的prompt
suffix = (
    "If you need to filter on a proper noun like a Name, you must ALWAYS first look up "
    "the filter value using the 'search_proper_nouns' tool! Do not try to "
    "guess at the proper name - use this function to find similar ones."
)

"""
中文翻译：
如果你需要过滤一个专有名词，比如名字，你必须首先查找使用'search_proper_nouns'
工具筛选值！不要尝试猜测正确的名称-使用此函数查找类似的名称。
"""

system = f"{system_message}\n\n{suffix}"
print(system)
tools.append(retriever_tool)

agent = create_react_agent(llm, tools, prompt=system)

# 测试
question = "How many albums does minas have?"

for step in agent.stream(
    {"messages": [{"role": "user", "content": question}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()

"""
因为省openai key的缘故，向量库只有20个专有名词，没效果
"""
```

