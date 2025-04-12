- [地址](https://platform.openai.com/docs/guides/batch?lang=python)
- 批量传输请求，api花费降低百分之50

<mark style="background: #FFB86CA6;">上传流程：</mark>
1. 生成一个jsonl文件，格式如下：
```jsonl
{"custom_id": "request-1", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "gpt-3.5-turbo-0125", "messages": [{"role": "system", "content": "You are a helpful assistant."},{"role": "user", "content": "Hello world!"}],"max_tokens": 1000}}
{"custom_id": "request-2", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "gpt-3.5-turbo-0125", "messages": [{"role": "system", "content": "You are an unhelpful assistant."},{"role": "user", "content": "Hello world!"}],"max_tokens": 1000}}
```
custom_id是唯一值
2. 上传jsonl文件 -> 创建batch
```python
from openai import OpenAI
client = OpenAI() # 需要api_key

# 上传文件
batch_input_file = client.files.create(
    file=open("batchinput.jsonl", "rb"),
    purpose="batch"
)

print(batch_input_file)

# 创建batch
batch_input_file_id = batch_input_file.id
batches = client.batches.create(
    input_file_id=batch_input_file_id,
    endpoint="/v1/chat/completions", # 还有/v1/completions生成式接口可用
    completion_window="24h", # 这个是固定的
    metadata={ # 自定义元数据信息，可以不要
        "description": "nightly eval job"
    }
)
print(batches)
"""
{
  "id": "batch_abc123", # batch的id，可以track这个id查询任务进度
  "object": "batch",
  "endpoint": "/v1/chat/completions",
  "errors": null,
  "input_file_id": "file-abc123",
  "completion_window": "24h",
  "status": "validating", # task状态，completed代表完成
  "output_file_id": null,
  "error_file_id": null,
  "created_at": 1714508499,
  "in_progress_at": null,
  "expires_at": 1714536634,
  "completed_at": null,
  "failed_at": null,
  "expired_at": null,
  "request_counts": {
    "total": 0,
    "completed": 0,
    "failed": 0
  },
  "metadata": null
}
"""
```
3. 查询task状态
```python
# 这个batch其实和上面的创建batches是一个对象
batch = client.batches.retrieve("batch_abc123")
print(batch)
```
batch的status状态如下表：
![[00-openai batch api.png]]
4. 任务完成获取结果
```python
# 完成后的batch status会变成completed，然后它里面有一个ouput_file_id作为参数填入下面
file_response = client.files.content("output_file_id")
print(file_response.text)
```
5. 请求batch 请求
```python
from openai import OpenAI
client = OpenAI()

client.batches.cancel("batch_id")
```
6. 获得所有batch请求
```python
from openai import OpenAI
client = OpenAI()

batches = client.batches.list(limit=10) # limit只看最近10个

batches.data[0] # 取的最新的一个batch任务
```