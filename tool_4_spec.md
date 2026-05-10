---
source_file: tools/tool_4_expert_reason.py
language: Python
lines: ~140
last_updated: 2026-05-05
---

# tool_4_expert_reason.py 技术说明书

## 1. 文件定位
RAG专家软推理工具（Tool 4）。接收结构化数据和硬计算结果，结合ChromaDB检索到的规范条文，生成符合工程勘察文风的学术推理段落。是系统中唯一同时使用RAG和结构化输出的节点。

## 2. 技术栈
- **LLM框架**: langchain_openai.ChatOpenAI
- **Prompt管理**: langchain_core.prompts.ChatPromptTemplate
- **输出解析**: langchain_core.output_parsers.JsonOutputParser
- **数据校验**: pydantic.BaseModel (StabilityAssessment)
- **模型**: Kimi-K2.6, temperature=0.1, max_tokens=1500
- **输出约束**: response_format=json_object

## 3. 核心数据结构

### StabilityAssessment (Pydantic模型)
强制LLM输出以下7个字段：
- `slope_id`: 边坡编号
- `thinking_process`: 思维链推理过程（强制CoT）
- `slope_type`: 边坡类型
- `stability_level`: 风险等级
- `key_factors`: 主要影响因素列表
- `reasoning`: 学术推理段落（核心输出）
- `suggested_action`: 防控对策

### SLOPE_CLASSIFICATION_RULES
三种边坡类型的分级逻辑字符串，注入System Prompt。

## 4. 关键函数

### generate_expert_reasoning(parsed_data, calc_result, retriever=None) -> Dict
核心推理函数，依赖注入模式：

**数据准备**:
1. 从parsed_data提取basic_info和score_data
2. 筛选A/B类扣分项（score>0），拼接disease_features
3. slope_type标准化（确保含"边坡"二字）

**RAG检索**:
- 构造查询："{slope_type}高风险评估，TS评分为{ts_value}，实测特征：{disease_features}"
- 调用retriever.invoke(query)获取k=3条最相关规范条文
- 若retriever为None或检索失败，降级为"未检索到额外条文"

**Prompt设计**:
- System Prompt：角色设定（"Geo-Agent科研搭档"+"湖南省顶尖地质勘察工程师"）+ 4条强制约束
  1. 坚守TS评价体系，严禁提及安全系数Fs
  2. I类风险最低，数字越大风险越高
  3. 辩证评估，禁止无脑"未见明显病害"
  4. 工程勘察文风，起手式固定为"按照《指南》..."
- Human Template：RAG上下文 + 多源输入数据 + 格式指令

**链式调用**: prompt | llm | parser

## 5. 数据流
```
parsed_data + calc_result → 提取disease_features
                              ↓
retriever.invoke(query) → RAG规范条文（k=3）
                              ↓
System Prompt + Human Template + RAG上下文 → ChatOpenAI → JsonOutputParser → StabilityAssessment
                              ↓
{thinking_process, reasoning, key_factors, suggested_action} → 写入state.reasoning_text
```

## 6. 注意事项
- 使用LangChain的ChatPromptTemplate而非直接拼接字符串，支持变量注入和模板复用
- JsonOutputParser(pydantic_object=StabilityAssessment)强制输出符合Pydantic模型的JSON
- thinking_process字段实现了隐式Chain-of-Thought：模型先梳理逻辑再给出结论
- retriever通过闭包注入（router_workflow.py中的node_tool_4_reason_with_retriever），不在本文件内创建数据库连接
