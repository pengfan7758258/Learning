- [地址](https://python.langchain.com/docs/how_to/chat_streaming/)
- token-by-token的流式方式看提供商是否支持，支持[列表](https://python.langchain.com/docs/integrations/chat/)

```python
from langchain.chat_models import init_chat_model

llm = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai"
)

for chunk in llm.stream("告诉我“守株待兔”的故事"):
    print(chunk.content, end="|", flush=True)
"""
------------- output -------------
|“|守|株|待|兔|”|是|一个|源|于|中国|古|代|的|寓|言|故事|，|出|自|《|韩|非|子|》。|这个|故事|传|达|了|对|懒|惰|和|依|赖|侥|幸|心理|的|批|评|。

|故事|的|主要|内容|是|：|古|时|有|一个|农|民|，他|在|田|里|耕|作|的时候|，|看到|一|只|兔|子|不|小|心|撞|到|树|桩|上|，|倒|地|而|死|。|这个|农|民|想|，|捡|到|这样的|兔|子|真|是|太|轻|松|了|，于|是|他|决定|守|在|树|桩|旁|，希望|再|能|遇|到|类似|的|好运|。|可是|，他|整|天|守|在|那|儿|，却|再|也|没有|看到|兔|子|出现|，|最终|却|耽|误|了|自己的|农|田|和|生活|。

|这个|故事|告诉|我们|，|依|赖|偶|然|的|好运|和|不|努力|工作|是|不可|行|的|，|成功|需要|付|出|实际|的|努力|，而|不是|坐|等|机会|的|降|临|。||
"""
# 异步实现
async for chunk in chat.astream("告诉我“守株待兔”的故事"):  
	print(chunk.content, end="|", flush=True)
```