---
source_file: tools/tool_2b_disease_vision.py
language: Python
lines: ~100
last_updated: 2026-05-05
---

# tool_2b_disease_vision.py 技术说明书

## 1. 文件定位
图文交叉感知工具（Tool 2b）。将结构化评分数据中的扣分项与现场全景照片进行交叉验证，生成约200字的病害综合描述段落。位于Tool 2和Tool 3之间。

## 2. 技术栈
- **视觉模型**: Kimi-K2.6 (Pro/moonshotai/Kimi-K2.6)
- **降级模型**: DeepSeek-V4-Flash (deepseek-ai/DeepSeek-V4-Flash，纯文本)
- **图片处理**: PIL.Image（RGB转换、Alpha通道剥离）
- **API调用**: openai.OpenAI

## 3. 关键函数

### encode_image(image_path: str) -> str
与Tool 1不同，此处使用PIL清洗图片：
- 剥离Alpha通道或3D深度图通道（img.convert("RGB")）
- 强制以纯净JPEG格式写入内存缓冲区
- 目的：过滤MPO等非标文件头，避免API调用失败

### generate_disease_description(parsed_data, input_path) -> Dict[str, str]
核心工具函数，两级容错机制：

**第一级（优先路径）**:
1. 从score_data中筛选score>0的扣分项，排除"/""无""无破损"等无效值
2. 在input_path目录中查找`image_`前缀的全景照片
3. 组装多模态Prompt：扣分项文本 + 全景图 → Kimi-K2.6
4. Prompt要求：交叉验证文字与图片、客观专业、约200字

**第二级（降级路径）**:
- 触发条件：视觉模型调用失败（网络异常、模型不可用等）
- 切换为DeepSeek-V4-Flash纯文本模型
- 仅基于扣分项文本生成连贯描述

**第三级（兜底）**:
- 若文本模型也失败，直接返回原始扣分项列表文本

## 4. 数据流
```
parsed_data.score_data → 筛选扣分项 → 拼接deduction_context
                                            ↓
input_path目录 → 查找image_*全景图 → Base64编码(PIL清洗)
                                            ↓
              deduction_context + 全景图 → 多模态Prompt → Kimi-K2.6 → 病害描述
                                                                  ↓ (失败时)
                                          deduction_context → 纯文本Prompt → DeepSeek-V4-Flash → 降级描述
```

## 5. 注意事项
- 仅取第一张image_开头的全景图进行分析（单图限制）
- PIL清洗是必要的：工程现场照片可能包含非标EXIF数据或MPO格式
- 扣分项筛选逻辑中，score>0表示有扣分（即存在问题），score=0表示该项正常
- 降级时标注"（注：无清晰全景图，依据调查表生成）"以提示用户
