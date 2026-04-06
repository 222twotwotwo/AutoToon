# DiffuseIndirectComposite.usf - 漫反射间接合成

## 文件信息
- **路径**: `Engine/Shaders/Private/DiffuseIndirectComposite.usf`
- **作用**: 漫反射间接合成着色器
- **MooaToon修改**: 在4处添加Toon间接光照支持

## 关键代码分析

### 1. 包含ReflectionEnvironmentShared.ush（第97-99行）

```cpp
// Mooa Indirect Lighting
#include "ReflectionEnvironmentShared.ush"
// Mooa End
```

#### 零基础解释

这是包含ReflectionEnvironmentShared.ush头文件。

**为什么要包含？**
- 需要ReflectionEnvironmentShared.ush里的函数
- 比如GetSkySHDiffuse
- 用于Toon间接光照

### 2. 动态AO强度控制（第368-379行）

```cpp
float DynamicAmbientOcclusion = AmbientOcclusionTexture.SampleLevel(AmbientOcclusionSampler, SceneBufferUV, 0).r; // Mooa Indirect Lighting

...

// Mooa Indirect Lighting
BRANCH if (Material.ShadingID == SHADINGMODELID_TOON)
	DynamicAmbientOcclusion = lerp(1, DynamicAmbientOcclusion, saturate(View.MooaDynamicAOIntensity));
// Mooa End
```

#### 零基础解释

这是Toon动态AO强度控制。

**这里做了什么？**
- 如果是Toon着色模型
- 把DynamicAmbientOcclusion在1和原值之间插值
- 插值因子是MooaDynamicAOIntensity

**为什么这么做？**
- 控制Toon渲染的动态AO强度
- 可以让AO更弱或更强
- 通过CVar调整

**lerp(1, DynamicAmbientOcclusion, Alpha)：**
```
如果Alpha = 0：
  DynamicAmbientOcclusion = 1（AO完全关闭）

如果Alpha = 1：
  DynamicAmbientOcclusion = 原值（AO完全打开）

如果Alpha = 0.5：
  DynamicAmbientOcclusion = (1 + 原值) / 2（AO减弱一半）
```

### 3. Toon间接光照混合（第614-638行）

```cpp
// Mooa Indirect Lighting
if (Material.GBufferData.ShadingModelID == SHADINGMODELID_TOON)
{
	float3 SHDiffuseAverage = GetSkySHDiffuse(0.0f) * View.SkyLightColor.rgb * View.PreExposure;
	float3 LumenGiDiffuse = IndirectLighting.Diffuse;
	FToonGBufferData ToonGBuffer = Material.GBufferData.MooaToonContext.ToonGBuffer;
	float3 mixedGIColor = lerp(SHDiffuseAverage * Material.GBufferData.DiffuseColor, LumenGiDiffuse, View.MooaGlobalIlluminationDirectionality);
	OutAddColor.rgb = lerp(mixedGIColor * View.MooaGlobalIlluminationColor.rgb, View.MooaGlobalIlluminationColor.rgb, View.MooaGlobalIlluminationColor.a) * View.MooaGlobalIlluminationIntensity;
	OutAddColor.rgb += IndirectLighting.Specular * Pow2(ToonGBuffer.ReflectionIntensity);
	OutAddColor.rgb *= View.MooaExposureScale;
}
else
{
	IndirectLighting.Specular *= GetSSSCheckerboadSpecularScale(PixelPos, Material.bNeedsSeparateLightAccumulation);
	FLightAccumulator LightAccumulator = (FLightAccumulator)0;
	...
}
// Mooa End
```

#### 零基础解释

这是Toon间接光照的混合逻辑，是最复杂的部分！

**这里做了什么？**
1. 计算SHDiffuseAverage（天空SH漫反射平均）
2. 获取LumenGiDiffuse（Lumen漫反射）
3. 获取ToonGBuffer（Toon GBuffer数据）
4. 混合SHDiffuseAverage和LumenGiDiffuse
5. 应用MooaGlobalIlluminationColor
6. 应用MooaGlobalIlluminationIntensity
7. 添加反射
8. 应用MooaExposureScale

**逐行解释：**

**第1行：SHDiffuseAverage**
```hlsl
float3 SHDiffuseAverage = GetSkySHDiffuse(0.0f) * View.SkyLightColor.rgb * View.PreExposure;
```
- GetSkySHDiffuse(0.0f)：获取天空SH漫反射（法线是0）
- 乘以SkyLightColor：天空光照颜色
- 乘以PreExposure：预曝光

