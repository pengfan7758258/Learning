[b站地址](https://www.bilibili.com/video/BV1ErPkeSEHn/?spm_id_from=333.337.search-card.all.click&vd_source=e4234f5ddbe45b813cf4296e06e14b9b)
能支持更长的context上下文推理（输入长度）

![[05-ROPE（旋转位置编码）.png]]
对两个向量做旋转并不影响两个向量之间的相关性，而在做旋转的时候加入了位置信息在里面

![[05-ROPE（旋转位置编码）-1.png]]
![[05-ROPE（旋转位置编码）-2.png]]
![[05-ROPE（旋转位置编码）-3.png]]
1. 原始embedding两两之间配对，得到n组的(1,2)的向量，n是向量维度的一半（因为两两配对）
2. 配对后的向量根据$m\theta$做旋转，m是位置，位置从1开始，$\theta$根据当前i和维度d计算得来
3. 旋转后的n组向量拼接恢复原始维度
4. 真实在做的时候是直接乘了一个旋转矩阵
5. 对q和k做的rope编码，原因：主要是想在计算注意力机制也就是词与词之间的相关性的时候考虑到位置因素