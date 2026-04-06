# SceneRendering.cpp - 渲染参数传递详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp` |
| **核心功能** | CVar 定义、ViewUniformShaderParameters 传递、Ramp 图集传递 |
| **重要性** | ⭐⭐⭐⭐⭐ |

---

## 写给零基础开发者

### 这个文件是做什么的？

**想象一下**：你是一个舞台监督，要把道具和布景交给演员。

这个文件就是那个**舞台监督**，它负责：
1. 定义 CVar（控制台变量）- 就像舞台上的调光器
2. 把 CVar 值传给着色器 - 就像告诉演员「现在用这个灯光」
3. 把 Ramp 图集传给着色器 - 就像把调色板递给画家

---

## 文件结构总览

| 部分 | 行号 | 内容 |
|------|------|------|
| 1 | 115-152 | CVar 定义（控制台变量） |
| 2 | 581-583 | ToonBufferA 显存标志 |
| 3 | 774-776 | 更新 ToonBufferA 显存标志 |
| 4 | 1435-1496 | ViewUniformShaderParameters 传递 |

---

## 第一部分：CVar 定义

### 1.1 什么是 CVar？

**CVar = Console Variable（控制台变量）**

**用大白话解释**：
- 就像游戏里的「设置」菜单
- 你可以在控制台输入命令来修改
- 修改后实时生效，不用重启

---

### 1.2 MooaToon 的 CVar 定义

**行号**: 120-152

```cpp
// Mooa Console Variable
static TAutoConsoleVariable<float> CMooaDebugValue(
    TEXT("r.Mooa.DebugValue"), 
    0.0f,
    TEXT("Set it to 0 to disable Debug View"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaDynamicAOIntensity(
    TEXT("r.Mooa.AmbientOcclusion.DynamicAOIntensity"), 
    0.0f,
    TEXT("Dynamic AO Intensity"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<int32> CMooaDiffuseColorRampEnablePostRampShadow(
    TEXT("r.Mooa.DiffuseColorRamp.EnablePostRampShadow"), 
    0,
    TEXT("If enabled, Shadows will not affect the Diffuse Color Ramp samples."), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<int32> CMooaDiffuseColorRampEnablePostRampMaterialAO(
    TEXT("r.Mooa.DiffuseColorRamp.EnablePostRampMaterialAO"), 
    0,
    TEXT("If enabled, Material AO will not affect the Diffuse Color Ramp samples."), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaDiffuseColorRampUVOffsetMaxRange(
    TEXT("r.Mooa.DiffuseColorRamp.UVOffsetMaxRange"), 
    0.25f,
    TEXT("Diffuse Color Ramp UV Offset Max Range"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaSpecularColorRampUVOffsetMaxRange(
    TEXT("r.Mooa.SpecularColorRamp.UVOffsetMaxRange"), 
    0.25f,
    TEXT("Specular Color Ramp UV Offset Max Range"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaRimLightMaxWidth(
    TEXT("r.Mooa.RimLight.MaxWidth"), 
    1.0f,
    TEXT("Screen Space RimLight Max Width"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaRimLightDepthTestThreshold(
    TEXT("r.Mooa.RimLight.DepthTestThreshold"), 
    2.0f,
    TEXT("Screen Space RimLight Depth Test Threshold"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaRimLightDepthTestFadeDistance(
    TEXT("r.Mooa.RimLight.DepthTestFadeDistance"), 
    10.0f,
    TEXT("Screen Space RimLight Depth Test Fade Distance"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaHairShadowIntensity(
    TEXT("r.Mooa.HairShadow.Intensity"), 
    1.0f,
    TEXT("Hair Shadow Intensity"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaHairShadowWidth(
    TEXT("r.Mooa.HairShadow.Width"), 
    1.0f,
    TEXT("Hair Shadow Width"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaHairShadowDepthTestThreshold(
    TEXT("r.Mooa.HairShadow.DepthTestThreshold"), 
    0.0f,
    TEXT("Hair Shadow Depth Test Threshold"), 
    ECVF_RenderThreadSafe);

static TAutoConsoleVariable<float> CMooaHairShadowDepthTestFadeDistance(
    TEXT("r.Mooa.HairShadow.DepthTestFadeDistance"), 
    5.0f,
    TEXT("Hair Shadow Depth Test Fade Distance"), 
    ECVF_RenderThreadSafe);
// Mooa End
```

**CVar 参数详解**：

