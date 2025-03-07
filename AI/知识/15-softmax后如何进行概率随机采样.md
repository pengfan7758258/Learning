代码：
```python
import random
import bisect

class WeightedSampler:
    def __init__(self, words, probabilities):
        self.words = words
        # 计算前缀和数组
        self.prefix_sums = []
        cumulative_sum = 0
        for p in probabilities:
            cumulative_sum += p
            self.prefix_sums.append(cumulative_sum)

    def sample(self):
        # 生成一个 0~1 之间的随机数
        r = random.random()
        # 使用二分查找在前缀和数组中找到对应的索引
        index = bisect.bisect_left(self.prefix_sums, r)
        return self.words[index]

# 示例
words = ["apple", "banana", "cherry"]
probabilities = [0.2, 0.5, 0.3]
sampler = WeightedSampler(words, probabilities)

# 采样10次
samples = [sampler.sample() for _ in range(10)]
print(samples)

```

![[15-softmax后如何进行概率随机采样.png]]