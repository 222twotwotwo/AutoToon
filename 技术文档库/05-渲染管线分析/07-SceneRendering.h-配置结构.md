# SceneRendering.h - 场景渲染配置结构

## 文件信息
- **路径**: `Engine/Source/Runtime/Renderer/Private/SceneRendering.h`
- **作用**: 定义场景渲染相关的配置结构体
- **MooaToon修改**: 在FFastVramConfig中添加ToonBufferA

## 关键代码分析

### 1. FFastVramConfig结构体 - 添加ToonBufferA（第2827-2829行）

```cpp
struct FFastVramConfig
{
    FFastVramConfig();
    void Update();
    void OnCVarUpdated();
    void OnSceneRenderTargetsAllocated();

    ETextureCreateFlags GBufferA;
    ETextureCreateFlags GBufferB;
    ETextureCreateFlags GBufferC;
    ETextureCreateFlags GBufferD;
    ETextureCreateFlags GBufferE;
    ETextureCreateFlags GBufferF;
    // Mooa GBuffer
    ETextureCreateFlags ToonBufferA;
    // Mooa End
    ETextureCreateFlags GBufferVelocity;
    ETextureCreateFlags HZB;
    ETextureCreateFlags SceneDepth;
    ETextureCreateFlags SceneColor;
    // ... 更多纹理
};
```

#### 零基础解释

这段代码在FFastVramConfig结构体中添加了ToonBufferA的纹理创建标志。

**什么是FFastVramConfig？**
- Fast VRAM配置
- VRAM = 显存（Video RAM）
- 这个结构体配置哪些纹理放在快速显存中
- 可以提高渲染性能

**什么是ETextureCreateFlags？**
- 纹理创建标志
- 比如：是否允许快速显存、是否需要UAV等
- 控制纹理的创建方式

**为什么要加ToonBufferA？**
- ToonBufferA是一个新的GBuffer
- 需要配置它的显存使用
- 所以在FFastVramConfig中加一个成员

**类比理解：**
```
想象一个仓库配置表：
- 记录每个货物放在哪个仓库
- 普通仓库（慢） vs 快速仓库（快）
- GBufferA、GBufferB... 都在表上
- 现在ToonBufferA也加入了表
```

## 技术细节

### FFastVramConfig的作用

```
FFastVramConfig的工作流程：

1. 初始化（构造函数）
   ↓
2. Update() - 更新配置
   ↓
3. OnCVarUpdated() - CVar更新时调用
   ↓
4. OnSceneRenderTargetsAllocated() - 渲染目标分配时调用
```

**为什么需要这个配置？**
- 显存分为不同类型
- 快速显存更快，但可能更小
- 需要决定哪些纹理放在快速显存中
- 优化性能

### ETextureCreateFlags

```cpp
ETextureCreateFlags ToonBufferA;
```

**这是什么？**
- 枚举类型，包含多个标志位
- 可以用位操作组合多个标志
- 比如：`TexCreate_FastVRAM | TexCreate_RenderTargetable`

**常见标志：**
- `TexCreate_FastVRAM`：放在快速显存
- `TexCreate_RenderTargetable`：可作为渲染目标
- `TexCreate_ShaderResource`：可作为着色器资源
- `TexCreate_UAV`：可作为无序访问视图

### ToonBufferA的位置

```cpp
ETextureCreateFlags GBufferF;
// Mooa GBuffer
ETextureCreateFlags ToonBufferA;
// Mooa End
ETextureCreateFlags GBufferVelocity;
```

**位置的选择：**
- 放在GBufferF之后
- GBufferVelocity之前
- 和其他GBuffer在一起
- 保持结构清晰

## MooaToon集成总结

### 修改内容
1. 在FFastVramConfig结构体中
2. 添加ToonBufferA成员变量
3. 类型为ETextureCreateFlags

### 设计意图
- 给ToonBufferA配置显存使用
- 和其他GBuffer保持一致
- 最小修改：只加一行变量声明

## 开发提示

### 如何添加自定义纹理的FastVRAM配置？

参考MooaToon的做法：

```cpp
struct FFastVramConfig
{
    // ... 其他纹理
    
    // 你的纹理
    ETextureCreateFlags YourTexture;
    
    // ... 其他纹理
};
```

### 还需要做什么？

只加成员变量不够，还需要：
1. 在构造函数中初始化
2. 在Update()中设置值
3. 在OnCVarUpdated()中更新
4. 在创建纹理时使用这些标志

（这些应该在SceneRendering.cpp中）

## 总结

SceneRendering.h是场景渲染的头文件，MooaToon在这里：
1. 在FFastVramConfig结构体中添加ToonBufferA

这个修改展示了：
- 如何给新纹理配置显存使用
- 结构体中添加新成员的简单方式
- 和现有代码保持一致的重要性

关键理解：
- 新GBuffer需要在多个地方配置
- FFastVramConfig是其中之一
- 还需要在.cpp文件中初始化和使用
