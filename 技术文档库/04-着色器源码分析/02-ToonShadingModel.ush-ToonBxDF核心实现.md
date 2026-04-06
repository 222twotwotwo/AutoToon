# ToonShadingModel.ush - ToonBxDF 核心实现详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Shaders/Private/ToonShadingModel.ush` |
| **核心功能** | ToonBxDF 核心函数、Ramp 采样、高光、描边 |
| **重要性** | ⭐⭐⭐⭐⭐ 最核心的着色器文件 |

---

## 写给零基础开发者

### 这个文件是做什么的？

**想象一下**：你是一个画家，要给 3D 物体上色。

这个文件就是那个**画家**，它负责：
1. **漫反射** - 物体的主要颜色（查 Ramp 纹理）
2. **高光** - 物体上的亮斑（可以是 PBR，也可以是 Ramp）
3. **描边** - 物体的轮廓线
4. **头发阴影** - 头发的自阴影
5. **面部阴影** - 角色面部的阴影

**这是 MooaToon 最核心的文件！**

---

## 文件结构总览

| 部分 | 行号 | 内容 |
|------|------|------|
| 1 | 11-60 | 工具函数（颜色、Ramp采样、阴影强度等） |
| 2 | 62-68 | LinearToDotProductSpace（坐标空间转换） |
| 3 | 129-154 | 屏幕空间头发阴影 |
| 4 | 156-170 | 距离场面部阴影 |
| 5 | 172-229 | 高光 Ramp UV 计算 |
| 6 | 231-270 | 屏幕空间描边 |
| 7 | 272-410 | **ToonBxDF（核心函数！）** |

---

## 第一部分：工具函数

### 1.1 ColorSaturationPowerAndScale（颜色饱和度调整）

**行号**: 11-19

```hlsl
void ColorSaturationPowerAndScale(
    inout float3 Color,
    float SaturationScale,
    float LuminanceScale = 1
)
{
    BRANCH if (any(Color > 0))
    {
        // 把 RGB 转成 HSV
        float3 ColorHSV = LinearRGB_2_HSV(Color);
        
        // 调整饱和度
        ColorHSV.y = saturate(ColorHSV.y * SaturationScale);
        
        // 转回 RGB，乘以亮度缩放
        Color = HSV_2_LinearRGB(ColorHSV) * LuminanceScale;
    }
}
```

**这是做什么的？**
- 调整颜色的饱和度
- 可以让颜色更鲜艳或更灰暗
- 也可以调整整体亮度

**用大白话解释**：
- 想象你在 Photoshop 里调整图片
- 这个函数就像是「饱和度」滑块 +「亮度」滑块

---

### 1.2 SampleGlobalRamp（采样全局 Ramp 纹理）

**行号**: 21-24

```hlsl
half4 SampleGlobalRamp(
    Texture2D Tex,      // Ramp 图集纹理
    float U,            // U 坐标（0-1，光照强度）
    uint index,         // Ramp 索引（选第几个 Ramp）
    float rampHeight    // Ramp 图集的高度（有多少行 Ramp）
)
{
    // 计算 V 坐标：(索引 + 0.5) / 总行数
    float2 UV = float2(U, float(index + 0.5f) / rampHeight);
    
    // 采样纹理
    return Texture2DSampleLevel(Tex, GlobalBilinearClampedSampler, UV, 0);
}
```

**这是做什么的？**
- 从 Ramp 图集中采样一个颜色
- U 坐标 = 光照强度（0-1）
- V 坐标 = 选第几个 Ramp

**用大白话解释**：
- 想象 Ramp 图集是一个表格，每一行是一个 Ramp
- index = 选第几行
- U = 在这一行里从左到右选哪个位置

**示例**：
```hlsl
// 采样第 3 个 Ramp，光照强度 0.5
half4 Color = SampleGlobalRamp(
    View.MooaGlobalDiffuseColorRampAtlas,
    0.5f,           // U = 0.5（中间位置）
    3,              // index = 3（第 4 个 Ramp）
    View.MooaGlobalDiffuseColorRampAtlasHeight
);
```

---

### 1.3 GetShadowColorIntensity（获取阴影颜色强度）

**行号**: 26-36

