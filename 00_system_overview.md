# Geo-Agent 系统技术综述

> 生成日期: 2026-05-05
> 代码文件总数: 12（含 __init__.py）
> 主要语言: Python 3.10+

## 1. 系统概述

Geo-Agent 是一套面向高速公路边坡风险评估的端到端自动化系统。系统以大语言模型（Kimi-K2.6）为核心推理引擎，结合 LangGraph 状态图编排、RAG 检索增强生成和确定性物理计算，实现从现场勘察数据（图片/文本）到标准 Word 评估报告的全流程自动化。

目标用户为公路运营管理单位的地质工程师。系统解决了传统人工评估模式下的三重瓶颈：单坡评估耗时长（3~5小时）、不同工程师评分一致性差、工程规范知识检索与对齐效率低。

## 2. 技术栈全景

| 层级 | 技术选型 | 用途 |
|------|---------|------|
| 语言 | Python 3.10+ | 主要开发语言 |
| 大模型 | Kimi-K2.6 (Pro/moonshotai) | 多模态推理（OCR、数据解析、图文交叉感知、专家推理） |
| 嵌入模型 | Qwen3-Embedding-8B | 文本向量化，用于 RAG 知识库 |
| 工作流引擎 | LangGraph (StateGraph) | 有向图编排，节点路由与状态传递 |
| LLM 框架 | LangChain (ChatOpenAI, ChatPromptTemplate) | Prompt 管理、输出解析、链式调用 |
| 向量数据库 | ChromaDB | 存储工程规范文本的语义向量 |
| 文本分割 | RecursiveCharacterTextSplitter | 规范文档切块（800字符/100重叠） |
| 前端 | Streamlit | Web UI（单次/批量评估、知识库管理） |
| 报告渲染 | docxtpl (Jinja2) + python-docx | Word 模板填充与图片插入 |
| 图片处理 | Pillow (PIL) | 图片格式清洗（RGB转换、Alpha剥离） |
| API 网关 | SiliconFlow (OpenAI 兼容) | 统一调用 Kimi-K2.6 和 Qwen3-Embedding |
| PDF 解析 | PyMuPDF (fitz) | 知识库文档文本提取 |
| 数据处理 | pandas, openpyxl | 辅助数据处理 |

## 3. 架构图

```
main_agent.py (Streamlit 前端)
├── 三个标签页：单次上传评估 / 批量目录扫描 / 知识库管理
├── load_vector_db() → ChromaDB retriever (k=3)
└── build_workflow(retriever) → LangGraph StateGraph

agent_core/ (核心编排层)
├── state_manager.py    — AgentState TypedDict (10字段状态定义)
├── router_workflow.py  — StateGraph 编译：路由→6节点→END
└── kb_manager.py       — 知识库入库流水线 (PDF/DOCX→ChromaDB)

tools/ (6个功能工具节点)
├── tool_1_vision_ocr.py      — 视觉OCR提取 (多模态LLM + Base64)
├── tool_2_data_parser.py     — 结构化解析 (System Prompt + JSON mode)
├── tool_2b_disease_vision.py — 图文交叉感知 (扣分项+全景图交叉验证)
├── tool_3_physics_calc.py    — 物理硬计算 (IS/CS/TS 确定性算法)
├── tool_4_expert_reason.py   — RAG专家推理 (ChromaDB检索 + CoT + Pydantic)
└── tool_5_report_writer.py   — 报告渲染 (docxtpl + Jinja2 + PIL)

config.py — 全局配置中心 (API_KEY, BASE_URL, MODEL_NAME)

knowledge_base/ — 知识库资源
├── chroma_db_qwen/         — Chroma 向量数据库持久化目录
├── *.txt                   — 指南文本分块
└── report_chunks/          — 历史勘察报告文本片段

templates/ — Word 报告模板 (土质/岩质/二元 三种)

data_io/ — 数据输入输出
├── temp_inputs/            — 临时上传文件落盘目录
└── output_reports/         — 生成的评估报告输出目录
```

## 4. 模块间数据流

```
输入（图片目录或文本）
    │
    ▼
route_input() 路由判断
    │
    ├── 图片目录/文件 → node_tool_1 (OCR)
    │       │
    │       │ ocr_text (Markdown格式文本)
    │       ▼
    └── 文本输入 ─────→ node_tool_2 (结构化解析)
                            │
                            │ parsed_data (Dict: basic_info + score_data)
                            ▼
                        node_tool_2b (图文交叉感知)
                            │
                            │ disease_description (约200字病害描述)
                            ▼
                        node_tool_3 (物理硬计算)
                            │
                            │ calculation_results (IS/CS/TS/Level)
                            ▼
                        node_tool_4 (RAG专家推理)
                            │  ← ChromaDB retriever (k=3规范条文)
                            │ reasoning_text (学术推理段落)
                            ▼
                        node_tool_5 (报告渲染)
                            │
                            │ report_path (.docx文件路径)
                            ▼
                          END
```

