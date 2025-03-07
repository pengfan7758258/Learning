# 1.简单版本
![[07-手写self-attetion.png]]
```python
import math
import torch
from torch import nn

class SelfAttentionOne(nn.Module):
    def __init__(self, input_dim=768, output_dim=768):
        """
        input_dim: 输入维度
        output_dim: 输出维度
        一般来说input_dim和output_dim一样
        """
        super().__init__()
        self.linear_q = nn.Linear(input_dim, output_dim) # 输入input_dim维，输出output_dim维, W_q维度: [input_dim, ouput_dim]
        self.linear_k = nn.Linear(input_dim, output_dim)
        self.linear_v = nn.Linear(input_dim, output_dim)

        self.softmax = nn.Softmax(dim=-1)

    def forward(self, X):
        """
        X: dim [bs, seq, embed_dim]
        """
        Q = self.linear_q(X) # [bs, seq, output_dim(768)]
        K = self.linear_k(X)
        V = self.linear_v(X)
        # print(Q.shape)
        # print(K.shape)
        # print(V.shape)
        # print(Q @ K.transpose(-1, -2)) # [bs, seq, seq]
        # print((Q @ K.transpose(-1, -2)) * (1 / math.sqrt(Q.shape[-1])) @ V)
        attention = self.softmax(((Q @ K.transpose(-1, -2)) * (1.0 / math.sqrt(Q.shape[-1])))) @ V

        return attention

X = torch.rand(3, 2, 4)
self_attention_one = SelfAttentionOne(input_dim=4)
self_attention_one(X)
```
# 2.做效率优化

- QKV 矩阵计算的时候，可以合并成一个大矩阵计算 -- 可能更高效
```python
class SelfAttentionTwo(nn.Module):
    def __init__(self, input_dim=768, output_dim=768):
        super().__init__()
        self.dim = output_dim
        self.linear_qkv = nn.Linear(input_dim, output_dim*3)
        self.softmax = nn.Softmax(dim=-1)

    def forward(self, X):
        QKV = self.linear_qkv(X)
        Q, K, V = torch.split(QKV, self.dim, dim=-1)
        attention = self.softmax((Q @ K.transpose(-1, -2)) * (1.0 / math.sqrt(Q.shape[-1]))) @ V

        return attention

X = torch.rand(3, 2, 4)
self_attention_two = SelfAttentionTwo(input_dim=4)
self_attention_two(X)
```

# 3.加入细节
- dropout：避免过拟合，让attention随机做一些遮蔽（例如总是关注相邻词，遮蔽后可以对其它位置更多关注）。消融实验表明，移除注意力Dropout会导致模型在验证集上的性能下降（过拟合迹象）
- attention_mask：padding部分的处理
- 多头最后还有一个全连接层
```python
class SelfAttentionThree(nn.Module):
    def __init__(self, input_dim=768, output_dim=768):
        super().__init__()
        self.dim = output_dim
        
        self.q = nn.Linear(input_dim, output_dim)
        self.k = nn.Linear(input_dim, output_dim)
        self.v = nn.Linear(input_dim, output_dim)
        self.attention_dropout = nn.Dropout(0.1) # 原始论文0.1
        self.softmax = nn.Softmax(dim=-1)

    def forward(self, X, attention_mask=None):

        Q = self.q(X)
        K = self.k(X)
        V = self.v(X)

        att_weight = (Q @ K.transpose(-1, -2)) * (1.0 / math.sqrt(Q.shape[-1]))
        if attention_mask is not None:
            # 给padding部分给一个很小的值，因为e^x是越-inf几乎为0
            att_weight = att_weight.masked_fill(attention_mask == 0, float("-1e20"))

        att_weight = self.attention_dropout(att_weight)

        attention = self.softmax(att_weight) @ V

        return attention
    

X = torch.rand(3, 4, 2)
b = torch.tensor(
    [
        [1, 1, 1, 0],
        [1, 1, 0, 0],
        [1, 0, 0, 0],
    ]
)
print(b.shape)
# b.unsqueeze(dim=1)在维度1上添加一维：[3, 4] -> [3, 1, 4]
attention_mask = b.unsqueeze(dim=1).repeat(1, 4, 1)
print(attention_mask)
self_attention_three = SelfAttentionThree(2)
self_attention_three(X, attention_mask)
```

# 4.MultiHead-Self-Attention
```python
import math
import torch
import torch.nn as nn

class MultiHeadAttention(nn.Module):
    def __init__(self, hidden_dim, nums_head) -> None:
        super().__init__()
        self.nums_head = nums_head

        # 一般来说，
        self.head_dim = hidden_dim // nums_head
        self.hidden_dim = hidden_dim

        # 一般默认有 bias，需要时刻主意，hidden_dim = head_dim * nums_head，所以最终是可以算成是 n 个矩阵
        self.q_proj = nn.Linear(hidden_dim, hidden_dim)
        self.k_proj = nn.Linear(hidden_dim, hidden_dim)
        self.v_proj = nn.Linear(hidden_dim, hidden_dim)

        # gpt2 和 bert 类都有，但是 llama 其实没有
        self.att_dropout = nn.Dropout(0.1)
        # 输出时候的 proj
        self.o_proj = nn.Linear(hidden_dim, hidden_dim)

    def forward(self, X, attention_mask=None):
        # 需要在 mask 之前 masked_fill
        # X shape is (batch, seq, hidden_dim)
        # attention_mask shape is (batch, seq)

        batch_size, seq_len, _ = X.size()

        Q = self.q_proj(X)
        K = self.k_proj(X)
        V = self.v_proj(X)

        # shape 变成 （batch_size, num_head, seq_len, head_dim）
        q_state = Q.view(batch_size, seq_len, self.nums_head, self.head_dim).permute(
            0, 2, 1, 3
        )
        k_state = K.view(batch_size, seq_len, self.nums_head, self.head_dim).transpose(
            1, 2
        )
        v_state = V.view(batch_size, seq_len, self.nums_head, self.head_dim).transpose(
            1, 2
        )
        # 主意这里需要用 head_dim，而不是 hidden_dim
        attention_weight = (
            q_state @ k_state.transpose(-1, -2) / math.sqrt(self.head_dim)
        )
        # print(type(attention_mask))
        if attention_mask is not None:
            attention_weight = attention_weight.masked_fill(
                attention_mask == 0, float("-1e20")
            )

        # 第四个维度 softmax
        attention_weight = torch.softmax(attention_weight, dim=3)
        # print(attention_weight)

        attention_weight = self.att_dropout(attention_weight)
        output_mid = attention_weight @ v_state

        # 重新变成 (batch, seq_len, num_head, head_dim)
        # 这里的 contiguous() 是相当于返回一个连续内存的 tensor，一般用了 permute/tranpose 都要这么操作
        # 如果后面用 Reshape 就可以不用这个 contiguous()，因为 view 只能在连续内存中操作
        output_mid = output_mid.transpose(1, 2).contiguous()

        # 变成 (batch, seq, hidden_dim),
        output = output_mid.view(batch_size, seq_len, -1)
        output = self.o_proj(output)
        return output

attention_mask = (
    torch.tensor(
        [
            [0, 1],
            [0, 0],
            [1, 0],
        ]
    )
    .unsqueeze(1)
    .unsqueeze(2)
    .expand(3, 8, 2, 2)
)

X = torch.rand(3, 2, 128)
net = MultiHeadAttention(128, 8)
net(X, attention_mask).shape
```