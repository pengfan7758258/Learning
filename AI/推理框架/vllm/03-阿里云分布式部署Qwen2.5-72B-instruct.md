# 租赁服务器
- 2台机器，每台机器两张l20卡的服务器
	- 保证有model size两倍还多的显存，这里2x2x48=192G的显存
		- 替代方案：8张24G显存的A10
	- 两台机器在同一个网段，阿里云中选同一区域即可。分布式对底层通信有要求，太慢的话影响推理效率。
	- 阿里云资源配置：
		- 系统磁盘给45G
		- 配置docker
		- ubuntu系统
		- 安装GPU驱动
		- 网络带宽按量：100M
- 1台没有GPU的服务器（2核4G）：下载model
	- 阿里云资源配置
		- 系统磁盘给150G
		- 网络带宽按量：100M
- 1台NAS：用来当做共享文件夹使用，专门放置model
	- 挂载到上面的三个机器的/mnt下


# 部署流程
1. 登录没有GPU的服务器
	- 安装python包：`python3 -m pip install modelscope --break-system-packages` 
	- vim编辑文件：`vim download_model.py`写入如下内容：
		```python
		#模型下载 
		from modelscope import snapshot_download
		model_dir = snapshot_download('Qwen/Qwen2.5-72B-Instruct', cache_dir="./")
		```
	- 将下载的model复制到/mnt下（rsync显示进度条，也可以用cp）：`rsync -a --progress ./Qwen /mnt/`
	- 最后呈现如下：![[03-阿里云分布式部署Qwen2.5-72B-instruct 6.png]]
