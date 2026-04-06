# MooaToon 技术详细报告2 - 引擎深度分析

## 目录
1. [引擎架构总览](#引擎架构总览)
2. [核心模块详解](#核心模块详解)
3. [GBuffer扩展设计](#gbuffer扩展设计)
4. [Toon着色模型实现](#toon着色模型实现)
5. [Ramp纹理系统](#ramp纹理系统)
6. [控制台变量系统](#控制台变量系统)
7. [后期处理参数](#后期处理参数)
8. [HybriToon深度魔改方案](#hybritoon深度魔改方案)

---

## 引擎架构总览

### MooaToon引擎修改概览

MooaToon不是一个纯插件，而是**深度修改UE5引擎源代码**的定制版本。核心修改包括：

| 模块 | 修改位置 | 功能 |
|------|---------|------|
| GBuffer扩展 | `SceneRenderTargetParameters.h` | 新增ToonBufferA |
| 渲染管线 | `SceneRendering.cpp/h` | 新增CVar和参数传递 |
| 着色器 | `ToonShadingCommon.ush` | 核心着色逻辑 |
| 着色器 | `ToonShadingModel.ush` | Toon BxDF实现 |
| 子系统 | `MooaToonSubsystem.h/cpp` | Ramp纹理管理 |
| 设置 | `MooaToonSettings.h/cpp` | 全局配置 |

### 关键文件位置

```
D:\MooaToon\MooaToon-Engine\Engine\
├── Source\Runtime\
│   ├── Engine\
│   │   ├── Public\Subsystems\MooaToonSubsystem.h
│   │   ├── Public\MooaToonSettings.h
│   │   └── Private\
│   │       ├── Subsystems\MooaToonSubsystem.cpp
│   │       └── MooaToonSettings.cpp
│   └── Renderer\
│       ├── Private\
│       │   ├── SceneRendering.cpp
│       │   ├── SceneRendering.h
│       │   ├── SceneTextures.cpp
│       │   └── SceneTextureParameters.h
│       └── Public\SceneRenderTargetParameters.h
└── Shaders\Private\
    ├── ToonShadingCommon.ush
    └── ToonShadingModel.ush
```

---

## 核心模块详解

### 1. MooaToonSubsystem - Ramp纹理管理

**文件**: `Engine/Source/Runtime/Engine/Public/Subsystems/MooaToonSubsystem.h`

```cpp
UCLASS()
class ENGINE_API UMooaToonSubsystem : public UEngineSubsystem
{
    GENERATED_BODY()

public:
    virtual void Initialize(FSubsystemCollectionBase& Collection) override;
    virtual void Deinitialize() override;

    UPROPERTY()
    TObjectPtr<UCurveLinearColorAtlas> GlobalDiffuseColorRampAtlas;

    UPROPERTY()
    TObjectPtr<UCurveLinearColorAtlas> GlobalSpecularColorRampAtlas;
};
```

**功能**:
- 引擎子系统，生命周期与引擎同步
- 管理全局漫反射Ramp纹理图集
- 管理全局高光Ramp纹理图集
- 自动从设置中加载纹理资源

### 2. MooaToonSettings - 全局配置

**文件**: `Engine/Source/Runtime/Engine/Public/MooaToonSettings.h`

```cpp
UCLASS(config = Engine, defaultconfig, meta = (DisplayName = "MooaToon"))
class ENGINE_API UMooaToonSettings : public UDeveloperSettings
{
    GENERATED_UCLASS_BODY()

public:
    UPROPERTY(config, EditAnywhere, Category = "MooaToon Settings")
    FSoftObjectPath GlobalDiffuseColorRampAtlas = 
        FSoftObjectPath(TEXT("/MooaToon/Assets/DiffuseColorRamps/CA_GlobalDiffuseColorRampAtlas.CA_GlobalDiffuseColorRampAtlas"));

    UPROPERTY(config, EditAnywhere, Category = "MooaToon Settings")
    FSoftObjectPath GlobalSpecularColorRampAtlas = 
        FSoftObjectPath(TEXT("/MooaToon/Assets/SpecularColorRamps/CA_GlobalSpecularColorRampAtlas.CA_GlobalSpecularColorRampAtlas"));
};
```

**功能**:
- 继承自`UDeveloperSettings`，在编辑器项目设置中显示
- 配置全局漫反射Ramp图集路径
- 配置全局高光Ramp图集路径
- 支持编辑器中实时修改并重新加载

---

## GBuffer扩展设计

### 1. GBuffer扩展 - ToonBufferA

**文件**: `Engine/Source/Runtime/Renderer/Public/SceneRenderTargetParameters.h`

```cpp
// Mooa GBuffer
ToonBufferA        = 1 << 9,
GBuffers        = GBufferA | GBufferB | GBufferC | GBufferD | GBufferE | GBufferF | ToonBufferA ,
// Mooa End
```

**修改点**:
- 新增`ToonBufferA`位标志（第9位）
- 将`ToonBufferA`加入`GBuffers`组合标志

### 2. ToonBufferA纹理创建

**文件**: `Engine/Source/Runtime/Renderer/Private/SceneTextures.cpp`

```cpp
// Mooa GBuffer
if (Bindings.ToonBufferA.Index >= 0)
{
    const FRDGTextureDesc Desc(FRDGTextureDesc::Create2D(
        Config.Extent, 
        Bindings.ToonBufferA.Format, 
        FClearValueBinding::Transparent, 
        Bindings.ToonBufferA.Flags | FlagsToAdd | GFastVRamConfig.ToonBufferA));
    SceneTextures.ToonBufferA = GraphBuilder.CreateTexture(Desc, TEXT("ToonBufferA"));
}
// Mooa End
```

**纹理格式**: RGBA Float 16（半精度浮点数）

### 3. ToonBufferA数据编码格式

**文件**: `Engine/Shaders/Private/ToonShadingCommon.ush:294-308`

```
ToonBufferA (RGBA Float 16)
x: SpecularColor.r(8)        SpecularColor.g(8)
y: SpecularColor.b(8)        SpecularColorRampUVOffset(8)
z: DiffuseColorRampUVOffset(8)    RimLightIntensity(4)    RimLightWidth(4)
w: DiffuseColorRampIndex(6)    SpecularColorRampIndex(5)    Stencil(5)

CustomData (GBufferD) (RGBA Float 8)
x: MainLightShadowColor.r(8)
y: MainLightShadowColor.g(8)
z: MainLightShadowColor.b(8)
w: ReflectionIntensity(4)    ShadingFeatureID(2)    RayTracingShadowFlag(2)

Metallic:    FacialShadowSdfLeft(8)
Anisotropy:  FacialShadowSdfRight(8)
```

**编码策略**:
- 充分利用Float16的精度
- 多个小参数打包进一个通道
- 复用GBufferD的CustomData、Metallic、Anisotropy通道

### 4. FToonGBufferData结构

**文件**: `Engine/Shaders/Private/ToonShadingCommon.ush:212-236`

```hlsl
struct FToonGBufferData
{
    uint ShadingFeatureID;
    
    // Diffuse
    float3 MainLightShadowColor;
    uint DiffuseColorRampIndex;
    float DiffuseColorRampUVOffset;

    // Specular
    float3 SpecularColor;
    uint SpecularColorRampIndex;
    float SpecularColorRampUVOffset;
    float ReflectionIntensity;
    float RimLightIntensity;
    float RimLightWidth;

    // Distance Field Facial Shadow
    float FacialShadowSdfLeft;
    float FacialShadowSdfRight;

    // Ray Tracing Shadow
    uint Stencil;
    uint RayTracingShadowFlag;
};
```

---

## Toon着色模型实现

### 1. 着色特性ID

**文件**: `Engine/Shaders/Private/ToonShadingCommon.ush:48-53`

```hlsl
#define MOOA_SHADING_FEATURE_ID_DEFAULT                         0
#define MOOA_SHADING_FEATURE_ID_PBR_SPECULAR                    1
// Require Tangents:
#define MOOA_SHADING_FEATURE_ID_KAJIYA_HAIR_SPECULAR            2
#define MOOA_SHADING_FEATURE_ID_DISTANCE_FIELD_FACIAL_SHADOW    3
```

**特性说明**:
- **DEFAULT**: 默认卡通渲染
- **PBR_SPECULAR**: 使用PBR高光（而非Ramp）
- **KAJIYA_HAIR_SPECULAR**: Kajiya-Kay头发高光（需要切线）
- **DISTANCE_FIELD_FACIAL_SHADOW**: 距离场面部阴影

### 2. 光线追踪阴影标志

**文件**: `Engine/Shaders/Private/ToonShadingCommon.ush:58-62`

```hlsl
#define MOOA_RAY_TRACING_SHADOW_FLAG_NONE                           0
#define MOOA_RAY_TRACING_SHADOW_FLAG_FACE                           1
#define MOOA_RAY_TRACING_SHADOW_FLAG_FACE_SCREEN_SPACE_HAIR_SHADOW    2
#define MOOA_RAY_TRACING_SHADOW_FLAG_HAIR                            3
```

### 3. ToonStep函数 - 卡通阶梯

**文件**: `Engine/Shaders/Private/ToonShadingCommon.ush:80-88`

```hlsl
float ToonStep(float Gradient, float Feather = 0.0f, float Threshold = 0.5f, 
               float FeatherPower = 2.0f, bool UseSaturate = true, bool UseSmoothstep = true)
{
    Feather = pow(saturate(Feather), FeatherPower);
    float Min = UseSaturate ? saturate(Threshold - Feather) : (Threshold - Feather);
    float Max = UseSaturate ? saturate(Threshold + Feather) : (Threshold + Feather);
    return UseSmoothstep
        ? smoothstep(Min, Max, Gradient)
        : InverseLerp(Min, Max, Gradient);
}
```

**参数说明**:
- `Gradient`: 输入梯度值（如NoL）
- `Feather`: 羽化宽度
- `Threshold`: 阈值
- `FeatherPower`: 羽化幂次（控制边缘硬度）
- `UseSaturate`: 是否使用saturate钳制
- `UseSmoothstep`: 是否使用smoothstep（或线性插值）

### 4. ToonBxDF - 核心着色函数

**文件**: `Engine/Shaders/Private/ToonShadingModel.ush:272-410`

这是MooaToon的核心函数，让我们分段分析：

#### 4.1 初始化

```hlsl
FDirectLighting ToonBxDF(FGBufferData GBuffer, half3 N, half3 V, half3 L, 
                          float Falloff, float NoL, FAreaLight AreaLight, FShadowTerms Shadow)
{
    FMooaToonContext MooaToonContext = GBuffer.MooaToonContext;
    FToonGBufferData ToonGBuffer = GBuffer.MooaToonContext.ToonGBuffer;
    FDirectLighting Lighting = (FDirectLighting)0;
    float3 LightColorAndAttenuation = AreaLight.FalloffColor * MooaToonContext.LightColor * Falloff;
    ColorSaturationPowerAndScale(LightColorAndAttenuation, View.MooaLightSaturationScale);
```

#### 4.2 坐标系统准备

```hlsl
    const float NoL_Full = dot(N, L);
    const float NoL_Half = NoL_Full * 0.5f + 0.5f;
    const float3 H = normalize(V + L);
    const float2 BufferUV = SvPositionToBufferUV(float4(MooaToonContext.PixelPos, 0, 0));
    const float2 ViewportUV = BufferUVToViewportUV(BufferUV);
    const float3 L_ClipSpace = mul(L, (float3x3)View.TranslatedWorldToClip).xyz;
    const float2 L_ViewportSpace = (L_ClipSpace.xy * float2(0.5, -0.5));
    const float3 N_ClipSpace = mul(N, (float3x3)View.TranslatedWorldToClip).xyz;
    const float2 N_ViewportSpace = (N_ClipSpace.xy * float2(0.5, -0.5));
    const float ViewportSpaceToWorldSpaceDir = rcp(GBuffer.Depth);
```

#### 4.3 漫反射计算（核心！）

```hlsl
    // Diffuse
    {
        const bool EnablePostRampShadow = View.MooaDiffuseColorRampEnablePostRampShadow;
        const bool EnablePostRampMaterialAO = View.MooaDiffuseColorRampEnablePostRampMaterialAO;

        // 计算阴影梯度
        float DiffuseColorRampUVOffset = (ToonGBuffer.DiffuseColorRampUVOffset * 2.0f - 1.0f) * View.MooaDiffuseColorRampUVOffsetMaxRange;
        float ShadowGradient = saturate(NoL_Half + DiffuseColorRampUVOffset);
        float HairLinearShadowGradient = GetScreenSpaceDepthTestHairShadow(...);
        
        ApplyDistanceFieldFacialShadow(MooaToonContext, GBuffer, ToonGBuffer, L, DiffuseColorRampUVOffset,
            /*inout*/ ShadowGradient);

        // 合并阴影梯度并采样漫反射Ramp
        float CustomLinearShadowGradient = min(
            EnablePostRampMaterialAO ? 1.0f : GBuffer.GBufferAO,
            EnablePostRampShadow ? 1.0f : min(Shadow.SurfaceShadow, HairLinearShadowGradient));
        
        float DiffuseColorRampU = min(ShadowGradient, LinearToDotProductSpace(CustomLinearShadowGradient));
        
        // RGB: 漫反射颜色，A: 漫反射颜色与阴影颜色的混合因子
        half4 DiffuseColorRamp = SampleGlobalRamp(
            View.MooaGlobalDiffuseColorRampAtlas, 
            DiffuseColorRampU, 
            ToonGBuffer.DiffuseColorRampIndex, 
            View.MooaGlobalDiffuseColorRampAtlasHeight);
        
        float DiffuseShadow = min3(
            DiffuseColorRamp.a,
            EnablePostRampMaterialAO ? GBuffer.GBufferAO : 1.0f,
            EnablePostRampShadow ? min(Shadow.SurfaceShadow, HairLinearShadowGradient) : 1.0f);

        // 混合漫反射颜色和阴影颜色
        half3 ToonDiffuseColor = GBuffer.DiffuseColor;
        half3 ToonShadowColor = ToonGBuffer.MainLightShadowColor * (1 - GBuffer.Metallic) * GetShadowColorIntensity(MooaToonContext);
        ToonDiffuseColor = lerp(ToonShadowColor, ToonDiffuseColor, DiffuseShadow);

        // 输出
        Lighting.Diffuse = Diffuse_Lambert(ToonDiffuseColor) * DiffuseColorRamp.rgb * LightColorAndAttenuation;
        Shadow.SurfaceShadow = min(Shadow.SurfaceShadow, HairLinearShadowGradient);
    }
```

**漫反射流程**:
1. 计算阴影梯度（NoL + UV偏移）
2. 应用头发阴影和面部阴影
3. 采样漫反射Ramp纹理
4. 混合基础色和阴影色
5. 输出最终漫反射

#### 4.4 各向异性BxDF上下文

```hlsl
    // Anisotropy Specular BxDF Context
    bool bHasAnisotropy;
    BxDFContext Context = (BxDFContext)0;
    {
    #if SUPPORTS_ANISOTROPIC_MATERIALS
        bHasAnisotropy = HasAnisotropy(GBuffer.SelectiveOutputMask) && GBuffer.Anisotropy != 0;
    #else
        bHasAnisotropy = false;
    #endif
        if (ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_DISTANCE_FIELD_FACIAL_SHADOW)
            bHasAnisotropy = false;
    
        BRANCH if (bHasAnisotropy)
        {
            half3 X = GBuffer.WorldTangent;
            half3 Y = normalize(cross(N, X));
            Init(Context, N, X, Y, V, L);
        }
        else
        {
            Init(Context, N, V, L);
            GBuffer.Anisotropy = 0;
        }
        Context.NoV = saturate(abs( Context.NoV ) + 1e-5);
    }
```

#### 4.5 高光计算

```hlsl
    // Specular
    BRANCH if (ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_PBR_SPECULAR)
    {
        // PBR高光模式
        Lighting.Specular = LightColorAndAttenuation * Shadow.SurfaceShadow * NoL * 
            SpecularGGX(GBuffer.Roughness, GBuffer.Anisotropy, GBuffer.SpecularColor, Context, NoL, AreaLight);
    }
    else
    {
        // Ramp高光模式
        float MaxSpecularValue;
        float SpecularColorRampU = GetSpecularColorRampUAndMaxSpecularValue(GBuffer, Context, N, H,
            /*out*/ MaxSpecularValue);
        float SpecularColorRampUVOffset = (ToonGBuffer.SpecularColorRampUVOffset * 2.0f - 1.0f) * View.MooaSpecularColorRampUVOffsetMaxRange;
        half3 SpecularColor = SampleGlobalRamp(
            View.MooaGlobalSpecularColorRampAtlas, 
            SpecularColorRampU + SpecularColorRampUVOffset, 
            ToonGBuffer.SpecularColorRampIndex,  
            View.MooaGlobalSpecularColorRampAtlasHeight).rgb;

        Lighting.Specular = GBuffer.SpecularColor * ToonGBuffer.SpecularColor * MaxSpecularValue * 
            SpecularColor * LightColorAndAttenuation * Shadow.SurfaceShadow;
    }
```

#### 4.6 边缘光计算

```hlsl
    // Rimlight
    float3 RimLight = GetScreenSpaceDepthTestRimlightColor(...) * LightColorAndAttenuation * Shadow.SurfaceShadow;
    ColorSaturationPowerAndScale(RimLight, View.MooaRimLightSaturationScale, View.MooaRimLightIntensity);
    Lighting.Specular += RimLight;

    return Lighting;
}
```

### 5. Kajiya-Kay头发高光

**文件**: `Engine/Shaders/Private/ToonShadingModel.ush:176-192`

```hlsl
BRANCH if (GBuffer.MooaToonContext.ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_KAJIYA_HAIR_SPECULAR)
{
    MaxSpecularValue = PI;
    
    // Kajiya Kay Hair Specular
    half ToH = 1 - abs(dot(GBuffer.WorldTangent, H));

    // 经验公式：将二分高光大小近似为PBR高光
    const float RoughnessGradientPow = 4.0f;
    const float SpecularGradientPowMin = 2.0f;
    const float SpecularGradientPowMax = 128.0f;
    half SpecularGradientPow = Remap01(
        pow(1 - GBuffer.Roughness, RoughnessGradientPow),
        SpecularGradientPowMin,
        SpecularGradientPowMax);
    return pow(ToH, SpecularGradientPow);
}
```

**Kajiya-Kay原理**:
- 使用切线方向计算高光
- `ToH = 1 - abs(dot(Tangent, H))`
- 通过幂次控制高光范围

### 6. 距离场面部阴影

**文件**: `Engine/Shaders/Private/ToonShadingModel.ush:156-170`

```hlsl
void ApplyDistanceFieldFacialShadow(FMooaToonContext MooaToonContext, FGBufferData GBuffer, 
                                    FToonGBufferData ToonGBuffer, float3 L, float DiffuseColorRampUVOffset,
                                    inout float FacialShadowGradient)
{
    BRANCH if (ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_DISTANCE_FIELD_FACIAL_SHADOW
        && GetEnableDistanceFieldFacial(MooaToonContext))
    {
        float3 FaceForwardDir = GBuffer.WorldTangent;
        float LightAngle = RadianToDegree(FastACos(dot(FaceForwardDir, L)));
        float RampUVOffsetByLightAngle = (1.0 - clamp(LightAngle / 180.0f, 1e-4, 1 - 1e-4)) - 0.5f;
        bool bLightAtRight = cross(FaceForwardDir, L).z >= 0;
        float shadowSdf = bLightAtRight ? ToonGBuffer.FacialShadowSdfRight : ToonGBuffer.FacialShadowSdfLeft;

        FacialShadowGradient = LinearToDotProductSpace(RampUVOffsetByLightAngle + shadowSdf + DiffuseColorRampUVOffset);
    }
}
```

**面部阴影原理**:
1. 使用世界切线作为面部朝向
2. 计算光源与面部的夹角
3. 根据光源在左侧/右侧选择对应的SDF
4. 将SDF值转换为阴影梯度

### 7. 屏幕空间头发阴影

**文件**: `Engine/Shaders/Private/ToonShadingModel.ush:129-154`

```hlsl
half GetScreenSpaceDepthTestHairShadow(FMooaToonContext MooaToonContext, FGBufferData GBuffer, 
                                        FToonGBufferData ToonGBuffer, float2 ViewportUV, 
                                        float2 L_ViewportSpace, float ViewportSpaceToWorldSpaceDir)
{
    float HairShadow = 1;
#if SHADING_PATH_DEFERRED && defined(MOOA_TOON_DEFERRED_LIGHTING) && !SUBSTRATE_ENABLED
    BRANCH if (!GetEnableScreenSpaceHairShadow(MooaToonContext))
        return HairShadow;
    
    const float HairShadowWidth = 2.0f * View.MooaHairShadowWidth;
    const float HairShadowIntensity = View.MooaHairShadowIntensity;
    BRANCH if(ToonGBuffer.RayTracingShadowFlag == MOOA_RAY_TRACING_SHADOW_FLAG_FACE_SCREEN_SPACE_HAIR_SHADOW &&
        HairShadowWidth > 0 && HairShadowIntensity > 0)
    {
        float2 ViewportUVOffset = L_ViewportSpace * ViewportSpaceToWorldSpaceDir * HairShadowWidth;
        float2 TargetBufferUV = ViewportUVToBufferUV(saturate(ViewportUV + ViewportUVOffset));
        FGBufferData HairGbuffer = GetGBufferData(TargetBufferUV);
        float DepthFade = saturate(max(0, GBuffer.Depth - HairGbuffer.Depth - View.MooaHairShadowDepthTestThreshold) 
            / max(1e-5, View.MooaHairShadowDepthTestFadeDistance));

        if (HairGbuffer.ShadingModelID == SHADINGMODELID_TOON &&
            HairGbuffer.MooaToonContext.ToonGBuffer.RayTracingShadowFlag == MOOA_RAY_TRACING_SHADOW_FLAG_HAIR)
        {
            HairShadow -= saturate(HairShadowIntensity * DepthFade);
        }
    }
#endif
    return HairShadow;
}
```

**头发阴影原理**:
1. 沿光源方向偏移UV
2. 采样偏移位置的GBuffer
3. 检查该位置是否是头发（通过RayTracingShadowFlag）
4. 根据深度差计算阴影衰减

---

## Ramp纹理系统

### 1. Ramp采样函数

**文件**: `Engine/Shaders/Private/ToonShadingModel.ush:21-24`

```hlsl
half4 SampleGlobalRamp(Texture2D Tex, float U, uint index, float rampHeight)
{
    return Texture2DSampleLevel(Tex, GlobalBilinearClampedSampler, 
        float2(U, float(index + 0.5f) / rampHeight), 0);
}
```

**采样方式**:
- `U`: 水平采样坐标（0-1）
- `index`: Ramp索引（选择第几条Ramp）
- `rampHeight`: Ramp图集高度
- 使用双线性采样和钳址寻址

### 2. 线性到点积空间转换

**文件**: `Engine/Shaders/Private/ToonShadingModel.ush:65-68`

```hlsl
float LinearToDotProductSpace(float gradient01)
{
    return saturate(cos(saturate(gradient01) * PI) * -0.5f + 0.5f);
}
```

**为什么需要这个转换**:
- NoL = dot(N, L) = cos(θ)，不是线性的
- 为了让Ramp采样结果一致，需要将线性梯度转换到相同的cos空间

### 3. Ramp图集管理（C++）

**文件**: `Engine/Source/Runtime/Engine/Private/Subsystems/MooaToonSubsystem.cpp`

```cpp
void UMooaToonSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);
    auto MooaToonSettings = GetDefault<UMooaToonSettings>();
    GlobalDiffuseColorRampAtlas = (UCurveLinearColorAtlas*)MooaToonSettings->GlobalDiffuseColorRampAtlas.TryLoad();
    GlobalSpecularColorRampAtlas = (UCurveLinearColorAtlas*)MooaToonSettings->GlobalSpecularColorRampAtlas.TryLoad();
}
```

---

## 控制台变量系统

### 1. 调试与AO

**文件**: `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:121-126`

```cpp
static TAutoConsoleVariable<float> CMooaDebugValue(
    TEXT("r.Mooa.DebugValue"), 0.0f,
    TEXT("Set it to 0 to disable Debug View"), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaDynamicAOIntensity(
    TEXT("r.Mooa.AmbientOcclusion.DynamicAOIntensity"), 0.0f,
    TEXT("Dynamic AO Intensity"), ECVF_RenderThreadSafe);
```

### 2. 漫反射Ramp

**文件**: `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:128-135`

```cpp
static TAutoConsoleVariable<int32> CMooaDiffuseColorRampEnablePostRampShadow(
    TEXT("r.Mooa.DiffuseColorRamp.EnablePostRampShadow"), 0,
    TEXT("If enabled, Shadows will not affect the Diffuse Color Ramp samples."), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<int32> CMooaDiffuseColorRampEnablePostRampMaterialAO(
    TEXT("r.Mooa.DiffuseColorRamp.EnablePostRampMaterialAO"), 0,
    TEXT("If enabled, Material AO will not affect the Diffuse Color Ramp samples."), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaDiffuseColorRampUVOffsetMaxRange(
    TEXT("r.Mooa.DiffuseColorRamp.UVOffsetMaxRange"), 0.25f,
    TEXT("Diffuse Color Ramp UV Offset Max Range"), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaSpecularColorRampUVOffsetMaxRange(
    TEXT("r.Mooa.SpecularColorRamp.UVOffsetMaxRange"), 0.25f,
    TEXT("Specular Color Ramp UV Offset Max Range"), ECVF_RenderThreadSafe);
```

### 3. 边缘光

**文件**: `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:137-142`

```cpp
static TAutoConsoleVariable<float> CMooaRimLightMaxWidth(
    TEXT("r.Mooa.RimLight.MaxWidth"), 1.0f,
    TEXT("Screen Space RimLight Max Width"), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaRimLightDepthTestThreshold(
    TEXT("r.Mooa.RimLight.DepthTestThreshold"), 2.0f,
    TEXT("Screen Space RimLight Depth Test Threshold"), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaRimLightDepthTestFadeDistance(
    TEXT("r.Mooa.RimLight.DepthTestFadeDistance"), 10.0f,
    TEXT("Screen Space RimLight Depth Test Fade Distance"), ECVF_RenderThreadSafe);
```

### 4. 头发阴影

**文件**: `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:144-151`

```cpp
static TAutoConsoleVariable<float> CMooaHairShadowIntensity(
    TEXT("r.Mooa.HairShadow.Intensity"), 1.0f,
    TEXT("Hair Shadow Intensity"), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaHairShadowWidth(
    TEXT("r.Mooa.HairShadow.Width"), 1.0f,
    TEXT("Hair Shadow Width"), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaHairShadowDepthTestThreshold(
    TEXT("r.Mooa.HairShadow.DepthTestThreshold"), 0.0f,
    TEXT("Hair Shadow Depth Test Threshold"), ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaHairShadowDepthTestFadeDistance(
    TEXT("r.Mooa.HairShadow.DepthTestFadeDistance"), 5.0f,
    TEXT("Hair Shadow Depth Test Fade Distance"), ECVF_RenderThreadSafe);
```

---

## 后期处理参数

### 1. ViewUniformShaderParameters中的Mooa参数

**文件**: `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:1436-1469`

```cpp
ViewUniformShaderParameters.MooaDebugValue = CMooaDebugValue.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaDynamicAOIntensity = CMooaDynamicAOIntensity.GetValueOnAnyThread();

ViewUniformShaderParameters.MooaDiffuseColorRampEnablePostRampShadow = CMooaDiffuseColorRampEnablePostRampShadow.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaDiffuseColorRampEnablePostRampMaterialAO = CMooaDiffuseColorRampEnablePostRampMaterialAO.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaDiffuseColorRampUVOffsetMaxRange = CMooaDiffuseColorRampUVOffsetMaxRange.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaSpecularColorRampUVOffsetMaxRange = CMooaSpecularColorRampUVOffsetMaxRange.GetValueOnAnyThread();

ViewUniformShaderParameters.MooaRimLightMaxWidth = CMooaRimLightMaxWidth.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaRimLightDepthTestThreshold = CMooaRimLightDepthTestThreshold.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaRimLightDepthTestFadeDistance = CMooaRimLightDepthTestFadeDistance.GetValueOnAnyThread();

ViewUniformShaderParameters.MooaHairShadowIntensity = CMooaHairShadowIntensity.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaHairShadowWidth = CMooaHairShadowWidth.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaHairShadowDepthTestThreshold = CMooaHairShadowDepthTestThreshold.GetValueOnAnyThread();
ViewUniformShaderParameters.MooaHairShadowDepthTestFadeDistance = CMooaHairShadowDepthTestFadeDistance.GetValueOnAnyThread();

// 后期处理设置
ViewUniformShaderParameters.MooaExposureScale = FinalPostProcessSettings.MooaExposureScale;
ViewUniformShaderParameters.MooaLightSaturationScale = FinalPostProcessSettings.MooaLightSaturationScale;
ViewUniformShaderParameters.MooaGlobalIlluminationDirectionality = FinalPostProcessSettings.MooaGlobalIlluminationDirectionality;
ViewUniformShaderParameters.MooaGlobalIlluminationLumenNormalFlatten = FinalPostProcessSettings.MooaGlobalIlluminationLumenNormalFlatten;
ViewUniformShaderParameters.MooaGlobalIlluminationColor = FinalPostProcessSettings.MooaGlobalIlluminationColor;
ViewUniformShaderParameters.MooaGlobalIlluminationIntensity = FinalPostProcessSettings.MooaGlobalIlluminationIntensity;
ViewUniformShaderParameters.MooaShadowBias = FinalPostProcessSettings.MooaShadowBias;
ViewUniformShaderParameters.MooaEnableScreenSpaceHairShadowDirectionalLights = FinalPostProcessSettings.MooaEnableScreenSpaceHairShadowDirectionalLights;
ViewUniformShaderParameters.MooaEnableScreenSpaceHairShadowPointLights = FinalPostProcessSettings.MooaEnableScreenSpaceHairShadowPointLights;
ViewUniformShaderParameters.MooaEnableScreenSpaceHairShadowSpotLights = FinalPostProcessSettings.MooaEnableScreenSpaceHairShadowSpotLights;
ViewUniformShaderParameters.MooaEnableScreenSpaceHairShadowRectLights = FinalPostProcessSettings.MooaEnableScreenSpaceHairShadowRectLights;
ViewUniformShaderParameters.MooaEnableDistanceFieldFacialShadowDirectionalLights = FinalPostProcessSettings.MooaEnableDistanceFieldFacialShadowDirectionalLights;
ViewUniformShaderParameters.MooaEnableDistanceFieldFacialShadowPointLights = FinalPostProcessSettings.MooaEnableDistanceFieldFacialShadowPointLights;
ViewUniformShaderParameters.MooaEnableDistanceFieldFacialShadowSpotLights = FinalPostProcessSettings.MooaEnableDistanceFieldFacialShadowSpotLights;
```

### 2. Ramp纹理传递到着色器

**文件**: `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:1484-1485`

```cpp
ViewUniformShaderParameters.MooaGlobalDiffuseColorRampAtlas = MooaToonSubsystem->GlobalDiffuseColorRampAtlas->TextureReference.TextureReferenceRHI;
ViewUniformShaderParameters.MooaGlobalDiffuseColorRampAtlasHeight = MooaToonSubsystem->GlobalDiffuseColorRampAtlas->TextureHeight;
```

---

## HybriToon深度魔改方案

### 方案概述

基于对MooaToon引擎的深度分析，我们可以设计以下几种深度魔改方案：

### 方案一：神经Ramp预测（推荐）

**难度**: ⭐⭐⭐
**性能**: ⭐⭐⭐⭐
**质量**: ⭐⭐⭐⭐

**架构**:
```
参考图 → 神经风格编码器 → Ramp索引 + UV偏移 → MooaToon渲染
              ↓
         风格特征向量
```

**修改点**:
1. 在材质中添加神经特征输入
2. 修改`ToonShadingModel.ush`，融合神经特征到Ramp采样
3. 保持现有GBuffer结构不变

**优点**:
- 最小化引擎修改
- 推理速度快
- 可解释性强

### 方案二：神经光照注入

**难度**: ⭐⭐⭐⭐
**性能**: ⭐⭐⭐
**质量**: ⭐⭐⭐⭐⭐

**架构**:
```
参考图 → CNN特征提取 → 特征图 → ToonBufferB（新增）→ 融合到光照计算
```

**修改点**:
1. 新增`ToonBufferB`（第二个Toon GBuffer）
2. 修改`SceneRenderTargetParameters.h`
3. 修改`ToonShadingModel.ush`，在光照计算中注入神经特征

**优点**:
- 更灵活的风格控制
- 可实现局部风格变化
- 质量更高

### 方案三：端到端神经渲染器

**难度**: ⭐⭐⭐⭐⭐
**性能**: ⭐⭐
**质量**: ⭐⭐⭐⭐⭐

**架构**:
```
3D场景 → GBuffer → 神经渲染器 → 最终图像
                ↓
         MooaToon特征
```

**修改点**:
1. 在延迟渲染后添加神经渲染Pass
2. 使用GBuffer和ToonBufferA作为输入
3. 输出最终风格化图像

**优点**:
- 最高质量
- 可实现任意风格
- 突破传统渲染限制

### 推荐实施路线图

#### Phase 1: 神经Ramp预测（1-2个月）
- 目标: 快速验证概念
- 实现: 风格编码器 → Ramp索引 + UV偏移
- 修改: 最小化引擎修改

#### Phase 2: 神经光照注入（2-3个月）
- 目标: 提升质量
- 实现: 特征图注入ToonBufferB
- 修改: 新增GBuffer，修改着色器

#### Phase 3: 端到端神经渲染（3-6个月）
- 目标: 手绘质感
- 实现: 完整神经渲染管线
- 修改: 深度集成

---

## 总结

### MooaToon核心技术亮点

1. **GBuffer扩展**: 巧妙地扩展GBuffer，新增ToonBufferA
2. **Ramp纹理系统**: 使用CurveLinearColorAtlas管理风格化Ramp
3. **Toon着色模型**: 完整的卡通BxDF实现
4. **头发/面部阴影**: 屏幕空间和距离场阴影
5. **控制台变量**: 丰富的运行时可调参数

### HybriToon魔改建议

1. **从方案一开始**: 神经Ramp预测，风险最低
2. **保持兼容性**: 尽量不破坏现有MooaToon功能
3. **渐进式增强**: 逐步添加神经渲染特性
4. **性能优先**: 确保实时渲染性能

### 关键文件速查表

| 功能 | 文件 |
|------|------|
| GBuffer定义 | `SceneRenderTargetParameters.h` |
| ToonBuffer创建 | `SceneTextures.cpp` |
| CVar定义 | `SceneRendering.cpp:120-152` |
| 参数传递 | `SceneRendering.cpp:1436-1485` |
| 着色器公共 | `ToonShadingCommon.ush` |
| Toon BxDF | `ToonShadingModel.ush` |
| Ramp管理 | `MooaToonSubsystem.h/cpp` |
| 全局设置 | `MooaToonSettings.h/cpp` |

---

## 参考资料

- MooaToon GitHub: https://github.com/JasonMa0012/MooaToon
- MooaToon 文档: https://mooatoon.com/docs/
- UE5 渲染文档: https://docs.unrealengine.com/

---

**报告生成时间**: 2026年4月6日
**报告版本**: v2.0
**分析深度**: 引擎级源代码分析
