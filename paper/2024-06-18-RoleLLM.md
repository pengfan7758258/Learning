a framework to benchmark, elicit, and enhance role-playing abilities in LLMs
（一个用于基准测试、引出和增强LLM角色扮演能力的框架）

## 原始论文
![[RoleLLM.pdf#height=400]]

## 目的
获得高质量的instruct角色扮演的数据集

## 四个阶段
![[RoleLLM.png]]
1. 构建100个角色的角色档案（Role Profile）
	- 95个英文角色和5个中文角色构建细粒度的角色档案，从916个英文剧本和24个中文剧本中选取
	- 角色选择的模板如下：
	- ![[RoleLLM 1.png]]
2. 基于上下文的指令生成（Context-Instruct）用于提取角色特定知识
	- 使用GPT从分割的档案中生成高质量的问答对，以提取特定角色的知识
		- 分割角色档案
			- 分成两块，第一块(a.Description and Catchphrases)角色描述和口头禅，第二块(b.Structured Dialogues)结构化对话
				- (a.Description and Catchphrases)角色描述和口头禅，使用gpt-4来生成审查，prompt模板如下：
					1. **使用GPT-4生成角色描述**：首先，研究者使用经过审核的GPT-4模型来生成角色的描述。这些描述包括角色的性格特征、生活经历、主要故事情节、重要事件等。生成的描述应该是简洁的，不包含角色的名字，并且以第三人称的角度来写。
					2. **生成第二人称描述**：将第一步生成的第三人称描述转换为第二人称，以便更适合作为系统指令的一部分。这一步通过将描述中的"he"或"she"替换为"you"，并以"Your description is:"开头。
					3. **生成口头禅**：对于角色的口头禅，如果角色在剧本或其他相关资料中有著名的口头禅，直接提取这些表达。如果没有，可以通过分析角色的语言风格创造性地生成一些符合角色特性的口头禅。
					4. **审核和调整**：由研究者审核GPT-4生成的描述和口头禅，确保它们的准确性和符合角色特性。必要时，研究者可以对生成的内容进行调整或手动添加额外的细节。
					5. ![[RoleLLM 2.png]]
			- 生成后的结果案例，如下图：
			- ![[RoleLLM 3.png]]
		- 生成（问题-置信度-答案） 三元组候选项
			- 使用LLM为每个角色和片段生成这些三元组
			- 提示模板包括角色描述、口头禅、少样本示例和说话风格模仿及三元组生成的任务指令。生成过程每个角色至少产生400个候选项，并通过多次模型运行进行生成。template如下（有基于剧本的和无剧本的）:
				- 无剧本：
				- ![[RoleLLM 4.png]]
				- 有剧本：
				- ![[RoleLLM 5.png]]
		- 过滤和后处理低质量数据
			- 根据上面的置信度直接过滤
			- TODO
3. 使用RoleGPT以模仿说话风格
	- 通过基于对话工程（dialogue engineering）的角色提示（role prompting），利用系统指令（system instruction）和检索增强（retrieval augmentation），激发GPT的角色扮演能力，生成模仿说话风格的响应（response）
		- 使用经过审查的GPT-4生成角色描述和口头禅，作为自定义指令的核心（即系统指令 system instruction）
			- 审查的GPT-4：向GPT-4提出一些关于每个角色的基本问题，由人工来验证，确保GPT-4对这个角色非常了解
		- 包含一个整体的角色扮演任务指令如 “Please speak like \[role_name]” ，并使用BM25在角色档案中检索前5个相关对话对作为少样本示例填入一下模板，RoleGPT的回应可以捕捉角色的说话风格，并包含一些角色特定知识
		- ![[RoleLLM 6.png]]
		- ![[RoleLLM 8.png]]
4. 通过角色条件指令调整RoCIT（Role-Conditioned Instruction Tuning ）对开源模型进行微调和角色定制
	- 通过使用168,093个由Context-Instruct和RoleGPT生成的角色扮演样本，在RoleBench上对开源的LLaMA和ChatGLM2进行上下文高效的角色调优，得到RoleLLaMA和RoleGLM
		- the chat markup language for RoleLLaMA is `“### Instruction:\n{**system instruction**}</s>\n\n### Input:\n{**user input**}</s>\n\n### Response:\n{**model response**}</s>”`，仅监督响应model response的特殊标记，在推理过程中，用户可以通过system instruction轻松修改大型语言模型的角色，实现不同角色的使用

通过2（Context-Instruct），3（RoleGPT）步骤，创建了RoleBench数据集，这是第一个系统且细粒度的角色扮演基准数据集，包含168,093个样本。根据论文描述，RoleBench是第一个用于细粒度角色扮演的系统指令调优数据集和基准。

## RoleBench数据集构建
1. 角色选择：从多个来源（包括NLP电影剧本、SummScreen和手动策划的中文剧本）中精心选择了100个具有代表性和独特性的角色
	- 95个英文角色和5个中文角色构建细粒度的角色档案，从916个英文剧本和24个中文剧本中选取
2. 角色档案构建：角色档案由GPT-4生成的角色描述和口头禅组成，并经过作者验证，同时从剧本中解析出结构化对话
3. 通用指令采样
	- 从多个数据集中随机抽取了1,500个英文一般指令（English general instructions），1,479个中文一般指令（Chinese general instructions）
	- 所有采样指令包含不超过100个单词
	- 基于BM25相似度去重
4. 原始RoleBench数据生成：使用RoleGPT为每个一般指令获取多个响应，并使用Context-Instruct生成角色特定的问题-回答对
5. RoleBench数据集的清理：原始数据集经过全面清理，以确保响应的完整性、AI和角色身份的隐藏以及不被拒绝
	- TODO

## 设计原则
1. **说话风格模仿**
	- **词汇一致性**：模型的回应应包含角色常用的口头禅或惯用表达
	- **对话忠实度**：模型生成的回应不仅要在上下文中适当，还要在风格上与角色的示例对话相似
2. **角色特定知识和记忆注入**
	- **基于剧本的知识**：包括剧本中记录的明确细节，如详细的角色背景、情景记忆以及角色经历的具体事件。例如，当扮演钢铁侠时，一个大型语言模型应该包含基于剧本的知识（如托尼·斯塔克在洞穴中被俘时创造的第一套钢铁侠战衣）
	- **与剧本无关的知识**：包括角色可能拥有的一般知识或专业知识。例如，作为钢铁侠，一个大型语言模型应该包含与成为企业家相关的知识（如商业头脑、领导素质和技术专业知识）

## GPT-4/RoleGPT参数设置
GPT4：temperature（0.7）、top-p（0.95）、max_generate_tokens（2000）、frequency（0）、penalties（0）
RoleGPT：temperature（0.7）、top-p（0.95）、max_generate_tokens（200）、frequency（0）、penalties（0）

## 评估方式
**三个Rouge-L指标（Lin，2004）、GPT和人工评估**来评估模型在说话风格模仿、回答准确性和角色特定知识捕捉方面的表现

## Role-playing
Role-playing aims to enable or customize LLMs to simulate various characters or personas with distinct attributes and conversational styles, which provides a more nuanced interaction experience for users, and renders LLMs more familiar and companionable

（角色扮演旨在使LLMs能够模拟**具有独特属性和对话风格的各种角色或人格**，为用户提供更细致的互动体验，并使LLMs更加亲切和友好）

## GPT-4 role-playing缺点
GPT-4（OpenAI，2023）LLM展示了高级的角色扮演能力，但其闭源性质带来了诸多限制，包括高昂的API费用、无法进行微调和有限的上下文窗口大小。

## 如何自己构建？

自己如何构建一个类似于Rolebench这样的数据集，每个角色具有自己鲜活的特色和口头禅

### 收集书籍
1. 在网上搜索、下载或爬取书籍
2. 购买纸质书籍进行扫描上传

### 角色选择
1. 手动选择你熟悉的角色
2. 使用prompt让GPT-4帮你来选择具有人物特色的角色 -- 可以让gpt4列出所有角色，然后你再让gpt4对这些角色全部生成一段描述，你从这些角色中挑选符合自己需求的角色
**角色选择模板：（这是论文里的模板，效果不好可以修改）**
![[RoleLLM 9.png]]


### 构建角色画像 - system instuction
1. 熟悉书籍里的人物，可以手动去写一段关于某个人物的描述和口头禅，这些描述包括角色的性格特征、生活经历、主要故事情节、重要事件等
2. 如果不熟悉，可以使用“Context Instruct”下的“Description and Catchphrases“来生成角色描述和口头禅，细节如下：
    1. **使用GPT-4生成角色描述**：首先，研究者使用经过审核的GPT-4模型来生成角色的描述。这些描述包括角色的性格特征、生活经历、主要故事情节、重要事件等。生成的描述应该是简洁的，不包含角色的名字，并且以第三人称的角度来写。
        - 审核的GPT-4模型：向GPT-4询问关于角色的基本问题。让人类注释者进行验证，以确保GPT-4很了解这个角色
            - 这一步可以忽略，因为这里的人类注释者首先得知道这个角色，知道这个角色的一些事情才能验证
            - TODO：如果能有更加智能的验证方式，这一步可以加上确保角色的真实性
    2. **生成第二人称描述**：将第一步生成的第三人称描述转换为第二人称，以便更适合作为系统指令的一部分。这一步通过将描述中的"he"或"she"替换为"you"，并以"Your description is:"开头。
    3. **生成口头禅**：对于角色的口头禅，如果角色在剧本或其他相关资料中有著名的口头禅，直接提取这些表达。如果没有，可以通过分析角色的语言风格创造性地生成一些符合角色特性的口头禅。
    4. **审核和调整**：由研究者审核GPT-4生成的描述和口头禅，确保它们的准确性和符合角色特性。必要时，研究者可以对生成的内容进行调整或手动添加额外的细节。

### 书籍分块
1. 如果模型能接受输入整本书籍则不需要分块 -- GPT-4
2. LLM有接受最大上下文tokens限制的话，则需要将很长的书籍分块，分块的规则论文中很麻烦，个人看法是如果是小说直接按照章节来分，其它书籍结构再具体分析

### 生成问答对
从书籍中提取和组织相关角色问答内容，使其成为可用于模型训练和评估的结构化数据
一.下面是论文给出的两种提取role-specific instruct的方式
不过在我们自己提取的时候gpt4不一定知道你的角色和角色所处的内容是什么样，需要修改prompt，统一基于内容提取问答对
1. 无剧本 - （问题，置信度，答案）生成模板 -- role-specific instruct
![[RoleLLM 10.png]]
![[RoleLLM 12.png]]
2. 有剧本 - （问题，置信度，答案）生成模板 -- role-specific instruct
![[RoleLLM 13.png]]
![[RoleLLM 14.png]]
二.general instructs 一般指令生成
调用rolegpt的api，设计prompt让角色保持原有风格来回答指令问题
![[RoleLLM 15.png]]
![[RoleLLM 16.png]]

### 遗留问题
1. 所有的构建方式都建立在gpt4在训练的时候已经融入了相关剧本角色的知识，如果gpt4不认识那么这段就需要重新设计
    - 解决方式一：将整本书籍喂入到gpt4当中让gpt4去生成你想要的内容 -- 需要设计prompt
2. 是否要加入一批instrcut，让模型禁止说自己是AI