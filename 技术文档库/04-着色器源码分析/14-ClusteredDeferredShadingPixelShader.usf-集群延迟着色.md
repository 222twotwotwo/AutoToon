# ClusteredDeferredShadingPixelShader.usf - 集群延迟着色

## 文件信息
- **路径**: `Engine/Shaders/Private/ClusteredDeferredShadingPixelShader.usf`
- **作用**: 实现集群化延迟着色（Clustered Deferred Shading）
- **MooaToon修改**: 将Toon着色模型加入单着色模型通道列表

## 关键代码分析

### 1. GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM宏调用（第536-538行）

```hlsl
// Mooa Toon Shading Model
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_TOON, 				PixelShadingModelID, CompositedLighting, ScreenUV, CulledLightGridHeader, Dither, FirstNonSimpleLightIndex);
// Mooa End
```

#### 零基础解释

这一行把Toon着色模型加入了单着色模型通道的列表。

**什么是集群延迟着色？**
- 普通延迟着色：每个光源画一个全屏四边形
- 集群延迟着色：把屏幕分成小块（集群），只处理影响该集群的光源
- 性能更好：减少不必要的计算

**什么是单着色模型通道（USE_PASS_PER_SHADING_MODEL）？**
- 给每个着色模型单独一个渲染通道
- 优点：代码更清晰，可能更优化
- 缺点：需要更多的Draw Call

**GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM做了什么？**
- 这是一个宏
- 为指定的着色模型计算光照
- 参数：着色模型ID、其他渲染参数

**类比理解：**
```
想象一个工厂流水线：
- 普通方式：所有产品在一条线上，混杂在一起
- 单着色模型通道：每种产品一条专门的线
  - 产品线1：默认光照
  - 产品线2：次表面散射
  - ...
  - 产品线N：Toon（MooaToon新加的）

GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM就是：
"给这个产品线分配任务！"
```

## 技术细节

### 完整的着色模型列表

让我们看一下所有使用单通道的着色模型：

```hlsl
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_DEFAULT_LIT, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_SUBSURFACE, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_PREINTEGRATED_SKIN, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_CLEAR_COAT, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_SUBSURFACE_PROFILE, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_TWOSIDED_FOLIAGE, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_HAIR, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_CLOTH, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_EYE, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_SINGLELAYERWATER, ...);
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_TOON, ...);  // MooaToon
```

**共11个着色模型！**

### USE_PASS_PER_SHADING_MODEL的工作原理

```hlsl
#if USE_PASS_PER_SHADING_MODEL
    // 每个着色模型单独处理
    GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_DEFAULT_LIT, ...);
    GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_TOON, ...);
    ...
#else
    // 统一处理所有着色模型
    CompositedLighting += GetLightGridLocalLighting(...);
#endif
```

**两种模式对比：**

| 模式 | 优点 | 缺点 |
|-----|------|------|
| **USE_PASS_PER_SHADING_MODEL** | 代码清晰、可优化 | 更多Draw Call |
| **统一处理** | 更少Draw Call | 代码复杂 |

### GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM宏展开

这个宏大概是这样的（简化版）：

```hlsl
#define GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(ShadingModelID, ...) \
    if (PixelShadingModelID == ShadingModelID) \
    { \
        CompositedLighting += GetLightGridLocalLightingForShadingModel(ShadingModelID, ...); \
    }
```

**工作流程：**
1. 检查当前像素是不是这个着色模型
2. 如果是，计算光照
3. 累加到结果中

## MooaToon集成总结

### 修改内容
1. 在单着色模型通道列表中
2. 添加Toon着色模型的宏调用
3. 让Toon也能使用单通道优化

### 设计意图
- 与其他着色模型保持一致
- 享受单通道的优化好处
- 最小修改：只加一行代码

## 开发提示

### 如何给你的着色模型添加单通道支持？

步骤1：在ClusteredDeferredShadingPixelShader.usf中添加
```hlsl
GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM(SHADINGMODELID_YOURS, ...);
```

步骤2：确保在ShadingModels.ush中集成了你的BxDF
```hlsl
case SHADINGMODELID_YOURS:
    return YourBxDF(...);
```

### 如何检查是否使用单通道？

```hlsl
#if USE_PASS_PER_SHADING_MODEL
    // 单通道模式
#else
    // 统一模式
#endif
```

### 两种模式都要支持！

重要：你的着色模型应该在两种模式下都能工作
- 单通道模式：通过GET_LIGHT_GRID_LOCAL_LIGHTING_SINGLE_SM
- 统一模式：通过GetLightGridLocalLighting

好消息：如果你的BxDF在ShadingModels.ush中正确集成了，两种模式都能自动工作！

## 总结

ClusteredDeferredShadingPixelShader.usf是集群延迟着色的实现，MooaToon在这里：
1. 把Toon加入了单着色模型通道列表

这个修改很小，但很重要：
- 没有它，Toon在单通道模式下不会计算光照
- Toon画面就会是黑的
- 这是完整集成的必要部分

关键理解：
- 引擎有不同的渲染路径
- 自定义着色模型需要在所有路径中都支持
- 通常只需要加一行代码（调用宏）
- 只要BxDF正确集成，其他都是自动的