2. 登录两台有GPU显卡的服务器：
	- 都pull 拉取镜像：`docker pull vllm/vllm-openai`
	- vim编辑：`vim run_cluster.sh`
		```bash
		#!/bin/bash
		#
		# Launch a Ray cluster inside Docker for vLLM inference.
		#
		# This script can start either a head node or a worker node, depending on the
		# --head or --worker flag provided as the third positional argument.
		#
		# Usage:
		# 1. Designate one machine as the head node and execute:
		#    bash run_cluster.sh \
		#         vllm/vllm-openai \
		#         <head_node_ip> \
		#         --head \
		#         /abs/path/to/huggingface/cache \
		#         -e VLLM_HOST_IP=<head_node_ip>
		#
		# 2. On every worker machine, execute:
		#    bash run_cluster.sh \
		#         vllm/vllm-openai \
		#         <head_node_ip> \
		#         --worker \
		#         /abs/path/to/huggingface/cache \
		#         -e VLLM_HOST_IP=<worker_node_ip>
		# 
		# Each worker requires a unique VLLM_HOST_IP value.
		# Keep each terminal session open. Closing a session stops the associated Ray
		# node and thereby shuts down the entire cluster.
		# Every machine must be reachable at the supplied IP address.
		#
		# The container is named "node-<random_suffix>". To open a shell inside
		# a container after launch, use:
		#       docker exec -it node-<random_suffix> /bin/bash
		#
		# Then, you can execute vLLM commands on the Ray cluster as if it were a
		# single machine, e.g. vllm serve ...
		#
		# To stop the container, use:
		#       docker stop node-<random_suffix>
		
		# Check for minimum number of required arguments.
		if [ $# -lt 4 ]; then
		    echo "Usage: $0 docker_image head_node_ip --head|--worker path_to_hf_home [additional_args...]"
		    exit 1
		fi
		
		# Extract the mandatory positional arguments and remove them from $@.
		DOCKER_IMAGE="$1"
		HEAD_NODE_ADDRESS="$2"
		NODE_TYPE="$3"  # Should be --head or --worker.
		PATH_TO_HF_HOME="$4"
		shift 4
		
		# Preserve any extra arguments so they can be forwarded to Docker.
		ADDITIONAL_ARGS=("$@")
		
		# Validate the NODE_TYPE argument.
		if [ "${NODE_TYPE}" != "--head" ] && [ "${NODE_TYPE}" != "--worker" ]; then
		    echo "Error: Node type must be --head or --worker"
		    exit 1
		fi
		
		# Generate a unique container name with random suffix.
		# Docker container names must be unique on each host.
		# The random suffix allows multiple Ray containers to run simultaneously on the same machine,
		# for example, on a multi-GPU machine.
		CONTAINER_NAME="node-${RANDOM}"
		
		# Define a cleanup routine that removes the container when the script exits.
		# This prevents orphaned containers from accumulating if the script is interrupted.
		# cleanup() {
		#    docker stop "${CONTAINER_NAME}"
		#    docker rm "${CONTAINER_NAME}"
		#}
		cleanup() {
		    echo "Container ${CONTAINER_NAME} left running in background."
		}
		trap cleanup EXIT
		
		# Build the Ray start command based on the node role.
		# The head node manages the cluster and accepts connections on port 6379, 
		# while workers connect to the head's address.
		RAY_START_CMD="ray start --block"
		if [ "${NODE_TYPE}" == "--head" ]; then
		    RAY_START_CMD+=" --head --port=6379"
		else
		    RAY_START_CMD+=" --address=${HEAD_NODE_ADDRESS}:6379"
		fi
		
		# Launch the container with the assembled parameters.
		# --network host: Allows Ray nodes to communicate directly via host networking
		# --shm-size 10.24g: Increases shared memory
		# --gpus all: Gives container access to all GPUs on the host
		# -v HF_HOME: Mounts HuggingFace cache to avoid re-downloading models
		docker run \
		    --entrypoint /bin/bash \
		    --network host \
		    --name "${CONTAINER_NAME}" \
		    --shm-size 10.24g \
		    --gpus all \
		    -v "${PATH_TO_HF_HOME}:/root/.cache/huggingface" \
		    -d \
		    "${ADDITIONAL_ARGS[@]}" \
		    "${DOCKER_IMAGE}" -c "${RAY_START_CMD}"

		```
	- 启动容器
		- 一台服务器当做head node：`bash run_cluster.sh vllm/vllm-openai 172.18.190.25 --head /mnt/Qwen -v /mnt/Qwen:/models`
			- 这里的ip是自己的etn0![[03-阿里云分布式部署Qwen2.5-72B-instruct.png]]
			- 这里的-v映射路径就是在没有GPU的机器下载模型的NAS共享文件夹路径
		- 另一台当做worker node：`bash run_cluster.sh vllm/vllm-openai 172.18.190.25 --worker /mnt/Qwen -v /mnt/Qwen:/models`
			- 这里ip也是head node的ip
		- 启动容器时可能报错![[03-阿里云分布式部署Qwen2.5-72B-instruct 1.png]]
			- 解决地址：https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html![[03-阿里云分布式部署Qwen2.5-72B-instruct 2.png]]
	- 进入head node container：`docker exec -it 容器id /bin/bash`
		- 进入容器后运行:`ray status`![[03-阿里云分布式部署Qwen2.5-72B-instruct 3.png]]
			- 4个gpu说明成功
		- 启动vllm服务:`vllm serve /models/Qwen2___5-72B-Instruct --served-model-name qwen2.5-72b-instruct --tensor-parallel-size 2 --pipeline_parallel_size 2`![[03-阿里云分布式部署Qwen2.5-72B-instruct 4.png]]
	- 阿里云安全组开放8000端口，使用下列脚本测试端口：
		```python
		from openai import OpenAI
		def openai_api():
			client = OpenAI(
				base_url="http://ip:8000/v1", # 服务地址
				api_key="test-key",
			)
			
			response = client.chat.completions.create(
				model="qwen2.5-72b-instruct", # 这里的名字要和你 vllm 启动时加载的模型一致
				messages=[
					{"role": "system", "content": "you are a assistant."},
					{"role": "user", "content": "你是谁!"},
				],
			)
			
			print(response.choices[0].message.content)
		
		openai_api()
		```
		![[03-阿里云分布式部署Qwen2.5-72B-instruct 5.png]]


# 参考
[vllm官网：Parallelism and Scaling](https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html)
[b站博主分布式部署实操](https://www.bilibili.com/video/BV1J5bJz2E9L/?spm_id_from=333.337.search-card.all.click&vd_source=e4234f5ddbe45b813cf4296e06e14b9b)