## 5. 关键设计决策

### 5.1 软硬分离原则
- **"硬"职能**（数学计算）全部由 Python 确定性实现：IS加权求和、CS查表插值、TS乘除、风险分级。LLM 不参与任何数值运算。
- **"软"职能**（语义理解与专业表述）由大语言模型承担：OCR识别、数据解析、病害描述、专家推理。
- **数据契约**：节点间通过 JSON/Markdown 格式的 AgentState 字段传递，不依赖 LLM 的输出格式。

### 5.2 两级容错机制（Tool 2b）
- 优先路径：多模态 LLM（Kimi-K2.6）同时读取扣分项文本和全景图，生成图文交叉验证的病害描述。
- 降级路径：若视觉模型调用失败，自动切换为纯文本 LLM（DeepSeek-V4-Flash），仅基于扣分项文本生成描述。
- 兜底：若文本模型也失败，直接返回原始扣分项列表。

### 5.3 文件名约定驱动的自动分流
- `image_` 前缀的图片被 Tool 1 跳过（视为全景图），被 Tool 2b 读取（用于图文交叉感知）。
- 非 `image_` 前缀的图片被 Tool 1 视为勘察表格，执行 OCR 提取。

### 5.4 依赖注入模式（Tool 4）
- ChromaDB retriever 由 main_agent.py 在启动时创建（单例缓存），通过 build_workflow(retriever=retriever) 注入到 Tool 4 的闭包节点中。
- Tool 4 内部不直接连接数据库，只接收 retriever 对象，实现了知识库与推理逻辑的解耦。

### 5.5 模板自动匹配（Tool 5）
- 按边坡类型（土质/岩质/二元）自动选择对应的 Word 模板。
- 模板使用 docxtpl（Jinja2 语法），所有复杂逻辑（条件判断、数据格式化、自然语言拼接）在 Python 侧预处理完成后，以扁平化的 context 字典传入模板。
- 模板内部仅做简单的占位符替换，不包含循环或条件分支。

## 6. 扩展点与限制

### 可扩展
- **模型替换**：config.py 中 MODEL_NAME 可直接切换为其他 OpenAI 兼容模型
- **知识库动态接入**：Streamlit 前端支持实时上传 PDF/DOCX 入库
- **模板扩展**：新增边坡类型只需添加对应的 Word 模板文件
- **工具节点插拔**：LangGraph 的节点注册机制支持新增或替换工具

### 硬限制
- **API 依赖**：所有 LLM 调用均通过 SiliconFlow API，离线环境无法运行
- **单图限制**：Tool 2b 目前仅取第一张全景图进行分析
- **模板静态**：模板占位符必须与 Python 侧 context 字典的键名严格一致
- **无持久化状态**：每次评估独立运行，无历史记录或增量更新能力

## 7. 文件索引

| 文件 | 类型 | 行数 | 说明文档 |
|------|------|------|---------|
| config.py | 配置 | 4 | [config_spec.md](config_spec.md) |
| main_agent.py | 入口 | ~180 | [main_agent_spec.md](main_agent_spec.md) |
| agent_core/__init__.py | 包 | 0 | - |
| agent_core/state_manager.py | 核心 | ~20 | [state_manager_spec.md](state_manager_spec.md) |
| agent_core/router_workflow.py | 核心 | ~120 | [router_workflow_spec.md](router_workflow_spec.md) |
| agent_core/kb_manager.py | 核心 | ~50 | [kb_manager_spec.md](kb_manager_spec.md) |
| tools/tool_1_vision_ocr.py | 工具 | ~60 | [tool_1_spec.md](tool_1_spec.md) |
| tools/tool_2_data_parser.py | 工具 | ~90 | [tool_2_spec.md](tool_2_spec.md) |
| tools/tool_2b_disease_vision.py | 工具 | ~100 | [tool_2b_spec.md](tool_2b_spec.md) |
| tools/tool_3_physics_calc.py | 工具 | ~110 | [tool_3_spec.md](tool_3_spec.md) |
| tools/tool_4_expert_reason.py | 工具 | ~140 | [tool_4_spec.md](tool_4_spec.md) |
| tools/tool_5_report_writer.py | 工具 | ~140 | [tool_5_spec.md](tool_5_spec.md) |
| requirements.txt | 配置 | 10 | - |
