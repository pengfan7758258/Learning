1. 无条件跳过：使用装饰器装饰要跳过的方法\@pytest.mark.skip(reason="跳过原因")
2. 有条件跳过：使用装饰器装饰要跳过的方法\@pytest.mark.skipif(age>18, reason="跳过原因") -- age变量大于18则跳过