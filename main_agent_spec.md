---
source_file: main_agent.py
language: Python
lines: ~180
last_updated: 2026-05-05
---

# main_agent.py 技术说明书

## 1. 文件定位
Streamlit前端主入口。提供Web界面，包含三个功能标签页：单次上传评估、批量目录扫描、知识库管理。是用户与Geo-Agent系统的唯一交互界面。

## 2. 技术栈
- **前端框架**: Streamlit (st.tabs, st.file_uploader, st.metric, st.spinner等)
- **向量数据库**: langchain_chroma.Chroma + langchain_openai.OpenAIEmbeddings
- **缓存机制**: @st.cache_resource（全局单例ChromaDB retriever）
- **工作流引擎**: agent_core.router_workflow.build_workflow

## 3. 核心数据结构

### StreamlitRedirect类
自定义的stdout重定向类，将print输出实时渲染到Streamlit的code块中，实现工作流执行过程的可视化日志。

## 4. 关键函数

### load_vector_db() -> Retriever (cached)
全局单例，利用@st.cache_resource保证Embedding模型和ChromaDB仅实例化一次。
- Embedding模型：Qwen3-Embedding-8B
- 数据库路径：knowledge_base/chroma_db_qwen/
- 检索参数：k=3（返回最相关的3条规范条文）

### init_ui()
主界面函数，构建三个标签页：

**标签页1：单次上传评估**
- file_uploader支持多文件上传（png/jpg/jpeg/txt）
- 创建时间戳临时目录（data_io/temp_inputs/run_{timestamp}）
- 将上传文件落盘到临时目录
- 构建initial_state，启动build_workflow(retriever)
- 使用app.stream()流式执行，每个节点完成后实时显示：
  - 图文交叉感知结果（st.info）
  - TS评分（st.metric）
  - 报告路径（st.success）

**标签页2：批量目录扫描**
- 输入本地根目录绝对路径
- 自动扫描子文件夹，逐个执行评估
- 每个子文件夹作为独立的评估任务

**标签页3：知识库管理**
- file_uploader支持PDF/DOCX上传
- 调用kb_manager.ingest_document_to_chroma()入库
- 入库后立即生效（ChromaDB持久化）

## 5. 数据流
```
用户上传文件 → 落盘到临时目录 → initial_state = {raw_input_path, output_dir}
    ↓
build_workflow(retriever) → app.stream(initial_state)
    ↓
逐节点执行并实时渲染日志/指标/结果到Streamlit界面
```

## 6. 注意事项
- @st.cache_resource确保ChromaDB全局单例，避免每次交互重新加载Embedding模型
- StreamlitRedirect劫持sys.stdout，执行完毕后必须finally恢复原始stdout
- 批量模式下每个文件夹独立执行一个完整的workflow，无并行处理
- 临时目录data_io/temp_inputs/不会自动清理（可能存在磁盘占用问题）
