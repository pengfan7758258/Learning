# 英文-raw
Your goal is to generate sophisticated and diverse web search queries. These queries are intended for an advanced automated web research tool capable of analyzing complex results, following links, and synthesizing information.

Instructions:
- Always prefer a single search query, only add another query if the original question requests multiple aspects or elements and one query is not enough.
- Each query should focus on one specific aspect of the original question.
- Don't produce more than {number_queries} queries.
- Queries should be diverse, if the topic is broad, generate more than 1 query.
- Don't generate multiple similar queries, 1 is enough.
- Query should ensure that the most current information is gathered. The current date is {current_date}.

Format:
- Format your response as a JSON object with ALL three of these exact keys:
- "rationale": Brief explanation of why these queries are relevant
- "query": A list of search queries

Example:
Topic: What revenue grew more last year apple stock or the number of people buying an iphone
```json
{{
"rationale": "To answer this comparative growth question accurately, we need specific data points on Apple's stock performance and iPhone sales metrics. These queries target the precise financial information needed: company revenue trends, product-specific unit sales figures, and stock price movement over the same fiscal period for direct comparison.",
"query": ["Apple total revenue growth fiscal year 2024", "iPhone unit sales growth fiscal year 2024", "Apple stock price growth fiscal year 2024"],
}}
```

Context: {research_topic}

# 中文-translate
你的目标是生成复杂多样的网络搜索查询。这些查询旨在用于能够分析复杂结果、跟踪链接和合成信息的高级自动化网络研究工具。

说明：
- 始终更喜欢单个搜索查询，只有在原始问题要求多个方面或元素并且一个查询不够时才添加另一个查询。
- 每个查询都应该关注原始问题的一个特定方面。
- 不要生成超过{number_queries}个查询。
- 查询应该多样化，如果主题广泛，则生成多个查询。
- 不要生成多个类似的查询，1个就足够了。
- 查询应确保收集到最新信息。当前日期为{current_date}。

格式：
- 将您的响应格式化为具有以下三个确切键的JSON对象：
- "rationale"：简要解释这些疑问的相关性
- "query"：搜索查询列表

例子：

主题：去年苹果股票或购买iphone的人数的收入增长更多
```json
{{
"rationale"："为了准确回答这个比较增长的问题，我们需要有关苹果股票表现和iPhone销售指标的具体数据点。这些查询针对所需的精确财务信息：公司收入趋势、特定产品的单位销售数据和同一财政期间的股价变动，以便直接比较。"，
"query"：["苹果2024财年总收入增长", "iPhone销量增长2024财年", "苹果股价增长2024财务年度"]，
}}
```

上下文：{research_topic}