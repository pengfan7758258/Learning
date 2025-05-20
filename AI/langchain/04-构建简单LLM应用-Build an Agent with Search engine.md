- [地址](https://python.langchain.com/docs/tutorials/agents/)
- 使用LLM作为推理引擎去决定所要执行的action和对应的input
- 经常需要和tool-calling（工具调用）配合使用
- 需要申请Tavily的[search api](https://docs.tavily.com/sdk/python/quick-start)，Tavily登录需要一个身份验证器，android身份验证器[下载](https://shouyou.3dmgame.com/android/461920.html)

<mark style="background: #BBFABBA6;">定义工具tools和LLM model：</mark>
```python
# 定义工具，tavily search engine是一个langchain内置的工具
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_core.messages import HumanMessage
from langchain.chat_models import init_chat_model

search = TavilySearchResults(max_results=2) # 初始化tavily搜索工具, max_results代表给几个结果

# search_results = search.invoke("what is the weather in beijing") # 搜索
# print(search_results)

# 你也可以创建其它的tools.
# 全部的工具放入list，后续当做agent的参考并使用.
tools = [search]

# 初始化llm
model = init_chat_model(
    model="gpt-4o-mini", # 模型名
    model_provider="openai" # 模型提供者
)
```
<mark style="background: #BBFABBA6;">langgraph创建agent：</mark>
```python
# langgraph构建agent
from langgraph.prebuilt import create_react_agent

agent_executor = create_react_agent(model, tools)
```
<mark style="background: #BBFABBA6;">两次不同输入验证tools的调用：</mark>
```python
# 尝试一次普通对话，看是否会调用tool
response = agent_executor.invoke({"messages": [HumanMessage(content="hi!")]})
response["messages"]
"""
[HumanMessage(content='hi!', additional_kwargs={}, response_metadata={}, id='b05475ea-876f-4357-88c2-ae391ad06db5'),

 AIMessage(content='Hello! How can I assist you today?', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 11, 'prompt_tokens': 81, 'total_tokens': 92, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_06737a9306', 'finish_reason': 'stop', 'logprobs': None}, id='run-77b9d8d6-1a3a-4ed9-9590-d4df17172bbe-0', usage_metadata={'input_tokens': 81, 'output_tokens': 11, 'total_tokens': 92, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]
"""

# 尝试不一样的input来让tool被使用
response = agent_executor.invoke(
    {"messages": [HumanMessage(content="whats the weather in sf?")]}
)
response["messages"]
"""
[HumanMessage(content='whats the weather in sf?', ...),

 AIMessage(content='', additional_kwargs={'tool_calls': [{'id': 'call_pZQtNQkM2DXL5D4qbjgEDcQz', 'function': {'arguments': '{"query":"current weather in San Francisco"}', 'name': 'tavily_search_results_json'}, ...),
 
 ToolMessage(content='[{"title": "Weather in San Francisco", "url": "https://www.weatherapi.com/", ..., "content": "...}),
 
 AIMessage(content='The current weather in San Francisco is partly cloudy with a temperature of approximately 52°F (11.1°C). ...)]
"""
# 可以看到，有ToolMessage的消息，证明tool被调用
```
<mark style="background: #BBFABBA6;">Streaming Messages</mark>
- 这里不是传统的流式输出，而是上述的.invoke是全部式的响应，包括了工具的调用和最终AI的整体返回，很耗费时间
- .stream的agent输出方式能让你看到每一步的输出，不至于等待太久
```python
for step in agent_executor.stream(
    {"messages": [HumanMessage(content="whats the weather in sf?")]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()

"""
=======[1m Human Message [0m=========
whats the weather in sf?

=======[1m Ai Message [0m===========
Tool Calls:
  tavily_search_results_json (call_4FmBRrKbXfF403Pleoatw1P3)
 Call ID: call_4FmBRrKbXfF403Pleoatw1P3
  Args:
    query: current weather in San Francisco

======[1m Tool Message [0m===============
Name: tavily_search_results_json
[{"title": "Weather in San Francisco", "url": "ht...]

=======[1m Ai Message [0m============
The current weather in San Francisco is as follows:
- **Temperature**: 11.1°C (52.0°F)
- **Condition**: Partly cloudy
- **Humidity**: 74%
- **Wind**: 9.2 mph from the WNW
- **Visibility**: 16 km (about 9 miles)
- **Pressure**: 1016 mb

For more detailed and real-time updates, you can check out [Weather.com](https://weather.com/weather/today/l/USCA0987:1:US) or [WeatherAPI](https://www.weatherapi.com/).
"""
```
<mark style="background: #BBFABBA6;">Streaming tokens</mark>
- 流式输出，.stream的stream_mode设置为messages
```python
for step, metadata in agent_executor.stream(
    {"messages": [HumanMessage(content="whats the weather in sf?")]},
    stream_mode="messages",
):

    if metadata["langgraph_node"] == "agent" and (text := step.text()): # TODO 还没看懂是为何这样写
        print(text, end="|")

"""
The| current| weather| in| San| Francisco| is| as| follows|:

|-| **|Temperature|**|:| |10|.|2|°C| (|50|.|4|°F|)
|-| **|Condition|**|:| Clear|
|-| **|Humidity|**|:| |86|%
|-| **|Wind|**|:| |7|.|6| mph| (|12|.|2| k|ph|)| from| the| W|NW|
|-| **|Pressure|**|:| |101|5| mb| (|29|.|98| in|)
|-| **|Visibility|**|:| |16| km| (|9| miles|)

|You| can| refer| to| [|Weather|API|](|https|://|www|.weather|api|.com|/)| for| more| details|.|
"""
```