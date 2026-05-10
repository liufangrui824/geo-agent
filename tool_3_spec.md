---
source_file: tools/tool_3_physics_calc.py
language: Python
lines: ~110
last_updated: 2026-05-05
---

# tool_3_physics_calc.py 技术说明书

## 1. 文件定位
物理规则硬计算工具（Tool 3）。基于《湖南省高风险边坡评估技术指南》的IS/CS/TS评分体系，执行确定性数学计算。是整条流水线中唯一不调用LLM的节点，所有计算由Python确定性实现。

## 2. 技术栈
- **标准库**: re（正则提取数值）
- **无第三方依赖**: 纯Python数学运算

## 3. 核心数据结构

### WEIGHTS（权重矩阵字典）
按边坡类型存储6套权重：
- `soil`: 12项（A1-A5, B1-B4, C1-C3）
- `binary`: 14项（A1-A4, B1-B3, C1-C7）
- `rock_collapse`: 12项（崩塌型）
- `rock_toppling`: 15项（倾倒型）
- `rock_planar`: 15项（平面滑动型）
- `rock_wedge`: 15项（楔形体滑动型）

岩质边坡取4种破坏模式中IS最大值。

### LOOKUP_TABLE（CS查表矩阵）
二维字典，键为size_val（破坏规模）和terr_val（地形参数），值为(alpha, beta)系数对。
- size_val: 0.3, 0.5, 0.7, 1.0
- terr_val: 0.0, 0.3, 0.6, 0.9, 1.2, 1.5

## 4. 关键函数

### safe_float(val, default=0.0) -> float
鲁棒的数值提取函数。处理None、int/float、字符串中的数字（正则匹配），失败返回默认值。

### calculate_slope_ts(parsed_data) -> Dict[str, Any]
核心计算函数，4个步骤：

**步骤1：提取边坡类型与高度**
- slope_type含"岩"→rock，含"二元"→binary，否则→soil
- 高度优先从score_data的C1项result字段提取，否则从basic_info.height_m

**步骤2：映射CS计算参数**
- 从score_dict中提取size_val、terr_val、坡顶设施(top_s/top_d)、坡脚设施(toe_s/toe_d)、交通量V
- 岩质与土质/二元的参数映射键名不同

**步骤3：核心计算**
- IS = Σ(score × weight)，按边坡类型选择对应权重矩阵；岩质取4种破坏模式最大值
- CS = size_val × (term1 + term2) × safe_H × V，其中term1/term2基于LOOKUP_TABLE的alpha/beta系数
- TS = IS × CS / 100

**步骤4：风险等级判定**
| 边坡类型 | I类 | II类 | III类 | IV类 |
|---------|-----|------|-------|------|
| 土质 | TS<40 | 40≤TS<50 | 50≤TS<60 | TS≥60 |
| 岩质 | TS<50 | 50≤TS<65 | 65≤TS<80 | TS≥80 |
| 二元 | TS<40 | 40≤TS<55 | 55≤TS<70 | TS≥70 |

## 5. 数据流
```
parsed_data → 提取slope_type/H/score_dict → 查LOOKUP_TABLE → 计算IS/CS/TS → 判定Level → {IS, CS, TS, Level}
```

## 6. 注意事项
- safe_H被限制在[1.0, 30.0]范围内，防止除零和极端值
- LOOKUP_TABLE使用最近邻匹配（min abs差值），非插值
- 岩质边坡取4种破坏模式IS的最大值，而非平均值
- 所有权重值均硬编码在源码中，与《指南》附录中的权重表一一对应
