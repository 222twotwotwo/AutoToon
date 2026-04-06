# SceneView.h - 视图Uniform缓冲区

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Public/SceneView.h`
- **作用**: 定义视图Uniform缓冲区
- **MooaToon修改**: 添加大量MooaToon参数和纹理

## 关键代码分析

### 1. VIEW_UNIFORM_BUFFER_MEMBER_TABLE宏（第793-830行）

```cpp
// Mooa View Uniform Buffer
// View uniform buffer member declarations
#define VIEW_UNIFORM_BUFFER_MEMBER_TABLE \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaDebugValue) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaDynamicAOIntensity) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaDiffuseColorRampEnablePostRampShadow) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaDiffuseColorRampEnablePostRampMaterialAO) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaDiffuseColorRampUVOffsetMaxRange) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaSpecularColorRampUVOffsetMaxRange) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaRimLightMaxWidth) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaRimLightDepthTestThreshold) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaRimLightDepthTestFadeDistance) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaHairShadowIntensity) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaHairShadowWidth) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaHairShadowDepthTestThreshold) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaHairShadowDepthTestFadeDistance) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaExposureScale) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaLightSaturationScale) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaGlobalIlluminationDirectionality) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaGlobalIlluminationLumenNormalFlatten) \
	VIEW_UNIFORM_BUFFER_MEMBER(FVector4f, MooaGlobalIlluminationColor) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaGlobalIlluminationIntensity) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaShadowBias) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaEnableScreenSpaceHairShadowDirectionalLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaEnableScreenSpaceHairShadowPointLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaEnableScreenSpaceHairShadowSpotLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaEnableScreenSpaceHairShadowRectLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaEnableDistanceFieldFacialShadowDirectionalLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaEnableDistanceFieldFacialShadowPointLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaEnableDistanceFieldFacialShadowSpotLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaEnableDistanceFieldFacialShadowRectLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaShadowColorIntensityDirectionalLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaShadowColorIntensityPointLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaShadowColorIntensitySpotLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaShadowColorIntensityRectLights) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaRimLightIntensity) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaRimLightSaturationScale) \
	VIEW_UNIFORM_BUFFER_MEMBER(uint32, MooaWorldType)
```

#### 零基础解释

这是MooaToon添加的视图Uniform缓冲区成员表。

**什么是VIEW_UNIFORM_BUFFER_MEMBER_TABLE？**
- 宏定义
- 列出所有视图Uniform缓冲区的成员
- 用于自动生成C++和HLSL代码

**MooaToon添加了多少参数？**
- **38个**参数！
- 涵盖了Toon渲染的所有方面

**参数分类：**

| 类别 | 参数数量 |
|------|---------|
| **调试** | 1个 |
| **Ramp纹理** | 4个 |
| **轮廓光（Rim Light）** | 5个 |
| **头发阴影** | 4个 |
| **全局光照（GI）** | 5个 |
| **光照** | 3个 |
| **屏幕空间头发阴影** | 4个 |
| **距离场面部阴影** | 4个 |
| **阴影颜色强度** | 4个 |
| **其他** | 4个 |

**部分重要参数说明：**

**1. 调试参数：**
- `MooaDebugValue`：调试用的值

**2. Ramp纹理参数：**
- `MooaDiffuseColorRampEnablePostRampShadow`：是否启用Ramp后阴影
- `MooaDiffuseColorRampEnablePostRampMaterialAO`：是否启用Ramp后材质AO
- `MooaDiffuseColorRampUVOffsetMaxRange`：漫反射Ramp UV偏移最大范围
- `MooaSpecularColorRampUVOffsetMaxRange`：高光Ramp UV偏移最大范围

**3. 轮廓光参数：**
- `MooaRimLightMaxWidth`：轮廓光最大宽度
- `MooaRimLightDepthTestThreshold`：轮廓光深度测试阈值
- `MooaRimLightDepthTestFadeDistance`：轮廓光深度测试衰减距离
- `MooaRimLightIntensity`：轮廓光强度
- `MooaRimLightSaturationScale`：轮廓光饱和度缩放

**4. 头发阴影参数：**
- `MooaHairShadowIntensity`：头发阴影强度
- `MooaHairShadowWidth`：头发阴影宽度
- `MooaHairShadowDepthTestThreshold`：头发阴影深度测试阈值
- `MooaHairShadowDepthTestFadeDistance`：头发阴影深度测试衰减距离

**5. 全局光照参数：**
- `MooaGlobalIlluminationDirectionality`：全局光照方向性
- `MooaGlobalIlluminationLumenNormalFlatten`：全局光照Lumen法线扁平化
- `MooaGlobalIlluminationColor`：全局光照颜色
- `MooaGlobalIlluminationIntensity`：全局光照强度

**6. 光照参数：**
- `MooaExposureScale`：曝光缩放
- `MooaLightSaturationScale`：光照饱和度缩放
- `MooaShadowBias`：阴影偏移

**7. 屏幕空间头发阴影：**
- `MooaEnableScreenSpaceHairShadowDirectionalLights`：是否启用方向光的屏幕空间头发阴影
- `MooaEnableScreenSpaceHairShadowPointLights`：是否启用点光源的屏幕空间头发阴影
- `MooaEnableScreenSpaceHairShadowSpotLights`：是否启用聚光灯的屏幕空间头发阴影
- `MooaEnableScreenSpaceHairShadowRectLights`：是否启用矩形光的屏幕空间头发阴影

**8. 距离场面部阴影：**
- `MooaEnableDistanceFieldFacialShadowDirectionalLights`：是否启用方向光的距离场面部阴影
- `MooaEnableDistanceFieldFacialShadowPointLights`：是否启用点光源的距离场面部阴影
- `MooaEnableDistanceFieldFacialShadowSpotLights`：是否启用聚光灯的距离场面部阴影
- `MooaEnableDistanceFieldFacialShadowRectLights`：是否启用矩形光的距离场面部阴影

**9. 阴影颜色强度：**
- `MooaShadowColorIntensityDirectionalLights`：方向光阴影颜色强度
- `MooaShadowColorIntensityPointLights`：点光源阴影颜色强度
- `MooaShadowColorIntensitySpotLights`：聚光灯阴影颜色强度
- `MooaShadowColorIntensityRectLights`：矩形光阴影颜色强度

**10. 其他：**
- `MooaWorldType`：世界类型

**类比理解：**
```
想象一个调色板：
- 有很多旋钮和开关
- 每个旋钮控制一个参数
- MooaToon加了38个旋钮！
- 可以精细控制Toon渲染的每个方面
```

### 2. FViewUniformShaderParameters结构体（第1095-1100行）

```cpp
// Mooa View Uniform Buffer Texture
SHADER_PARAMETER_TEXTURE(Texture2D, MooaGlobalDiffuseColorRampAtlas)
SHADER_PARAMETER(int, MooaGlobalDiffuseColorRampAtlasHeight)
SHADER_PARAMETER_TEXTURE(Texture2D, MooaGlobalSpecularColorRampAtlas)
SHADER_PARAMETER(int, MooaGlobalSpecularColorRampAtlasHeight)
// Mooa End
```

#### 零基础解释

这是在FViewUniformShaderParameters中添加的MooaToon纹理参数。

**添加了4个参数：**
1. `MooaGlobalDiffuseColorRampAtlas`：全局漫反射Ramp图集
2. `MooaGlobalDiffuseColorRampAtlasHeight`：漫反射Ramp图集高度
3. `MooaGlobalSpecularColorRampAtlas`：全局高光Ramp图集
4. `MooaGlobalSpecularColorRampAtlasHeight`：高光Ramp图集高度

**为什么需要这些？**
- 着色器需要访问Ramp图集
- 要在Uniform缓冲区中传递
- HLSL才能采样这些纹理

## 技术细节

### VIEW_UNIFORM_BUFFER_MEMBER_TABLE的工作原理

```cpp
// 你写的：
#define VIEW_UNIFORM_BUFFER_MEMBER_TABLE \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaDebugValue) \
	VIEW_UNIFORM_BUFFER_MEMBER(float, MooaDynamicAOIntensity) \
	...

