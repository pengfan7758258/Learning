# 英文-raw
```
SQL Evaluation Prompt:
-----------------------
You are tasked with determining if the SQL generated appropiately answers a given instruction
taking into account its generated query and response.

Data:
-----
- [Instruction]: {question}
  This section contains the specific task or problem that the sql query is intended to solve.

- [Reference Query]: {query_gen}
  This is the sql query submitted for evaluation. Analyze it in the context of the provided
  instruction.

Evaluation:
-----------
Your response should be a single word: either "correct" or "incorrect".
You must assume that the db exists and that columns are appropiately named.
You must take into account the response as additional information to determine the correctness.

- "correct" indicates that the sql query correctly solves the instruction.
- "incorrect" indicates that the sql query correctly does not solve the instruction correctly.

Note: Your response should contain only the word "correct" or "incorrect" with no additional text
or characters.
```


# 中文-translate
```
SQL评估提示：
-----------------------
您的任务是确定生成的SQL是否正确地响应了给定的指令
考虑其生成的查询和响应。

数据：
-----
- [Instruction]: {question}
本节包含sql查询要解决的特定任务或问题。

- [Reference Query]: {query_gen}
这是提交用于评估的sql查询。在提供的上下文中进行分析
指令。

评价：
-----------
你的回答应该是一个词：“正确”或“不正确”。
您必须假设数据库存在，并且列的名称正确。
您必须将响应视为确定正确性的附加信息。

-"correct"表示sql查询正确地解决了指令。
-"incorrect"表示sql查询正确地解决不了指令。

注意：您的回复应仅包含"corret"或"incorrect"一词，无需额外文本
或字符。
```