**第2行：LumenGiDiffuse**
```hlsl
float3 LumenGiDiffuse = IndirectLighting.Diffuse;
```
- Lumen计算的漫反射间接光照

**第3行：ToonGBuffer**
```hlsl
FToonGBufferData ToonGBuffer = Material.GBufferData.MooaToonContext.ToonGBuffer;
```
- 从GBuffer获取Toon数据

**第4行：混合GI**
```hlsl
float3 mixedGIColor = lerp(SHDiffuseAverage * Material.GBufferData.DiffuseColor, LumenGiDiffuse, View.MooaGlobalIlluminationDirectionality);
```
- 在SH和Lumen之间插值
- 插值因子是MooaGlobalIlluminationDirectionality

**第5行：应用GI颜色**
```hlsl
OutAddColor.rgb = lerp(mixedGIColor * View.MooaGlobalIlluminationColor.rgb, View.MooaGlobalIlluminationColor.rgb, View.MooaGlobalIlluminationColor.a) * View.MooaGlobalIlluminationIntensity;
```
- 乘以MooaGlobalIlluminationColor.rgb
- 在这个结果和MooaGlobalIlluminationColor.rgb之间插值
- 插值因子是MooaGlobalIlluminationColor.a
- 乘以MooaGlobalIlluminationIntensity

**第6行：添加反射**
```hlsl
OutAddColor.rgb += IndirectLighting.Specular * Pow2(ToonGBuffer.ReflectionIntensity);
```
- 加上间接光照高光
- 乘以ReflectionIntensity的平方

**第7行：应用曝光**
```hlsl
OutAddColor.rgb *= View.MooaExposureScale;
```
- 乘以MooaExposureScale

**类比理解：**
```
想象调鸡尾酒：
1. 基酒1：SHDiffuseAverage（天空GI）
2. 基酒2：LumenGiDiffuse（Lumen GI）
3. 混合基酒：lerp(基酒1, 基酒2, 混合比例)
4. 加调料：MooaGlobalIlluminationColor
5. 加甜度：MooaGlobalIlluminationIntensity
6. 加冰块：反射
7. 加冰块倍数：Pow2(ReflectionIntensity)
8. 加水：MooaExposureScale

这杯鸡尾酒就是Toon的间接光照！
```

## 技术细节

### Toon间接光照流程

```
Toon间接光照：
1. 计算SHDiffuseAverage（天空GI）
   ↓
2. 获取LumenGiDiffuse（Lumen GI）
   ↓
3. 混合：lerp(SH, Lumen, MooaGlobalIlluminationDirectionality)
   ↓
4. 应用MooaGlobalIlluminationColor
   ↓
5. 应用MooaGlobalIlluminationIntensity
   ↓
6. 添加反射：IndirectLighting.Specular * Pow2(ReflectionIntensity)
   ↓
7. 应用MooaExposureScale
```

## MooaToon集成总结

### 修改内容
1. 包含ReflectionEnvironmentShared.ush
2. 动态AO强度控制
3. Toon间接光照混合（最复杂的部分）

### 设计意图
- 给Toon提供完整的间接光照控制
- 支持SH和Lumen混合
- 支持颜色、强度、曝光等多维度控制

## 开发提示

### 如何调整Toon间接光照？

**1. 动态AO：**
```
r.MooaDynamicAOIntensity 0.5
```

**2. GI方向性：**
```
r.MooaGlobalIlluminationDirectionality 0.5
```

**3. GI颜色：**
```
r.MooaGlobalIlluminationColor "1,1,1,1"
```

**4. GI强度：**
```
r.MooaGlobalIlluminationIntensity 1.0
```

**5. 曝光：**
```
r.MooaExposureScale 1.0
```

## 总结

DiffuseIndirectComposite.usf是漫反射间接合成着色器，MooaToon在这里有**4处**修改：
1. 包含ReflectionEnvironmentShared.ush
2. 动态AO强度控制（lerp 1和DynamicAmbientOcclusion）
3. Toon间接光照混合（最复杂的部分，7步）

这个文件展示了：
- Toon间接光照的完整控制
- SH和Lumen混合
- 颜色、强度、曝光多维度控制
- 反射强度平方

关键理解：
- Toon间接光照非常复杂
- 支持SH和Lumen混合
- 有超多CVar可以调
- 这是Toon渲染的核心之一！
