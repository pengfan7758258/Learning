![[19-KV cache.png]]

从图中看出将已经计算的attention值不重复计算，把这部分存储起来，只计算新推理出来的token与前面的token产生的新的attention值

步骤分解：
![[19-KV cache-1.png]]