
# DeferredLightingCommon.ush - 延迟光照公共函数详细分析

## 写给零基础开发者

大家好！这是 MooaToon 引擎中另一个非常核心的文件 —— `DeferredLightingCommon.ush`。

### 这个文件是做什么的？

`DeferredLightingCommon.ush` 是 UE5 **延迟光照**的公共函数库，它负责：
- 遍历所有光源
- 对每个光源调用 BxDF 函数计算光照
- 累加光照结果

### MooaToon 在这个文件里改了什么？

MooaToon 在这个文件里做了以下重要修改：
1. 在光照计算条件中新增 Toon 着色模型
2. 在光照计算前设置 MooaToon 上下文（LightType、LightColor、PixelPos）
3. 对 Toon 着色模型强制设置 Shadow 和 MaskedLightColor 为 1

---

## 文件信息表

| 项目 | 说明 |
|------|------|
| **文件路径** | `Engine/Shaders/Private/DeferredLightingCommon.ush` |
| **主要功能** | 延迟光照计算、光源遍历 |
| **重要性** | ⭐⭐⭐⭐⭐ 核心文件，延迟光照的核心 |
| **代码行数** | 约 500+ 行 |
| **MooaToon 修改** | 3 处 |

---

## MooaToon 修改总览

| 位置 | 功能 | 代码行数 |
|------|------|---------|
| **修改 1** | 光照计算条件新增 Toon 着色模型 | 398 |
| **修改 2** | 设置 MooaToon 上下文（LightType/LightColor/PixelPos） | 423-427 |
| **修改 3** | 对 Toon 着色模型强制设置 Shadow 和 MaskedLightColor | 458-465 |

---

## 逐块详细解释

---

### 修改 1：光照计算条件新增 Toon 着色模型

```hlsl
if( (Shadow.SurfaceShadow + Shadow.TransmissionShadow &gt; 0) || GBuffer.ShadingModelID == SHADINGMODELID_TOON)// Mooa Toon Shading
```

**原来的条件：**
```hlsl
if( (Shadow.SurfaceShadow + Shadow.TransmissionShadow &gt; 0) )
```

**修改后的条件：**
```hlsl
if( (Shadow.SurfaceShadow + Shadow.TransmissionShadow &gt; 0) || GBuffer.ShadingModelID == SHADINGMODELID_TOON )
```

| 条件 | 说明 |
|------|------|
| `Shadow.SurfaceShadow + Shadow.TransmissionShadow &gt; 0` | 表面有阴影或透射阴影 |
| `GBuffer.ShadingModelID == SHADINGMODELID_TOON` | 是 Toon 着色模型 |

**写给零基础：**
- 为什么要加这个？
  - 因为 Toon 着色模型即使没有阴影，也需要计算光照
  - 传统的 PBR 着色模型可能在没有阴影时跳过光照计算，但 Toon 不行

---

### 修改 2：设置 MooaToon 上下文

```hlsl
// Mooa GBuffer
SetMooaToonContext_LightType(GBuffer, LightData);
SetMooaToonContext_LightColor(GBuffer, MaskedLightColor);
SetMooaToonContext_PixelPos(GBuffer, SVPos);
// Mooa End
```

| 函数 | 说明 | 参数 |
|------|------|------|
| `SetMooaToonContext_LightType()` | 设置光源类型 | GBuffer, LightData |
| `SetMooaToonContext_LightColor()` | 设置光源颜色 | GBuffer, MaskedLightColor |
| `SetMooaToonContext_PixelPos()` | 设置像素位置 | GBuffer, SVPos |

**写给零基础：**
- 这三个函数在 `ToonShadingCommon.ush` 中定义
- 它们把光源信息和像素位置存储到 `GBuffer.MooaToonContext` 中
- 这样 `ToonBxDF()` 函数就能访问这些信息了

---

### 修改 3：对 Toon 着色模型强制设置 Shadow 和 MaskedLightColor

```hlsl
// Mooa Toon Shading
BRANCH
if (GBuffer.ShadingModelID == SHADINGMODELID_TOON)
{
	Shadow.SurfaceShadow = 1;
	MaskedLightColor = 1;
}
// Mooa End
```

| 设置 | 值 | 说明 |
|------|-----|------|
| `Shadow.SurfaceShadow` | 1 | 强制表面阴影为 1（不衰减） |
| `MaskedLightColor` | 1 | 强制光源颜色为 1（不衰减） |

**写给零基础：**
- 为什么要强制设置为 1？
  - 因为 Toon 着色模型自己在 `ToonBxDF()` 中处理阴影和光照衰减
  - 不需要传统的阴影计算
  - 这样可以确保 Toon 渲染完全由自己的逻辑控制

---

## 数据流图（延迟光照）

```
DeferredLightingCommon.ush
    ↓
遍历所有光源
    ↓
对每个光源：
    ↓
    检查是否需要计算光照（包括 Toon 着色模型）
        ↓
        设置 MooaToon 上下文：
            - LightType
            - LightColor
            - PixelPos
        ↓
        如果是 Toon 着色模型：
            - Shadow.SurfaceShadow = 1
            - MaskedLightColor = 1
        ↓
        调用 IntegrateBxDF()
            ↓
            IntegrateBxDF() 中调用 ToonBxDF()
                ↓
                ToonBxDF() 使用 MooaToonContext 计算光照
            ↓
        返回光照结果
    ↓
累加所有光源的光照
```

---

## 相关函数说明

### IntegrateBxDF()

这个函数在 `ShadingModels.ush` 中定义，它的作用是：
- 根据 `ShadingModelID` 选择对应的 BxDF 函数
- 对 Toon 着色模型，调用 `ToonBxDF()`

### ToonBxDF()

这个函数在 `ToonShadingModel.ush` 中定义，它是 Toon 着色的核心！

---

## 总结

### 关键点回顾

1. **光照条件** - 新增 Toon 着色模型，确保即使没有阴影也计算光照
2. **MooaToon 上下文** - 设置 LightType、LightColor、PixelPos
3. **强制 Shadow 和 MaskedLightColor** - 设置为 1，让 ToonBxDF 完全控制

### 记忆要点

- **核心作用**：延迟光照阶段为 Toon 着色模型准备上下文
- **三个设置函数**：SetMooaToonContext_LightType/LightColor/PixelPos
- **强制值**：Shadow.SurfaceShadow = 1, MaskedLightColor = 1

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `ToonShadingCommon.ush` | SetMooaToonContext_* 函数定义 |
| `ToonShadingModel.ush` | ToonBxDF() 核心实现 |
| `ShadingModels.ush` | IntegrateBxDF() 函数 |
| `DeferredShadingCommon.ush` | DecodeGBufferData() 函数 |

---

祝你学习顺利！🚀

