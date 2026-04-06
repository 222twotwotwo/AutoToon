
# ShadingModels.ush - 着色模型集成详细分析

## 写给零基础开发者

大家好！这是 MooaToon 引擎中非常重要的一个文件 —— `ShadingModels.ush`。

### 这个文件是做什么的？

`ShadingModels.ush` 是 UE5 **着色模型**的集成文件，它定义了：
- 所有着色模型的 BxDF 函数
- `IntegrateBxDF()` 函数，根据着色模型 ID 选择对应的 BxDF

### MooaToon 在这个文件里改了什么？

MooaToon 在这个文件里做了以下修改：
1. 包含 `ToonShadingModel.ush`（非移动端）
2. 在 `IntegrateBxDF()` 函数的 switch 语句中新增 `SHADINGMODELID_TOON` case

---

## 文件信息表

| 项目 | 说明 |
|------|------|
| **文件路径** | `Engine/Shaders/Private/ShadingModels.ush` |
| **主要功能** | 着色模型集成、IntegrateBxDF 函数 |
| **重要性** | ⭐⭐⭐⭐ 重要文件，连接着色模型和 BxDF |
| **代码行数** | 约 1000+ 行 |
| **MooaToon 修改** | 2 处 |

---

## MooaToon 修改总览

| 位置 | 功能 | 代码行数 |
|------|------|---------|
| **修改 1** | 包含 ToonShadingModel.ush（非移动端） | 973-977 |
| **修改 2** | IntegrateBxDF() 中新增 SHADINGMODELID_TOON case | 1003-1007 |

---

## 逐块详细解释

---

### 修改 1：包含 ToonShadingModel.ush

```hlsl
// Mooa Toon Shading Model
#if !SHADING_PATH_MOBILE
	#include "ToonShadingModel.ush"
#endif
// Mooa End
```

| 条件 | 说明 |
|------|------|
| `!SHADING_PATH_MOBILE` | 非移动端路径 |

**写给零基础：**
- 为什么要加 `#if !SHADING_PATH_MOBILE`？
  - 因为 MooaToon 目前不支持移动端
  - 移动端用的是简化的渲染路径，没有完整的延迟着色
- 这样可以避免在移动端编译时出错

---

### 修改 2：IntegrateBxDF() 中新增 SHADINGMODELID_TOON case

这是最重要的修改！

```hlsl
// Mooa Toon Shading Model
#if !SHADING_PATH_MOBILE
	case SHADINGMODELID_TOON:
		return ToonBxDF(GBuffer, N, V, L, Falloff, NoL, AreaLight, Shadow);
#endif
```

**IntegrateBxDF() 函数总览：**

```hlsl
FDirectLighting IntegrateBxDF( FGBufferData GBuffer, half3 N, half3 V, half3 L, float Falloff, half NoL, FAreaLight AreaLight, FShadowTerms Shadow )
{
	switch( GBuffer.ShadingModelID )
	{
		case SHADINGMODELID_DEFAULT_LIT:
		case SHADINGMODELID_SINGLELAYERWATER:
		case SHADINGMODELID_THIN_TRANSLUCENT:
			// ... 默认光照 ...
		case SHADINGMODELID_EYE:
			return EyeBxDF( GBuffer, N, V, L, Falloff, NoL, AreaLight, Shadow );
		// Mooa Toon Shading Model
		#if !SHADING_PATH_MOBILE
		case SHADINGMODELID_TOON:
			return ToonBxDF(GBuffer, N, V, L, Falloff, NoL, AreaLight, Shadow);
		#endif
		default:
			return (FDirectLighting)0;
	}
}
```

| 参数 | 说明 |
|------|------|
| `GBuffer` | GBuffer 数据（包含 MooaToonContext） |
| `N` | 法线 |
| `V` | 视线方向 |
| `L` | 光照方向 |
| `Falloff` | 光照衰减 |
| `NoL` | 法线点乘光照方向 |
| `AreaLight` | 区域光数据 |
| `Shadow` | 阴影数据 |

**写给零基础：**
- `IntegrateBxDF()` 就像一个"调度员"
- 它看一下 `GBuffer.ShadingModelID` 是多少
- 如果是 `SHADINGMODELID_TOON`，就调用 `ToonBxDF()`
- 否则调用其他对应的 BxDF 函数

---

## 数据流图（完整）

```
材质编辑器
    ↓
设置 MSM_Toon 着色模型
    ↓
设置 MP_MooaEncodedAttribute0-4
    ↓
BasePass
    ↓
调用 GetMaterialMooaEncodedAttribute0-4()
    ↓
DecodeToonGBufferDataFromMaterialAttribute()
    ↓
EncodeToonGBufferDataToMRT()
    ↓
写入 GBuffer + ToonBufferA
    ↓
延迟光照阶段
    ↓
DecodeGBufferData()
    ↓
DecodeToonGBufferDataFromMRT()
    ↓
InitMooaToonContext()
    ↓
DeferredLightingCommon.ush
    ↓
设置 MooaToonContext（LightType/LightColor/PixelPos）
    ↓
强制 Shadow.SurfaceShadow = 1, MaskedLightColor = 1
    ↓
调用 IntegrateBxDF()
    ↓
IntegrateBxDF() 中 switch 到 SHADINGMODELID_TOON
    ↓
调用 ToonBxDF()
    ↓
ToonBxDF() 使用 MooaToonContext 计算光照
    ↓
返回 FDirectLighting（漫反射 + 高光）
    ↓
累加所有光源
    ↓
输出最终画面
```

---

## 总结

### 关键点回顾

1. **包含 ToonShadingModel.ush** - 非移动端
2. **IntegrateBxDF()** - switch 语句中新增 SHADINGMODELID_TOON case
3. **ToonBxDF()** - 实际的 Toon 着色逻辑

### 记忆要点

- **核心作用**：把 Toon 着色模型集成到 UE5 的着色模型系统
- **关键函数**：IntegrateBxDF()
- **条件**：!SHADING_PATH_MOBILE（非移动端）

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `ToonShadingModel.ush` | ToonBxDF() 核心实现 |
| `DeferredLightingCommon.ush` | 调用 IntegrateBxDF() |
| `DeferredShadingCommon.ush` | DecodeGBufferData() |
| `MaterialTemplate.ush` | GetMaterialMooaEncodedAttribute0-4() |

---

祝你学习顺利！🚀

