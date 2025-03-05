通常用于输入密码等敏感信息
它的主要特点是不会在屏幕上显示用户输入的内容，避免密码被他人看到

```python
import getpass

pwd = getpass.getpass("请输入密码:") # 输入时不会显示在交互终端
print("你输入的密码是:", pwd) # 打印输入的密码
```