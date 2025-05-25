# 完整example（verl）
```Dockerfile
# Start from the official NVIDIA CUDA image: https://hub.docker.com/r/nvidia/cuda/tags
# Use the CUDA 12.4.0 base image with Ubuntu 22.04
# This image includes the CUDA toolkit and development libraries
FROM nvidia/cuda:12.4.0-devel-ubuntu22.04

# Install system dependencies
RUN apt-get update && apt-get install -y \
    python3.10 python3.10-venv python3.10-dev python3-pip \
    build-essential curl git git-lfs vim \
    && ln -s /usr/bin/python3.10 /usr/bin/python \
    && git lfs install \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
RUN python -m pip install --upgrade pip && \
    pip install packaging && \
    pip install modelscope[framework] && \
    pip install torch==2.6.0 --index-url https://download.pytorch.org/whl/cu124 && \
    pip install flash-attn --no-build-isolation

# Install verl
WORKDIR /workspace
RUN git clone https://github.com/volcengine/verl.git
WORKDIR /workspace/verl
RUN pip install -e .[vllm]
WORKDIR /workspace

CMD ["bash"]
```



#  流程拆解（带有注释）
1. <mark style="background: #FFB86CA6;">First</mark>: 选一个带有nvidia环境的ubuntu：[地址](https://hub.docker.com/r/nvidia/cuda/tags)，可以选择`12.4.0`较稳定
	- 12.9.0-base-ubuntu24.04：最基础的 CUDA 镜像，只包含 CUDA 运行时和驱动相关依赖
	- 12.9.0-runtime-ubuntu24.04：比 base 多了完整的 CUDA **运行环境**，适合只用 GPU 运行程序，不做开发
	- 12.9.0-devel-ubuntu24.04：开发环境镜像，用于<mark style="background: #FF5582A6;">训练模型</mark>、编译 CUDA/C++ 代码
	- 12.9.0-cudnn-runtime-ubuntu24.04：用于模型推理，如 TensorFlow、PyTorch 的运行环境
	- 12.9.0-cudnn-devel-ubuntu24.04：<mark style="background: #FF5582A6;">训练 + 推理通吃</mark>，用于深度学习开发全流程
```Dockerfile
# Start from the official NVIDIA CUDA image: https://hub.docker.com/r/nvidia/cuda/tags
# Use the CUDA 12.4.0 base image with Ubuntu 22.04
# This image includes the CUDA toolkit and development libraries
FROM nvidia/cuda:12.4.0-devel-ubuntu22.04
# 12.4.0是CUDA Toolkit版本
```
2. <mark style="background: #FFB86CA6;">Second</mark>: 安装系统环境，python环境与其它基本使用工具
```Dockerfile
# Install system dependencies
RUN apt-get update && apt-get install -y \
    python3.10 python3.10-venv python3.10-dev python3-pip \
    build-essential curl git git-lfs vim \
    && ln -s /usr/bin/python3.10 /usr/bin/python \
    && git lfs install \
    && rm -rf /var/lib/apt/lists/*
# apt-get update：同步软件包索引，确保获取的是最新版本
# apt-get install -y: 安装后续列出的所有软件包，`-y` 表示自动确认安装（不交互）
# python3.10-dev: 包含 Python 头文件和静态库，用于编译依赖 C 扩展的包（如uvloop, numpy）
# python3-pip: Python 的包管理工具 pip
# build-essential：包含 gcc, g++, make 等构建工具，用于编译 C/C++ 程序
# ln -s /usr/bin/python3.10 /usr/bin/python：创建一个符号链接，把 `python3.10` 映射为 python，这样用户在命令行中输入 python 时会调用 Python3.10
# rm -rf /var/lib/apt/lists/*：删除 apt 的缓存文件，减少最终镜像大小（这是构建 Docker 镜像的最佳实践）删除
```
3. <mark style="background: #FFB86CA6;">Third</mark>：安装python依赖，这部分主要是把指定cuda版本的torch安装好
```Dockerfile
# Install Python dependencies
RUN python -m pip install --upgrade pip && \
    pip install torch==2.6.0 --index-url https://download.pytorch.org/whl/cu124 && \
    pip install flash-attn --no-build-isolation
# --no-build-isolation：别用临时隔离环境，就直接用我当前环境里的依赖来构建这个包
```
4. <mark style="background: #FFB86CA6;">Four</mark>：准备开发目录、具体项目的开发环境
```Dockerfile
# example：Install verl
WORKDIR /workspace
RUN git clone https://github.com/volcengine/verl.git
WORKDIR /workspace/verl
RUN pip install -e .[vllm]

# 关键字`WORKDIR`：设置 工作目录；后续所有的命令（如 `RUN`, `COPY`, `CMD`, `ENTRYPOINT` 等）都将在这个目录下执行
```
5. <mark style="background: #FFB86CA6;">Five</mark>：开启一个程序或多个
```Dockerfile
# 方式一：什么都不写
CMD ["bash"]

# 方式二：开启一个fastapi服务
CMD ["python", "app.py"]

# 方式三：写一个sh文件，依次执行，可以执行多个文件
CMD ["bash", "example.sh"]
```



