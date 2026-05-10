---
source_file: tools/tool_2_data_parser.py
language: Python
lines: ~90
last_updated: 2026-05-05
---

# tool_2_data_parser.py 技术说明书

## 1. 文件定位
结构化解析工具（Tool 2）。将OCR输出的Markdown文本或直接输入的文本解析为结构化的JSON字典（basic_info + score_data）。是连接"感知层"与"计算层"的关键桥梁。

## 2. 技术栈
- **API调用**: openai.OpenAI (SiliconFlow 端点)
- **模型**: Kimi-K2.6
- **输出约束**: response_format={"type": "json_object"}（强制JSON输出）
- **数据解析**: json.loads()

## 3. 核心数据结构

### SYSTEM_PROMPT（约1500字的强约束Prompt）
包含三部分：
1. **强制对照字典**: 18对中英文字段映射（如"边坡类型"→"slope_type"），确保LLM不会自造字段名
2. **score_data提取规则**: 每个评分项必须包含item/weight/score/result四个键，覆盖A-V全部指标
3. **输出格式约束**: 纯JSON，禁止```json标记

### 输出JSON结构
```json
{
  "basic_info": {
    "route_name": "...",
    "slope_code": "SS80K46R336",
    "slope_type": "土质/岩质/二元",
    "height_m": 41.3,
    "length_m": 138.0,
    ...（共18个字段）
  },
  "score_data": [
    {"item": "A1", "weight": 100, "score": 40, "result": "现场描述..."},
    ...（覆盖A-V所有指标）
  ]
}
```

## 4. 关键函数

### parse_slope_text_to_dict(raw_content: str) -> Dict[str, Any]
1. 输入校验（非空、字符串类型）
2. 构造消息：system=SYSTEM_PROMPT, user=raw_content
3. 调用Kimi-K2.6，强制JSON输出模式
4. json.loads()解析返回结果
5. 异常处理：JSON解析失败或API调用异常返回含error键的字典

## 5. 数据流
```
OCR文本/直接文本 → SYSTEM_PROMPT约束 → Kimi-K2.6(JSON mode) → json.loads → {basic_info, score_data} → 写入state.parsed_data
```

## 6. 注意事项
- temperature=0.1 保证输出格式稳定
- JSON mode 是关键：强制模型输出合法JSON，但仍需try/except处理边界情况
- "强制对照字典"是本工具的核心设计：通过Prompt Engineering约束LLM的输出字段，避免自造键名导致下游Tool 3计算失败
- basic_info中slope_type严格限定为"土质""岩质""二元"三个值，与Tool 3的权重矩阵和Tool 5的模板选择直接关联
