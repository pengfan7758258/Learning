- 企业级提示词的一个特征：通过冗余来确保关键指令被遵循。在真实的商业环境中，AI 的一个微小偏差都可能导致客户体验的显著差异。
- LLM 有时会为了满足输出格式要求而「一本正经地胡说八道」，即产生幻觉。因此，必须为 LLM 提供一个「逃生出口」(escape hatch)。**明确告知「我不知道」**：需要告诉 LLM，如果信息不足以做出判断，就不要臆造，而是停下来询问。


# 层次化内容生成
剧本生成：Plot Summary(故事概要) → Act Summaries(动作摘要) -> Scenes Summaries(场景摘要) -> Script Dialog(脚本对话框)
