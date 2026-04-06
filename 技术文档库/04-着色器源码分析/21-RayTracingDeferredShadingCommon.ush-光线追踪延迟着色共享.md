# RayTracingDeferredShadingCommon.ush - 光线追踪延迟着色共享

## 文件信息
- **路径**: `Engine/Shaders/Private/RayTracing/RayTracingDeferredShadingCommon.ush`
- **作用**: 光线追踪延迟着色共享函数
- **MooaToon修改**: 添加ToonBufferA加载和传递

## 关键代码分析

### 1. ToonBufferA加载（第17-19行）

```cpp
// Mooa GBuffer
float4 ToonBufferA = ToonBufferATexture.Load(int3(PixelCoord, 0));
```

#### 零基础解释

这是加载ToonBufferA的代码。

**这里做了什么？**
- 从ToonBufferATexture加载数据
- 位置是PixelCoord（像素坐标）
- mip level是0

**为什么需要加载ToonBufferA？**
- 光线追踪也需要Toon数据
- 要解码ToonGBufferData
- 所以必须加载ToonBufferA

### 2. 传递ToonBufferA到DecodeGBufferData（第33-34行）

```cpp
// Mooa GBuffer
return DecodeGBufferData(GBufferA, GBufferB, GBufferC, GBufferD, GBufferE, GBufferF, ToonBufferA, GBufferVelocity, CustomNativeDepth, CustomStencil, SceneDepth, bGetNormalizedNormal, CheckerFromPixelPos(PixelCoord));
```

#### 零基础解释

这是把ToonBufferA传给DecodeGBufferData函数。

**DecodeGBufferData的参数：**
```
1. GBufferA
2. GBufferB
3. GBufferC
4. GBufferD
5. GBufferE
6. GBufferF
7. ToonBufferA ← 新加的！
8. GBufferVelocity
9. CustomNativeDepth
10. CustomStencil
11. SceneDepth
12. bGetNormalizedNormal
13. CheckerFromPixelPos(PixelCoord)
```

**为什么要传ToonBufferA？**
- DecodeGBufferData需要ToonBufferA
- 才能解码出FToonGBufferData
- 光线追踪才能用Toon数据

## 技术细节

### 光线追踪延迟着色流程

```
光线追踪延迟着色：
1. 加载所有GBuffer（包括ToonBufferA）
   ↓
2. 调用DecodeGBufferData解码
   ↓
3. 得到FGBufferData（包含Toon数据）
   ↓
4. 进行光线追踪计算
```

## MooaToon集成总结

### 修改内容
1. 加载ToonBufferA
2. 传递ToonBufferA到DecodeGBufferData

### 设计意图
- 光线追踪也需要Toon数据
- 保持与延迟着色一致
- ToonBufferA是必需的

## 总结

RayTracingDeferredShadingCommon.ush是光线追踪延迟着色共享文件，MooaToon在这里：
1. 加载ToonBufferA
2. 传递ToonBufferA到DecodeGBufferData

这个文件展示了：
- 光线追踪也需要ToonBufferA
- 保持与延迟着色一致
- Toon数据在多种渲染路径中都需要

关键理解：
- ToonBufferA是必需的
- 光线追踪也用Toon数据
- DecodeGBufferData需要ToonBufferA