// 宏展开后（C++端）：
struct FViewUniformShaderParameters
{
	float MooaDebugValue;
	float MooaDynamicAOIntensity;
	...
};

// 宏展开后（HLSL端）：
cbuffer ViewUniformBuffer
{
	float MooaDebugValue;
	float MooaDynamicAOIntensity;
	...
}
```

**好处：**
- 一次定义，两端都有
- C++和HLSL自动同步
- 不用手动写两遍

### 38个参数的意义

MooaToon添加了**38个**参数，说明：
1. Toon渲染需要大量控制
2. MooaToon非常灵活
3. 可以精细调整每个方面
4. 支持多种光源类型

## MooaToon集成总结

### 修改内容
1. 在VIEW_UNIFORM_BUFFER_MEMBER_TABLE中添加38个参数
2. 在FViewUniformShaderParameters中添加4个纹理参数

### 设计意图
- 给Toon渲染提供全面的参数控制
- 通过CVar实时调整
- 支持多种光源类型

## 开发提示

### 如何在HLSL中使用这些参数？

```hlsl
// 在着色器中
float RimLightWidth = View.MooaRimLightMaxWidth;
float HairShadowIntensity = View.MooaHairShadowIntensity;

// 采样Ramp图集
Texture2D DiffuseRampAtlas = View.MooaGlobalDiffuseColorRampAtlas;
int RampAtlasHeight = View.MooaGlobalDiffuseColorRampAtlasHeight;
```

### 如何在C++中设置这些参数？

这些参数通常通过CVar设置，在SceneRendering.cpp中更新。

## 总结

SceneView.h是视图Uniform缓冲区的头文件，MooaToon在这里：
1. 添加了**38个**MooaToon参数（VIEW_UNIFORM_BUFFER_MEMBER_TABLE）
2. 添加了**4个**纹理参数（FViewUniformShaderParameters）

这个修改展示了：
- Toon渲染需要大量参数控制
- MooaToon非常灵活和强大
- 如何通过Uniform缓冲区传递大量参数

关键理解：
- VIEW_UNIFORM_BUFFER_MEMBER_TABLE自动生成C++和HLSL
- MooaToon有38个可调节参数
- 还传递了Ramp图集纹理
