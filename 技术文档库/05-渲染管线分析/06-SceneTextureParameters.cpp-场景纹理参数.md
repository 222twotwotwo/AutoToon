# SceneTextureParameters.cpp - 场景纹理参数

## 文件信息
- **路径**: `Engine/Source/Runtime/Renderer/Private/SceneTextureParameters.cpp`
- **作用**: 设置场景纹理参数，传递给着色器
- **MooaToon修改**: 传递ToonBufferA纹理到着色器参数

## 关键代码分析

### 1. GetSceneTextureParameters函数1 - 从SceneTextures获取（第33-35行）

```cpp
// Mooa GBuffer
Parameters.ToonBufferATexture = GetIfProduced(SceneTextures.ToonBufferA);
// Mooa End
```

#### 零基础解释

这段代码把ToonBufferA纹理从SceneTextures传递到Parameters。

**GetSceneTextureParameters函数做什么？**
- 把渲染图（RDG）中的纹理
- 打包成FSceneTextureParameters结构体
- 传递给着色器使用

**GetIfProduced函数做什么？**
- 获取纹理，如果它被生成了的话
- 如果没生成，返回一个默认值
- 安全地处理可选纹理

**类比理解：**
```
想象打包行李：
- 先放GBufferA、GBufferB、GBufferC...
- 然后放ToonBufferA（MooaToon新加的）
- 最后放其他纹理
- 打包好递给着色器
```

### 2. GetSceneTextureParameters函数2 - 从UniformBuffer获取（第54-56行）

```cpp
// Mooa GBuffer
Parameters.ToonBufferATexture = (*SceneTextureUniformBuffer)->ToonBufferATexture;
// Mooa End
```

#### 零基础解释

这段代码从UniformBuffer中获取ToonBufferA纹理。

**为什么有两个GetSceneTextureParameters函数？**
- 函数1：从RDG（渲染图）中获取
  - 用于渲染过程中
- 函数2：从UniformBuffer中获取
  - 用于已经准备好的UniformBuffer

**两者都要改！**
- 函数1：不改的话，渲染过程中ToonBufferA传不过去
- 函数2：不改的话，UniformBuffer方式ToonBufferA传不过去
- 两个都改，才能保证所有情况都工作

## 技术细节

### 纹理传递的完整流程

```
1. 创建纹理（SceneTextures.cpp）
   ↓
2. 传递到Parameters（SceneTextureParameters.cpp）
   ↓
3. 设置到UniformBuffer（SceneRendering.cpp）
   ↓
4. 着色器中使用（DeferredShadingCommon.ush等）
```

### GetIfProduced函数

```cpp
Parameters.ToonBufferATexture = GetIfProduced(SceneTextures.ToonBufferA);
```

**这个函数的作用：**
- 安全地获取纹理
- 如果纹理没被创建，返回默认值
- 不会崩溃

**什么时候纹理没被创建？**
- 某些渲染路径可能不用ToonBufferA
- 这时候GetIfProduced返回默认值
- 着色器中可以判断是否可用

### FSceneTextureParameters结构体

这个结构体应该在SceneTextureParameters.h中定义，包含：
- 所有GBuffer纹理
- ToonBufferA纹理（MooaToon新加的）
- 其他场景纹理

## MooaToon集成总结

### 修改内容
1. 在第一个GetSceneTextureParameters函数中添加ToonBufferA
2. 在第二个GetSceneTextureParameters函数中添加ToonBufferA

### 设计意图
- 把ToonBufferA纹理传递给着色器
- 确保两种获取方式都支持
- 最小修改：各加一行代码

## 开发提示

### 如何添加自定义纹理参数？

参考MooaToon的做法：

```cpp
// 1. 在第一个函数中添加
Parameters.YourTexture = GetIfProduced(SceneTextures.YourTexture);

// 2. 在第二个函数中添加
Parameters.YourTexture = (*SceneTextureUniformBuffer)->YourTexture;
```

### 为什么要改两个函数？

**只改第一个，不改第二个：**
- RDG方式工作
- UniformBuffer方式不工作
- 有些渲染路径会出错

**只改第二个，不改第一个：**
- UniformBuffer方式工作
- RDG方式不工作
- 有些渲染路径会出错

**两个都改：**
- 所有情况都工作 ✓

## 总结

SceneTextureParameters.cpp是场景纹理参数的设置文件，MooaToon在这里：
1. 在两个GetSceneTextureParameters函数中都添加ToonBufferA

这个修改展示了：
- 如何把新纹理传递给着色器
- 为什么有两个类似的函数
- 确保所有路径都支持

关键理解：
- 纹理创建后需要传递给着色器
- 有多种传递方式，都要支持
- ToonBufferA需要在所有路径中都可用
