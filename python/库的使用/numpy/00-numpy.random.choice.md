1. 指定每个元素被选中的概率：（可以被用来对softmax后的值进行采样）
```python
choice = np.random.choice(arr, size=3, p=[0.1, 0.2, 0.3, 0.2, 0.2])
print(choice)  # 按照指定的概率输出3个元素
```