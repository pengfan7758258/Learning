1. 微调：llamafactory-cli train examples/train_lora/llama3_lora_sft.yaml
2. 推理：llamafactory-cli chat examples/inference/llama3_lora_sft.yaml
3. 合并：llamafactory-cli export examples/merge_lora/llama3_lora_sft.yaml

examples文件夹有：
1. train_full：全参数sft yaml模版
2. train_lora：lora yaml模版
	- dpo、ppo、ds3、pretrain、reward
3. train_qlora：qlora yaml模版
	- gptq、awq、bnb、aqlm
4. inference：
	- full、lora、sglang、vllm
5. merge lora：合并lora和原始模型参数
	- full、lora、gptq

<mark style="background: #ADCCFFA6;">我们要做的是：</mark>
1. 找一个你要train的模式，复制一份它的yaml文件修改，譬如lora微调qwen2.5：
	- 红色框是我修改的：模型路径、我自己的数据集（data/data_info.json的key，也需要手动上传的数据集后面会讲）、输出的保存路径、template模版（这个根据自己模型的选择修改）
	- 其它超参数可以根据自己需求修改![[01-微调、推理、合并（命令版） 6.png]]
2. 自己的数据集：
	1. 根据选择微调的模型，参考data下的格式的demo.json文件，或者自己网上搜索模型的指令微调数据格式和字段，改好后放入data下
	2. 在data/data_info.json添加自己的数据集，具体添加哪些字段可以再搜索看看教程![[01-微调、推理、合并（命令版） 5.png]]
