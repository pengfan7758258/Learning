1. 安装
	- `pip install allure-pytest`
	- 安装allure
		- https://github.com/allure-framework/allure2/releases
		- 下载.zip格式
		- 解压到一个路径下
		- 添加环境变量 -- 例如：D:\install\allure-2.32.0\bin
	- 验证安装是否成功：`allure --version`
		- 这里如果出现java问题，安装jdk
		- 有时候安装后还是不能再pycharm中查看版本号，重启pycharm
2. 生成json临时报告：`pytest --alluredir ./temp`
3. 生成allure报告：`allure generate ./temp -o ./report --clean`
4. 在./report下打开html可以查看allure生成的测试报告详细信息