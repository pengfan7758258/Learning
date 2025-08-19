# TavilySearch
```python
"""
env: 
	TAVILY_API_KEY=
"""
from langchain_tavily import TavilySearch
tavily_tool = TavilySearch(max_results=5) # max_results最大返回结果数量
```