| CVar 名称 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `r.Mooa.DebugValue` | float | 0.0 | 调试值，0 禁用调试视图 |
| `r.Mooa.AmbientOcclusion.DynamicAOIntensity` | float | 0.0 | 动态 AO 强度 |
| `r.Mooa.DiffuseColorRamp.EnablePostRampShadow` | int | 0 | 启用后，阴影不影响漫反射 Ramp 采样 |
| `r.Mooa.DiffuseColorRamp.EnablePostRampMaterialAO` | int | 0 | 启用后，材质 AO 不影响漫反射 Ramp 采样 |
| `r.Mooa.DiffuseColorRamp.UVOffsetMaxRange` | float | 0.25 | 漫反射 Ramp UV 偏移最大范围 |
| `r.Mooa.SpecularColorRamp.UVOffsetMaxRange` | float | 0.25 | 高光 Ramp UV 偏移最大范围 |
| `r.Mooa.RimLight.MaxWidth` | float | 1.0 | 屏幕空间边缘光最大宽度 |
| `r.Mooa.RimLight.DepthTestThreshold` | float | 2.0 | 屏幕空间边缘光深度测试阈值 |
| `r.Mooa.RimLight.DepthTestFadeDistance` | float | 10.0 | 屏幕空间边缘光深度测试衰减距离 |
| `r.Mooa.HairShadow.Intensity` | float | 1.0 | 头发阴影强度 |
| `r.Mooa.HairShadow.Width` | float | 1.0 | 头发阴影宽度 |
| `r.Mooa.HairShadow.DepthTestThreshold` | float | 0.0 | 头发阴影深度测试阈值 |
| `r.Mooa.HairShadow.DepthTestFadeDistance` | float | 5.0 | 头发阴影深度测试衰减距离 |

**如何在控制台使用？**
```
// 在 Unreal 控制台输入：
r.Mooa.HairShadow.Intensity 2.0    // 把头发阴影强度设为 2
r.Mooa.RimLight.MaxWidth 3.0        // 把边缘光最大宽度设为 3
```

---

## 第二部分：ToonBufferA 显存标志

### 2.1 什么是 FASTVRAM_CVAR？

**FASTVRAM_CVAR = Fast Video Memory Console Variable（显存控制台变量）**

**用大白话解释**：
- 控制哪些渲染目标放在「快速显存」里
- 快速显存 = GPU 里最快的内存
- 把常用的渲染目标放进去，速度更快

---

### 2.2 ToonBufferA 的显存标志

**行号**: 581-583

```cpp
// Mooa GBuffer
FASTVRAM_CVAR(ToonBufferA, 0);
// Mooa End
```

**这是做什么的？**
- 给 ToonBufferA 定义一个 CVar
- 默认值 = 0（不强制放在快速显存）
- 你可以在控制台修改

**行号**: 774-776

```cpp
// Mooa GBuffer
bDirty |= UpdateTextureFlagFromCVar(CVarFastVRam_ToonBufferA, ToonBufferA);
// Mooa End
```

**这是做什么的？**
- 从 CVar 更新 ToonBufferA 的显存标志
- 如果 CVar 变了，就标记为「脏」（需要重新设置）

---

## 第三部分：ViewUniformShaderParameters 传递（核心！）

### 3.1 什么是 ViewUniformShaderParameters？

**ViewUniformShaderParameters = 视图 uniform 着色器参数**

**用大白话解释**：
- 这是一个「快递包」，CPU 把参数打包好发给 GPU
- 里面包含：CVar 值、后处理设置、Ramp 图集等
- 每一帧都发一次

---

### 3.2 传递 CVar 值

**行号**: 1435-1453

```cpp
//------------------------------------------------------------------------------------------------------------------
// Mooa View Uniform Buffer
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
```

**这是做什么的？**
- 把所有 CVar 值读到 ViewUniformShaderParameters 里
- `GetValueOnAnyThread()` = 从任何线程都可以读（线程安全）

---

### 3.3 传递后处理设置

**行号**: 1455-1476

```cpp
// Post Processing
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
ViewUniformShaderParameters.MooaEnableDistanceFieldFacialShadowRectLights = FinalPostProcessSettings.MooaEnableDistanceFieldFacialShadowRectLights;
ViewUniformShaderParameters.MooaShadowColorIntensityDirectionalLights = FinalPostProcessSettings.MooaShadowColorIntensityDirectionalLights;
ViewUniformShaderParameters.MooaShadowColorIntensityPointLights = FinalPostProcessSettings.MooaShadowColorIntensityPointLights;
ViewUniformShaderParameters.MooaShadowColorIntensitySpotLights = FinalPostProcessSettings.MooaShadowColorIntensitySpotLights;
ViewUniformShaderParameters.MooaShadowColorIntensityRectLights = FinalPostProcessSettings.MooaShadowColorIntensityRectLights;
ViewUniformShaderParameters.MooaRimLightIntensity = FinalPostProcessSettings.MooaRimLightIntensity;
ViewUniformShaderParameters.MooaRimLightSaturationScale = FinalPostProcessSettings.MooaRimLightSaturationScale;
```

