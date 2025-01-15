## 主函数模式 -- 一般不用
1. 运行所有：`pytest.main()`
2. 运行模块：`pytest.main(["-vs", "test_register.py"])`
3. 运行目录：`pytest.main(["-vs", "./web_testcase"])`
4. 运行nodeId：nodeId是`函数名::类名::方法名` --> `pytest.main(["-vs", "./web_testcase/test_login.py::TestLogin::test_01_login"])`

## 命令行模式 -- 调试使用
1. 运行所有：`pytest`
2. 运行模块：`pytest -vs ./testcase/test_register.py`
3. 运行目录：`pytest -vs web_testcase`
4. 运行nodeId：`pytest -vs ./web_testcase/test_login.py::TestLogin::test_01_login

## 参数详解
- -v：输出调试信息，print打印的信息
- -s：输出具体的用例信息，哪个文件的哪个类的哪个方法
- -vs：合并使用
- -n：需要安装`pip install pytest-xdist`，多线程运行 --> `pytest -vs ./testcase -n 2`
- --reruns：需要安装`pip install pytest-rerunfailures`，失败用例重跑 --> `pytest -vs ./testcase --reruns 2`
- -x：只要有一个用例报错，测试停止
- --maxfail：出现最大次数失败用例则停止 --> `pytest -vs ./testcase --maxfail 2`
- -k：根据测试用例的名字部分匹配指定测试用例，例如测试用例包含"ao" --> `pytest -vs ./testcase -k "ao"`
- -html：需要安装`pip install pytest-html`。生成html的测试报告 --> `pytest --html ./report.html`

## pytest.init配置文件运行 -- 实际使用
1. pytest.init是pytest单元测试框架的核心配置文件
	- 位置：项目根目录
	- 作用：改变pytest的默认规则（在介绍里的默认规则，文件名必须以test开头等等）
	- 运行规则：不管是主函数运行模式还是命令行运行模式，都会读取这个配置文件
2. 配置事例：
```ini
[pytest]  
# 可添加多个命令行参数，用空格分隔  
addopts = -vs  
# 测试用例文件夹，可自己配置，…/pytestproject为上一层的pytestproject文件夹。  
testpaths = ./testcase  
# 配置测试搜索的模块文件名称  
python_files = test_*.py  
# 配置测试搜索的测试类名  
python_classes = Test*  
# 配置测试搜索的测试函数名,以某个字符串开头  
python_functions = test

# 如果出现gbk报错，参考博客https://blog.csdn.net/m0_56130892/article/details/136166210
```