```hlsl
half GetShadowColorIntensity(FMooaToonContext MooaToonContext)
{
    // 根据光源类型返回不同的 CVar 值
    if (MooaToonContext.LightType == MOOA_LIGHT_TYPE_DIRECTIONAL)
        return View.MooaShadowColorIntensityDirectionalLights;
    else if (MooaToonContext.LightType == MOOA_LIGHT_TYPE_POINT)
        return View.MooaShadowColorIntensityPointLights;
    else if (MooaToonContext.LightType == MOOA_LIGHT_TYPE_SPOT)
        return View.MooaShadowColorIntensitySpotLights;
    else
        return View.MooaShadowColorIntensityRectLights;
}
```

**这是做什么的？**
- 不同类型的光源可以有不同的阴影颜色强度
- 平行光、点光源、聚光灯、矩形光各自独立控制

---

### 1.4 LinearToDotProductSpace（线性空间 → 点积空间）

**行号**: 62-68

```hlsl
// NoL == dot(NormalDir, LightDir) == cos(theta)
// 换句话说，NoL 不是线性的，
// 所以其他线性的 ShadowGradient 必须转换到同一空间，
// 确保 Ramp 采样结果一致
float LinearToDotProductSpace(float gradient01)
{
    return saturate(cos(saturate(gradient01) * PI) * -0.5f + 0.5f);
}
```

**这是做什么的？**
- 把线性的 0-1 转换成 cos 曲线的 0-1
- 因为 NoL = dot(N, L) = cos(theta)，不是线性的
- 所以其他阴影梯度也要转到同一空间

**用大白话解释**：
- 想象你有两把尺子，刻度不一样
- 这个函数就是把其中一把尺子的刻度转换成另一把
- 这样它们才能在一起使用

---

## 第二部分：屏幕空间头发阴影

### GetScreenSpaceDepthTestHairShadow

**行号**: 129-154

