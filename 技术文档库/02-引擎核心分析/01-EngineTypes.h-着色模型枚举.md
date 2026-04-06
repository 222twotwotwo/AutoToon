# EngineTypes.h - 着色模型枚举详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h` |
| **核心功能** | 定义材质着色模型枚举 |
| **MooaToon修改** | 新增 `MSM_Toon` 着色模型 |
| **修改位置** | 第 676 行 |

---

## 源码分析

### 完整的 EMaterialShadingModel 枚举

**文件位置**: `EngineTypes.h:661-683`

```cpp
UENUM()
enum EMaterialShadingModel : int
{
    // UE5 原生着色模型
    MSM_Unlit                    UMETA(DisplayName="Unlit"),
    MSM_DefaultLit               UMETA(DisplayName="Default Lit"),
    MSM_Subsurface               UMETA(DisplayName="Subsurface"),
    MSM_PreintegratedSkin        UMETA(DisplayName="Preintegrated Skin"),
    MSM_ClearCoat                UMETA(DisplayName="Clear Coat"),
    MSM_SubsurfaceProfile        UMETA(DisplayName="Subsurface Profile"),
    MSM_TwoSidedFoliage          UMETA(DisplayName="Two Sided Foliage"),
    MSM_Hair                     UMETA(DisplayName="Hair"),
    MSM_Cloth                    UMETA(DisplayName="Cloth"),
    MSM_Eye                      UMETA(DisplayName="Eye"),
    MSM_SingleLayerWater         UMETA(DisplayName="SingleLayerWater"),
    MSM_ThinTranslucent          UMETA(DisplayName="Thin Translucent"),
    MSM_Strata                   UMETA(DisplayName="Substrate", Hidden),
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // MooaToon 新增着色模型
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    MSM_Toon                     UMETA(DisplayName="Toon"), // Mooa Toon Shading Model
    
    // 系统枚举
    MSM_NUM                      UMETA(Hidden),
    MSM_FromMaterialExpression   UMETA(DisplayName="From Material Expression"),
    MSM_MAX
};
```

---

## MooaToon 修改详解

### 修改内容

在 UE5 原生的 15 个着色模型后，新增了第 16 个着色模型：

```cpp
MSM_Toon UMETA(DisplayName="Toon"), // Mooa Toon Shading Model
```

### 为什么要新增着色模型？

#### 原因1：触发专用的 Toon 着色逻辑

当材质的 Shading Model 设置为 `MSM_Toon` 时，UE5 会：
1. 在编译着色器时定义 `MATERIAL_SHADINGMODEL_TOON` 宏
2. 在 DeferredLighting 中调用 `ToonBxDF()` 而不是 `DefaultBxDF()`
3. 在 BasePass 中写入 ToonBufferA

**相关代码**：`ShaderGenerationUtil.cpp:135`
```cpp
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_TOON); // Mooa Toon Shading Model
```

#### 原因2：与传统 PBR 着色模型解耦

- `MSM_DefaultLit` - 用于 PBR 真实感渲染
- `MSM_Hair` - 用于头发渲染
- `MSM_Cloth` - 用于布料渲染
- `MSM_Toon` - **专门用于 Toon 卡通渲染**

这样设计的好处是：
- 不会破坏传统 PBR 渲染
- 可以在同一个场景混合 PBR 和 Toon 材质
- 代码逻辑清晰，易于维护

---

## 着色模型数量限制

**文件位置**: `EngineTypes.h:685`

```cpp
static_assert(MSM_NUM <= 16, "Do not exceed 16 shading models without expanding FMaterialShadingModelField to support uint32 instead of uint16!");
```

### 重要提示

⚠️ **MooaToon 已经用了第 16 个着色模型！**

| 着色模型 | 索引 |
|---------|------|
| MSM_Unlit | 0 |
| MSM_DefaultLit | 1 |
| ... | ... |
| MSM_Strata | 14 |
| **MSM_Toon** | **15** |
| MSM_NUM | 16 |

**注意**：如果 HybriToon 需要新增更多着色模型，需要：
1. 扩展 `FMaterialShadingModelField` 从 `uint16` 到 `uint32`
2. 修改这个 `static_assert`

---

## FMaterialShadingModelField 结构体

**文件位置**: `EngineTypes.h:688-710`

