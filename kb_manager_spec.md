---
source_file: agent_core/kb_manager.py
language: Python
lines: ~50
last_updated: 2026-05-05
---

# kb_manager.py 技术说明书

## 1. 文件定位
知识库管理模块。提供文档解析→文本切块→向量化→入库的完整流水线。由 main_agent.py 的"知识库管理"标签页调用。

## 2. 技术栈
- **PDF解析**: PyMuPDF (fitz)
- **DOCX解析**: python-docx
- **文本分割**: langchain_text_splitters.RecursiveCharacterTextSplitter
- **向量化**: langchain_openai.OpenAIEmbeddings (Qwen3-Embedding-8B)
- **向量数据库**: langchain_chroma.Chroma

## 3. 关键函数

### extract_text_from_file(file_path: str) -> str
根据文件后缀自动选择解析引擎：
- .pdf → PyMuPDF 逐页提取纯文本
- .docx/.doc → python-docx 逐段落提取

### ingest_document_to_chroma(file_path: str, chroma_db_path: str) -> (bool, str)
核心入库流水线，三步：
1. **文本抽取**: 调用 extract_text_from_file()
2. **文本切块**: RecursiveCharacterTextSplitter，chunk_size=800, chunk_overlap=100，分隔符优先级：段落→句子→句号→叹号→问号→分号→逗号→空格
3. **向量化入库**: OpenAIEmbeddings (Qwen3-Embedding-8B) → ChromaDB，每块附带 source 元数据

## 4. 外部依赖
- SiliconFlow API（Embedding 模型调用）
- 本地 ChromaDB 持久化目录（knowledge_base/chroma_db_qwen/）

## 5. 注意事项
- 与 main_agent.py 中的 load_vector_db() 共用同一 Embedding 模型和同一 ChromaDB 路径
- 入库操作是追加式的（add_texts），不会清除已有数据
- 中文分隔符（句号、叹号等）确保中文文档的切块不会在句子中间截断
