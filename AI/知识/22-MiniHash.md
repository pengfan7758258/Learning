MinHash（**最小哈希**）是一种用于**近似集合相似度计算**的算法，尤其在计算**Jaccard 相似度**方面非常高效。它被广泛用于**去重、聚类、搜索引擎、推荐系统**等领域。

---

## 🎯 MinHash 的目标

给定两个集合 `A` 和 `B`，我们希望高效地估算它们的 **Jaccard 相似度**：
$$
J(A,B)=∣A∩B∣∣A∪B∣J(A, B) = \frac{|A \cap B|}{|A \cup B|}
$$
MinHash 的核心思想是：通过构造多个哈希函数，将集合表示为签名向量（signature），这些签名之间的相似性就能很好地近似原始集合的 Jaccard 相似度。

---

## 🧠 MinHash 的原理（简化步骤）

### 假设：

我们有两个集合（比如两个文档的词集）：

- A = {apple, banana, cat, dog}
- B = {apple, banana, dog, elephant}

我们用哈希函数将这些词映射到数字上，然后找出每个集合在这些哈希函数下的最小值。

---

### ✅ 核心步骤：

1. **预处理：集合 -> 特征集合（Shingling）**
    - 对文档或文本进行分词、n-gram 等操作，得到集合表示。
2. **构造多个哈希函数（如 H1, H2, ..., Hk）**
    - 每个哈希函数都对应一个签名向量中的位置。
3. **对每个集合应用哈希函数，取最小值（Min Hash）**
    - 对于每个集合和每个哈希函数，记录该集合中元素被哈希后的最小值。
    举个例子（用整数哈希模拟）：
    - `H1(apple) = 5, H1(banana) = 3, H1(cat) = 7, H1(dog) = 2`
    - 对集合 A = {apple, banana, cat, dog}，MinHash = min(5, 3, 7, 2) = 2
4. **得到“签名向量”**
    - 每个集合在多个哈希函数下的最小值，组合成向量。例如：
        ```
        Sig(A) = [2, 1, 9, ...]  # k个哈希函数
        Sig(B) = [2, 1, 13, ...]
        ```
5. **计算签名向量的相等比例 ≈ Jaccard 相似度**
    - 比较两个签名向量中相等的位置个数，占总长度的比例：
$$Similarity=相等的维度数总维度数≈J(A,B)\text{Similarity} = \frac{\text{相等的维度数}}{\text{总维度数}} ≈ J(A, B)$$

---

## ✅ MinHash 的优点：

- **压缩表示**：可以把大集合表示成一个较短的签名向量。
    
- **快速比较**：比较两个集合时，不需要访问原始集合，只比较签名向量。
    
- **适合大规模去重与搜索**（常用于 LSH：局部敏感哈希）
    

---

## 📌 小结：

| 步骤              | 说明               |
| --------------- | ---------------- |
| 1. 分词/Shingling | 把数据变成集合（如单词集合）   |
| 2. 多个哈希函数       | 模拟“洗牌”操作，每次选择最小值 |
| 3. 生成签名向量       | 每个集合的 MinHash 签名 |
| 4. 签名向量比较       | 估计 Jaccard 相似度   |

---

# python实现
pip install datasketch scikit-learn