```hlsl
half GetScreenSpaceDepthTestHairShadow(
    FMooaToonContext MooaToonContext,
    FGBufferData GBuffer,
    FToonGBufferData ToonGBuffer,
    float2 ViewportUV,
    float2 L_ViewportSpace,
    float ViewportSpaceToWorldSpaceDir
)
{
    float HairShadow = 1;  // 1 = 无阴影，0 = 全黑
    
#if SHADING_PATH_DEFERRED && defined(MOOA_TOON_DEFERRED_LIGHTING) && !SUBSTRATE_ENABLED
    
    // 如果没启用头发阴影，直接返回
    BRANCH if (!GetEnableScreenSpaceHairShadow(MooaToonContext))
        return HairShadow;
    
    const float HairShadowWidth = 2.0f * View.MooaHairShadowWidth;
    const float HairShadowIntensity = View.MooaHairShadowIntensity;
    
    // 只有当标记是 FACE_SCREEN_SPACE_HAIR_SHADOW 时才计算
    BRANCH if(ToonGBuffer.RayTracingShadowFlag == MOOA_RAY_TRACING_SHADOW_FLAG_FACE_SCREEN_SPACE_HAIR_SHADOW &&
        HairShadowWidth > 0 && HairShadowIntensity > 0)
    {
        // 沿光源方向偏移 UV
        float2 ViewportUVOffset = L_ViewportSpace * ViewportSpaceToWorldSpaceDir * HairShadowWidth;
        float2 TargetBufferUV = ViewportUVToBufferUV(saturate(ViewportUV + ViewportUVOffset));
        
        // 读取偏移位置的 GBuffer
        FGBufferData HairGbuffer = GetGBufferData(TargetBufferUV);
        
        // 计算深度衰减
        float DepthFade = saturate(
            max(0, GBuffer.Depth - HairGbuffer.Depth - View.MooaHairShadowDepthTestThreshold) 
            / max(1e-5, View.MooaHairShadowDepthTestFadeDistance)
        );
        
        // 如果偏移位置是头发，就应用阴影
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

**这是做什么的？**
- 沿光源方向采样，看前面有没有头发
- 如果有，就把当前像素变暗

**用大白话解释**：
- 想象你站在阳光下，前面有一棵树
- 树的影子会落在你身上
- 这个函数就是检查「前面有没有头发」，如果有，就加阴影

**工作原理**：
1. 沿光源方向偏移 UV
2. 读取偏移位置的深度
3. 如果那里是头发（RayTracingShadowFlag = HAIR）
4. 就把当前像素变暗

---

## 第三部分：距离场面部阴影

### ApplyDistanceFieldFacialShadow

**行号**: 156-170

```hlsl
void ApplyDistanceFieldFacialShadow(
    FMooaToonContext MooaToonContext,
    FGBufferData GBuffer,
    FToonGBufferData ToonGBuffer,
    float3 L,
    float DiffuseColorRampUVOffset,
    inout float FacialShadowGradient
)
{
    // 只有当 ShadingFeatureID 是 DISTANCE_FIELD_FACIAL_SHADOW 时才启用
    BRANCH if (ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_DISTANCE_FIELD_FACIAL_SHADOW
        && GetEnableDistanceFieldFacial(MooaToonContext))
    {
        // 面部朝向 = WorldTangent
        float3 FaceForwardDir = GBuffer.WorldTangent;
        
        // 计算光源角度
        float LightAngle = RadianToDegree(FastACos(dot(FaceForwardDir, L)));
        
        // 根据光源角度计算 UV 偏移
        float RampUVOffsetByLightAngle = (1.0 - clamp(LightAngle / 180.0f, 1e-4, 1 - 1e-4)) - 0.5f;
        
        // 判断光源在左边还是右边
        bool bLightAtRight = cross(FaceForwardDir, L).z >= 0;
        
        // 选择左边或右边的 SDF
        float shadowSdf = bLightAtRight ? ToonGBuffer.FacialShadowSdfRight : ToonGBuffer.FacialShadowSdfLeft;
        
        // 计算最终的阴影梯度
        FacialShadowGradient = LinearToDotProductSpace(
            RampUVOffsetByLightAngle + shadowSdf + DiffuseColorRampUVOffset
        );
    }
}
```

**这是做什么的？**
- 用距离场（SDF）给角色面部画阴影
- 光源在左边 → 用左边的 SDF
- 光源在右边 → 用右边的 SDF

**用大白话解释**：
- 想象你有一个面具，左边和右边各有一个阴影形状
- 太阳在左边 → 用左边的阴影形状
- 太阳在右边 → 用右边的阴影形状
- 这样角色的面部阴影会跟着太阳转

---

## 第四部分：高光计算

### GetSpecularColorRampUAndMaxSpecularValue

**行号**: 172-229

这是一个比较复杂的函数，让我们分两部分看：

#### 部分A：Kajiya-Kay 头发高光

```hlsl
BRANCH if (GBuffer.MooaToonContext.ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_KAJIYA_HAIR_SPECULAR)
{
    MaxSpecularValue = PI;
    
    // Kajiya Kay 头发高光
    half ToH = 1 - abs(dot(GBuffer.WorldTangent, H));
    
    // 根据粗糙度调整高光大小
    const float RoughnessGradientPow = 4.0f;
    const float SpecularGradientPowMin = 2.0f;
    const float SpecularGradientPowMax = 128.0f;
    
    half SpecularGradientPow = Remap01(
        pow(1 - GBuffer.Roughness, RoughnessGradientPow),
        SpecularGradientPowMin,
        SpecularGradientPowMax
    );
    
    return pow(ToH, SpecularGradientPow);
}
```

**Kajiya-Kay 是什么？**
- 专门用于头发的高光模型
- 特点：沿发丝方向的长条高光

---

#### 部分B：普通高光（PBR 或 Ramp）

```hlsl
else
{
    // 计算 MaxSpecularValue（从 SpecularGGX 修改而来）
    {
        float a2 = Pow4(GBuffer.Roughness);
        float D = rcp(a2 * PI);
        float Vis = 0.25f;
        float F = 1;
        
        MaxSpecularValue = D * Vis * F;
    }
    
    // 计算高光梯度（从 D_GGXaniso 修改而来）
    {
        float3 V = float3(
            saturate(-GBuffer.Anisotropy) * Context.XoH,
            saturate(GBuffer.Anisotropy) * Context.YoH,
            (1 - abs(GBuffer.Anisotropy)) * (1 - Context.NoH)
        );
        
        float SpecularGradient = saturate(1 - dot(V , V));
        
        // 当粗糙度接近 0 时，高光向 NoH 收缩
        SpecularGradient *= lerp(1, Context.NoH, Pow14(1 - GBuffer.Roughness));
        
        // 多项式拟合（根据粗糙度调整幂次）
        const float a = -12.41463;
        const float b = 44.39455;
        const float c = -58.20974;
        const float d = 31.87271;
        const float e = -4.56996;
        const float f = -0.0729301;
        
        float PowerGradient01 = saturate(
            a * Pow5(GBuffer.Roughness) + 
            b * Pow4(GBuffer.Roughness) + 
            c * Pow3(GBuffer.Roughness) + 
            d * Pow2(GBuffer.Roughness) + 
            e * GBuffer.Roughness + 
            f
        );
        
        // 风格化高光范围
        const float SpecularPowRangeMin = 2000.0f;  // 粗糙度 0
        const float SpecularPowRangeMax = 10.0f;    // 粗糙度 1
        
        float SpecularGradientPow = Remap01(PowerGradient01, SpecularPowRangeMin, SpecularPowRangeMax);
        
        return pow(SpecularGradient, SpecularGradientPow);
    }
}
```

**这是做什么的？**
- 把 PBR 的高光参数转换成 Ramp 的 UV 坐标
- 这样既能用 PBR 的粗糙度控制，又能用 Ramp 的风格化

---

## 第五部分：屏幕空间描边

### GetScreenSpaceDepthTestRimlightColor

**行号**: 231-270

```hlsl
float3 GetScreenSpaceDepthTestRimlightColor(
    FGBufferData GBuffer,
    FToonGBufferData ToonGBuffer,
    float2 ViewportUV,
    float2 L_ViewportSpace,
    float ViewportSpaceToWorldSpaceDir
)
{
    float3 RimLightColor = 0;
    
    // 只有当边缘光强度和宽度都大于 0 时才计算
    BRANCH if (ToonGBuffer.RimLightIntensity > 0 && ToonGBuffer.RimLightWidth > 0)
    {
        const float MaxRimLightWidth = 3.0f * View.MooaRimLightMaxWidth;
        const float DepthFadeDistance = View.MooaRimLightDepthTestFadeDistance;
        const float RimLightDepthThreshold = View.MooaRimLightDepthTestThreshold;
        
        const float RimLightWidth = Pow2(ToonGBuffer.RimLightWidth);
        const float RimLightIntensity = ToonGBuffer.RimLightIntensity;
        
        // 沿光源方向偏移 UV
        float2 ViewportUVOffset = L_ViewportSpace * ViewportSpaceToWorldSpaceDir * MaxRimLightWidth * RimLightWidth;
        float2 TargetBufferUV = ViewportUVToBufferUV(saturate(ViewportUV + ViewportUVOffset));
        
        // 读取偏移位置的深度
        float NewDepth = 0;
#if MATERIALBLENDING_TRANSLUCENT || MATERIALBLENDING_ADDITIVE || MATERIALBLENDING_MODULATE
        // 半透明物体用半透明深度
        float SingleLayerDeviceZ = Texture2DSampleLevel(TranslucentBasePass.SceneDepth, GlobalPointClampedSampler, TargetBufferUV, 0).r;
        NewDepth = ConvertFromDeviceZ(SingleLayerDeviceZ);
#elif SHADING_PATH_DEFERRED
        // 不透明物体用场景深度
        NewDepth = CalcSceneDepth(TargetBufferUV);
#endif
        
        // 计算深度衰减
        float DepthFade = saturate(
            max(0, NewDepth - GBuffer.Depth - RimLightDepthThreshold) 
            / max(1e-5, DepthFadeDistance)
        );
        
        // 计算边缘光颜色
        RimLightColor = DepthFade * ComputeF0(RimLightIntensity, GBuffer.BaseColor, GBuffer.Metallic);
    }
    
    return RimLightColor;
}
```

**这是做什么的？**
- 沿光源方向看，如果深度有突变，就画边缘光
- 用来模拟卡通的描边效果

**用大白话解释**：
- 想象你用铅笔描物体的轮廓
- 这个函数就是自动找轮廓（深度突变的地方）
- 然后在轮廓上画边缘光

---

## 第六部分：ToonBxDF（核心函数！）

### ToonBxDF

**行号**: 272-410

这是最核心的函数！让我们分部分解析：

---

#### 部分1：初始化和准备

```hlsl
FDirectLighting ToonBxDF(
    FGBufferData GBuffer,
    half3 N,
    half3 V,
    half3 L,
    float Falloff,
    float NoL,
    FAreaLight AreaLight,
    FShadowTerms Shadow
)
{
    // 从 GBuffer 中获取 MooaToon 上下文
    FMooaToonContext MooaToonContext = GBuffer.MooaToonContext;
    FToonGBufferData ToonGBuffer = GBuffer.MooaToonContext.ToonGBuffer;
    
    FDirectLighting Lighting = (FDirectLighting)0;
    
    // 计算光源颜色和衰减
    float3 LightColorAndAttenuation = AreaLight.FalloffColor * MooaToonContext.LightColor * Falloff;
    
    // 调整光源颜色的饱和度
    ColorSaturationPowerAndScale(LightColorAndAttenuation, View.MooaLightSaturationScale);
    
    // 计算各种向量和坐标
    const float NoL_Full = dot(N, L);
    const float NoL_Half = NoL_Full * 0.5f + 0.5f;  // 从 [-1,1] 转到 [0,1]
    const float3 H = normalize(V + L);  // 半角向量
    const float2 BufferUV = SvPositionToBufferUV(float4(MooaToonContext.PixelPos, 0, 0));
    const float2 ViewportUV = BufferUVToViewportUV(BufferUV);
    const float3 L_ClipSpace = mul(L, (float3x3)View.TranslatedWorldToClip).xyz;
    const float2 L_ViewportSpace = (L_ClipSpace.xy * float2(0.5, -0.5));
    const float3 N_ClipSpace = mul(N, (float3x3)View.TranslatedWorldToClip).xyz;
    const float2 N_ViewportSpace = (N_ClipSpace.xy * float2(0.5, -0.5));
    const float ViewportSpaceToWorldSpaceDir = rcp(GBuffer.Depth);
```

---

#### 部分2：漫反射计算（核心！）

```hlsl
// -----------------------------------------------------------------------------------------------------------------
// Diffuse（漫反射）
{
    const bool EnablePostRampShadow = View.MooaDiffuseColorRampEnablePostRampShadow;
    const bool EnablePostRampMaterialAO = View.MooaDiffuseColorRampEnablePostRampMaterialAO;
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 步骤1：获取各种阴影梯度
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    // 漫反射 Ramp UV 偏移
    float DiffuseColorRampUVOffset = (ToonGBuffer.DiffuseColorRampUVOffset * 2.0f - 1.0f) * View.MooaDiffuseColorRampUVOffsetMaxRange;
    
    // 基础阴影梯度 = NoL + UV偏移
    float ShadowGradient = saturate(NoL_Half + DiffuseColorRampUVOffset);
    
    // 头发阴影
    float HairLinearShadowGradient = GetScreenSpaceDepthTestHairShadow(
        MooaToonContext, GBuffer, ToonGBuffer, ViewportUV, L_ViewportSpace, ViewportSpaceToWorldSpaceDir
    );
    
    // 面部阴影（会修改 ShadowGradient）
    ApplyDistanceFieldFacialShadow(
        MooaToonContext, GBuffer, ToonGBuffer, L, DiffuseColorRampUVOffset,
        /*inout*/ ShadowGradient
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 步骤2：组合所有阴影梯度
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    float CustomLinearShadowGradient = min(
        // 如果启用 PostRampMaterialAO，就不用 GBufferAO
        EnablePostRampMaterialAO ? 1.0f : GBuffer.GBufferAO,
        // 如果启用 PostRampShadow，就不用 SurfaceShadow 和 HairShadow
        EnablePostRampShadow ? 1.0f : min(Shadow.SurfaceShadow, HairLinearShadowGradient)
    );
    
    // 最终的 Ramp U = min(ShadowGradient, CustomLinearShadowGradient转空间后)
    float DiffuseColorRampU = min(ShadowGradient, LinearToDotProductSpace(CustomLinearShadowGradient));
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 步骤3：采样 Ramp 纹理
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    // RGB = 漫反射颜色，A = 漫反射颜色和阴影颜色的混合因子
    half4 DiffuseColorRamp = SampleGlobalRamp(
        View.MooaGlobalDiffuseColorRampAtlas,
        DiffuseColorRampU,
        ToonGBuffer.DiffuseColorRampIndex,
        View.MooaGlobalDiffuseColorRampAtlasHeight
    );
    
    // 计算最终的阴影混合因子
    float DiffuseShadow = min3(
        DiffuseColorRamp.a,
        EnablePostRampMaterialAO ? GBuffer.GBufferAO : 1.0f,
        EnablePostRampShadow ? min(Shadow.SurfaceShadow, HairLinearShadowGradient) : 1.0f
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 步骤4：混合漫反射颜色和阴影颜色
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    half3 ToonDiffuseColor = GBuffer.DiffuseColor;
    half3 ToonShadowColor = ToonGBuffer.MainLightShadowColor * (1 - GBuffer.Metallic) * GetShadowColorIntensity(MooaToonContext);
    
#if USE_DEVELOPMENT_SHADERS
    ToonShadowColor *= View.DiffuseOverrideParameter.w + View.DiffuseOverrideParameter.xyz;
#endif
    
    // lerp(阴影颜色, 漫反射颜色, 阴影混合因子)
    ToonDiffuseColor = lerp(ToonShadowColor, ToonDiffuseColor, DiffuseShadow);
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 步骤5：输出最终漫反射
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    Lighting.Diffuse = Diffuse_Lambert(ToonDiffuseColor) * DiffuseColorRamp.rgb * LightColorAndAttenuation;
    Shadow.SurfaceShadow = min(Shadow.SurfaceShadow, HairLinearShadowGradient);
}
```

**漫反射流程图**：
```
NoL + UV偏移 → ShadowGradient
    ↓
+ 头发阴影 + 面部阴影
    ↓
组合所有阴影 → CustomLinearShadowGradient
    ↓
采样 Ramp 纹理 → DiffuseColorRamp
    ↓
混合漫反射颜色和阴影颜色 → ToonDiffuseColor
    ↓
输出最终漫反射
```

---

#### 部分3：各向异性 BxDF 上下文

```hlsl
// -----------------------------------------------------------------------------------------------------------------
// Anisotropy Specular BxDF Context（各向异性高光上下文）
bool bHasAnisotropy;
BxDFContext Context = (BxDFContext)0;
{
#if SUPPORTS_ANISOTROPIC_MATERIALS
    bHasAnisotropy = HasAnisotropy(GBuffer.SelectiveOutputMask) && GBuffer.Anisotropy != 0;
#else
    bHasAnisotropy = false;
#endif
    
    // 面部阴影模式下禁用各向异性
    if (ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_DISTANCE_FIELD_FACIAL_SHADOW)
        bHasAnisotropy = false;
    
    BRANCH if (bHasAnisotropy)
    {
        // 有各向异性：用 WorldTangent 构建切线空间
        half3 X = GBuffer.WorldTangent;
        half3 Y = normalize(cross(N, X));
        Init(Context, N, X, Y, V, L);
    }
    else
    {
        // 无各向异性：简单初始化
        Init(Context, N, V, L);
        GBuffer.Anisotropy = 0;
    }
    
    Context.NoV = saturate(abs(Context.NoV) + 1e-5);
}
```

---

#### 部分4：高光计算

```hlsl
// -----------------------------------------------------------------------------------------------------------------
// Specular（高光）
BRANCH if (ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_PBR_SPECULAR)
{
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 模式A：PBR 高光
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
#if SUPPORTS_ANISOTROPIC_MATERIALS
    BRANCH if (bHasAnisotropy)
    {
        // 各向异性 PBR 高光
        Lighting.Specular = LightColorAndAttenuation * Shadow.SurfaceShadow * NoL * 
            SpecularGGX(GBuffer.Roughness, GBuffer.Anisotropy, GBuffer.SpecularColor, Context, NoL, AreaLight);
    }
    else
#endif
    {
        BRANCH if(IsRectLight(AreaLight))
        {
            // 矩形光
            Lighting.Specular = MooaToonContext.LightColor * Shadow.SurfaceShadow * 
                RectGGXApproxLTC(GBuffer.Roughness, GBuffer.SpecularColor, N, V, AreaLight.Rect, AreaLight.Texture);
        }
        else
        {
            // 普通 PBR 高光
            Lighting.Specular = LightColorAndAttenuation * Shadow.SurfaceShadow * NoL * 
                SpecularGGX(GBuffer.Roughness, GBuffer.SpecularColor, Context, NoL, AreaLight);
        }
    }
}
else
{
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 模式B：Ramp 高光（默认）
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    float MaxSpecularValue;
    float SpecularColorRampU = GetSpecularColorRampUAndMaxSpecularValue(
        GBuffer, Context, N, H,
        /*out*/ MaxSpecularValue
    );
    
    float SpecularColorRampUVOffset = (ToonGBuffer.SpecularColorRampUVOffset * 2.0f - 1.0f) * View.MooaSpecularColorRampUVOffsetMaxRange;
    
    // 采样高光 Ramp
    half3 SpecularColor = SampleGlobalRamp(
        View.MooaGlobalSpecularColorRampAtlas,
        SpecularColorRampU + SpecularColorRampUVOffset,
        ToonGBuffer.SpecularColorRampIndex,
        View.MooaGlobalSpecularColorRampAtlasHeight
    ).rgb;
    
    // 输出最终高光
    Lighting.Specular = GBuffer.SpecularColor * ToonGBuffer.SpecularColor * MaxSpecularValue * 
        SpecularColor * LightColorAndAttenuation * Shadow.SurfaceShadow;
}
```

---

#### 部分5：边缘光

```hlsl
// -----------------------------------------------------------------------------------------------------------------
// Rimlight（边缘光）
float3 RimLight = GetScreenSpaceDepthTestRimlightColor(
    GBuffer, ToonGBuffer, ViewportUV, L_ViewportSpace, ViewportSpaceToWorldSpaceDir
) * LightColorAndAttenuation * Shadow.SurfaceShadow;

// 调整边缘光的饱和度
ColorSaturationPowerAndScale(RimLight, View.MooaRimLightSaturationScale, View.MooaRimLightIntensity);

// 边缘光加到高光上
Lighting.Specular += RimLight;

return Lighting;
```

---

## 完整的 ToonBxDF 流程图

```
ToonBxDF()
    │
    ├─ 初始化
    │   ├─ 获取 MooaToonContext
    │   ├─ 计算 LightColorAndAttenuation
    │   └─ 计算各种向量（NoL, H, UV等）
    │
    ├─ 漫反射计算
    │   ├─ 计算 ShadowGradient（NoL + UV偏移）
    │   ├─ 计算头发阴影
    │   ├─ 计算面部阴影
    │   ├─ 组合所有阴影
    │   ├─ 采样漫反射 Ramp
    │   ├─ 混合漫反射颜色和阴影颜色
    │   └─ 输出 Lighting.Diffuse
    │
    ├─ 各向异性上下文
    │   └─ 初始化 BxDFContext
    │
    ├─ 高光计算
    │   ├─ 模式A：PBR 高光
    │   │   ├─ 各向异性 PBR
    │   │   └─ 普通 PBR
    │   └─ 模式B：Ramp 高光（默认）
    │       ├─ 计算 SpecularColorRampU
    │       ├─ 采样高光 Ramp
    │       └─ 输出 Lighting.Specular
    │
    ├─ 边缘光
    │   ├─ 计算屏幕空间深度测试边缘光
    │   ├─ 调整饱和度
    │   └─ 加到 Lighting.Specular
    │
    └─ 返回 Lighting
```

---

## 总结

### 关键点

1. **ToonBxDF 是最核心的函数**，所有 Toon 渲染逻辑都在这里
2. **漫反射** = 查 Ramp 纹理 + 混合阴影颜色
3. **高光** 有两种模式：PBR 或 Ramp
4. **头发阴影** = 屏幕空间深度测试
5. **面部阴影** = 距离场 SDF
6. **边缘光** = 屏幕空间深度测试描边

### 记忆要点

- ✅ ToonBxDF = 画家
- ✅ 漫反射 = 查 Ramp 纹理
- ✅ 高光 = PBR 或 Ramp
- ✅ 头发/面部阴影 = 屏幕空间深度测试
- ✅ 边缘光 = 描边

---

**文档版本**: v1.0  
**分析深度**: 源码级（逐行解释）  
**目标读者**: 零基础开发者  
**最后更新**: 2026年4月6日
