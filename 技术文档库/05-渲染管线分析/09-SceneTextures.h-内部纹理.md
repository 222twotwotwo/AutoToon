# SceneTextures.h - 场景纹理（内部）

## 文件信息
- **路径**: `Engine/Source/Runtime/Renderer/Internal/SceneTextures.h`
- **作用**: 定义场景纹理的内部结构体
- **MooaToon修改**: 在FSceneTextures中添加ToonBufferA

## 关键代码分析

### 1. FSceneTextures结构体（第132-139行）

```cpp
FRDGTextureRef GBufferB{};
FRDGTextureRef GBufferC{};
FRDGTextureRef GBufferD{};
FRDGTextureRef GBufferE{};
FRDGTextureRef GBufferF{};
// Mooa GBuffer
FRDGTextureRef ToonBufferA{};
// Mooa End
```

#### 零基础解释

这是FSceneTextures结构体，定义了所有场景纹理。

**什么是FRDGTextureRef？**
- RDG纹理引用
- Render Dependency Graph（渲染依赖图）
- 现代渲染资源管理
- 智能指针，自动管理生命周期

**{}是什么？**
- 列表初始化
- C++11的语法
- 初始化为nullptr

**这个结构体做什么？**
- 存储所有场景纹理
- 包括GBufferA到GBufferF
- 包括ToonBufferA（MooaToon新加的）
- 渲染管线中传递纹理

**为什么要加ToonBufferA？**
- Toon渲染需要额外的缓冲区
- 要和其他GBuffer放在一起
- 方便统一管理

**类比理解：**
```
想象一个工具箱：
- 里面有各种工具：
  - 锤子（GBufferA）
  - 螺丝刀（GBufferB）
  - 扳手（GBufferC）
  - ...
  - 特殊工具（ToonBufferA，新加的）

这个结构体就是工具箱！
```

## 技术细节

### FRDGTextureRef的特点

```cpp
// 普通指针：
FRDGTexture* Texture;
// 缺点：手动管理生命周期

// RDG引用：
FRDGTextureRef Texture;
// 优点：自动管理生命周期
```

**RDG的好处：**
1. **自动生命周期管理
2. **自动资源回收
3. **自动依赖追踪
4. **更好的性能优化

### FSceneTextures的使用场景

```
渲染管线：
1. BasePass → 写入GBuffer和ToonBufferA
   ↓
2. 延迟光照 → 读取GBuffer和ToonBufferA
   ↓
3. 后处理 → 读取场景纹理
```

**FSceneTextures在这个流程中：**
- 在各个Pass之间传递纹理
- 统一管理所有场景纹理
- RDG自动处理依赖关系

## MooaToon集成总结

### 修改内容
1. 在FSceneTextures结构体中
2. 添加ToonBufferA成员
3. 类型是FRDGTextureRef
4. 初始化为{}（nullptr）

### 设计意图
- ToonBufferA是场景纹理的一部分
- 与其他GBuffer统一管理
- 使用RDG系统

## 开发提示

### 如何添加自定义RDG纹理？

参考MooaToon的做法：

```cpp
struct FSceneTextures
{
    // 其他纹理...
    
    // Mooa GBuffer
    FRDGTextureRef ToonBufferA{};
    // Mooa End
    
    // 其他纹理...
};
```

### 如何使用FSceneTextures？

```cpp
// 在渲染管线中
void Render(FRDGBuilder&amp; GraphBuilder, FSceneTextures&amp; SceneTextures)
{
    // 使用ToonBufferA
    if (SceneTextures.ToonBufferA)
    {
        // 读取或写入
    }
}
```

## 总结

SceneTextures.h是场景纹理的内部头文件，MooaToon在这里：
1. 在FSceneTextures中添加ToonBufferA

这个修改展示了：
- 如何扩展UE5的场景纹理系统
- 如何使用RDG纹理引用
- 新纹理如何与现有系统集成

关键理解：
- FSceneTextures是场景纹理的容器
- RDG自动管理资源
- ToonBufferA是场景纹理的一部分
