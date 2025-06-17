# letta-ai ：为 LLM 应用注入记忆能力的开发框架
https://github.com/letta-ai/letta

# Mem0：新一代AI Agent的持久化记忆体系
- **记忆处理**：利用大型语言模型自动从对话中提取并存储关键信息，同时保持完整的上下文语境
- **记忆管理**：持续更新存储信息并消除矛盾点，确保数据准确性
- **双重存储架构**：结合向量数据库（用于记忆存储）和图数据库（用于关系追踪）的混合存储方案
- **智能检索系统**：采用语义搜索与图查询技术，根据信息重要性和时效性检索相关记忆
- **便捷API集成**：提供简单易用的记忆添加（add）与检索（search）接口端点
- [微信公众号讲解](https://mp.weixin.qq.com/s/1TUoquan8aCFx-aK6_J1-Q?poc_token=HNYLUGijpBRwCN_FbI3PumOCFmsEP4OsVhLBDvK2)
- [github地址]()
# 方案
- rag搜索query相关的QA作为history
- 保留前n轮对话作为history
- 总结历史对话，保存关键信息，利用大型语言模型自动从对话中提取并存储关键信息，同时保持完整的上下文语境
- 图数据库：用户关系追踪、更新最新信息（e.g. 年龄的更新：18->19）