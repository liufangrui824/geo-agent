---
source_file: agent_core/router_workflow.py
language: Python
lines: ~120
last_updated: 2026-05-05
---

# router_workflow.py 技术说明书

## 1. 文件定位
LangGraph 状态图的核心编排文件。定义了6个节点函数、1个路由函数和1个图构建函数。是系统的"中枢神经"。

## 2. 技术栈
- **语言特性**: 类型注解 (Literal), 闭包（node_tool_4_reason_with_retriever 捕获 retriever）
- **第三方依赖**: langgraph.graph (StateGraph, END)

## 3. 关键函数

### route_input(state) -> Literal["node_tool_1", "node_tool_2"]
路由入口。判断 raw_input_path 是目录/图片文件还是文本：
- 目录或 .png/.jpg/.jpeg → node_tool_1 (OCR路径)
- 其他 → node_tool_2 (直接解析路径)

### node_tool_1_ocr(state) -> dict
调用 Tool 1 的 extract_text_from_image()，将 ocr_text 写入 state。

### node_tool_2_parser(state) -> dict
调用 Tool 2 的 parse_slope_text_to_dict()。输入优先读 ocr_text（OCR路径），否则读 raw_input_path（文本路径）。

### node_tool_2b_vision(state) -> dict
调用 Tool 2b 的 generate_disease_description()，传入 parsed_data 和 raw_input_path。

### node_tool_3_calc(state) -> dict
调用 Tool 3 的 calculate_slope_ts()，传入 parsed_data。

### node_tool_4_reason_with_retriever(state) -> dict
闭包节点，捕获外部传入的 retriever。调用 Tool 4 的 generate_expert_reasoning()。

### node_tool_5_report(state) -> dict
调用 Tool 5 的 generate_word_report()，聚合所有上游数据。

### build_workflow(retriever=None) -> CompiledStateGraph
构建并编译工作流图。

## 4. 数据流（图结构）

```
入口 → route_input
  ├── 图片 → node_tool_1 → node_tool_2 → node_tool_2b → node_tool_3 → node_tool_4 → node_tool_5 → END
  └── 文本 → node_tool_2 → node_tool_2b → node_tool_3 → node_tool_4 → node_tool_5 → END
```

**边的定义**:
- node_tool_1 → node_tool_2（OCR结果传给解析器）
- node_tool_2 → node_tool_2b（解析结果传给图文感知）
- node_tool_2b → node_tool_3（感知结果传给硬计算）
- node_tool_3 → node_tool_4（计算结果传给推理）
- node_tool_4 → node_tool_5（推理结果传给报告渲染）

## 5. 注意事项
- Tool 4 使用闭包模式注入 retriever，而非直接在节点内创建数据库连接
- 图是线性的（无分支/循环），唯一的条件逻辑在入口路由
- 使用 app.stream() 执行（流式模式），每个节点完成后立即返回状态更新
