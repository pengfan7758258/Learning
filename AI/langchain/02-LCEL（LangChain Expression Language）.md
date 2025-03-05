- [地址](https://python.langchain.com/docs/concepts/lcel/#should-i-use-lcel)
- 链式表达语言：LCEL is an orchestration solution（编排解决）
- 从现有的Runnables构建新的Runnables

<mark style="background: #FFB86CA6;">LCEL优势：</mark>
- 并行执行优化：使用RunnableParallel或者Runnable Batch API并行多个input chain
- 支持async：使用Runnable Async API可以异步运行任何基于LCEL的chain，能有效处理大规模的并发
- 流式：支持流式输出，优化LLMS的第一个token出来之前经过的时间

<mark style="background: #FFB8EBA6;">langGraph支持多个node，每个node可以写一个LCEL，node与node之间可以组合成chain，形成一个complex的graphs来解决非常复杂的任务</mark>

<mark style="background: #FFB86CA6;">两个重要的组成元素：</mark>
- RunnableSequence
	- 当前runnable的输出可以作为下一个runnable的输入
	- 使用`|`特殊符号即可实现
		- `runnable1 | runnable2`等价于`RunnableSequence([runnable1, runnable2])`
- RunnableParallel
	- 多个runnable可以并发执行，只要输入相同
	- 支持同步、异步执行
- 其它的一些可以是上述两个的变体（e.g.,RunnableAssign,RunnableLambda）