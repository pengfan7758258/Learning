# 配置
显卡：一张4090
模型：Hunyuan-0.5B-Instruct

# nginx安装
1. `apt install nginx`
2. 修改nginx.conf文件的http{...}：`vim /etc/nginx/nginx.conf`
	```
	# 输出格式
	log_format upstreamlog 'TESTLOG $remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" '
                        'upstream: $upstream_addr '
                        'request_time: $request_time '
                        'upstream_time: $upstream_response_time';
	# 日志添加输出格式
    access_log /var/log/nginx/access.log upstreamlog;
	```
3. 添加一个新的nginx的config文件，vllm.conf：`vim /etc/nginx/vllm.conf`
	```
	upstream backend {
	    least_conn;
	    server 127.0.0.1:8001 max_fails=3 fail_timeout=30s;
	    server 127.0.0.1:8002 max_fails=3 fail_timeout=30s;
	    server 127.0.0.1:8003 max_fails=3 fail_timeout=30s;
	}
	server {
	    listen 6006;
	
	    location / {
	        proxy_pass http://backend;
	        proxy_set_header Host $host;
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	        proxy_set_header X-Forwarded-Proto $scheme;
	    }
	}
	```
	1. upsteam backend：定义上游服务组 backend
		- 包含三个后端服务：127.0.0.1:8001、127.0.0.1:8002、127.0.0.1:8003
		- 负载均衡策略：least_conn，会将请求分配给当前活跃连接数最少的后端。比默认round-robin（轮询）更适合长请求（例如LLM推理）
		- max_fails=3、fail_timeout=30s：某个后端在 30 秒内连续失败 3 次，Nginx 就会把它标记为“不可用”，30 秒后再尝试恢复。避免请求重复打到坏的请求。
	2. server：
		- listen 6006：本机6006端口监听。外部访问`http://ip:6006`时触发
		- location的proxy_pass：所有6006上的端口路径都会代理到upstream backend上
		- proxy_set_header：给转发的请求加HTTP头，方便后端识别真实客户端
			- `Host $host`：保留原始请求的 Host 头
			- `X-Real-IP $remote_addr`：客户端真实 IP
			- `X-Forwarded-For $proxy_add_x_forwarded_for`：代理链上的所有客户端 IP
			- `X-Forwarded-Proto $scheme`：记录客户端使用的协议（http/https）
4. 重启nginx：`pkill -9 nginx && systemctl start nginx`

# 启动多个vllm服务

- nohup vllm serve Hunyuan-0___5B-Instruct --served-model-name hunyuan-0.5B-instruct --port 8001 --max-model-len 4096 --gpu-memory-utilization 0.3 > vllm0.log 2>&1 &
- nohup vllm serve Hunyuan-0___5B-Instruct --served-model-name hunyuan-0.5B-instruct --port 8002 --max-model-len 4096 --gpu-memory-utilization 0.3 > vllm1.log 2>&1 &	    
- nohup vllm serve Hunyuan-0___5B-Instruct --served-model-name hunyuan-0.5B-instruct --port 8003 --max-model-len 4096 --gpu-memory-utilization 0.3 > vllm2.log 2>&1 &

多个同时启动有问题的话，可以一个一个启动

# 日志观察
- nginx日志：`tail -f /var/log/nginx/access.log`
- vllm0日志：`tail -f vllm0.log`
- vllm1日志：`tail -f vllm1.log`
- vllm2日志：`tail -f vllm2.log`

# 并发测试：
并发少，用的线程池的方式
运行下面的程序，观察nginx的access.log请求分发，观察vllm0.log、vllm1.log、vllm2.log日志情况
```python
def openai_api():
		client = OpenAI(
		base_url="https://ip:port/v1", # 服务地址
	)

	response = client.chat.completions.create(
		model="hunyuan-0.5B-instruct", # 这里的名字要和你 vllm 启动时加载的模型一致
		messages=[
			{"role": "system", "content": "you are a assistant."},
			{"role": "user", "content": "/no_think给我讲讲小红帽的故事"},
		],
	)
	
	# print(response.choices[0].message.content)
	
	return response.choices[0].message.content


def run_concurrent(n: int):
	results = []
	with ThreadPoolExecutor(max_workers=n) as executor:
		futures = [executor.submit(openai_api) for _ in range(n)]
		for future in as_completed(futures):
			try:
				results.append(future.result())
			except Exception as e:
				results.append(f"Error: {e}")
	return results

concurrent_num = 16
outputs = run_concurrent(concurrent_num)
print(outputs)
```