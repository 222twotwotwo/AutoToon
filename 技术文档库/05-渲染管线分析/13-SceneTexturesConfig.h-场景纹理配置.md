# SceneTexturesConfig.h - 场景纹理配置

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Public/SceneTexturesConfig.h`
- **作用**: 定义场景纹理Uniform参数
- **MooaToon修改**: 添加ToonBufferATexture

## 关键代码分析

### 1. FSceneTextureUniformParameters结构体（第22-29行）

```cpp
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

**这个文件和SceneTextureParameters.h的关系：**
- `SceneTexturesConfig.h`：在Engine模块
- `SceneTextureParameters.h`：在Renderer模块
- 两个文件可能有重复定义

**SHADER_PARAMETER_RDG_TEXTURE宏：**
- 和之前看到的一样
- 定义RDG纹理参数
- 自动生成C++和HLSL代码

**为什么有两个文件？**
- 不同模块需要访问
- Engine模块：材质系统等
- Renderer模块：渲染管线
- 所以两个地方都定义了

**类比理解：**
```
想象两个部门：
- 部门A（Engine）：需要一份清单
- 部门B（Renderer）：也需要一份清单
- 清单内容一样，但两个部门各有一份

这两个文件就是两个部门的清单！
```

## 技术细节

### 为什么Engine和Renderer都需要？

```
Engine模块需要：
- 材质系统
- 着色器编译
- 等等...

Renderer模块需要：
- 渲染管线
- 场景纹理管理
- 等等...

所以两个模块都需要FSceneTextureUniformParameters！
```

### 两个文件的内容

| 文件 | 模块 | 用途 |
|------|------|------|
| `SceneTexturesConfig.h` | Engine | 材质系统、着色器编译 |
| `SceneTextureParameters.h` | Renderer | 渲染管线、场景纹理 |

## MooaToon集成总结

### 修改内容
1. 在FSceneTextureUniformParameters结构体中
2. 添加ToonBufferATexture参数
3. 和其他GBuffer放在一起

### 设计意图
- Engine模块也需要访问ToonBufferA
- 所以两个文件都要加
- 保持一致

## 总结

SceneTexturesConfig.h是Engine模块的场景纹理配置头文件，MooaToon在这里：
1. 在FSceneTextureUniformParameters中添加ToonBufferATexture

这个修改展示了：
- Engine和Renderer模块可能有重复定义
- ToonBufferA需要在多个地方定义
- 保持一致性很重要

关键理解：
- 不同模块可能需要相同的定义
- ToonBufferA不是只在一个地方用
- 要确保所有地方都同步修改
