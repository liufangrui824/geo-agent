# Geo-Agent: 基于大语言模型的高速公路边坡风险自动评估系统

## 简介
基于 LLM + LangGraph + RAG 技术构建的边坡风险智能评估 Agent，
实现从现场勘察数据到风险评估报告的自动化全流程生成。

## 系统架构
（插入 workflow_diagram.png）

## 环境要求
- Python 3.10+
- SiliconFlow API Key（用于调用 Kimi-K2.6）

## 快速开始
1. 克隆仓库
2. 安装依赖：pip install -r requirements.txt
3. 配置 API Key：cp .env.example .env（然后编辑 .env）
4. 启动：streamlit run main_agent.py

## 工具模块说明
| 工具 | 功能 |
|------|------|
| Tool 1 | 视觉 OCR 提取 |
| Tool 2 | 数据结构化解析 |
| Tool 2b | 图文交叉病害感知 |
| Tool 3 | 物理力学硬计算（IS/CS/TS） |
| Tool 4 | RAG 专家软推理 |
| Tool 5 | Word 报告渲染 |

## 参考标准
《湖南省高风险边坡评估技术指南（修订稿）》HNGSYH 001-2019

## 论文
本系统为本科毕业论文「基于大语言模型的高速公路边坡安全风险
自动评估方法研究」的核心成果。
