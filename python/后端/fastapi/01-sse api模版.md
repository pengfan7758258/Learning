```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

class StreamTokenCounter:
    def __init__(self, tokenizer):
        """
        tokenizer: 一个 callable，输入 str，返回 token 数
        """
        self.tokenizer = tokenizer
        self.prompt_tokens = 0
        self.completion_text = []
        self.completion_tokens = 0

    def set_prompt(self, messages: list[dict]):
        """
        在请求前调用
        """
        text = ""
        for m in messages:
            text += f"{m['role']}: {m['content']}\n"
        self.prompt_tokens = self.tokenizer(text)

    def on_delta(self, delta_text: str):
        """
        stream 时，每来一段就调用
        """
        self.completion_text.append(delta_text)

    def finalize(self):
        """
        stream 结束时调用
        """
        full_text = "".join(self.completion_text)
        self.completion_tokens = self.tokenizer(full_text)

    @property
    def total_tokens(self):
        return self.prompt_tokens + self.completion_tokens

import tiktoken
enc = tiktoken.get_encoding("cl100k_base")

def count_tokens(text: str) -> int:
    return len(enc.encode(text))

def stream_chat_completion(client, model, messages):
    counter = StreamTokenCounter(count_tokens)
    counter.set_prompt(messages)

    response_iter = client.chat.completions.create(
        model=model,
        messages=messages,
        stream=True,
    )

    try:
        for chunk in response_iter:
            delta = chunk.choices[0].delta

            if not delta:
                continue

            if "content" in delta:
                text = delta["content"]
                counter.on_delta(text)

                yield f"data: {json.dumps({'type': 'delta', 'content': text})}\n\n"

        counter.finalize()
        # TODO 这里存数据库

        # stream 结束，发送 token 统计
        yield f"data: {json.dumps({
            'type': 'usage',
            'prompt_tokens': counter.prompt_tokens,
            'completion_tokens': counter.completion_tokens,
            'total_tokens': counter.total_tokens,
        })}\n\n"

    except Exception as e:
        yield f"data: {json.dumps({'type': 'error', 'message': str(e)})}\n\n"


app = FastAPI()

@app.post("/chat/stream")
async def chat_stream():
    messages = [
        {"role": "system", "content": "你是一个非常聪明的助手"},
        {"role": "user", "content": "你叫什么？"},
    ]

    return StreamingResponse(
        stream_chat_completion(
            client=client,
            model=os.getenv("SELECT_MODEL"),
            messages=messages,
        ),
        media_type="text/event-stream",
    )

```