# BasePassPixelShader.usf - BasePass 集成详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Shaders/Private/BasePassPixelShader.usf` |
| **核心功能** | BasePass 中集成 Toon、解码 MooaEncodedAttribute、编码 ToonBufferA |
| **重要性** | ⭐⭐⭐⭐⭐ |

---

## 写给零基础开发者

### 这个文件是做什么的？

**想象一下**：你是一个厨师，要把食材做成菜。

这个文件就是那个**厨师**，它负责 BasePass（基础渲染通道），具体到 MooaToon：
1. 从材质属性中解码 Toon 数据
2. 初始化 MooaToon 上下文
3. 把 Toon 数据编码到 ToonBufferA
4. Forward 渲染时应用全局光照强度

**什么是 BasePass？**
- BasePass = 基础渲染通道
- 这是渲染的第一步
- 把几何体渲染到 GBuffer（包括 ToonBufferA）
- 延迟渲染：后面的光照 pass 会读取 GBuffer
- 前向渲染：在这个 pass 里就计算光照

---

## 文件结构总览（Mooa 部分）

| 部分 | 行号 | 内容 |
|------|------|------|
| 1 | 1078-1092 | 从材质属性解码 ToonGBuffer |
| 2 | 1160-1165 | 设置 PixelPos 到 MooaToonContext |
| 3 | 1538-1543 | Forward 渲染时应用全局光照强度 |
| 4 | 2083-2092 | 编码 ToonGBuffer 到 MRT（ToonBufferA） |
| 5 | 2102-2104 | Unlit 模式下清空 ToonBufferA |

---

## 第一部分：从材质属性解码 ToonGBuffer

### 1078-1092 行：核心代码

```hlsl
// Mooa Encode GBuffer
#if MATERIAL_SHADINGMODEL_TOON
    BRANCH if (ShadingModel == SHADINGMODELID_TOON)
    {
        GBuffer.MooaToonContext.ToonGBuffer = DecodeToonGBufferDataFromMaterialAttribute(
            GetMaterialMooaEncodedAttribute0(PixelMaterialInputs),
            GetMaterialMooaEncodedAttribute1(PixelMaterialInputs),
            GetMaterialMooaEncodedAttribute2(PixelMaterialInputs),
            GetMaterialMooaEncodedAttribute3(PixelMaterialInputs),
            GetMaterialMooaEncodedAttribute4(PixelMaterialInputs));
        
        InitMooaToonContext(GBuffer);
    }
#endif
// Mooa End
```

**这是做什么的？**
- 只有当 `MATERIAL_SHADINGMODEL_TOON` 宏定义时才编译这段代码
- 只有当 ShadingModel 是 SHADINGMODELID_TOON 时才执行
- 从 5 个材质属性（MooaEncodedAttribute0-4）中解码出 ToonGBuffer
- 初始化 MooaToon 上下文

**逐句解释**：

| 代码 | 说明 |
|------|------|
| `#if MATERIAL_SHADINGMODEL_TOON` | 只有定义了这个宏才编译 |
| `ShadingModel == SHADINGMODELID_TOON` | 只有当前材质是 Toon 着色模型才执行 |
| `DecodeToonGBufferDataFromMaterialAttribute(...)` | 从 5 个材质属性解码 ToonGBuffer |
| `GetMaterialMooaEncodedAttribute0(PixelMaterialInputs)` | 获取材质属性 0 |
| `InitMooaToonContext(GBuffer)` | 初始化 MooaToon 上下文 |

**什么是 DecodeToonGBufferDataFromMaterialAttribute？**
- 这个函数在 ToonShadingCommon.ush 中定义
- 输入：5 个 Float4 材质属性
- 输出：FToonGBufferData 结构体
- 详细内容见：01-ToonShadingCommon.ush-GBuffer编码解码.md

---

## 第二部分：设置 PixelPos 到 MooaToonContext

### 1160-1165 行

```hlsl
// Mooa GBuffer
if (ShadingModel == SHADINGMODELID_TOON)
{
    SetMooaToonContext_PixelPos(GBuffer, MaterialParameters.SvPosition.xy);
}
// Mooa End
```

**这是做什么的？**
- 把像素位置（SvPosition.xy）存到 MooaToonContext 里
- 后面的屏幕空间效果（头发阴影、边缘光）需要用到像素位置

**什么是 SvPosition？**
- SvPosition = 系统值：像素位置
- SvPosition.xy = 像素在屏幕上的坐标（单位：像素）
- 范围：(0, 0) 到 (ViewSize.x, ViewSize.y)

---

## 第三部分：Forward 渲染时应用全局光照强度

### 1538-1543 行

```hlsl
// Mooa Forward Shading
if (GBuffer.ShadingModelID == SHADINGMODELID_TOON)
{
    DiffuseColor *= View.MooaGlobalIlluminationIntensity;
}
// Mooa End
```

