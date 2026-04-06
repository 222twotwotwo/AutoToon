# PlanarReflectionRendering.cpp - 平面反射渲染

## 文件信息
- **路径**: `Engine/Source/Runtime/Renderer/Private/PlanarReflectionRendering.cpp`
- **作用**: 平面反射渲染的实现
- **MooaToon修改**: 在平面反射Pass中传递ToonBufferATexture

## 关键代码分析

### 1. ToonBufferATexture传递（第848行）

```cpp
PassParameters-&gt;SceneTextures.GBufferFTexture = SceneTextures.GBufferFTexture;
PassParameters-&gt;SceneTextures.ToonBufferATexture = SceneTextures.ToonBufferATexture;// Mooa GBuffer
PassParameters-&gt;SceneTextures.GBufferVelocityTexture = SceneTextures.GBufferVelocityTexture;
```

#### 零基础解释

这是在平面反射Pass中传递ToonBufferATexture的代码。

**什么是平面反射（Planar Reflection）？**
- 比如镜子、水面的反射
- UE5的平面反射系统
- 渲染场景的反射版本

**这段代码在做什么？**
- 设置平面反射Pass的参数
- 把所有场景纹理传递给着色器
- 包括ToonBufferA（MooaToon新加的）

**为什么要传递ToonBufferA？**
- 平面反射需要渲染Toon物体
- Toon渲染需要ToonBufferA
- 所以反射Pass也需要访问ToonBufferA

**类比理解：**
```
想象拍照片：
- 你要拍一张反射的照片
- 你需要所有原照片的材料
- 包括ToonBufferA（特殊材料）
- 这样才能拍出正确的反射照片
```

## 技术细节

### 平面反射的工作原理

```
1. 渲染场景到反射纹理
   ↓
2. 读取场景纹理（包括ToonBufferA）
   ↓
3. 应用反射效果
   ↓
4. 输出反射结果
```

**ToonBufferA的作用：**
- 反射中的Toon物体也需要正确渲染
- 所以反射Pass也要能访问ToonBufferA

### 与其他文件的关系

```
SceneTextures.cpp: 创建ToonBufferA
    ↓
SceneTextureParameters.cpp: 传递给大多数Pass
    ↓
PlanarReflectionRendering.cpp: 传递给平面反射Pass
    ↓
HLSL着色器: 使用ToonBufferA
```

## MooaToon集成总结

### 修改内容
1. 在平面反射Pass的参数设置中
2. 添加ToonBufferATexture的传递
3. 与其他GBuffer保持一致

### 设计意图
- 平面反射也需要渲染Toon物体
- 所以反射Pass也要能访问ToonBufferA
- 与其他Pass保持一致

## 开发提示

### 如何在自定义Pass中传递ToonBufferA？

参考MooaToon的做法：

```cpp
// 在你的Pass参数中
FYourPass::FParameters* PassParameters = GraphBuilder.AllocParameters&lt;FYourPass::FParameters&gt;();

// 传递其他GBuffer
PassParameters-&gt;SceneTextures.GBufferATexture = SceneTextures.GBufferATexture;
PassParameters-&gt;SceneTextures.GBufferBTexture = SceneTextures.GBufferBTexture;
// ...

// 传递ToonBufferA
PassParameters-&gt;SceneTextures.ToonBufferATexture = SceneTextures.ToonBufferA;

// 传递其他纹理
PassParameters-&gt;SceneTextures.GBufferVelocityTexture = SceneTextures.GBufferVelocityTexture;
```

## 总结

PlanarReflectionRendering.cpp是平面反射渲染的实现文件，MooaToon在这里：
1. 在平面反射Pass中传递ToonBufferATexture

这个修改展示了：
- ToonBufferA需要在所有使用GBuffer的Pass中传递
- 平面反射也需要ToonBufferA
- 保持一致性很重要

关键理解：
- ToonBufferA不是只在一个地方用
- 所有渲染Toon的Pass都需要它
- 平面反射也要支持Toon
