# HLSLMaterialTranslator.cpp - 材质属性编译详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp` |
| **核心功能** | 材质属性编译、MooaEncodedAttribute 编译、着色模型宏定义 |
| **重要性** | ⭐⭐⭐⭐⭐ |

---

## 写给零基础开发者

### 这个文件是做什么的？

**想象一下**：你是一个翻译，要把中文翻译成英文。

这个文件就是那个**翻译**，它负责：
1. 把材质编辑器里的节点（中文）翻译成 HLSL 代码（英文）
2. 当使用 Toon 着色模型时，自动编译 MooaEncodedAttribute0-4
3. 定义着色模型宏（如 `MATERIAL_SHADINGMODEL_TOON`）

---

## 文件结构总览（Mooa 部分）

| 部分 | 行号 | 内容 |
|------|------|------|
| 1 | 767-773 | SharedPixelProperties - 标记 Mooa 属性为像素着色器属性 |
| 2 | 1631-1640 | 编译 MooaEncodedAttribute0-4 |
| 3 | 1820-1826 | 各向异性检测（Toon 特殊处理） |
| 4 | 2851-2856 | 定义 MATERIAL_SHADINGMODEL_TOON 宏 |

---

## 第一部分：SharedPixelProperties

### 什么是 SharedPixelProperties？

**SharedPixelProperties = 共享像素属性数组**

**用大白话解释**：
- 这是一个布尔数组，标记哪些属性是「像素着色器属性」
- 像素着色器属性 = 在像素着色器阶段计算的属性
- 标记为 true 的属性，会在像素着色器中使用

---

### Mooa 属性标记

**行号**: 767-773

```cpp
// Mooa Material Editor
SharedPixelProperties[MP_MooaEncodedAttribute0] = true;
SharedPixelProperties[MP_MooaEncodedAttribute1] = true;
SharedPixelProperties[MP_MooaEncodedAttribute2] = true;
SharedPixelProperties[MP_MooaEncodedAttribute3] = true;
SharedPixelProperties[MP_MooaEncodedAttribute4] = true;
// Mooa End
```

**这是做什么的？**
- 把 MP_MooaEncodedAttribute0-4 都标记为像素着色器属性
- 这样它们会在像素着色器阶段计算和使用

**为什么要这样做？**
- MooaEncodedAttribute 包含 Toon 渲染专用数据（Ramp 索引、UV 偏移等）
- 这些数据是每个像素都不同的，所以要在像素着色器中计算

---

## 第二部分：编译 MooaEncodedAttribute0-4

### 1631-1640 行：核心逻辑

```cpp
// Mooa Material Editor
if (MaterialShadingModels.HasShadingModel(MSM_Toon))
{
    Chunk[MP_MooaEncodedAttribute0] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute0, this);
    Chunk[MP_MooaEncodedAttribute1] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute1, this);
    Chunk[MP_MooaEncodedAttribute2] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute2, this);
    Chunk[MP_MooaEncodedAttribute3] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute3, this);
    Chunk[MP_MooaEncodedAttribute4] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute4, this);
}
// Mooa End
```

**这是做什么的？**
- 如果材质使用了 Toon 着色模型（MSM_Toon）
- 就编译 MooaEncodedAttribute0-4 这 5 个属性
- 把编译结果存到 Chunk 数组里

**逐句解释**：

| 代码 | 说明 |
|------|------|
| `MaterialShadingModels.HasShadingModel(MSM_Toon)` | 检查材质是否使用 Toon 着色模型 |
| `Material->CompilePropertyAndSetMaterialProperty(...)` | 编译这个材质属性 |
| `Chunk[MP_MooaEncodedAttribute0] = ...` | 把编译结果存到 Chunk 数组里 |

**什么是 Chunk？**
- Chunk = 编译后的代码块
- 每个属性对应一个 Chunk
- Chunk 数组是所有属性编译结果的集合

**流程图**：
```
材质编辑器节点
    ↓
CompilePropertyAndSetMaterialProperty（编译）
    ↓
Chunk 数组（存储编译结果）
    ↓
生成 HLSL 代码
```

---

## 第三部分：各向异性检测（Toon 特殊处理）

### 1820-1826 行

