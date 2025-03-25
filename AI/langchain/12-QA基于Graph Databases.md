- [地址](https://python.langchain.com/docs/tutorials/graph/)
- 需要安装neo4j

![[12-QA基于Graph Databases.png]]

<mark style="background: #FFB86CA6;">neo4j配置：</mark>
```python
# neo4j数据库配置
import os
os.environ["NEO4J_URI"] = "bolt://192.168.1.55:7687"
os.environ["NEO4J_USERNAME"] = "neo4j"
os.environ["NEO4J_PASSWORD"] = "11111111"

# 创建与 Neo4j 数据库的连接，并使用有关电影及其演员的示例数据填充该数据库
from langchain_neo4j import Neo4jGraph
graph = Neo4jGraph()

# 导入 movie information 事例数据
movies_query = """
LOAD CSV WITH HEADERS FROM
'https://raw.githubusercontent.com/tomasonjo/blog-datasets/main/movies/movies_small.csv'
AS row
MERGE (m:Movie {id:row.movieId})
SET m.released = date(row.released),
    m.title = row.title,
    m.imdbRating = toFloat(row.imdbRating)
FOREACH (director in split(row.director, '|') |
    MERGE (p:Person {name:trim(director)})
    MERGE (p)-[:DIRECTED]->(m))
FOREACH (actor in split(row.actors, '|') |
    MERGE (p:Person {name:trim(actor)})
    MERGE (p)-[:ACTED_IN]->(m))
FOREACH (genre in split(row.genres, '|') |
    MERGE (g:Genre {name:trim(genre)})
    MERGE (m)-[:IN_GENRE]->(g))
"""
# MERGE：合并关键字，在图谱中没有则会添加该数据
# SET：设置节点属性关键字
# FOREACH：循环关键字
# date()、toFloat()、split()、trim()一些neo4j内置的函数

graph.query(movies_query)
# output: []

# 获取当前图结构 - 后续如果更改了图结构可以使用此命令刷新图结构
graph.refresh_schema()
print(graph.schema)
"""
-------- output --------
Node properties:
Movie {imdbRating: FLOAT, id: STRING, released: DATE, title: STRING}
Person {name: STRING}
Genre {name: STRING}
Relationship properties:

The relationships:
(:Movie)-[:IN_GENRE]->(:Genre)
(:Person)-[:DIRECTED]->(:Movie)
(:Person)-[:ACTED_IN]->(:Movie)
"""

# enhanced_schema为True获得增强版的图结构
enhanced_graph = Neo4jGraph(enhanced_schema=True)
print(enhanced_graph.schema)
"""
-------- output --------
Node properties:
- **Movie**
  - `imdbRating`: FLOAT Min: 2.4, Max: 9.3
  - `id`: STRING Example: "1"
  - `released`: DATE Min: 1964-12-16, Max: 1996-09-15
  - `title`: STRING Example: "Toy Story"
- **Person**
  - `name`: STRING Example: "John Lasseter"
- **Genre**
  - `name`: STRING Example: "Adventure"
Relationship properties:

The relationships:
(:Movie)-[:IN_GENRE]->(:Genre)
(:Person)-[:DIRECTED]->(:Movie)
(:Person)-[:ACTED_IN]->(:Movie)
"""
```

<mark style="background: #FFB86CA6;">GraphQACypherChain:</mark>
![[12-QA基于Graph Databases 1.png]]
```python
from langchain_neo4j import GraphCypherQAChain

from langchain.chat_models import init_chat_model

# 加载模型，temperature=0确保每次都能复现结果
llm = init_chat_model(model="gpt-4o", model_provider="openai", temperature=0)

# 实例化GraphCypherQAChain
chain = GraphCypherQAChain.from_llm(
    graph=enhanced_graph, # 图数据库对象
    llm=llm,
    verbose=True, # 用于控制是否输出详细的调试信息
    allow_dangerous_requests=True # 用于控制是否允许执行可能具有潜在危险的请求(如删除数据、修改结构等)
)

response = chain.invoke({"query": "What was the cast of the Casino?"})
response
"""
-------- output --------
Entering new GraphCypherQAChain chain...
Generated Cypher:
mcypher
MATCH (p:Person)-[:ACTED_IN]->(m:Movie {title: "Casino"})
RETURN p.name

Full Context:
[{'p.name': 'James Woods'}, {'p.name': 'Joe Pesci'}, {'p.name': 'Robert De Niro'}, {'p.name': 'Sharon Stone'}]

Finished chain.

{'query': 'What was the cast of the Casino?',
 'result': 'James Woods, Joe Pesci, Robert De Niro, and Sharon Stone were the cast of Casino.'}
"""
```

<mark style="background: #FFB86CA6;">Advanced implementation with LangGraph:</mark>
![[12-QA基于Graph Databases 2.png]]
```python
# 定义state
from pydantic import BaseModel
class OverallState(BaseModel):
    question: str # 问题
    next_action: str # 下一个动作
    cypher_statement: str # cypher state
    cypher_errors: list[str] # cypher错误
    database_records: str # 数据库记录
    steps: list[str] # 步骤
    answer: str # 答案
```
1. guardrails：
	- 一个意图识别函数，判断是否和movie业务相关
```python
# 1.设置`护栏`，看问题是否和电影、演员相关
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

  
"""
guardrails_system中文翻译：
作为一名智能助理，你的主要目标是决定一个给定的问题是否与movie有关。
如果问题与movie有关，则输出"movie"。否则，输出"end"。
为了做出这个决定，评估问题的内容，并确定它是否涉及任何movie, actor, director, film行业，
或相关主题。仅提供指定的输出："movie" or "end"。
"""

guardrails_system = """
As an intelligent assistant, your primary objective is to decide whether a given question is related to movies or not.
If the question is related to movies, output "movie". Otherwise, output "end".
To make this decision, assess the content of the question and determine if it refers to any movie, actor, director, film industry,
or related topics. Provide only the specified output: "movie" or "end".
"""

  
guardrails_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            guardrails_system,
        ),
        (
            "human",
            ("{question}"),
        ),
    ]
)

class GuardrailsOutput(BaseModel):
    decision: str = Field(
        description="Decision on whether the question is related to movies",
        enum=["movie", "end"]
    )

guardrails_chain = guardrails_prompt | llm.with_structured_output(GuardrailsOutput)
  
def guardrails(state: OverallState):
    """
    决定你的question和电影有关或者无关.
    """
    guardrails_output = guardrails_chain.invoke({"question": state.question})
    database_records = ""
    if guardrails_output.decision == "end":
        database_records = "This questions is not about movies or their cast. Therefore I cannot answer this question." # 最终输出要用
    return {
        "next_action": guardrails_output.decision,
        "database_records": database_records,
        "steps": ["guardrail"],
    }

# guardrails条件函数，用来判断流程走向
def guardrails_condition(
    state: OverallState,
):
    if state.next_action == "end":
        return "generate_final_answer"
    elif state.next_action == "movie":
        return "generate_cypher"
```
2. generate_cypher：
	- 生成cypher
	- SemanticSimilarityExampleSelector：语义相似example样例选择器指导增强LLM的生成能力
```python
# SemanticSimilarityExampleSelector
from langchain_core.example_selectors import SemanticSimilarityExampleSelector
from langchain_neo4j import Neo4jVector
from langchain_openai import OpenAIEmbeddings

examples = [
    {
        "question": "How many artists are there?",
        "query": "MATCH (a:Person)-[:ACTED_IN]->(:Movie) RETURN count(DISTINCT a)",
    },
    {
        "question": "Which actors played in the movie Casino?",
        "query": "MATCH (m:Movie {title: 'Casino'})<-[:ACTED_IN]-(a) RETURN a.name",
    },
    {
        "question": "How many movies has Tom Hanks acted in?",
        "query": "MATCH (a:Person {name: 'Tom Hanks'})-[:ACTED_IN]->(m:Movie) RETURN count(m)",
    },
    {
        "question": "List all the genres of the movie Schindler's List",
        "query": "MATCH (m:Movie {title: 'Schindler's List'})-[:IN_GENRE]->(g:Genre) RETURN g.name",
    },
    {
        "question": "Which actors have worked in movies from both the comedy and action genres?",
        "query": "MATCH (a:Person)-[:ACTED_IN]->(:Movie)-[:IN_GENRE]->(g1:Genre), (a)-[:ACTED_IN]->(:Movie)-[:IN_GENRE]->(g2:Genre) WHERE g1.name = 'Comedy' AND g2.name = 'Action' RETURN DISTINCT a.name",
    },
    {
        "question": "Which directors have made movies with at least three different actors named 'John'?",
        "query": "MATCH (d:Person)-[:DIRECTED]->(m:Movie)<-[:ACTED_IN]-(a:Person) WHERE a.name STARTS WITH 'John' WITH d, COUNT(DISTINCT a) AS JohnsCount WHERE JohnsCount >= 3 RETURN d.name",
    },
    {
        "question": "Identify movies where directors also played a role in the film.",
        "query": "MATCH (p:Person)-[:DIRECTED]->(m:Movie), (p)-[:ACTED_IN]->(m) RETURN m.title, p.name",
    },
    {
        "question": "Find the actor with the highest number of movies in the database.",
        "query": "MATCH (a:Actor)-[:ACTED_IN]->(m:Movie) RETURN a.name, COUNT(m) AS movieCount ORDER BY movieCount DESC LIMIT 1",
    },
]

example_selector = SemanticSimilarityExampleSelector.from_examples(
    examples, OpenAIEmbeddings(), Neo4jVector, k=5,
    input_keys=["question"] #  表示将使用输入问题中的 "question" 字段与示例中的 "question" 字段进行比较
)

# Cypher generation node
from langchain_core.output_parsers import StrOutputParser

text2cypher_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            (
                "Given an input question, convert it to a Cypher query. No pre-amble."
                "Do not wrap the response in any backticks or anything else. Respond with a Cypher statement only!"
            ),
        ),
        (

            "human",
            (
                """You are a Neo4j expert. Given an input question, create a syntactically correct Cypher query to run.
Do not wrap the response in any backticks or anything else. Respond with a Cypher statement only!
Here is the schema information
{schema}

Below are a number of examples of questions and their corresponding Cypher queries.
{fewshot_examples}

User input: {question}
Cypher query:"""
            ),
        ),
    ]
)

text2cypher_chain = text2cypher_prompt | llm | StrOutputParser()

def generate_cypher(state: OverallState):
    """
    生成 a cypher statement 基于提供的 schema and user input
    """
    NL = "\n"
    fewshot_examples = (NL * 2).join(
        [
            f"Question: {el['question']}{NL}Cypher:{el['query']}"
            for el in example_selector.select_examples(
                {"question": state.question} # 搜索question和examples语义相似高的案例
            )
        ]
    )

    generated_cypher = text2cypher_chain.invoke(
        {
            "question": state.question,
            "fewshot_examples": fewshot_examples,
            "schema": enhanced_graph.schema,
        }
    )

    return {"cypher_statement": generated_cypher, "steps": ["generate_cypher"]}
```
3. Query validation
	- cypher query验证
```python
# 创建一个链，用于检测 Cypher 语句中的任何错误并提取它引用的属性值
from typing import Optional

validate_cypher_system = """
You are a Cypher expert reviewing a statement written by a junior developer.
"""

validate_cypher_user = """You must check the following:
* Are there any syntax errors in the Cypher statement?
* Are there any missing or undefined variables in the Cypher statement?
* Are any node labels missing from the schema?
* Are any relationship types missing from the schema?
* Are any of the properties not included in the schema?
* Does the Cypher statement include enough information to answer the question?

Examples of good errors:
* Label (:Foo) does not exist, did you mean (:Bar)?
* Property bar does not exist for label Foo, did you mean baz?
* Relationship FOO does not exist, did you mean FOO_BAR?

Schema:
{schema}

The question is:
{question}

The Cypher statement is:
{cypher}

Make sure you don't make any mistakes!"""

validate_cypher_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            validate_cypher_system,
        ),
        (
            "human",
            (validate_cypher_user),
        ),
    ]
)

class Property(BaseModel):
    """
    Represents a 条件过滤基于一个特定node属性 in a graph in a Cypher声明.
    """
  
    node_label: str = Field(
        description="The label of the node to which this property belongs."
    )
    property_key: str = Field(description="The key of the property being filtered.")
    property_value: str = Field(
        description="The value that the property is being matched against."
    )

  
class ValidateCypherOutput(BaseModel):
    """
    Represents the 验证结果 of a Cypher query's 输出,
    including any errors and applied filters.
    """

    errors: Optional[list[str]] = Field(
        description="A list of syntax or semantical errors in the Cypher statement. Always explain the discrepancy between schema and Cypher statement"
    )
    filters: Optional[list[Property]] = Field(
        description="A list of property-based filters applied in the Cypher statement."
    )

validate_cypher_chain = validate_cypher_prompt | llm.with_structured_output(
    ValidateCypherOutput
)

# 实验性功能 CypherQueryCorrector 可以辅助Cypher语句中的关系方向
from langchain_neo4j.chains.graph_qa.cypher_utils import CypherQueryCorrector, Schema

corrector_schema = [
    Schema(el["start"], el["type"], el["end"])
    for el in enhanced_graph.structured_schema.get("relationships")
]
cypher_query_corrector = CypherQueryCorrector(corrector_schema)

from neo4j.exceptions import CypherSyntaxError

def validate_cypher(state: OverallState):
    """
    验证 the Cypher statements and 映射任何属性值 to the database.
    """
    errors = []
    mapping_errors = []

    # 检查语法errors
    try: # EXPLAIN 会返回查询的执行计划，包括数据库引擎如何遍历图、使用哪些索引、执行哪些操作。它不会实际执行查询，也不会返回查询结果
        enhanced_graph.query(f"EXPLAIN {state.cypher_statement}")
    except CypherSyntaxError as e:
        errors.append(e.message)
        
    # 实验性功能 for 正确的关系路径
    corrected_cypher = cypher_query_corrector(state.cypher_statement)
    if not corrected_cypher:
        errors.append("The generated Cypher statement doesn't fit the graph schema")
    if not corrected_cypher == state.cypher_statement:
        print("Relationship direction was corrected")
        
    # 使用LLM去发现额外隐藏的errors and get the mapping for values
    llm_output = validate_cypher_chain.invoke(
        {
            "question": state.question,
            "schema": enhanced_graph.schema,
            "cypher": state.cypher_statement,
        }
    )
    if llm_output.errors:
        errors.extend(llm_output.errors)
    if llm_output.filters:
        for filter in llm_output.filters:
            # Do mapping only for string values
            if (
                not [
                    prop
                    for prop in enhanced_graph.structured_schema["node_props"][
                        filter.node_label
                    ]
                    if prop["property"] == filter.property_key
                ][0]["type"]
                == "STRING"
            ):
                continue
            mapping = enhanced_graph.query(
                f"MATCH (n:{filter.node_label}) WHERE toLower(n.`{filter.property_key}`) = toLower($value) RETURN 'yes' LIMIT 1",
                {"value": filter.property_value},
            )
            if not mapping:
                print(
                    f"Missing value mapping for {filter.node_label} on property {filter.property_key} with value {filter.property_value}"
                )
                mapping_errors.append(
                    f"Missing value mapping for {filter.node_label} on property {filter.property_key} with value {filter.property_value}"
                )
    if mapping_errors:
        next_action = "end"
    elif errors:
        next_action = "correct_cypher"
    else:
        next_action = "execute_cypher"

    return {
        "next_action": next_action,
        "cypher_statement": corrected_cypher,
        "cypher_errors": errors,
        "steps": ["validate_cypher"],
    }

# validate_cypher条件边
def validate_cypher_condition(
    state: OverallState,
):
    if state.next_action == "end": # 生成的cypher node的属性key、value缺失，数据缺失直接跳到结尾
        return "generate_final_answer"
    elif state.next_action == "correct_cypher": # cypher node没问题，但是不符合neo4j的schema结构，需要修正
        return "correct_cypher"
    elif state.next_action == "execute_cypher": # 执行cypher
        return "execute_cypher"
```
4. correct_cypher：
	- 纠正cypher
```python
correct_cypher_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            (
                "You are a Cypher expert reviewing a statement written by a junior developer. "
                "You need to correct the Cypher statement based on the provided errors. No pre-amble."
                "Do not wrap the response in any backticks or anything else. Respond with a Cypher statement only!"
            ),
        ),
        (
            "human",
            (
                """Check for invalid syntax or semantics and return a corrected Cypher statement.
  
Schema:
{schema}

Note: Do not include any explanations or apologies in your responses.
Do not wrap the response in any backticks or anything else.
Respond with a Cypher statement only!

Do not respond to any questions that might ask anything else than for you to construct a Cypher statement.

The question is:
{question}

The Cypher statement is:
{cypher}

The errors are:
{errors}

Corrected Cypher statement: """
            ),
        ),
    ]
)

correct_cypher_chain = correct_cypher_prompt | llm | StrOutputParser()

# 纠正cypher
def correct_cypher(state: OverallState):
    """
    Correct the Cypher statement based on the provided errors.
    """
    corrected_cypher = correct_cypher_chain.invoke(
        {
            "question": state.question,
            "errors": state.cypher_errors,
            "cypher": state.cypher_statement,
            "schema": enhanced_graph.schema,
        }
    )

    return {
        "next_action": "validate_cypher",
        "cypher_statement": corrected_cypher,
        "steps": ["correct_cypher"],
    }
```
5. execute_cypher：
	- 执行cypher
```python
no_results = "I couldn't find any relevant information in the database"

def execute_cypher(state: OverallState):
    """
    Executes the given Cypher statement.
    """
    records = enhanced_graph.query(state.cypher_statement)

    return {
        "database_records": str(records) if records else no_results,
        "next_action": "end",
        "steps": ["execute_cypher"],
    }
```
6. generate_final_answer：
	- 生成最终答案
```python
# 生成答案。这涉及将初始问题与cypher语句查询结果相结合，以产生相关的回答。
generate_final_prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are a helpful assistant",
        ),
        (
            "human",
            (
                """Use the following results retrieved from a database to provide a succinct, definitive answer to the user's question.
                
Respond as if you are answering the question directly.

Results: {results}

Question: {question}"""
            ),
        ),
    ]
)

generate_final_chain = generate_final_prompt | llm | StrOutputParser()

def generate_final_answer(state: OverallState):
    """
    Decides if the question is related to movies.
    """
    final_answer = generate_final_chain.invoke(
        {"question": state.question, "results": state.database_records}
    )

    return {"answer": final_answer, "steps": ["generate_final_answer"]}
```
7. langgraph work flow
```python
from langgraph.graph import END, StateGraph

# 初始化图
langgraph = StateGraph(OverallState)

# 添加node
langgraph.add_node(guardrails)
langgraph.add_node(generate_cypher)
langgraph.add_node(validate_cypher)
langgraph.add_node(correct_cypher)
langgraph.add_node(execute_cypher)
langgraph.add_node(generate_final_answer)

# 设置入口
langgraph.set_entry_point("guardrails")

# 添加edge
langgraph.add_conditional_edges(
    "guardrails",
    guardrails_condition,
)
langgraph.add_edge("generate_cypher", "validate_cypher")
langgraph.add_conditional_edges(
    "validate_cypher",
    validate_cypher_condition,
)
langgraph.add_edge("execute_cypher", "generate_final_answer")
langgraph.add_edge("correct_cypher", "validate_cypher")

# 设置出口
langgraph.set_finish_point("generate_final_answer")

# 编译图，让langgraph变为一个runnable的graph
langgraph = langgraph.compile()
```
8. 测试
```python
# 使用一个和movie不相关的问题进行测试
initial_state = OverallState(
    question="What's the weather in Spain?",
    next_action="",
    cypher_statement="",
    cypher_errors=[],
    database_records="",
    steps=[],
    answer=""
)

langgraph.invoke(initial_state)
"""
-------- output --------
{'question': "What's the weather in Spain?",
 'next_action': 'end',
 'cypher_statement': '',
 'cypher_errors': [],
 'database_records': 'This questions is not about movies or their cast. Therefore I cannot answer this question.',
 'steps': ['generate_final_answer']}
"""


# 问一个和电影相关的问题
initial_state = OverallState(
    question="What was the cast of the Casino?",
    next_action="",
    cypher_statement="",
    cypher_errors=[],
    database_records="",
    steps=[],
    answer=""
)

langgraph.invoke(initial_state)
"""
-------- output --------
{'question': 'What was the cast of the Casino?',
 'next_action': 'end',
 'cypher_statement': "MATCH (m:Movie {title: 'Casino'})<-[:ACTED_IN]-(a:Person) RETURN a.name",
 'cypher_errors': [],
 'database_records': "[{'a.name': 'James Woods'}, {'a.name': 'Joe Pesci'}, {'a.name': 'Robert De Niro'}, {'a.name': 'Sharon Stone'}]",
 'steps': ['generate_final_answer'],
 'answer': 'The cast of "Casino" includes James Woods, Joe Pesci, Robert De Niro, and Sharon Stone.'}
"""
```