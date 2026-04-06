# DeferredLightPixelShaders.usf - 延迟光照像素着色器

## 文件信息
- **路径**: `Engine/Shaders/Private/DeferredLightPixelShaders.usf`
- **作用**: 实现延迟渲染的光照计算
- **MooaToon修改**: 定义MOOA_TOON_DEFERRED_LIGHTING宏、设置Toon上下文、应用Toon曝光

## 关键代码分析

### 1. MOOA_TOON_DEFERRED_LIGHTING宏定义（第6-11行）

```hlsl
// Mooa Toon Shading Model
#define MOOA_TOON_DEFERRED_LIGHTING

// Debug
// #define SUBSTRATE_ENABLED 0
// Mooa End
```

#### 零基础解释

这个宏用于标识MooaToon的延迟光照功能。

**什么是宏定义？**
- 宏就像一个开关或者标签
- 定义后，其他地方可以用`#ifdef MOOA_TOON_DEFERRED_LIGHTING`来检查
- 可以用来条件编译代码

**为什么需要这个宏？**
- 标识这是MooaToon修改过的版本
- 其他文件可能会检查这个宏来决定是否包含Toon代码
- 方便调试和维护

### 2. 设置Toon光照上下文（第359行）

```hlsl
SetMooaToonContext_LightType(ScreenSpaceData.GBuffer, LightData);// Mooa GBuffer
```

#### 零基础解释

在光照计算之前，先设置Toon的光照上下文。

**上下文是什么？**
- 上下文就是一组相关的数据
- 比如：光照类型、光照颜色、像素位置等
- Toon渲染需要这些信息来做特殊处理

**SetMooaToonContext_LightType做了什么？**
- 把当前光照的类型（方向光/点光源/聚光灯）存起来
- Toon渲染可能根据不同光照类型做不同处理
- 比如：方向光可能需要面部阴影，点光源可能不需要

**调用时机：**
```
1. 初始化光照数据
2. ↓
3. 设置Toon上下文（这里）← MooaToon
4. ↓
5. 计算光照
```

### 3. 应用Toon曝光（第398-411行）

```hlsl
// Mooa Toon Deferred Shading
BRANCH
if(ScreenSpaceData.GBuffer.ShadingModelID == SHADINGMODELID_TOON)
{
    OutColor.rgba *= ScreenSpaceData.GBuffer.MooaToonContext.Exposure;
}
else
{
    // So we need PreExposure for both color and alpha
    OutColor.rgba *= GetExposure();
}
// Mooa End
```

#### 零基础解释

这是Toon曝光的特殊处理，与默认曝光不同。

**什么是曝光？**
- 曝光就是调整画面的亮度
- 就像相机的曝光设置
- 太暗了加曝光，太亮了减曝光

**Toon为什么用特殊曝光？**
- 卡通渲染可能需要更灵活的曝光控制
- 可以给每个Toon材质单独设置曝光
- 而不是用全局的曝光

**代码逻辑：**
```
如果是Toon材质：
    颜色 × Toon自己的曝光值
否则：
    颜色 × 全局曝光值
```

**类比理解：**
```
想象拍照：
- 普通照片：用相机的全局曝光设置
- Toon照片：每个角色可以有自己的曝光设置
  - 角色A：+0.5曝光（亮一点）
  - 角色B：-0.3曝光（暗一点）
```

## 技术细节

### 延迟光照的流程

```
DeferredLightPixelShaders.usf的工作流程：

1. 读取GBuffer数据
   ├─ 位置、法线、基础颜色
   └─ ToonBufferA（如果是Toon材质）

2. 初始化光照数据
   ├─ 光照类型、颜色、强度
   └─ 设置Toon上下文（MooaToon）← 这里

3. 计算光照
   ├─ 调用ToonBxDF（如果是Toon）
   └─ 或调用其他BxDF

4. 应用曝光
   ├─ Toon用自己的曝光（MooaToon）← 这里
   └─ 其他用全局曝光

5. 输出颜色
```

### 为什么需要两处Toon修改？

**第一处（SetMooaToonContext_LightType）：**
- 位置：光照计算之前
- 作用：准备Toon需要的上下文数据
- 必须：没有上下文，Toon渲染无法正常工作

**第二处（曝光处理）：**
- 位置：光照计算之后
- 作用：应用Toon特殊的曝光
- 可选：但这是Toon效果的一部分

### BRANCH的性能考虑

```hlsl
BRANCH if (条件)
{
    ...
}
```

**BRANCH的作用：**
- 告诉GPU：这个分支可能不执行
- GPU可以跳过不需要的分支

**为什么用BRANCH？**
- Toon材质可能只占场景的一小部分
- 大多数像素不需要执行Toon分支
- 可以节省性能

## MooaToon集成总结

### 修改内容
1. 定义MOOA_TOON_DEFERRED_LIGHTING宏
2. 在光照计算前设置Toon上下文
3. 在光照计算后应用Toon曝光

### 设计意图
- 在延迟光照流程中插入Toon逻辑
- 保持与引擎现有流程的集成
- 提供Toon专用的曝光控制

## 开发提示

### 如何在延迟着色器中添加自定义逻辑？

参考MooaToon的做法：

```hlsl
// 1. 在文件开头定义宏
#define YOUR_CUSTOM_FEATURE

// 2. 在光照计算前设置上下文
SetYourContext(GBuffer, LightData);

// 3. 在光照计算后做处理
BRANCH if (GBuffer.ShadingModelID == SHADINGMODELID_YOURS)
{
    OutColor *= YourCustomFactor;
}
```

### 曝光的不同方式

```hlsl
// 方式1：全局曝光（默认）
OutColor *= GetExposure();

// 方式2：材质专用曝光（Toon方式）
OutColor *= GBuffer.YourContext.Exposure;

// 方式3：后期处理曝光
// 在PostProcess中做，不是这里
```

### 如何调试Toon曝光？

```hlsl
// 调试：显示曝光值
BRANCH if (GBuffer.ShadingModelID == SHADINGMODELID_TOON)
{
    OutColor.rgb = GBuffer.MooaToonContext.Exposure;
}
```

## 总结

DeferredLightPixelShaders.usf是延迟光照的核心文件，MooaToon在这里：
1. 定义了功能宏
2. 设置了Toon光照上下文
3. 应用了Toon特殊曝光

这个文件展示了：
- 如何在延迟光照流程中插入自定义逻辑
- 如何给着色模型提供专用参数
- 如何使用分支优化性能

关键理解：
- 延迟光照是一个标准流程
- 自定义着色模型可以在各个环节插入
- 上下文数据很重要，需要提前准备
