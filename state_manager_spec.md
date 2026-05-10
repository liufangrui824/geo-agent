---
source_file: agent_core/state_manager.py
language: Python
lines: 20
last_updated: 2026-05-05
---

# state_manager.py 技术说明书

## 1. 文件定位
定义 LangGraph 状态图的核心数据结构 AgentState。所有节点函数的输入和输出都基于此 TypedDict。

## 2. 技术栈
- **语言特性**: Python typing 模块的 TypedDict、Optional
- **标准库**: typing

## 3. 核心数据结构

### AgentState(TypedDict)
智能体内存状态定义，10个字段：

| 字段名 | 类型 | 写入节点 | 读取节点 | 说明 |
|--------|------|---------|---------|------|
| slope_id | str | 初始 | 全部 | 边坡标识符 |
| raw_input_path | str | 初始 | T1, T2b, T5 | 输入路径（图片目录或文本文件） |
| output_dir | Optional[str] | 初始 | T5 | 报告输出目录，None时用默认路径 |
| ocr_text | Optional[str] | T1 | T2 | OCR提取的Markdown文本 |
| parsed_data | Optional[Dict] | T2 | T2b, T3, T4, T5 | 结构化边坡参数（basic_info + score_data） |
| disease_description | Optional[str] | T2b | T5 | 图文交叉感知的病害描述文本 |
| calculation_results | Optional[Dict] | T3 | T4, T5 | 硬计算结果（IS/CS/TS/Level） |
| reasoning_text | Optional[str] | T4 | T5 | RAG专家推理的学术段落 |
| report_path | Optional[str] | T5 | END | 生成的Word报告文件路径 |
| errors | List[str] | - | - | 错误信息累积列表（当前未使用） |

## 4. 数据流
每个节点从 state 读取上游字段，计算后将结果写入新字段，下游节点读取。LangGraph 的 StateGraph 机制自动合并节点返回的 dict 到全局 state。
