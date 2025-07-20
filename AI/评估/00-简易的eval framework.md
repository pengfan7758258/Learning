```python
# 智能体工作流程评估  
def evaluate_agent_workflow(agent, test_scenarios):  
    results = []  
    for scenario in test_scenarios:  
        # 运行智能体  
        output = agent.run(scenario["input"])  
  
        # 检查关键步骤  
        step_results = {  
            "正确理解任务": check_task_understanding(output),  
            "调用了正确工具": check_tool_usage(output),  
            "给出合理答案": check_answer_quality(output)  
        }  
        results.append(step_results)  
  
    return analyze_results(results)
```