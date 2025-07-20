- 适合用来跑 **大量 IO 密集型操作**（如数据库访问、API 请求、文件读写等）
- 下面是一个案例，需要修改核心功能部分实现自己的需求
```python
async def process_contract(contract, semaphore):
    async with semaphore: # This controls access to the resource
        structured_data = await         llm.with_structured_output(Contract).ainvoke(contract["text"])

	    try:
			structured_data = json.loads(structured_data.model_dump_json())
		except: # When LLM breaks
			return {"file_id": contract["file_id"]}

		structured_data["file_id"] = contract["file_id"]
		# Clean dates
		structured_data["effective_date"] = structured_data["effective_date"] if is_valid_date(structured_data["effective_date"]) else None

		structured_data["end_date"] = structured_data["end_date"] if is_valid_date(structured_data["end_date"]) else None

		# Infer end date
		if not structured_data["end_date"] and (structured_data["effective_date"] and structured_data["duration"]):
			try:
				structured_data["end_date"] = add_duration_to_date(structured_data["effective_date"], structured_data["duration"])
			except:
				pass
		return structured_data

async def process_all(contracts, max_workers=10):
	# Create a semaphore with the desired number of workers
	semaphore = asyncio.Semaphore(max_workers)
	# Create tasks with the semaphore
	tasks = [process_contract(contract, semaphore) for contract in contracts]
	# Use tqdm with asyncio.as_completed to show progress
	results = []
	for future in tqdm(asyncio.as_completed(tasks), total=len(tasks), desc="Processing contracts"):
		result = await future
		results.append(result)

	return results

results = await process_all(contracts)
```