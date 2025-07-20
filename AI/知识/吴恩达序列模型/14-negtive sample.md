一定程度解决了softmax分母计算量过大

1. 选取一个正确的context和word当做正例，target设置为1
2. 选择同样的context，然后从字典里随机抽取k个词作为word组成k个负例，target设置为0（即使正好出现在了正确的context与word的组合也不要紧，概率很小）
3. k的选择，小数据5-20；大数据集2-5
4. 构建算法，将context和word作为input，target作为output
5. 每次训练迭代k+1个逻辑回归模型而不去计算最后的softmax
![[14-negtive sample.png]]
![[14-negtive sample-1.png]]
![[14-negtive sample-2.png]]