**这是做什么的？**
- 只有在 Forward（前向）渲染时才执行
- 把漫反射颜色乘以全局光照强度
- 这个强度在 PostProcessSettings 里设置

**为什么只在 Forward 渲染？**
- 延迟渲染（Deferred）：在光照 Pass 里计算，不是在 BasePass
- 前向渲染（Forward）：在 BasePass 里计算光照，所以要在这里应用

---

## 第四部分：编码 ToonGBuffer 到 MRT

### 2083-2092 行：核心代码

```hlsl
// Mooa Encode GBuffer
#if MATERIAL_SHADINGMODEL_TOON
    BRANCH if (GBuffer.ShadingModelID == SHADINGMODELID_TOON)
    {
        GBuffer.GenericAO = MaterialAO;
        EncodeToonGBufferDataToMRT(GBuffer.MooaToonContext.ToonGBuffer,
            GBuffer.MooaToonContext.EncodedToonBufferA, GBuffer.CustomData, GBuffer.Metallic);
    }
#endif
// Mooa End
```

**这是做什么的？**
- 把 ToonGBuffer 编码到 MRT（多渲染目标）
- 主要写入：EncodedToonBufferA（ToonBufferA）、CustomData、Metallic

**逐句解释**：

| 代码 | 说明 |
|------|------|
| `GBuffer.GenericAO = MaterialAO` | 把材质 AO 存到 GenericAO |
| `EncodeToonGBufferDataToMRT(...)` | 编码 ToonGBuffer 到 MRT |
| `GBuffer.MooaToonContext.EncodedToonBufferA` | 输出：ToonBufferA |
| `GBuffer.CustomData` | 输出：CustomData |
| `GBuffer.Metallic` | 输入/输出：Metallic（可能会被修改） |

**什么是 EncodeToonGBufferDataToMRT？**
- 这个函数在 ToonShadingCommon.ush 中定义
- 输入：FToonGBufferData
- 输出：EncodedToonBufferA、CustomData、Metallic
- 详细内容见：01-ToonShadingCommon.ush-GBuffer编码解码.md

---

## 第五部分：Unlit 模式下清空 ToonBufferA

### 2102-2104 行

```hlsl
// Mooa Encode GBuffer
Out.MRT[1 + GBUFFER_HAS_VELOCITY ? 6 : 5] = 0;
// Mooa End
```

**这是做什么的？**
- 如果是 Unlit（无光照）着色模型
- 就把 ToonBufferA 清 0
- 因为 Unlit 不需要 Toon 渲染

**MRT 索引计算**：
- `GBUFFER_HAS_VELOCITY` = 是否有 GBufferVelocity
- 如果有：索引 = 1 + 6 = 7
- 如果没有：索引 = 1 + 5 = 6
- 这个索引对应 ToonBufferA

---

## 完整的 BasePass 数据流（Mooa 部分）

```
┌─────────────────────────────────────────────────────────────┐
│  1. 解码材质属性                                             │
│  └─ DecodeToonGBufferDataFromMaterialAttribute(0-4)         │
│     └─ → FToonGBufferData                                    │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  2. 初始化上下文                                             │
│  └─ InitMooaToonContext(GBuffer)                            │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  3. 设置 PixelPos                                            │
│  └─ SetMooaToonContext_PixelPos(PixelPos)                   │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  4. Forward 渲染？                                           │
│  ├─ 是 → DiffuseColor *= MooaGlobalIlluminationIntensity   │
│  └─ 否 → 跳过（在光照 Pass 计算）                           │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  5. 编码到 MRT                                               │
│  └─ EncodeToonGBufferDataToMRT()                            │
│     ├─ → EncodedToonBufferA（ToonBufferA）                  │
│     ├─ → CustomData                                          │
│     └─ → Metallic（可能修改）                                │
└─────────────────────────────────────────────────────────────┘
```

---

## 总结

### 关键点

1. **解码阶段** - 从 MooaEncodedAttribute0-4 解码出 ToonGBuffer
2. **初始化阶段** - InitMooaToonContext 初始化上下文
3. **PixelPos** - 设置像素位置，用于屏幕空间效果
4. **Forward 渲染** - 应用全局光照强度
5. **编码阶段** - EncodeToonGBufferDataToMRT 编码到 ToonBufferA
6. **Unlit 模式** - 清空 ToonBufferA

### 记忆要点

- ✅ BasePassPixelShader.usf = 厨师
- ✅ 解码 = 把食材拆包
- ✅ 初始化 = 准备工具
- ✅ 编码 = 把菜装盘（写到 GBuffer）
- ✅ Forward = 在这个 Pass 就炒菜（计算光照）
- ✅ Deferred = 先装盘，后面再炒菜

---

**文档版本**: v1.0  
**分析深度**: 源码级（逐行解释）  
**目标读者**: 零基础开发者  
**最后更新**: 2026年4月7日
