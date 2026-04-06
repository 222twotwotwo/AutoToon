# SceneTextureParameters.h - 场景纹理参数（内部）

## 文件信息
- **路径**: `Engine/Source/Runtime/Renderer/Private/SceneTextureParameters.h`
- **作用**: 定义场景纹理参数的内部头文件
- **MooaToon修改**: 在FSceneTextureUniformParameters中添加ToonBufferATexture

## 关键代码分析

### 1. FSceneTextureUniformParameters结构体（第14-21行）

```cpp
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferATexture)
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferBTexture)
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferCTexture)
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferDTexture)
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferETexture)
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferFTexture)
// Mooa GBuffer
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ToonBufferATexture)
// Mooa End
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, GBufferVelocityTexture)
```

#### 零基础解释

这是场景纹理Uniform参数结构体，定义了传递给着色器的纹理。

**什么是SHADER_PARAMETER_RDG_TEXTURE宏？**
- UE5的着色器参数宏
- 定义一个RDG（渲染图）纹理参数
- 自动生成C++和HLSL代码

**RDG是什么？**
- Render Dependency Graph（渲染依赖图）
- UE5的现代渲染架构
- 自动管理渲染资源和依赖

**这个结构体做什么？**
- 定义所有场景纹理
- 包括GBufferA到GBufferF
- 包括ToonBufferA（MooaToon新加的）
- 包括Velocity纹理等

**为什么要加ToonBufferATexture？**
- 着色器需要访问ToonBufferA
- 必须在Uniform参数中定义
- 这样C++才能把纹理传递给HLSL

**类比理解：**
```
想象一个包裹清单：
- 要寄的东西：
  - GBufferA
  - GBufferB
  - GBufferC
  - ...
  - ToonBufferA（新加的）
  - GBufferVelocity

这个结构体就是包裹清单！
```

## 技术细节

### SHADER_PARAMETER_RDG_TEXTURE宏的展开

```cpp
// 你写的：
SHADER_PARAMETER_RDG_TEXTURE(Texture2D, ToonBufferATexture)

// 宏展开后（简化版）：
struct FSceneTextureUniformParameters
{
    FRDGTextureRef ToonBufferATexture;
};

// HLSL端：
Texture2D ToonBufferATexture;
```

**这个宏的好处：**
- 一次定义，两端都有
- C++和HLSL自动同步
- 不用手动写两遍

### Uniform参数的传递流程

```
C++端：
1. 创建FSceneTextureUniformParameters
   ↓
2. 设置ToonBufferATexture
   ↓
3. 传递给着色器

HLSL端：
1. 接收Uniform参数
   ↓
2. 使用ToonBufferATexture
```

## MooaToon集成总结

### 修改内容
1. 在FSceneTextureUniformParameters结构体中
2. 添加ToonBufferATexture参数
3. 使用SHADER_PARAMETER_RDG_TEXTURE宏

### 设计意图
- 让着色器能访问ToonBufferA
- 使用UE5的RDG系统
- 与其他GBuffer保持一致

## 开发提示

### 如何添加自定义RDG纹理参数？

参考MooaToon的做法：

```cpp
// 在SHADER_PARAMETER_STRUCT中添加
SHADER_PARAMETER_STRUCT(FYourParameters, )
{
    // 其他参数...
    
    // 你的纹理
    SHADER_PARAMETER_RDG_TEXTURE(Texture2D, YourTexture)
    
    // 其他参数...
}
END_SHADER_PARAMETER_STRUCT()
```

### RDG vs 传统方式

| 特性 | RDG方式 | 传统方式 |
|-----|---------|---------|
| **资源管理** | 自动 | 手动 |
| **依赖关系** | 自动处理 | 手动跟踪 |
| **代码复杂度** | 低 | 高 |
| **性能** | 更好 | 一般 |

## 总结

SceneTextureParameters.h是场景纹理参数的内部头文件，MooaToon在这里：
1. 在FSceneTextureUniformParameters中添加ToonBufferATexture

这个修改展示了：
- 如何使用UE5的SHADER_PARAMETER宏
- 如何添加RDG纹理参数
- C++和HLSL如何自动同步

关键理解：
- Uniform参数是C++和HLSL通信的桥梁
- SHADER_PARAMETER宏让这变得简单
- ToonBufferA需要在多个地方定义
