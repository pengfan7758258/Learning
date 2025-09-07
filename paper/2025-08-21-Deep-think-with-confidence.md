
**DeepConf**:自信的深度思考

# 目的
一种简单且高效的方法提高推理效率和性能

# 原始论文
![[Deep-think-with-confidence.pdf#height=400]]

# DeefConf伪代码
这个伪代码看起来是online thinking的
```python
After generating next token x: 
	# Store token confidence 存储token置信度
	conf_list.append(compute_conf(x.logprob)) 
	# Calculate group confidence 计算最近group_size个token的平均置信度
	gc = average(conf_list[x-group_size:x]) 
	# threshold is set by offline warmup 
	# 设置threshold，这个值是通过`离线实验`（offline warmup）调出来的
	If gc >= threshold: 
		continue generation 
	else: 
		stop generation
```
模型一边写答案，一边心里打分：
- 每个token生成后，它都会问自己，这一次生成token靠不靠谱
- 取最近的group_size个token
- 如果平均分还不错，就一直生成token
- 如果平均分越来越低，低于threshold，它就会察觉自己开始瞎编了，立马停止。


# offline thinking and online thinking

## offline thinking
![[2025-08-21-Deep-think-with-confidence 11.png]]
![[2025-08-21-Deep-think-with-confidence 16.png]]
1. 输入P，设定过滤百分比η
2. 生成N条traces，每条trace有自己的C（Average Trace Confidence）
3. 使用η过滤低C的trace，统计计算V(a)（Confidence-Weighted Majority Voting）
4. 拿到最高投票answer

# online thinking
![[2025-08-21-Deep-think-with-confidence 12.png]]
提出两种基于最低组置信度（lowest group confidence）的算法：DeepConf-low and DeepConf-high，在线推理时提前停止生成，它们依赖于两个组件，offline warmup and adaptive sampling
![[2025-08-21-Deep-think-with-confidence 17.png]]
1. 输入P
2. offline warmup：初始化$N_{init}$个traces，计算每个trace的C（Average Trace Confidence）
3. 计算停止阈值s，从集合$\{C_0,C_1,...,C_{init-1}\}$取出在η百分比阶段的C当做s
4. 初始化一个T，这个T已经生成了k个traces。用k个traces得到所有的答案的投票数，和投票数最多的$\hat{a}$
5. 在线推理：两层while循环
	- 每个P最多生成B个traces，设置是否继续生成trace的阈值$\tau$。
	- 是否生成当前trace的下一个token取决于当前token生成后的group confidence与阈值s
	- 当前trace停止生成后，将这个trace加入到T中，计算并更新每个答案的投票数和投票数最多的$\hat{a}$DeepConf-low

# 公式
# Token Entropy
![[2025-08-21-Deep-think-with-confidence 1.png]]
i：第i个token（Given a language model’s predicted token distribution Pi at position i）
j：vocabulary第j个token概率（where $P_i(j)$ represents the probability of the j-th vocabulary token）
# Token Confidence
![[2025-08-21-Deep-think-with-confidence 2.png]]
$C_i$：当前token的token confidence
k：取top k的概率
chatgpt的QA：
![[2025-08-21-Deep-think-with-confidence.png]]
# Average Trace Confidence
![[2025-08-21-Deep-think-with-confidence 3.png]]
$C_{avg}$：token级别的metric，评估整个推理轨迹
N：生成的所有token个数
# Group Confidence
![[2025-08-21-Deep-think-with-confidence 4.png]]
$G_i$：包括当前生成的token一共n个token的token confidence C（group $G_i$ consisting of n previous tokens (e.g., n = 1024 or 2048) with overlapping adjacent windows.）
# Bottom 10% Group Confidence
![[2025-08-21-Deep-think-with-confidence 5.png]]
$G_b$：一条trace经过overlapping adjacent windows计算出一个C的集合，排序后，取百分之10垫底的C的平均值作为这条trace的confidence
# Bottom-10% Conf@512
对每条trace，取它内部置信度最低的 10% 窗口的平均值作为权重，再在 512 条轨迹上做加权投票。
![[2025-08-21-Deep-think-with-confidence 18.png]]
如果是最弱的环节也是靠谱的，那它的权重自然就大，那投票也就更准确。
# Lowest Group Confidence
![[2025-08-21-Deep-think-with-confidence 6.png]]
$C_{least}$：一条trace经过overlapping adjacent windows计算出一个C的集合，取最小为当前trace的C
# Tail Confidence
![[2025-08-21-Deep-think-with-confidence 7.png]]
$C_{tail}$：尾部置信度，固定尾部T个token（where $T_{tail}$ represents a fixed number of tokens (e.g., 2048).）
# Majority Voting
![[2025-08-21-Deep-think-with-confidence 8.png]]
![[2025-08-21-Deep-think-with-confidence 9.png]]
多数投票$V(a)$：对于每个answer统计票数
final answer $\hat{a}$：选取票数最多的answer作为最终答案
# Confidence-Weighted Majority Voting
![[2025-08-21-Deep-think-with-confidence 10.png]]
$C_t$：上面提到的针对整个推理轨迹的confidence - Average Trace Confidence
# Confidence Filtering
置信度过滤：选择top-$\eta$ 百分比的高置信度trace，确保最终答案更可靠
提供两种$\eta$: η = 10% and η = 90%
# Offline Warmup
![[2025-08-21-Deep-think-with-confidence 13.png]]
计算停止阈值s
初始化$N_{init}$个完整的reasoning trace（For each new prompt, we generate Ninit reasoning traces (e.g., Ninit = 16).）
s：停止阈值
$T_{warmup}$：表示所有warmup trace
$C_t$：trace 的置信度
{$C_t$}：$N_{init}$个置信度集合
$\eta$：百分比，DeepConf-low用 top η = 10%，那就是100-10=90%，对应于取第90百分位数作为s；DeepConf-high用top η = 90%，那就是100-90=10%，对应于取第10百分位数作为s
chatgpt的QA：
![[2025-08-21-Deep-think-with-confidence 14.png]]
# Adaptive Sampling
![[2025-08-21-Deep-think-with-confidence 15.png]]
基于question的难度生成多少traces
$\tau$：预设的共识阈值，论文中通常设定为0.95.
$\hat{a}$：majority answer（当前最多轨迹（加权后）支持的答案）
$V(\hat{a})$：Confidence-Weighted Majority Voting
$\sum_{a} V(a)$：所有的 Confidence-Weighted Majority Voting累和
B：固定的一个trace数量，达到这个数量后不再生成新的trace
operate：如果$\tau>\beta$   ，会再生成新的trace，直到生成固定大小B个trace。相反如果$\tau<\beta$，停止生成trace，在已有traces中分析answer
