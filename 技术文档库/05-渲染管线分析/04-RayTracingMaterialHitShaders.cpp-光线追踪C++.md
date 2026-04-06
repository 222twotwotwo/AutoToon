# RayTracingMaterialHitShaders.cpp - 光线追踪材质命中着色器（C++）

## 文件信息
- **路径**: `Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingMaterialHitShaders.cpp`
- **作用**: 光线追踪材质命中着色器的C++端管理
- **MooaToon修改**: 让Toon着色模型也编译任意命中着色器（AHS）

## 关键代码分析

### 1. 编译时判断是否需要任意命中着色器（第148行）

```cpp
const bool bWantAnyHitShader = ((CVarCompileMaterialAHS.Get(Parameters.Platform) != 0) && (Parameters.MaterialParameters.bIsMasked || IsTranslucentOnlyBlendMode(Parameters.MaterialParameters) || Parameters.MaterialParameters.ShadingModels.HasShadingModel(MSM_Toon))); // Mooa Ray Tracing Shadow
```

#### 零基础解释

这段代码在编译着色器时判断：是否需要编译任意命中着色器（AHS）。

**什么是任意命中着色器（Any Hit Shader, AHS）？**
- 光线追踪时，光线可能穿过多个物体
- AHS在每次命中时调用
- 可以控制是否忽略这次命中
- 用于实现特殊的阴影效果

**原来的判断条件：**
```
需要AHS吗？
├─ 遮罩材质？
└─ 半透明材质？
```

**修改后的判断条件：**
```
需要AHS吗？
├─ 遮罩材质？
├─ 半透明材质？
└─ Toon着色模型？ ← MooaToon新加的
```

**为什么Toon需要AHS？**
- Toon有特殊的阴影控制
- 比如：禁用自阴影、禁用Toon之间投射阴影
- 这些都需要在AHS中实现
- 所以Toon材质也需要编译AHS

**类比理解：**
```
想象一家餐厅：
- 原来只给"特殊顾客"（遮罩、半透明）提供"定制服务"（AHS）
- 现在"Toon顾客"也需要定制服务
- 所以把Toon也加入了名单
```

### 2. 运行时判断是否使用任意命中着色器（第333行）

```cpp
const bool UseAnyHitShader = (MaterialResource.IsMasked() || IsTranslucentOnlyBlendMode(MaterialResource) || MaterialResource.GetShadingModels().HasShadingModel(MSM_Toon)) && GCompileRayTracingMaterialAHS; // Mooa Ray Tracing Shadow
```

#### 零基础解释

这段代码在运行时判断：是否使用任意命中着色器。

**和编译时的区别：**
- 编译时：决定是否编译这个shader
- 运行时：决定是否使用这个shader

**判断条件和编译时一样：**
- 遮罩材质，或
- 半透明材质，或
- Toon着色模型 ← MooaToon新加的

**为什么两处都要改？**
- 编译时：如果不改，Toon的AHS根本不会被编译
- 运行时：如果不改，即使编译了也不会被使用
- 两处都要改，才能正常工作

## 技术细节

### 任意命中着色器的完整流程

```
编译阶段：
1. 判断是否需要编译AHS（第148行）
   ├─ 遮罩？半透明？Toon？
   └─ 是 → 编译AHS
   └─ 否 → 不编译AHS

运行阶段：
2. 判断是否使用AHS（第333行）
   ├─ 遮罩？半透明？Toon？
   └─ 是 → 使用AHS
   └─ 否 → 不使用AHS

3. 如果使用AHS
   ├─ 光线追踪时调用AHS
   ├─ 执行Toon的阴影控制逻辑
   └─ 可能调用IgnoreHit()
```

### HasShadingModel函数

```cpp
Parameters.MaterialParameters.ShadingModels.HasShadingModel(MSM_Toon)
```

**这个函数做什么？**
- 检查材质是否使用了指定的着色模型
- 返回true/false

**ShadingModels是一个集合：**
- 一个材质可以使用多个着色模型
- HasShadingModel检查是否包含某个模型

### CVarCompileMaterialAHS

```cpp
static FShaderPlatformCachedIniValue<int32> CVarCompileMaterialAHS(TEXT("r.RayTracing.CompileMaterialAHS"));
```

**这是什么？**
- 控制台变量（CVar）
- 可以在运行时修改
- 控制是否编译材质AHS

**使用方法：**
```
r.RayTracing.CompileMaterialAHS 1  // 启用
r.RayTracing.CompileMaterialAHS 0  // 禁用
```

## MooaToon集成总结

### 修改内容
1. 在编译时判断中添加Toon着色模型
2. 在运行时判断中添加Toon着色模型

### 设计意图
- 让Toon材质也能使用任意命中着色器
- 支持Toon的特殊阴影控制
- 最小修改：只加两个条件判断

## 开发提示

### 如何让你的着色模型也使用AHS？

参考MooaToon的做法：

```cpp
// 1. 在编译时判断中添加
const bool bWantAnyHitShader = (... || Parameters.MaterialParameters.ShadingModels.HasShadingModel(MSM_Yours));

// 2. 在运行时判断中添加
const bool UseAnyHitShader = (... || MaterialResource.GetShadingModels().HasShadingModel(MSM_Yours));
```

### 为什么要改两处？

**只改编译时，不改运行时：**
- AHS会被编译
- 但运行时不会使用
- 白费功夫

**只改运行时，不改编译时：**
- 运行时想使用AHS
- 但AHS根本没编译
- 会出错

**两处都改：**
- 完美工作 ✓

### HasAnyShadingModel vs HasShadingModel

```cpp
// 检查是否有多个中的任意一个
ShadingModels.HasAnyShadingModel({ MSM_A, MSM_B, MSM_C });

// 检查是否有指定的一个
ShadingModels.HasShadingModel(MSM_A);
```

## 总结

RayTracingMaterialHitShaders.cpp是光线追踪命中着色器的C++管理文件，MooaToon在这里：
1. 在编译时判断中添加Toon
2. 在运行时判断中添加Toon

这个修改展示了：
- 如何让自定义着色模型使用特殊的shader
- 编译时和运行时判断的区别
- 最小修改原则

关键理解：
- 有些shader不是所有材质都需要
- 需要时才编译和使用
- Toon需要AHS来做特殊阴影控制
