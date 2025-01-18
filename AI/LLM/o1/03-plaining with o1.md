o1模型制定一个解决任务的计划
1. 给定一组执行计划并施加约束的工具
2. 设置任务的边界
如果使用o1来执行每一步会非常慢，所以这里使用o1-mini来生成一个计划，gpt-4o-mini来执行每一步具体的任务


Plan Generation + Execution Architecture（计划生成+执行结构）
![[03-plaining with o1.png|450]]