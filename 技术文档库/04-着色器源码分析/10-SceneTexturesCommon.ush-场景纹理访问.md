# SceneTexturesCommon.ush - 场景纹理公共访问

## 文件信息
- **路径**: `Engine/Shaders/Private/SceneTexturesCommon.ush`
- **作用**: 定义场景纹理的公共访问接口和采样器
- **MooaToon修改**: 新增ToonBufferA纹理采样器定义

## 关键代码分析

### 1. ToonBufferA采样器定义（第34-36行）

```hlsl
// Mooa GBuffer
#define SceneTexturesStruct_ToonBufferATextureSampler SceneTexturesStruct.PointClampSampler
// Mooa End
```

#### 零基础解释

这段代码定义了ToonBufferA纹理的采样器。让我一步一步解释：

**什么是采样器？**
- 采样器（Sampler）告诉GPU如何从纹理中读取像素
- 就像给相机设置滤镜一样，决定读取纹理时的效果

**PointClampSampler是什么？**
- **Point（点采样）**: 直接读取最近的像素，不进行插值
  - 优点：速度快，保持像素化风格
  - 适合：卡通渲染，因为我们需要清晰的颜色边界
- **Clamp（钳制）**: 纹理坐标超出[0,1]范围时，使用边界像素
  - 优点：不会出现纹理重复的瑕疵

**为什么ToonBufferA用PointClamp？**
1. ToonBufferA存储的是编码数据，不是颜色
2. 我们需要精确读取每个像素的原始值
3. 点采样能保证数据的准确性

### 2. 其他GBuffer采样器参考（第21-32行）

```hlsl
#define SceneTexturesStruct_GBufferATextureSampler SceneTexturesStruct.PointClampSampler
#define SceneTexturesStruct_GBufferBTextureSampler SceneTexturesStruct.PointClampSampler
#define SceneTexturesStruct_GBufferCTextureSampler SceneTexturesStruct.PointClampSampler
#define SceneTexturesStruct_GBufferDTextureSampler SceneTexturesStruct.PointClampSampler
#define SceneTexturesStruct_GBufferETextureSampler SceneTexturesStruct.PointClampSampler
#define SceneTexturesStruct_GBufferFTextureSampler SceneTexturesStruct.PointClampSampler
```

#### 零基础解释

可以看到，所有的GBuffer纹理都使用PointClamp采样器。这是因为：

**GBuffer存储的是数据，不是图片**
- GBufferA：法线、粗糙度
- GBufferB：基础颜色、金属度
- GBufferC：自定义数据
- 这些数据需要精确读取，不能模糊

**类比理解**
- 想象GBuffer是一张数据表格
- 每个格子存的是数字（比如法线坐标）
- 我们需要精确读取每个格子的数字
- 不能把相邻格子的数字平均起来

## 技术细节

### 采样器类型对比

| 采样器类型 | 用途 | 适用场景 |
|-----------|------|---------|
| **PointClamp** | 精确读取数据 | GBuffer、ToonBuffer |
| **LinearClamp** | 平滑过渡 | 普通纹理、颜色 |
| **PointWrap** | 精确读取+重复 | 像素风格重复纹理 |
| **LinearWrap** | 平滑+重复 | 普通重复纹理 |

### 为什么不用线性采样？

**线性采样（Linear）的问题：**
```
假设ToonBufferA存的是Ramp索引：
像素1: 索引5（红色）
像素2: 索引10（蓝色）

用线性采样读取中间位置：
结果 = (5 + 10) / 2 = 7.5 → 索引7或8（紫色）

但我们想要的是：
要么是红色，要么是蓝色，不要中间色！
```

**点采样（Point）的优势：**
```
读取中间位置时：
直接选最近的像素 → 要么索引5，要么索引10
保持数据的完整性
```

## MooaToon集成总结

### 修改内容
1. 在GBuffer采样器定义区域之后
2. 新增ToonBufferA采样器宏
3. 使用与其他GBuffer相同的PointClamp采样器

### 设计意图
- 保持一致性：ToonBufferA与其他GBuffer使用相同的采样策略
- 数据精确性：点采样保证编码数据不被破坏
- 性能优化：点采样比线性采样稍快

## 开发提示

### 如何访问ToonBufferA？

```hlsl
// 方法1：通过SceneTexturesStruct访问
float4 ToonBufferValue = Texture2DSample(
    SceneTexturesStruct.ToonBufferATexture,
    SceneTexturesStruct_ToonBufferATextureSampler,
    ScreenUV
);

// 方法2：直接Load（更快，需要像素坐标）
float4 ToonBufferValue = SceneTexturesStruct.ToonBufferATexture.Load(
    int3(PixelPos, 0)
);
```

### 什么时候用哪种方法？

| 方法 | 适用场景 | 性能 |
|-----|---------|------|
| **Sample** | 需要UV坐标，可能有缩放 | 中等 |
| **Load** | 精确像素坐标，1:1映射 | 快 |

## 总结

SceneTexturesCommon.ush是场景纹理的公共配置文件，MooaToon在这里只做了一件小事：
- 给ToonBufferA定义了采样器

但这件小事很重要，因为：
1. 它让ToonBufferA能被正确访问
2. 它保证了数据读取的精确性
3. 它保持了与引擎其他部分的一致性

这就是引擎修改的特点：
- 很多修改都是在现有系统中"加一点东西"
- 不需要大改，就能集成新功能
- 关键是找到正确的"切入点"
