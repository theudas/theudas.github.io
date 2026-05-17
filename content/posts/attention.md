+++
date = '2026-05-17T09:19:44+08:00'
draft = true
title = 'Attention 机制'
+++

Attention 机制简单来说就是给定 Q(Query), K(Key), V(Value)，通过 Query 和 Key 的匹配程度来决定从 Value 中提取多少信息(就是一个加权求和的过程)。这个可以参考数据库中的查询，根据查询键 Q 去匹配数据库中的键 K，找到对应的记录并取出该记录的值 V。Attention 机制与此类似，先通过 $ Q \cdot K^T $ 来计算相似度，由于$Q \cdot K^T$的结果是一个实数值向量，它的取值范围可能会很大，所以跟 V 相乘之前还需要先进行 softmax，softmax 会把它们归一化成一个和为 1 的概率分布，因此可以写成 $softmax(Q \cdot K^T) \cdot V$。不过 Transformer 的原论文[《Attention Is All You Need》](https://arxiv.org/abs/1706.03762)中还对$Q \cdot K^T$后的结果除了一个缩放因子$\sqrt{d_k}$，因为随着 $d_k$ 增大，点积的方差会变大，softmax 更容易饱和，导致梯度变小，而除以 $\sqrt{d_k}$ 可以避免这种饱和。由此可以得出最终的公式为 $Attention(Q,K,V)=softmax(\frac {Q \cdot K^T}{\sqrt{d_k}}) \cdot V$。由于加了“缩放因子”，所以这种Attention机制也叫**Scaled Dot-Product Attention**机制。

## Self-Attention vs. Cross-Attention
Self-Attention 之所以叫 "Self"，就是因为它的 Query、Key、Value 都是由同一个 x 分别通过不同的线性层（k_proj、q_proj、v_proj）来得到的。而 Cross-Attention 的 "Cross" 则是因为在 Transformer 中，它的 Key、Value 来自 Encoder 中的输出，而 Query 则是 Decoder 中的输入。

## Multi-Head Attention（多头注意力机制, MHA）
Transformer 中用的其实是 MHA，他就是把 Q、K、V 都分成多个头，在每个头中都分别进行一次 Attention 操作，最后再把所有头的结果拼起来。这样的好处是可以不同的头关注不同的子空间，从而提取出不同的特征信息。具体的做法如下：
```python
import math
import torch
import torch.nn as nn

class MultiHeadAttention(nn.Module):
    def __init__(self, hidden_size, num_heads):
        super().__init__()
        self.hidden_size = hidden_size
        self.num_heads = num_heads               # 头数
        self.head_dim = hidden_size // num_heads # 每个头的维度

        # Q、K、V 投影矩阵
        self.q_proj = nn.Linear(hidden_size, hidden_size)
        self.k_proj = nn.Linear(hidden_size, hidden_size)
        self.v_proj = nn.Linear(hidden_size, hidden_size)
        
        # 输出线性层
        self.o_proj = nn.Linear(hidden_size, hidden_size)
    
    def forward(self, x, attention_mask=None):
        batch_size = x.size()[0]

        # 输入的 x 形状为（batch_size, seq_len, hidden_size）

        # 计算 Q、K、V
        q = self.q_proj(x)
        k = self.k_proj(x)
        v = self.v_proj(x)
        # 经过投影层后，Q、K、V 的形状为（batch_size, seq_len, hidden_size）

        # 把 Q、K、V 都分成多个头
        q = q.view(batch_size, -1, self.num_heads, self.head_dim).transpose(1, 2)
        k = k.view(batch_size, -1, self.num_heads, self.head_dim).transpose(1, 2)
        v = v.view(batch_size, -1, self.num_heads, self.head_dim).transpose(1, 2)
        # 经过分头操作后，Q、K、V 的形状为（batch_size, num_heads, seq_len, head_dim）

        # 计算 Attention
        attn_scores = q @ k.transpose(-2, -1) / math.sqrt(self.head_dim)
        if attention_mask is not None: # attention_mask 矩阵中为 0 的位置不关注
            attn_scores = attn_scores.masked_fill(attention_mask == 0, float("-inf"))
        # attn_scores 的形状为（batch_size, num_heads, seq_len, seq_len）
        attn_probs = torch.softmax(attn_scores, dim=-1)
        # attn_probs 的形状为（batch_size, num_heads, seq_len, seq_len）
        output = attn_probs @ v
        # output 的形状为（batch_size, num_heads, seq_len, head_dim）

        # 合并所有头的结果
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.hidden_size)
        # output 的形状变回（batch_size, seq_len, hidden_size）
        output = self.o_proj(output)
        return output
```

## MQA（Multi-Query Attention）
MQA 是 MHA 的一个变体，它的多个 Query 共享同一套 Key、Value，这样只需要保留一组 Key、Value 即可，可以减少参数量、显著减少推理时的 KV cache 和内存带宽开销，但表达能力可能会降低。具体的实现如下（大部分跟 MHA 是一样的，除了 k_proj/v_proj 的输出维度和分头操作）：
```python
import math
import torch
import torch.nn as nn

class MultiQueryAttention(nn.Module):
    def __init__(self, hidden_size, num_heads):
        super().__init__()
        self.hidden_size = hidden_size
        self.num_heads = num_heads
        self.head_dim = hidden_size // num_heads

        # Q、K、V 投影矩阵
        self.q_proj = nn.Linear(hidden_size, hidden_size)
        self.k_proj = nn.Linear(hidden_size, self.head_dim)
        self.v_proj = nn.Linear(hidden_size, self.head_dim)

        # 输出线性层
        self.o_proj = nn.Linear(hidden_size, hidden_size)

    def forward(self, x, attention_mask=None):
        batch_size = x.size()[0]
        # 输入的 x 形状为（batch_size, seq_len, hidden_size）

        # 计算 Q、K、V
        q = self.q_proj(x)
        k = self.k_proj(x)
        v = self.v_proj(x)
        # 经过投影后，Q 的形状为（batch_size, seq_len, hidden_size）
        # K、V 的形状为（batch_size, seq_len, head_dim）

        # 把 Q 分成多个头， K、V reshape 成一个共享头（之后广播给所有的 query head 使用）
        q = q.view(batch_size, -1, self.num_heads, self.head_dim).transpose(1, 2)
        k = k.view(batch_size, -1, 1, self.head_dim).transpose(1, 2)
        v = v.view(batch_size, -1, 1, self.head_dim).transpose(1, 2)
        # 经过分头操作后，Q 的形状为（batch_size, num_heads, seq_len, head_dim）
        # K、V 的形状为（batch_size, 1, seq_len, head_dim）

        # 计算 Attention
        attn_scores = q @ k.transpose(-2, -1) / math.sqrt(self.head_dim)
        # 计算 q @ k^T 时，k^T 会在 num_heads 维度进行广播
        # 最后得到的 attn_scores 的形状为（batch_size, num_heads, seq_len, seq_len）
        if attention_mask is not None:
            attn_scores = attn_scores.masked_fill(attention_mask == 0, float("-inf"))
        attn_probs = torch.softmax(attn_scores, dim=-1)
        output = attn_probs @ v

        # 合并所有头的结果
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.hidden_size)
        output = self.o_proj(output)
        return output
```

## GQA（Group-Query Attention）
GQA 实际是 MHA 和 MQA 之间的一种 trade-off，它把 Query 分成多个 group，每个 group 里的 Query 共享同一套 Key、Value。 具体的实现如下所示
```python
import math
import torch
import torch.nn as nn

class GroupQueryAttention(nn.Module):
    def __init__(self, hidden_size, num_heads, num_groups):
        super().__init__()
        self.hidden_size = hidden_size           # 隐藏层维度
        self.num_heads = num_heads               # 头数
        self.num_groups = num_groups             # 组数
        self.head_dim = hidden_size // num_heads # 每个头的维度
        # 每个组有 self.num_heads // self.num_groups 个头

        self.q_proj = nn.Linear(hidden_size, hidden_size)
        self.k_proj = nn.Linear(hidden_size, self.head_dim * num_groups)
        self.v_proj = nn.Linear(hidden_size, self.head_dim * num_groups)

        self.o_proj = nn.Linear(hidden_size, hidden_size)

    def forward(self, x, attention_mask=None):
        batch_size = x.size()[0]
        
        q = self.q_proj(x)
        k = self.k_proj(x)
        v = self.v_proj(x)

        q = q.view(batch_size, -1, self.num_heads, self.head_dim).transpose(1, 2)
        k = (k.view(batch_size, -1, self.num_groups, 1, self.head_dim)
            .expand(batch_size, -1, self.num_groups, self.num_heads // self.num_groups, self.head_dim)
            .reshape(batch_size, -1, self.num_heads, self.head_dim)
            .transpose(1, 2))
        v = (v.view(batch_size, -1, self.num_groups, 1, self.head_dim)
            .expand(batch_size, -1, self.num_groups, self.num_heads // self.num_groups, self.head_dim)
            .reshape(batch_size, -1, self.num_heads, self.head_dim)
            .transpose(1, 2))

        # 计算 Attention
        attn_scores = q @ k.transpose(-2, -1) / math.sqrt(self.head_dim)
        if attention_mask is not None:
            attn_scores = attn_scores.masked_fill(attention_mask == 0, float("-inf"))
        attn_probs = torch.softmax(attn_scores, dim=-1)
        output = attn_probs @ v

        # 合并所有头的结果
        output = output.transpose(1, 2).contiguous().view(batch_size, -1, self.hidden_size)
        output = self.o_proj(output)
        return output
```