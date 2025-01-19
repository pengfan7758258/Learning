作用：迭代器切割
`itertools.islice(iterable, start, stop[, step])`
 可以返回从迭代器中的start位置到stop位置的元素。如果stop为None，则一直迭代到最后位置

三种情况：
1. `itertools.islice(iterable, stop)`：两个参数，第二个参数是停止位置
2. `itertools.islice(iterable, start, stop)`：三参数
3. `itertools.islice(iterable, start, stop, step)`：四参数

总结来说和list切片一样


