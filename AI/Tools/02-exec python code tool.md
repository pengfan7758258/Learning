# python_repl_tool
```python
from langchain_core.tools import tool
from langchain_experimental.utilities import PythonREPL

# 警告：这将在本地执行代码，如果不进行sandboxed处理，则可能不安全
repl = PythonREPL()

@tool
def python_repl_tool(
	code: Annotated[str, "The python code to execute to generate your chart."],
):
"""Use this to execute python code. If you want to see the output of a value,
you should print it out with `print(...)`. This is visible to the user."""
	try:
		result = repl.run(code)
	except BaseException as e:
		return f"Failed to execute. Error: {repr(e)}"

	result_str = f"Successfully executed:\n```python\n{code}\n```\nStdout: {result}"
	return (
	result_str + "\n\nIf you have completed all tasks, respond with FINAL ANSWER."
	
	)
```