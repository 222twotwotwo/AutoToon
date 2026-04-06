# MooaToon 技术详细报告3 - 材质属性与编码系统

## 目录
1. [材质属性系统](#材质属性系统)
2. [MooaEncodedAttribute映射](#mooaencodedattribute映射)
3. [完整数据编码流程](#完整数据编码流程)
4. [着色模型集成](#着色模型集成)
5. [HybriToon接口设计](#hybritoon接口设计)

---

## 材质属性系统

### 1. MooaEncodedAttribute0-4

MooaToon新增了**5个Float4材质属性**，用于在材质编辑器中暴露Toon渲染参数：

| 属性 | 类型 | 默认值 | 着色频率 | GUID |
|------|------|---------|---------|------|
| MP_MooaEncodedAttribute0 | Float4 | (0,0,0,0) | 像素着色器 | C1A0EA03-15C74A55-B852BAD5-9A20789A |
| MP_MooaEncodedAttribute1 | Float4 | (0,0,0,0) | 像素着色器 | C6A0AB14-15D75A65-B132ABD4-0B20789C |
| MP_MooaEncodedAttribute2 | Float4 | (0,0,0,0) | 像素着色器 | C7A1EA12-23C54A54-A851BCD1-5A2C784C |
| MP_MooaEncodedAttribute3 | Float4 | (0,0,0,0) | 像素着色器 | C1C0BC01-1AC73C53-B152BAB5-9A21389B |
| MP_MooaEncodedAttribute4 | Float4 | (0,0,0,0) | 像素着色器 | C320DA20-15D75E62-D252BCD7-9A22489D |

**文件位置**: `Engine/Source/Runtime/Engine/Private/Materials/MaterialAttributeDefinitionMap.cpp:280-284`

```cpp
Add(FGuid(0xC1A0EA03, 0x15C74A55, 0xB852BAD5, 0x9A20789A), 
    TEXT("MooaEncodedAttribute0"), MP_MooaEncodedAttribute0, 
    MCT_Float4, FVector4(0,0,0,0), SF_Pixel);

Add(FGuid(0xC6A0AB14, 0x15D75A65, 0xB132ABD4, 0x0B20789C), 
    TEXT("MooaEncodedAttribute1"), MP_MooaEncodedAttribute1, 
    MCT_Float4, FVector4(0,0,0,0), SF_Pixel);

Add(FGuid(0xC7A1EA12, 0x23C54A54, 0xA851BCD1, 0x5A2C784C), 
    TEXT("MooaEncodedAttribute2"), MP_MooaEncodedAttribute2, 
    MCT_Float4, FVector4(0,0,0,0), SF_Pixel);

Add(FGuid(0xC1C0BC01, 0x1AC73C53, 0xB152BAB5, 0x9A21389B), 
    TEXT("MooaEncodedAttribute3"), MP_MooaEncodedAttribute3, 
    MCT_Float4, FVector4(0,0,0,0), SF_Pixel);

Add(FGuid(0xC320DA20, 0x15D75E62, 0xD252BCD7, 0x9A22489D), 
    TEXT("MooaEncodedAttribute4"), MP_MooaEncodedAttribute4, 
    MCT_Float4, FVector4(0,0,0,0), SF_Pixel);
```

### 2. 材质属性在HLSLMaterialTranslator中的编译

**文件位置**: `Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp:1632-1639`

```cpp
if (MaterialShadingModels.HasShadingModel(MSM_Toon))
{
    Chunk[MP_MooaEncodedAttribute0] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute0, this);
    Chunk[MP_MooaEncodedAttribute1] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute1, this);
    Chunk[MP_MooaEncodedAttribute2] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute2, this);
    Chunk[MP_MooaEncodedAttribute3] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute3, this);
    Chunk[MP_MooaEncodedAttribute4] = Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute4, this);
}
```

---

## MooaEncodedAttribute映射

### FToonGBufferData → MooaEncodedAttribute映射

让我们看一下这5个Float4属性是如何与FToonGBufferData对应的：

**文件位置**: `Engine/Shaders/Private/ToonShadingCommon.ush:242-262`

```hlsl
void DecodeToonGBufferDataFromEncodedAttributes(
    out float4 MooaEncodedAttribute0, out float4 MooaEncodedAttribute1, 
    out float4 MooaEncodedAttribute2, out float4 MooaEncodedAttribute3, 
    out float4 MooaEncodedAttribute4)
{
    MooaEncodedAttribute0.x    = ToonGBuffer.ShadingFeatureID;
    MooaEncodedAttribute0.yzw  = ToonGBuffer.MainLightShadowColor;
    
    MooaEncodedAttribute1.x  = ToonGBuffer.DiffuseColorRampIndex;
    MooaEncodedAttribute1.y  = ToonGBuffer.DiffuseColorRampUVOffset;
    MooaEncodedAttribute1.zw = 0;
    
    MooaEncodedAttribute2.xyz  = ToonGBuffer.SpecularColor;
    MooaEncodedAttribute2.w    = ToonGBuffer.SpecularColorRampIndex;
    
    MooaEncodedAttribute3.x  = ToonGBuffer.SpecularColorRampUVOffset;
    MooaEncodedAttribute3.y  = ToonGBuffer.ReflectionIntensity;
    MooaEncodedAttribute3.z  = ToonGBuffer.RimLightIntensity;
    MooaEncodedAttribute3.w  = ToonGBuffer.RimLightWidth;
    
    MooaEncodedAttribute4.x  = ToonGBuffer.FacialShadowSdfLeft;
    MooaEncodedAttribute4.y  = ToonGBuffer.FacialShadowSdfRight;
    MooaEncodedAttribute4.z  = ToonGBuffer.Stencil;
    MooaEncodedAttribute4.w  = ToonGBuffer.RayTracingShadowFlag;
}
```

### 详细映射表

| 属性 | Float4通道 | FToonGBufferData字段 | 说明 |
|------|-----------|----------------------|------|
| **MooaEncodedAttribute0** | x | ShadingFeatureID | 着色特性ID |
| | yzw | MainLightShadowColor | 主光源阴影颜色 |
| **MooaEncodedAttribute1** | x | DiffuseColorRampIndex | 漫反射Ramp索引 |
| | y | DiffuseColorRampUVOffset | 漫反射Ramp UV偏移 |
| | zw | 0 | 保留 |
| **MooaEncodedAttribute2** | xyz | SpecularColor | 高光颜色 |
| | w | SpecularColorRampIndex | 高光Ramp索引 |
| **MooaEncodedAttribute3** | x | SpecularColorRampUVOffset | 高光Ramp UV偏移 |
| | y | ReflectionIntensity | 反射强度 |
| | z | RimLightIntensity | 边缘光强度 |
| | w | RimLightWidth | 边缘光宽度 |
| **MooaEncodedAttribute4** | x | FacialShadowSdfLeft | 面部阴影SDF（左） |
| | y | FacialShadowSdfRight | 面部阴影SDF（右） |
| | z | Stencil | 模板值 |
| | w | RayTracingShadowFlag | 光线追踪阴影标志 |

---

## 完整数据编码流程

### 流程1：材质属性 → FToonGBufferData → ToonBufferA

```
材质编辑器
    ↓
MooaEncodedAttribute0-4 (5个Float4)
    ↓
BasePassPixelShader.usf:1083-1087
    ↓
DecodeToonGBufferDataFromEncodedAttributes()
    ↓
FToonGBufferData
    ↓
EncodeToonGBufferDataToMRT()
    ↓
ToonBufferA (RGBA Float16)
    ↓
GBuffer
```

### 流程2：ToonBufferA → FToonGBufferData → 着色

```
GBuffer
    ↓
ToonBufferA (RGBA Float16)
    ↓
DecodeToonGBufferDataFromMRT()
    ↓
FToonGBufferData
    ↓
ToonBxDF()
    ↓
最终渲染
```

### EncodeToonGBufferDataToMRT函数

**文件位置**: `Engine/Shaders/Private/ToonShadingCommon.ush:309-330`

这是将FToonGBufferData编码到ToonBufferA的函数（与ToonBufferA编码格式一致）。

---

## 着色模型集成

### 1. EMaterialShadingModel枚举

**文件位置**: `Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h:661-683`

```cpp
enum EMaterialShadingModel : int
{
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
    MSM_Toon                     UMETA(DisplayName="Toon"), // Mooa Toon Shading Model
    MSM_NUM                      UMETA(Hidden),
    MSM_FromMaterialExpression   UMETA(DisplayName="From Material Expression"),
    MSM_MAX
};
```

### 2. 着色器编译定义

**文件位置**: `Engine/Source/Runtime/Engine/Private/ShaderCompiler/ShaderGenerationUtil.cpp:135`

```cpp
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_TOON); // Mooa Toon Shading Model
```

### 3. 材质编译器集成

**文件位置**: `Engine/Source/Runtime/Engine/Private/Materials/Material.cpp` (多处)

```cpp
// 各向异性支持
bool bUsesAnisotropy = MaterialResource->GetShadingModels().HasAnyShadingModel(
    { MSM_DefaultLit, MSM_ClearCoat, MSM_Toon }) && // Mooa Anisotropy
    ...;

// 光线追踪阴影
if (GetShadingModels().HasShadingModel(MSM_Toon)) // Mooa Ray Tracing Shadow
    ...;
```

---

## HybriToon接口设计

### 方案A：通过MooaEncodedAttribute注入（推荐）

**优点**: 
- 不需要修改引擎
- 可以在现有MooaToon上叠加
- 完全兼容材质系统

**架构**:
```
神经风格编码器
    ↓
MooaEncodedAttribute0-4 (5个Float4)
    ↓
现有MooaToon渲染管线
    ↓
最终风格化图像
```

**材质参数映射**:
- **MooaEncodedAttribute0.x**: 神经风格ID
- **MooaEncodedAttribute0.yzw**: 风格化阴影颜色
- **MooaEncodedAttribute1.x**: 风格化Ramp索引
- **MooaEncodedAttribute1.y**: 风格化UV偏移
- ...等等

### 方案B：扩展MooaEncodedAttribute（新增5-9）

**优点**:
- 不破坏现有MooaToon
- 有足够空间注入神经特征

**架构**:
```
MP_MooaEncodedAttribute0-4: 原有MooaToon参数
MP_MooaEncodedAttribute5-9: 新增神经特征 (5个Float4)
    ↓
融合到ToonBxDF中
```

### 方案C：新增专用神经材质属性

**优点**:
- 命名清晰
- 完全独立

**需要新增**:
- MP_HybriToonNeuralFeature0
- MP_HybriToonNeuralFeature1
- ...

---

## 完整修改文件清单

### 引擎核心修改（完整列表）

| 序号 | 文件 | 功能 |
|------|------|------|
| 1 | `Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h` | 新增MSM_Toon着色模型枚举 |
| 2 | `Engine/Source/Runtime/Engine/Public/SceneTypes.h` | 新增MP_MooaEncodedAttribute0-4材质属性 |
| 3 | `Engine/Source/Runtime/Engine/Private/Materials/MaterialAttributeDefinitionMap.cpp` | 注册MooaEncodedAttribute0-4 |
| 4 | `Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.cpp` | 编译MooaEncodedAttribute0-4 |
| 5 | `Engine/Source/Runtime/Engine/Private/ShaderCompiler/ShaderGenerationUtil.cpp` | 新增MATERIAL_SHADINGMODEL_TOON定义 |
| 6 | `Engine/Source/Runtime/Engine/Private/Materials/Material.cpp` | 集成Toon着色模型到材质系统 |
| 7 | `Engine/Source/Runtime/Engine/Public/Subsystems/MooaToonSubsystem.h` | MooaToon子系统（Ramp管理） |
| 8 | `Engine/Source/Runtime/Engine/Private/Subsystems/MooaToonSubsystem.cpp` | MooaToon子系统实现 |
| 9 | `Engine/Source/Runtime/Engine/Public/MooaToonSettings.h` | MooaToon设置 |
| 10 | `Engine/Source/Runtime/Engine/Private/MooaToonSettings.cpp` | MooaToon设置实现 |
| 11 | `Engine/Source/Runtime/Renderer/Public/SceneRenderTargetParameters.h` | 新增ToonBufferA位标志 |
| 12 | `Engine/Source/Runtime/Renderer/Private/SceneTextures.cpp` | 创建ToonBufferA纹理 |
| 13 | `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp` | CVar定义和参数传递 |
| 14 | `Engine/Source/Runtime/Renderer/Private/SceneRendering.h` | FFastVramConfig中新增ToonBufferA |
| 15 | `Engine/Shaders/Private/ToonShadingCommon.ush` | Toon GBuffer编码/解码 |
| 16 | `Engine/Shaders/Private/ToonShadingModel.ush` | Toon BxDF实现 |
| 17 | `Engine/Shaders/Private/MaterialTemplate.ush` | GetMaterialMooaEncodedAttributeX函数 |
| 18 | `Engine/Shaders/Private/BasePassPixelShader.usf` | 调用EncodeToonGBufferDataToMRT |
| 19 | `Engine/Shaders/Private/AnisotropyPassShader.usf` | MooaEncodedAttribute支持 |
| 20 | `Engine/Shaders/Private/RayTracing/RayTracingMaterialHitShaders.usf` | 光线追踪支持 |

**总计**: 20个核心文件被修改！

---

## 总结

### 关键发现

1. **MooaToon不是纯插件，是深度魔改引擎**
   - 修改了20+个核心引擎文件
   - 新增着色模型、材质属性、GBuffer

2. **5个MooaEncodedAttribute是与材质系统的接口**
   - 在材质编辑器中暴露
   - 映射到FToonGBufferData
   - 编码到ToonBufferA

3. **HybriToon的最佳切入点是MooaEncodedAttribute**
   - 方案A：通过MooaEncodedAttribute0-4注入
   - 不需要修改引擎
   - 完全兼容现有MooaToon

---

## 参考资料

- MooaToon GitHub: https://github.com/JasonMa0012/MooaToon
- MooaToon 文档: https://mooatoon.com/docs/

---

**报告生成时间**: 2026年4月6日
**报告版本**: v3.0
**分析深度**: 材质系统级深度分析
