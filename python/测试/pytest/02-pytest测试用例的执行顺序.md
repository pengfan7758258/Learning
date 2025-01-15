### 默认方式
从上到下

### 指定顺序
1. 安装`pip install pytest-ordering`
2. 对用户方法进行\@pytest.mark.run(order=1)装饰，被装饰的方法先运行，按照order从小到大依次先运行，未被装饰的方法按照默认从上至下的方式后运行