Attention is all your need
- self-attention
- multi-head attention：特征融合的一个过程，一组qkv一组特征表达，类似CNN中的卷积核，每个核学习到的信息不一样，代码中是将最后一个维度大小除以头数n_head作为每个头的特征维度大小，最后的特征融合是一个拼接然后经过一个全连接网络

- 注意力计算的过程就是相关度计算的过程
- 注意力矩阵大小是\[seq,seq]
- decoder解码器在解码的时候，self-attention只计算q，计算的时候只考虑decoder端，不考虑encoder的word，kv由encoder编码器提供
- 多层堆叠，原始论文中是6层encoder，6层decoder

- position encoding:位置编码，采用正余弦的方式来模拟位置信息
- Layer Normalization：从layer层（特征维度，最后一个维度，word本身的embedding）进行归一化，方差为0，标准差为1，稳定数值，让训练更平稳
- ADD：残差网络，对input和当前output做一个对位相加，让模型的结果至少不会更差，在设计的时候考虑经过了一系列的操作，self-attention，全连接后你能保证效果一定好吗，不一定，所以拼接原始信息，至少不会更差
- masked-self-attention：decoder解码器里，只对当前词之前的词做self-attention，后面的mask遮蔽掉，不泄露信息，因为你本身就要预测后面的词，你还对它做self-attention那不就告诉他答案了。底层代码这部分就是一个list切片

self-attention的时间复杂度和空间复杂度：
- 空间复杂度分析主要针对运行时的动态存储需求，所以模型容量的一些参数不参与（W_q,W_k,W_v等）
- 假设输入的序列长度为n,特征维度是d,那么输入则是nxd
- 因为要保持最后的输入维度和输出维度一致，那么W_q,W_k,W_v的维度是dxd
- nxd与dxd做矩阵相乘，那么时间复杂度是nxdxd，也就是nd^2，计算得到的Q，K，V矩阵nxd，空间复杂度是3nd
- Q和K^T做矩阵乘法，时间复杂度为nxdxn，也就是dn^2，空间复杂度为n^2
- softmax是在注意力矩阵的维度上做的，也就是说空间复杂度也是n^2，时间复杂度是最后要与V矩阵nxd做一个特征融合的阶段，也就是nxn的矩阵与nxd的矩阵做矩阵相乘，得到的时间复杂度为nxnxd，也就是dn^2
- 最后也就是说，主要的时间复杂度体现在Q,K,V的计算，注意力矩阵的计算和最后的softmax加权求和、特征融合和的计算：nd^2+dn^2+dn^2，近似为n\^2d
- 空间复杂度体现在存储QKV，注意力分数矩阵和softmax结果：3nd+n^2+n^2，近似为n^2
![[24-transformer.png]]
![[24-transformer-1.png]]
![[24-transformer-2.png]]


