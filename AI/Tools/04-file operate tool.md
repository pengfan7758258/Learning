
# read_file
```python
from pathlib import Path
from langchain_core.tools import tool

WORKING_DIRECTORY = Path("/home/username/workdir")

@tool
def read_file(
	file_name: Annotated[str, "File path to read the file from."]
) -> str:
	"""Read the specified file."""
	if not (WORKING_DIRECTORY / file_name).exists():
		return f"Error: {WORKING_DIRECTORY / file_name} not found."
	# WORKING_DIRECTORY是from pathlib import Path对象
	with (WORKING_DIRECTORY / file_name).open("r", encoding="utf-8") as file:
		content = file.read()
	return content
```

# write_file
```python
from pathlib import Path
from langchain_core.tools import tool

WORKING_DIRECTORY = Path("/home/username/workdir")

@tool
def write_file(
	content: Annotated[str, "Text content to be written into the file."],
	file_name: Annotated[str, "File path to save the content."],
) -> Annotated[str, "Path of the saved content detail."]:
	"""Create and save a text file."""
	with (WORKING_DIRECTORY / file_name).open("w") as file:
		file.write(content)
	return f"Document saved to {file_name}"
```