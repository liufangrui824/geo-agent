---
source_file: tools/tool_5_report_writer.py
language: Python
lines: ~140
last_updated: 2026-05-05
---

# tool_5_report_writer.py 技术说明书

## 1. 文件定位
报告渲染工具（Tool 5）。流水线的终端节点。聚合上游所有数据（parsed_data、calc_result、reasoning_text、disease_description），匹配边坡类型模板，渲染并输出标准Word评估报告。

## 2. 技术栈
- **模板引擎**: docxtpl (DocxTemplate, Jinja2语法)
- **Word操作**: docx.shared.Mm（图片宽度控制）
- **图片处理**: PIL.Image（RGB转换、Alpha剥离、JPEG清洗）
- **图片嵌入**: docxtpl.InlineImage
- **文件操作**: os, re, io

## 3. 关键函数

### generate_word_report(...) -> Dict[str, Any]
核心渲染函数，5个逻辑块：

**逻辑1：输出路径管理**
- 接受output_dir参数，None时默认"data_io/output_reports"
- 文件命名：{文件夹名}_{slope_code}_评估报告.docx

**逻辑2：Context字典构建**
从上游数据构建扁平化的context字典，供模板占位符使用：
- basic_info直接展开为context字段
- 自然语言拼接：protection_desc（防护+加固描述）、disease_desc（排水描述）
- score_data展平：每个评分项展开为{item}_weight、{item}_result、{item}_score三个context键
- 破坏历史提取：从B3/B4项的result字段中正则匹配括号内容
- 融合计算结果：IS/CS/TS/Level
- 融合推理结果：reasoning（推理段落）、suggested_action（按等级分类的对策建议）
- 注入病害描述：disease_description

**逻辑3：模板匹配**
- 三种模板：各边坡风险评估报告样例（土质/岩质/二元）.docx
- 匹配逻辑：slope_type中包含"二元""土质""岩质"关键字则选对应模板

**逻辑4：图片处理与插入**
- 扫描input_dir中`image_`前缀的图片文件
- PIL清洗：convert("RGB") → save as JPEG → BytesIO
- InlineImage嵌入：width=140mm
- 预留image_1到image_10共10个占位符

**逻辑5：渲染与落盘**
- doc.render(context)执行Jinja2模板填充
- doc.save()写入文件

## 4. 数据流
```
parsed_data + calc_result + reasoning_text + disease_description + raw_input_path
    ↓
构建context字典（扁平化、自然语言拼接、展平评分项）
    ↓
匹配Word模板（土质/岩质/二元）
    ↓
扫描image_*图片 → PIL清洗 → InlineImage嵌入
    ↓
docxtpl渲染 → .docx文件落盘 → report_path写入state
```

## 5. 注意事项
- 模板内部仅做简单的占位符替换（{{variable}}），不包含循环或条件分支
- 所有复杂逻辑（条件判断、数据格式化、自然语言拼接）在Python侧完成
- PIL的LOAD_TRUNCATED_IMAGES=True强制加载损坏/截断的图片，防止现场照片格式异常导致中断
- suggested_action按等级分两类：I/II类→"日常巡查"，III/IV类→"专业维修加固"
- 破坏历史的正则匹配：r'[（(](.*?)[)）]'提取括号中的具体内容
