- [知乎讲解](https://zhuanlan.zhihu.com/p/1888676890828583054)
- 安装：`python3 -m pip install -U yt-dlp`
- demo:
```python
# 下载 - 不需要认证
yt-dlp "https://www.youtube.com/watch?v=jWQx2f-CErU"
# 下载 - 需要认证
# google浏览器安装Cookie-Editor -> 导出Netscape的cookies.txt文件
yt-dlp "https://x.com/jingjishasha/status/1933927198781796574/video/1" --cookies cookies.txt 
```
