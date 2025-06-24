# letta-ai ：为 LLM 应用注入记忆能力的开发框架
https://github.com/letta-ai/letta

# Mem0：新一代AI Agent的持久化记忆体系
- **记忆处理**：利用大型语言模型自动从对话中提取并存储关键信息，同时保持完整的上下文语境
- **记忆管理**：持续更新存储信息并消除矛盾点，确保数据准确性
- **双重存储架构**：结合向量数据库（用于记忆存储）和图数据库（用于关系追踪）的混合存储方案
- **智能检索系统**：采用语义搜索与图查询技术，根据信息重要性和时效性检索相关记忆
- **便捷API集成**：提供简单易用的记忆添加（add）与检索（search）接口端点
- [微信公众号讲解](https://mp.weixin.qq.com/s/1TUoquan8aCFx-aK6_J1-Q?poc_token=HNYLUGijpBRwCN_FbI3PumOCFmsEP4OsVhLBDvK2)
- [github地址](https://github.com/mem0ai/mem0)
- demo案例：
```python
#...省略全局配置  
#从文本中提取并添加新的记忆  
memory.add(  
        "林峰于2022年创办了星辰智能科技。这家总部位于深圳的公司专注于人工智能领域，其核心产品是名为“星语”的智能对话助手",  
        user_id="aUser")  
#基于文本比对，从记忆库中检索出与文本相关的记忆  
relevant_memories = memory.search(query="星辰智能科技的创始人是谁", user_id="aUser")  
memories = "\n".join(f"- {entry['memory']}"for entry in relevant_memories["results"])  
 relations = "\n".join( f"- {entry['source']}-{entry['relationship']}-{entry['destination']}"for entry in relevant_memories["relations"])  
print(f"memories:\n{memories}")  
print(f"relations:\n{relations}")

"""
output:
memories:  
- 星辰智能科技专注于人工智能领域  
- 星辰智能科技总部位于深圳  
- 林峰于2022年创办了星辰智能科技  
- 星辰智能科技的核心产品是名为“星语”的智能对话助手  
relations:  
- 林峰-founded-星辰智能科技  
- 星辰智能科技-headquarters_located_in-深圳  
- 星辰智能科技-specializes_in-人工智能  
- 星辰智能科技-core_product-星语
"""
```

# 方案
- rag搜索query相关的QA作为history
- 保留前n轮对话作为history
- 总结历史对话，保存关键信息，利用大型语言模型自动从对话中提取并存储关键信息，同时保持完整的上下文语境
- 图数据库：用户关系追踪、更新最新信息（e.g. 年龄的更新：18->19）