```python
from datasketch import MinHash, MinHashLSH
from sklearn.feature_extraction.text import CountVectorizer

# 示例文本（你可以替换为自己的文本列表）
documents = [
    "Today is a sunny day in New York.",
    "It is sunny in New York today.",
    "Rain is expected tomorrow in Boston.",
    "Tomorrow, it will rain in Boston.",
    "Completely different text that should not match."
]

# 初始化 LSH（局部敏感哈希）对象
# threshold: 两个文本签名之间相似度超过多少才算重复（范围：0~1）
# num_perm: 哈希函数数量，越多精度越高但计算越慢
lsh = MinHashLSH(threshold=0.8, num_perm=128)

# 存储每个文本对应的 MinHash 签名对象
minhashes = {}

# n-gram 分词函数：将文本拆成 n 个字符组成的小片段（shingles）
# n 设置为 3 表示使用三字符为单位，适合处理中英文短文本
def get_ngrams(text, n=3):
    vectorizer = CountVectorizer(analyzer='char', ngram_range=(n, n))
    analyzer = vectorizer.build_analyzer()
    return analyzer(text)

# 遍历每篇文本，生成它的 MinHash 签名，并放入 LSH 中
for i, doc in enumerate(documents):
    shingles = get_ngrams(doc)  # 提取 n-gram 特征
    m = MinHash(num_perm=128)   # 创建 MinHash 签名对象
    for s in shingles:
        m.update(s.encode('utf8'))  # 把每个特征添加进 MinHash（必须是字节）
    
    key = f"doc_{i}"              # 为每个文档起个唯一的键
    lsh.insert(key, m)            # 插入到 LSH 索引结构中
    minhashes[key] = m            # 本地保存，方便后面查询

# 查询重复文档对（可能重复）
print("可能重复的文档对：")
checked = set()  # 避免重复输出
for i in range(len(documents)):
    key = f"doc_{i}"
    result = lsh.query(minhashes[key])  # 查询与当前文档相似的其他文档
    for match in result:
        if match != key and (match, key) not in checked:
            print(f"{key} ≈ {match}")
            checked.add((key, match))

```
|参数|含义|
|`threshold=0.8`|相似度阈值（MinHash 签名相同的比例），高说明更严格|
|`num_perm=128`|MinHash 使用的哈希函数数量，常用值为 64/128/256，越高越精确|
|`ngram_range=(n, n)`|表示将文本切分为长度为 n 的子串。n=3 是常见默认值|
|`lsh.insert(key, minhash)`|将文档对应的签名插入到 LSH 索引中|
|`lsh.query(minhash)`|查询与当前签名相似的文档集合（近似搜索）|

# 追问
## 问题一：num_perm代表哈希函数数量，它这个哈希函数能有256种这么多吗，咋做的hash？
非常好的问题！你提到了 MinHash 中的核心参数之一：`num_perm`（哈希函数数量），确实这个细节值得深入了解。

---

## 🧠 问题回顾：

> `num_perm=256` 是说用了 256 个哈希函数，那这么多哈希函数是怎么来的？真的有 256 种不一样的哈希函数吗？怎么生成的？

---

## ✅ 简短回答：

MinHash 中并不是真的去手动写 256 个不同的哈希函数，而是用**一个基础哈希函数 + 多个参数变化**（比如线性组合）来**模拟出很多不同的哈希函数**。

---

## 🔍 更深入的解释：

### 🔧 MinHash 中哈希函数的数学形式：

通常构造一组伪随机哈希函数的方式如下（对整数元素）：

$$h_i(x)=(a_i⋅x+b_i) \mod p$$

其中：

- x：输入值（如 token 的哈希）
- $a_i, b_i$：随机生成的系数，对每个 i（第 i 个“哈希函数”）不同
- p：一个足够大的质数，通常大于最大输入值
- $h_i$：模拟出来的第 i 个哈希函数

虽然你只用一个实际的哈希函数（比如 Python 的 `hash()` 或 MurmurHash），但通过这种方式，结合不同的 $(a_i, b_i)$，你可以**模拟出任意数量的“独立哈希函数”**，理论上不限于 256 个。

---

## ✅ 总结：

| 问题                     | 回答                                      |
| ---------------------- | --------------------------------------- |
| `num_perm` 是几个真正的哈希函数？ | 不是“真正不同的函数”，而是通过随机参数构造的多个线性变换           |
| 这些函数怎么生成的？             | 通常是 $h_i(x)=(a_i⋅x+b_i) \mod p$，模拟哈希函数族 |
| 为什么能用这么多？              | 数学上这类函数族有良好的随机性和分布特性，可以安全使用成百上千个        |
| datasketch 库是怎么实现的？    | 使用 NumPy 向量化生成多个 `a, b`，用线性组合快速计算多个哈希结果 |
