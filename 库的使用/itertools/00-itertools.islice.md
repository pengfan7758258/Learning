作用：迭代器切割
`itertools.islice(iterable, start, stop[, step])`
 可以返回从迭代器中的start位置到stop位置的元素。如果stop为None，则一直迭代到最后位置

三种情况：
1. `itertools.islice(iterable, stop)`：两个参数，第二个参数是停止位置
2. `itertools.islice(iterable, start, stop)`：三参数
3. `itertools.islice(iterable, start, stop, step)`：四参数

用法和list切片一样

主要应用：大文件读取及处理，不可能直接加载到内存中，因此进行分批次小量读取及处理
```python
from itertools import islice
def read_itertools(path):
    with open(path, 'r', encoding='utf-8') as fout:
        list_gen = islice(fout, 0, 5)  # 两个参数分别表示开始行和结束行
        for line in list_gen:
            print(line)
```
每次从fout文件对象读取五行数据，下一次读取不会从头开始读取，是接着读取的