```cpp
// Mooa Anisotropy
// MooaToon uses 0 as the default value for Tangent and converts it to World Tangent before output.
// Non-zero values are output as-is.
bUsesAnisotropy = bUsesAnisotropy != 0
                    || (IsMaterialPropertyUsed(MP_Tangent, Chunk[MP_Tangent], FLinearColor(0, 0, 0, 0), 3)
                            && MaterialShadingModels.HasShadingModel(MSM_Toon));
// Mooa End
```

**这是做什么的？**
- 检测材质是否使用各向异性
- 对于 Toon 着色模型，有特殊处理

**注释解释**：
```
MooaToon 使用 0 作为 Tangent 的默认值，
在输出前把它转换成 World Tangent。
非零值按原样输出。
```

**逐句解释**：

| 代码 | 说明 |
|------|------|
| `bUsesAnisotropy != 0` | 如果已经标记为使用各向异性，保持 true |
| `IsMaterialPropertyUsed(MP_Tangent, ...)` | 检查 MP_Tangent 属性是否被使用 |
| `MaterialShadingModels.HasShadingModel(MSM_Toon)` | 并且材质使用 Toon 着色模型 |
| `bUsesAnisotropy = ...` | 如果满足任一条件，就标记为使用各向异性 |

**为什么要这样处理？**
- Toon 着色模型的 WorldTangent 有特殊用途（面部朝向）
- 即使 Tangent 的默认值是 0，也要标记为使用各向异性

---

## 第四部分：定义着色模型宏

### 2851-2856 行

```cpp
// Mooa Toon Shading Model
if (EnvironmentDefines->HasShadingModel(MSM_Toon))
{
    OutEnvironment.SetDefine(TEXT("MATERIAL_SHADINGMODEL_TOON"), TEXT("1"));
}
// Mooa End
```

**这是做什么的？**
- 如果材质使用 Toon 着色模型
- 就定义一个宏：`#define MATERIAL_SHADINGMODEL_TOON 1`

**什么是宏？**
- 宏 = 一个开关
- 在着色器代码中，可以用 `#ifdef MATERIAL_SHADINGMODEL_TOON` 来检查
- 如果定义了，就执行特定代码

**示例**：
```hlsl
// 在着色器中：
#ifdef MATERIAL_SHADINGMODEL_TOON
    // 执行 Toon 特定代码
    ToonBxDF(...);
#else
    // 执行普通 PBR 代码
    StandardBxDF(...);
#endif
```

---

## 完整的材质编译流程（Mooa 部分）

```
┌─────────────────────────────────────────────────────────────┐
│  1. 初始化                                                   │
│  └─ SharedPixelProperties[MP_MooaEncodedAttribute*] = true │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  2. 编译属性                                                 │
│  └─ 如果是 MSM_Toon：                                       │
│     ├─ 编译 MP_MooaEncodedAttribute0                        │
│     ├─ 编译 MP_MooaEncodedAttribute1                        │
│     ├─ 编译 MP_MooaEncodedAttribute2                        │
│     ├─ 编译 MP_MooaEncodedAttribute3                        │
│     └─ 编译 MP_MooaEncodedAttribute4                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  3. 检测各向异性                                             │
│  └─ 如果是 MSM_Toon 且使用了 Tangent：                      │
│     └─ bUsesAnisotropy = true                               │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  4. 定义宏                                                   │
│  └─ 如果是 MSM_Toon：                                       │
│     └─ #define MATERIAL_SHADINGMODEL_TOON 1                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  5. 生成 HLSL 代码                                           │
│  └─ 所有 Chunk 组合成最终的着色器代码                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 总结

### 关键点

1. **SharedPixelProperties** - 标记 Mooa 属性为像素着色器属性
2. **条件编译** - 只有当材质是 MSM_Toon 时，才编译 MooaEncodedAttribute0-4
3. **各向异性特殊处理** - Toon 着色模型的 Tangent 有特殊用途
4. **宏定义** - 定义 `MATERIAL_SHADINGMODEL_TOON`，着色器可以用 `#ifdef` 检查

### 记忆要点

- ✅ HLSLMaterialTranslator.cpp = 翻译
- ✅ SharedPixelProperties = 标记像素属性
- ✅ 只有 MSM_Toon 才编译 MooaEncodedAttribute
- ✅ 定义 MATERIAL_SHADINGMODEL_TOON 宏
- ✅ 着色器用 #ifdef 检查这个宏

---

**文档版本**: v1.0  
**分析深度**: 源码级（逐行解释）  
**目标读者**: 零基础开发者  
**最后更新**: 2026年4月6日
