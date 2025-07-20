文本转换语音

# 运行中出现的问题和解决
1. 安装环境requirements出现各种问题，查找到的原因是Python版本太高，一开始使用的3.12，后面创建了3.9的python环境解决
2. pip install soundfile PySoundFile 解决下面的问题![[29-ChatTTS.png]]
3. 在ChatTTS/norm.py中特殊token因为”\[“和”\]“被替换无法使用，在代码中注释掉![[29-ChatTTS 1.png]]


# 运行demo几个有趣的点

1. 几个有用的参数![[29-ChatTTS 2.png]]
2. 支持的几种特殊token![[29-ChatTTS 3.png]]
	- 简单的实现逻辑
		- 先把text重新生成具有特殊token的text -- 添加了一些停顿、语气
		- 再把新的text生成音频