```cpp
USTRUCT()
struct FMaterialShadingModelField
{
    GENERATED_USTRUCT_BODY()

public:
    FMaterialShadingModelField() {}
    FMaterialShadingModelField(EMaterialShadingModel InShadingModel) 
    { 
        AddShadingModel(InShadingModel); 
    }

    void AddShadingModel(EMaterialShadingModel InShadingModel) 
    { 
        check(InShadingModel < MSM_NUM); 
        ShadingModelField |= (1 << (uint16)InShadingModel); 
    }
    
    void RemoveShadingModel(EMaterialShadingModel InShadingModel) 
    { 
        ShadingModelField &= ~(1 << (uint16)InShadingModel); 
    }
    
    void ClearShadingModels() 
    { 
        ShadingModelField = 0; 
    }
    
    // ... 更多方法
};
```

### 这是什么？

`FMaterialShadingModelField` 是一个**位域（Bitfield）**，用来表示一个材质支持哪些着色模型。

**工作原理**：
- 每个着色模型占 1 位
- `MSM_Unlit` = 第 0 位
- `MSM_DefaultLit` = 第 1 位
- ...
- `MSM_Toon` = 第 15 位

**示例**：
```cpp
// 材质支持 DefaultLit 和 Toon
FMaterialShadingModelField Field;
Field.AddShadingModel(MSM_DefaultLit);  // 第 1 位置 1
Field.AddShadingModel(MSM_Toon);         // 第 15 位置 1

// Field 的二进制: ...0001000000000010
```

---

## 这个文件在整个系统中的位置

```
EngineTypes.h (定义枚举)
    ↓
Material.cpp (检查着色模型)
    ↓
HLSLMaterialTranslator.cpp (编译属性)
    ↓
ShaderGenerationUtil.cpp (定义宏)
    ↓
着色器 (ToonShadingModel.ush)
```

---

## 实际使用示例

### 在材质编辑器中使用

1. 打开材质编辑器
2. 在详情面板找到「Material」→「Shading Model」
3. 选择「Toon」

### 在 C++ 代码中检查

```cpp
// 检查材质是否使用 Toon 着色模型
if (Material->GetShadingModels().HasShadingModel(MSM_Toon))
{
    // 是 Toon 材质
}
```

### 在着色器中检查

```hlsl
#if MATERIAL_SHADINGMODEL_TOON
    // Toon 着色逻辑
    FDirectLighting Lighting = ToonBxDF(...);
#else
    // 传统 PBR 着色逻辑
    FDirectLighting Lighting = DefaultBxDF(...);
#endif
```

---

## 相关文件索引

| 文件 | 作用 |
|------|------|
| `ShaderGenerationUtil.cpp:135` | 定义 `MATERIAL_SHADINGMODEL_TOON` 宏 |
| `HLSLMaterialTranslator.cpp:1632` | 编译 MooaEncodedAttribute |
| `Material.cpp` | 检查 Toon 着色模型 |
| `ToonShadingModel.ush` | ToonBxDF 实现 |

---

## HybriToon 扩展建议

### 如果需要新增更多着色模型

**方案A：继续往后加（不推荐）**
```cpp
// 第 17 个着色模型
MSM_HybriToonNeural UMETA(DisplayName="HybriToon Neural"),
```
⚠️ 需要修改 `static_assert` 和 `FMaterialShadingModelField`

**方案B：复用于 MSM_Toon（推荐）**
- 不需要新增着色模型
- 通过 MooaEncodedAttribute 区分
- 风险更低，兼容性更好

---

## 总结

### 关键点

1. **MSM_Toon** 是 MooaToon 新增的第 16 个着色模型
2. 它的作用是**触发专用的 Toon 着色逻辑**
3. 着色模型数量限制在 16 个（因为用了 uint16）
4. HybriToon 建议复用于 MSM_Toon，不需要新增

### 记忆要点

- ✅ 用 MSM_Toon 触发 Toon 渲染
- ✅ 检查 MATERIAL_SHADINGMODEL_TOON 宏
- ⚠️ 着色模型已经用完 16 个了
- ✅ HybriToon 用 MooaEncodedAttribute 区分就好

---

**文档版本**: v1.0  
**分析深度**: 源码级  
**最后更新**: 2026年4月6日
