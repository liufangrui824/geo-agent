# Geo-Agent: 基于大语言模型的高速公路边坡风险自动评估系统

基于 LLM + LangGraph + RAG 技术构建的边坡风险智能评估 Agent，实现从现场勘察数据到风险评估报告的自动化全流程生成。

## 系统架构

```
输入 → 路由判断 → [Tool 1: 视觉 OCR] → [Tool 2: 数据解析]
                 → [Tool 2b: 图文交叉感知] → [Tool 3: 物理计算]
                 → [Tool 4: RAG 专家推理] → [Tool 5: 报告渲染] → 输出
```

## 工具模块

| 工具 | 功能 | 输入 | 输出 |
|------|------|------|------|
| Tool 1 | 视觉 OCR 提取 | 勘察表格照片 | 结构化 OCR 文本 |
| Tool 2 | 数据解析 | OCR 文本 / JSON | 结构化边坡参数字典 |
| Tool 2b | 图文交叉感知 | 数据字典 + 全景照片 | 病害综合描述 |
| Tool 3 | 物理硬计算 | 边坡参数字典 | IS / CS / TS 分数 + 风险等级 |
| Tool 4 | RAG 专家推理 | 数据 + 计算结果 + ChromaDB | 专业评估推理文本 |
| Tool 5 | 报告渲染 | 全部上游输出 | 标准 Word 评估报告 |

## 环境要求

- Python 3.10+
- SiliconFlow API Key（用于调用 Kimi-K2.6 大模型）

## 快速开始

```bash
# 1. 克隆仓库
git clone https://github.com/your-username/geo-agent-slope.git
cd geo-agent-slope

# 2. 安装依赖
pip install -r requirements.txt

# 3. 配置 API Key
cp .env.example .env
# 编辑 .env，填入你的 SiliconFlow API Key

# 4. 启动 Streamlit 前端
streamlit run main_agent.py
```

## 配置说明

系统通过环境变量读取 API 配置，需在 `.env` 文件或系统环境变量中设置：

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `SILICONFLOW_API_KEY` | SiliconFlow API 密钥 | （必填） |
| `SILICONFLOW_BASE_URL` | API 地址 | `https://api.siliconflow.cn/v1` |
| `SILICONFLOW_MODEL` | 模型名称 | `Pro/moonshotai/Kimi-K2.6` |

## 技术栈

- **大模型**: Kimi-K2.6（通过 SiliconFlow API）
- **Embedding**: Qwen3-Embedding-8B
- **工作流引擎**: LangGraph（StateGraph 有向图）
- **向量数据库**: ChromaDB
- **前端**: Streamlit
- **报告生成**: docxtpl（Jinja2 模板 + Word 渲染）

## 评估标准

本系统依据《湖南省高风险边坡评估技术指南（修订稿）》（HNGSYH 001-2019）进行边坡风险评估，支持土质边坡、岩质边坡和二元边坡三种类型。

### 风险分级

| 边坡类型 | I 类(正常) | II 类(监控) | III 类(加固) | IV 类(紧急) |
|----------|-----------|------------|-------------|------------|
| 土质     | TS < 40   | 40≤TS<50   | 50≤TS<60    | TS ≥ 60    |
| 岩质     | TS < 50   | 50≤TS<65   | 65≤TS<80    | TS ≥ 80    |
| 二元     | TS < 40   | 40≤TS<55   | 55≤TS<70    | TS ≥ 70    |

## 项目背景

本系统为本科毕业论文「基于大语言模型的高速公路边坡安全风险自动评估方法研究」的核心成果，已累计生成 55+ 份标准化评估报告。

## License

MIT
