
# DeferredShadingCommon.ush - 延迟着色公共函数详细分析

## 写给零基础开发者

大家好！这是 MooaToon 引擎中非常核心的一个文件 —— `DeferredShadingCommon.ush`。

### 这个文件是做什么的？

`DeferredShadingCommon.ush` 是 UE5 **延迟着色**的公共函数库，它定义了：
- GBuffer 的解码函数
- FGBufferData 结构体
- 各种 GBuffer 相关的工具函数

### MooaToon 在这个文件里改了什么？

MooaToon 在这个文件里做了以下重要修改：
1. 新增 `ToonBufferATexture` 纹理声明
2. 在 `FGBufferData` 结构体中新增 `MooaToonContext` 成员
3. 在 `DecodeGBufferData()` 函数中新增 ToonBufferA 的解码逻辑

---

## 文件信息表

| 项目 | 说明 |
|------|------|
| **文件路径** | `Engine/Shaders/Private/DeferredShadingCommon.ush` |
| **主要功能** | 延迟着色公共函数、GBuffer 解码 |
| **重要性** | ⭐⭐⭐⭐⭐ 核心文件，延迟着色的基础 |
| **代码行数** | 约 1200+ 行 |
| **MooaToon 修改** | 5 处 |

---

## MooaToon 修改总览

| 位置 | 功能 | 代码行数 |
|------|------|---------|
| **修改 1** | 包含 ToonShadingCommon.ush | 15-16 |
| **修改 2** | 声明 ToonBufferATexture | 45-48 |
| **修改 3** | FGBufferData 新增 MooaToonContext | 401-403 |
| **修改 4** | DecodeGBufferData 参数新增 InToonBufferA | 965-967 |
| **修改 5** | DecodeGBufferData 中解码 ToonBufferA | 1017-1023 |

---

## 逐块详细解释

---

### 修改 1：包含 ToonShadingCommon.ush

```hlsl
// Mooa
#include "ToonShadingCommon.ush"
```

**说明：**
- 这是第一行修改，引入了 ToonShadingCommon.ush
- 这样后面就可以使用 `FToonGBufferData`、`DecodeToonGBufferDataFromMRT` 等函数了

---

### 修改 2：声明 ToonBufferATexture

```hlsl
// Mooa GBuffer
Texture2D ToonBufferATexture;
#define ToonBufferATextureSampler GlobalPointClampedSampler
// Mooa End
```

| 声明 | 说明 |
|------|------|
| `Texture2D ToonBufferATexture` | ToonBufferA 纹理 |
| `ToonBufferATextureSampler` | 采样器，使用 GlobalPointClampedSampler |

**写给零基础：**
- `GlobalPointClampedSampler` 是 UE5 预定义的采样器
- Point = 点采样（不插值）
- Clamped = 超出范围时钳制（不重复）

---

### 修改 3：FGBufferData 新增 MooaToonContext

```hlsl
// Mooa GBuffer
FMooaToonContext MooaToonContext;
// Mooa End
```

**FGBufferData 结构体总览：**

| 成员 | 说明 |
|------|------|
| `WorldNormal` | 世界空间法线 |
| `BaseColor` | 基础色 |
| `Metallic` | 金属度 |
| `Specular` | 高光 |
| `Roughness` | 粗糙度 |
| `ShadingModelID` | 着色模型 ID |
| `CustomData` | 自定义数据 |
| `PerObjectGBufferData` | 逐对象 GBuffer 数据 |
| `Depth` | 深度 |
| **`MooaToonContext`** | **MooaToon 上下文（新增）** |

**写给零基础：**
- `FMooaToonContext` 是在 `ToonShadingCommon.ush` 中定义的
- 它包含了 `ToonGBuffer` 成员，存储解码后的 Toon 数据

---

### 修改 4：DecodeGBufferData 参数新增 InToonBufferA

```hlsl
float4 InGBufferE,
float4 InGBufferF,
// Mooa GBuffer
float4 InToonBufferA,
// Mooa End
float4 InGBufferVelocity,
```

| 参数 | 说明 |
|------|------|
| `InGBufferA` - `InGBufferF` | 传统 GBuffer 输入 |
| **`InToonBufferA`** | **ToonBufferA 输入（新增）** |
| `InGBufferVelocity` | 速度缓冲输入 |

---

### 修改 5：DecodeGBufferData 中解码 ToonBufferA

这是最重要的修改！

```hlsl
// Mooa GBuffer Decode
BRANCH if (GBuffer.ShadingModelID == SHADINGMODELID_TOON)
{
	GBuffer.MooaToonContext.ToonGBuffer = DecodeToonGBufferDataFromMRT(InToonBufferA, GBuffer.CustomData, GBuffer.Metallic, InGBufferF.a);
	InitMooaToonContext(GBuffer);
}
// Mooa End
```

| 步骤 | 说明 |
|------|------|
| 1 | 检查 `ShadingModelID` 是否是 `SHADINGMODELID_TOON` |
| 2 | 如果是，调用 `DecodeToonGBufferDataFromMRT()` 解码 |
| 3 | 调用 `InitMooaToonContext()` 初始化上下文 |

**写给零基础：**
- 这个逻辑在延迟着色的 GBuffer 解码阶段执行
- 只有当材质使用 Toon 着色模型时，才会解码 ToonBufferA
- 解码后的数据存储在 `GBuffer.MooaToonContext.ToonGBuffer` 中

---

## 数据流图（延迟着色）

```
BasePass
    ↓
写入 GBufferA-F + ToonBufferA
    ↓
延迟光照阶段
    ↓
DecodeGBufferData()
    ↓
读取 GBufferA-F + ToonBufferA
    ↓
解码传统 GBuffer 数据
    ↓
如果是 Toon 着色模型
    ↓
DecodeToonGBufferDataFromMRT(InToonBufferA, ...)
    ↓
InitMooaToonContext(GBuffer)
    ↓
得到完整的 FGBufferData（包含 MooaToonContext）
    ↓
ToonBxDF() 使用这些数据计算光照
```

---

## 相关函数说明

### DecodeToonGBufferDataFromMRT()

这个函数在 `ToonShadingCommon.ush` 中定义，它的作用是：
- 输入：`InToonBufferA`、`CustomData`、`Metallic`、`InGBufferF.a`
- 输出：`FToonGBufferData`

### InitMooaToonContext()

这个函数也在 `ToonShadingCommon.ush` 中定义，它的作用是：
- 初始化 `FMooaToonContext` 的其他成员
- 设置 `PixelPos`、`LightType`、`LightColor` 等

---

## 总结

### 关键点回顾

1. **ToonBufferATexture** - 新增的纹理声明
2. **FMooaToonContext** - FGBufferData 新增的成员
3. **DecodeGBufferData()** - 解码 ToonBufferA 的逻辑
4. **条件判断** - 只有 Toon 着色模型才解码

### 记忆要点

- **核心作用**：在延迟着色阶段解码 ToonBufferA
- **修改位置**：DeferredShadingCommon.ush
- **关键函数**：DecodeGBufferData()
- **条件**：ShadingModelID == SHADINGMODELID_TOON

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `ToonShadingCommon.ush` | FMooaToonContext、DecodeToonGBufferDataFromMRT 定义 |
| `DeferredLightingCommon.ush` | 延迟光照计算 |
| `ShadingModels.ush` | IntegrateBxDF() 调用 ToonBxDF() |

---

祝你学习顺利！🚀

