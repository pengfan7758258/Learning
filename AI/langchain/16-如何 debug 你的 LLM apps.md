- [地址](https://python.langchain.com/docs/how_to/debugging/)
- 三种方式
	- verbose
	- debug
	- langsmith（付费）

<mark style="background: #FFB86CA6;">基础配置:</mark>
```python
from langchain.chat_models import init_chat_model
# 初始化模型
llm = init_chat_model("gpt-4o-mini", model_provider="openai")

from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_core.prompts import ChatPromptTemplate

tools = [TavilySearchResults(max_results=1)]

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are a helpful assistant.",
        ),
        ("placeholder", "{chat_history}"),
        ("human", "{input}"),
        ("placeholder", "{agent_scratchpad}"),
    ]

)

# 构建Tools agent
# agent 负责生成决策和工具调用请求
agent = create_tool_calling_agent(llm, tools, prompt)

# agent_executor 负责实际执行这些请求，并管理整个执行流程。
agent_executor = AgentExecutor(agent=agent, tools=tools)
response = agent_executor.invoke(
    {"input": "《龙猫》这个电影的导演是谁，今天多少岁了?"}
)

chat_history = []
# 更新聊天历史
chat_history.append(("human", "《龙猫》这个电影的导演是谁，今天多少岁了?"))
chat_history.append(("ai", response["output"]))

agent_executor.invoke(
    {"input": "我刚刚问了什么问题", "chat_history": chat_history}
)
"""
{'input': '我刚刚问了什么问题',
 'chat_history': [('human', '《龙猫》这个电影的导演是谁，今天多少岁了?'),
  ('ai',
   '《龙猫》的导演是宫崎骏（Hayao Miyazaki），他于1951年1月5日出生。因此，截止到今天（2023年10月），他是72岁。')],
 'output': '你问的是《龙猫》这部电影的导演是谁，以及他今天多少岁了。'}
"""
```

<mark style="background: #FFB86CA6;">verbose：</mark>
- 关键步骤的调试信息
```python
from langchain.globals import set_verbose

set_verbose(True)
agent_executor = AgentExecutor(agent=agent, tools=tools)
agent_executor.invoke(
    {"input": "《龙猫》这个电影的导演是谁，今天多少岁了?"}
)
"""
------------ verbose执行过程 ------------
Entering new AgentExecutor chain...

Invoking: `tavily_search_results_json` with `{'query': '《龙猫》 导演'}`
[{'title': '龙猫_百度百科', 'url': 'https://baike.baidu.com/item/%E9%BE%99%E7%8C%AB/12015836', 'content': '《龙猫》是由宫崎骏自编自导，日高法子、坂本千夏、糸井重里、岛本须美、北林谷荣等参与配音的动画电影。该片于1988年4月16日在日本上映，2018年12月14日数码修复版在', 'score': 0.86052185}]

Invoking: `tavily_search_results_json` with `{'query': '宫崎骏 生日'}`
[{'title': '84岁生日，动画大师宫崎骏，为中华蛇年，送上手绘生肖图！ - 网易', 'url': 'https://www.163.com/dy/article/JL5RGT7Q0517BURM.html', 'content': '1951年1月5日出生于日本东京的动画大师宫崎骏（HayaoMiyazaki）今日迎来了自己84岁的生日！请每一位欣赏过并热爱着这位寿星的影迷观众，一同默默祝福其', 'score': 0.92074}]
《龙猫》的导演是宫崎骏。他出生于1951年1月5日，今天他84岁。

Finished chain

------------- output -------------
{'input': '《龙猫》这个电影的导演是谁，今天多少岁了?',
 'output': '《龙猫》的导演是宫崎骏。他出生于1951年1月5日，今天他84岁。'}
"""
```
<mark style="background: #FFB86CA6;">debug：</mark>
- 信息很全
```python
from langchain.globals import set_debug
set_debug(True)
```