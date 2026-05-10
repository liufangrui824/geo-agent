---
source_file: tools/tool_1_vision_ocr.py
language: Python
lines: ~60
last_updated: 2026-05-05
---

# tool_1_vision_ocr.py 技术说明书

## 1. 文件定位
视觉OCR提取工具（Tool 1）。将勘察表格图片转换为Markdown格式文本，供Tool 2解析。位于流水线的最前端（图片输入路径）。

## 2. 技术栈
- **API调用**: openai.OpenAI (SiliconFlow 端点)
- **模型**: Kimi-K2.6 (Pro/moonshotai/Kimi-K2.6)
- **图片编码**: base64 标准库

## 3. 关键函数

### encode_image(image_path: str) -> str
将本地图片文件读取为二进制，Base64编码后返回字符串。

### extract_text_from_image(input_path: str) -> Dict[str, Any]
核心工具函数。输入为图片文件路径或包含图片的目录路径。

**处理流程**:
1. **图片收集**: 扫描目录中的 .png/.jpg/.jpeg 文件，过滤掉 `image_` 前缀的文件（这些是全景图，留给Tool 2b使用）
2. **逐张OCR**: 对每张图片，构造多模态消息（text + image_url），调用Kimi-K2.6
3. **结果拼接**: 多张图片的OCR结果以 `\n\n` 拼接为完整长文本

**Prompt设计**:
- 角色设定："专业的地质工程资料录入员"
- 任务："提取图片中的所有表格信息和文字，转换为Markdown格式的表格输出"
- temperature=0.1（抑制幻觉，保证识别准确性）

**返回**: `{"status": "success", "ocr_text": "..."}` 或 `{"error": "..."}`

## 4. 数据流
```
图片目录 → 过滤image_前缀 → Base64编码 → Kimi-K2.6多模态推理 → Markdown文本 → 写入state.ocr_text
```

## 5. 注意事项
- 多图场景下按文件名字典序排序后逐张处理，结果按序拼接
- `image_` 前缀约定是整个系统的核心分流机制：Tool 1跳过，Tool 2b读取
- temperature=0.1 是关键参数，高于此值可能导致OCR输出不稳定
