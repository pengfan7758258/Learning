![[32-GQA分组查询注意力.png]]
分组查询注意力 (GQA) 是一种在大型语言模型中的多查询注意力 (MQA) 和多头注意力 (MHA) 之间进行插值的方法，它的目标是在保持 MQA 速度的同时实现 MHA 的质量。

标准多头注意力(MHA)由H个查询头、键头和值头组成。每个头都有D个维度。
MHA:
```python
# shapes: (batch_size, seq_len, num_heads, head_dim)
query = torch.randn(1, 256, 8, 64) 
key = torch.randn(1, 256, 8, 64) 
value = torch.randn(1, 256, 8, 64)
```

GQA将查询头分成G组，每组共享一个键和值。
GQA:
```python
# shapes: (batch_size, seq_len, num_heads, head_dim) 
query = torch.randn(1, 256, 8, 64) 
key = torch.randn(1, 256, 2, 64) 
value = torch.randn(1, 256, 2, 64)

query_heads = query.shape[2]
kv_heads = key.shape[2]
assert query_heads % kv_heads == 0, f"query_heads ({query_heads}) must be divisible by kv_heads ({kv_heads})"
```

Qwen源码：维度是假设的
hidden_size=768,num_attention_heads=12,num_key_value_heads=2,head_dim=64
seq_len=512
![[32-GQA分组查询注意力 1.png]]
![[32-GQA分组查询注意力 2.png]]
num_key_value_groups=num_attention_heads // num_key_value_heads=6
![[32-GQA分组查询注意力 3.png]]
![[32-GQA分组查询注意力 4.png]]