**这是做什么的？**
- 把后处理设置（FinalPostProcessSettings）传给着色器
- 这些是在「后处理体积」里设置的

---

### 3.4 传递 Ramp 图集（最重要！）

**行号**: 1478-1496

```cpp
// Global Ramps
if (auto MooaToonSubsystem = GEngine->GetEngineSubsystem<UMooaToonSubsystem>())
{
    FString NullRampName = TEXT("");
    
    // 传递漫反射 Ramp 图集
    if (MooaToonSubsystem->GlobalDiffuseColorRampAtlas != nullptr)
    {
        ViewUniformShaderParameters.MooaGlobalDiffuseColorRampAtlas = 
            MooaToonSubsystem->GlobalDiffuseColorRampAtlas->TextureReference.TextureReferenceRHI;
        ViewUniformShaderParameters.MooaGlobalDiffuseColorRampAtlasHeight = 
            MooaToonSubsystem->GlobalDiffuseColorRampAtlas->TextureHeight;
    }
    else
    {
        NullRampName = TEXT("GlobalDiffuseColorRampAtlas");
    }
    
    // 传递高光 Ramp 图集
    if (MooaToonSubsystem->GlobalSpecularColorRampAtlas != nullptr)
    {
        ViewUniformShaderParameters.MooaGlobalSpecularColorRampAtlas = 
            MooaToonSubsystem->GlobalSpecularColorRampAtlas->TextureReference.TextureReferenceRHI;
        ViewUniformShaderParameters.MooaGlobalSpecularColorRampAtlasHeight = 
            MooaToonSubsystem->GlobalSpecularColorRampAtlas->TextureHeight;
    }
    else
    {
        NullRampName = TEXT("GlobalSpecularColorRampAtlas");
    }
    
    if (NullRampName != TEXT(""))
    {
        // 如果 Ramp 为空，可能会有警告（代码被截断了）
    }
}
```

**这是做什么的？**
- 从 MooaToonSubsystem 获取 Ramp 图集
- 把 Ramp 图集的 RHI 引用传给着色器
- 把 Ramp 图集的高度（有多少行 Ramp）也传给着色器

**数据流**：
```
MooaToonSubsystem（引擎子系统）
    ↓
GlobalDiffuseColorRampAtlas / GlobalSpecularColorRampAtlas
    ↓
TextureReference.TextureReferenceRHI（RHI 引用）
    ↓
ViewUniformShaderParameters（快递包）
    ↓
GPU 着色器
```

---

## 完整的数据流图

```
┌─────────────────────────────────────────────────────────────┐
│                        CPU 端                                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────┐    ┌──────────────────┐              │
│  │  CVar 定义       │    │  后处理设置       │              │
│  │  (控制台变量)    │    │  (Post Process)   │              │
│  └────────┬─────────┘    └────────┬─────────┘              │
│           │                         │                         │
│           └──────────┬──────────────┘                         │
│                      │                                        │
│  ┌───────────────────▼───────────────────┐                   │
│  │  ViewUniformShaderParameters           │                   │
│  │  (把所有参数打包好)                    │                   │
│  └───────────────────┬───────────────────┘                   │
│                      │                                        │
│  ┌───────────────────▼───────────────────┐                   │
│  │  MooaToonSubsystem                     │                   │
│  │  (引擎子系统)                          │                   │
│  │  - GlobalDiffuseColorRampAtlas         │                   │
│  │  - GlobalSpecularColorRampAtlas        │                   │
│  └───────────────────┬───────────────────┘                   │
│                      │                                        │
│                      ▼                                        │
├─────────────────────────────────────────────────────────────┤
│                        GPU 端                                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  着色器 (ToonShadingModel.ush)                        │  │
│  │  - View.MooaGlobalDiffuseColorRampAtlas               │  │
│  │  - View.MooaGlobalSpecularColorRampAtlas              │  │
│  │  - View.MooaHairShadowIntensity                        │  │
│  │  - View.MooaRimLightMaxWidth                           │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 总结

### 关键点

1. **CVar** = 控制台变量，可以实时修改
2. **ViewUniformShaderParameters** = 快递包，CPU 把参数打包发给 GPU
3. **MooaToonSubsystem** = 管理 Ramp 图集的引擎子系统
4. **每一帧**都把参数传给着色器

### 记忆要点

- ✅ SceneRendering.cpp = 舞台监督
- ✅ CVar = 控制台调光器
- ✅ ViewUniformShaderParameters = 快递包
- ✅ MooaToonSubsystem = 管理 Ramp 图集
- ✅ 每一帧都传一次参数

---

**文档版本**: v1.0  
**分析深度**: 源码级（逐行解释）  
**目标读者**: 零基础开发者  
**最后更新**: 2026年4月6日
