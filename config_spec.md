---
source_file: config.py
language: Python
lines: 4
last_updated: 2026-05-05
---

# config.py 技术说明书

## 1. 文件定位
全局配置中心。所有需要调用 SiliconFlow API 的模块（tool_1~tool_5、kb_manager、main_agent）均从此文件导入 API_KEY、BASE_URL 和 MODEL_NAME。

## 2. 技术栈
- **标准库**: 无
- **第三方依赖**: 无（纯常量定义）

## 3. 核心数据结构
- `API_KEY` (str): SiliconFlow API 密钥
- `BASE_URL` (str): API 端点地址 `https://api.siliconflow.cn/v1`
- `MODEL_NAME` (str): 大语言模型标识 `Pro/moonshotai/Kimi-K2.6`

## 4. 注意事项
- API_KEY 硬编码在源码中，生产环境应改为环境变量加载
- MODEL_NAME 被 Tool 1、Tool 2、Tool 2b、Tool 4 共用；Tool 2b 额外定义了 VISION_MODEL_NAME（同为 Kimi-K2.6）
- kb_manager.py 和 tool_4.py 中的 Embedding 模型名（Qwen3-Embedding-8B）未在此文件统一管理，存在分散风险
