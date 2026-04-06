# SceneTextures.cpp - GBuffer 创建详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Source/Runtime/Renderer/Private/SceneTextures.cpp` |
| **核心功能** | ToonBufferA 创建、GBuffer 绑定、Uniform 参数设置 |
| **重要性** | ⭐⭐⭐⭐⭐ |

---

## 写给零基础开发者

### 这个文件是做什么的？

**想象一下**：你是一个建筑工人，要盖房子。

这个文件就是那个**建筑工人**，它负责：
1. 买地（分配显存）
2. 盖房子（创建渲染目标）
3. 把钥匙交给住户（绑定到着色器参数）

具体到 MooaToon：
- 创建 ToonBufferA 渲染目标
- 把 ToonBufferA 绑定到 MRT（多渲染目标）
- 把 ToonBufferA 传给着色器

---

## 文件结构总览（Mooa 部分）

| 部分 | 行号 | 内容 |
|------|------|------|
| 1 | 739-745 | 创建 ToonBufferA 渲染目标 |
| 2 | 858-860 | 把 ToonBufferA 加入 GBuffer 绑定列表 |
| 3 | 1056 | 设置 ToonBufferA 默认值（黑色） |
| 4 | 1110-1115 | 把 ToonBufferA 传给着色器 Uniform 参数 |

---

## 第一部分：创建 ToonBufferA 渲染目标

### 739-745 行：核心代码

```cpp
// Mooa GBuffer
if (Bindings.ToonBufferA.Index >= 0)
{
    const FRDGTextureDesc Desc(FRDGTextureDesc::Create2D(
        Config.Extent,                          // 纹理大小（宽度、高度）
        Bindings.ToonBufferA.Format,            // 像素格式
        FClearValueBinding::Transparent,        // 清除值（透明）
        Bindings.ToonBufferA.Flags | FlagsToAdd | GFastVRamConfig.ToonBufferA  // 纹理标志
    ));
    SceneTextures.ToonBufferA = GraphBuilder.CreateTexture(Desc, TEXT("ToonBufferA"));
}
// Mooa End
```

**这是做什么的？**
- 检查是否需要创建 ToonBufferA（Index >= 0）
- 如果需要，就创建一个 2D 纹理
- 把创建好的纹理存到 SceneTextures.ToonBufferA

**逐句解释**：

| 代码 | 说明 |
|------|------|
| `Bindings.ToonBufferA.Index >= 0` | 检查 ToonBufferA 是否在绑定列表中 |
| `FRDGTextureDesc::Create2D(...)` | 创建 2D 纹理描述符 |
| `Config.Extent` | 纹理大小（和屏幕一样大） |
| `Bindings.ToonBufferA.Format` | 像素格式（如 PF_A32B32G32R32F） |
| `FClearValueBinding::Transparent` | 清除值（初始化为透明） |
| `GraphBuilder.CreateTexture(...)` | 用 RDG 图构建器创建纹理 |
| `TEXT("ToonBufferA")` | 纹理的名字（调试用） |

**什么是 FRDGTextureDesc？**
- FRDGTextureDesc = RDG 纹理描述符
- RDG = Render Dependency Graph（渲染依赖图）
- 描述符 = 告诉 GPU 「要创建什么样的纹理」

**什么是 GraphBuilder？**
- GraphBuilder = RDG 图构建器
- 它管理所有渲染 pass 的依赖关系
- 用它创建的纹理会自动管理生命周期

---

## 第二部分：把 ToonBufferA 加入 GBuffer 绑定列表

### 858-860 行

```cpp
// Mooa GBuffer
{ TEXT("ToonBufferA"), ToonBufferA, Bindings.ToonBufferA.Index },
// Mooa End
```

**这是做什么的？**
- 把 ToonBufferA 加入 GBufferEntries 数组
- GBufferEntries 是一个数组，包含所有 GBuffer 的信息

**完整的 GBufferEntries 数组**：
```cpp
const FGBufferEntry GBufferEntries[] =
{
    { TEXT("GBufferA"), GBufferA, Bindings.GBufferA.Index },
    { TEXT("GBufferB"), GBufferB, Bindings.GBufferB.Index },
    { TEXT("GBufferC"), GBufferC, Bindings.GBufferC.Index },
    { TEXT("GBufferD"), GBufferD, Bindings.GBufferD.Index },
    { TEXT("GBufferE"), GBufferE, Bindings.GBufferE.Index },
    // Mooa GBuffer
    { TEXT("ToonBufferA"), ToonBufferA, Bindings.ToonBufferA.Index },
    // Mooa End
    { TEXT("Velocity"), Velocity, Bindings.GBufferVelocity.Index }
};
```

**FGBufferEntry 结构体**（推测）：
```cpp
struct FGBufferEntry
{
    const TCHAR* Name;      // 名字（调试用）
    FRDGTextureRef Texture;  // 纹理引用
    int32 Index;             // MRT 索引（渲染目标索引）
};
```

---

## 第三部分：设置 ToonBufferA 默认值

### 1056 行

```cpp
SceneTextureParameters.ToonBufferATexture = SystemTextures.Black; // Mooa GBuffer
```

**这是做什么的？**
- 给 ToonBufferATexture 设置默认值
- 默认值 = SystemTextures.Black（黑色纹理）

**为什么要设置默认值？**
- 如果 ToonBufferA 没有被创建（比如 Forward 渲染路径）
- 着色器仍然可以访问这个参数，不会报错
- 只是读到的是黑色

**SystemTextures.Black 是什么？**
- SystemTextures = 系统纹理集合
- Black = 一个 1x1 的黑色纹理
- 用来当默认值或占位符

---

## 第四部分：把 ToonBufferA 传给着色器 Uniform 参数

### 1110-1115 行

```cpp
// Mooa GBuffer
if (EnumHasAnyFlags(SetupMode, ESceneTextureSetupMode::ToonBufferA) && HasBeenProduced(SceneTextures->ToonBufferA))
{
    SceneTextureParameters.ToonBufferATexture = SceneTextures->ToonBufferA;
}
// Mooa End
```

**这是做什么的？**
- 检查 SetupMode 是否包含 ToonBufferA
- 检查 ToonBufferA 是否已被创建（HasBeenProduced）
- 如果都满足，就把 ToonBufferA 传给 SceneTextureParameters

**逐句解释**：

| 代码 | 说明 |
|------|------|
| `EnumHasAnyFlags(SetupMode, ESceneTextureSetupMode::ToonBufferA)` | 检查 SetupMode 是否包含 ToonBufferA |
| `HasBeenProduced(SceneTextures->ToonBufferA)` | 检查 ToonBufferA 是否已被创建 |
| `SceneTextureParameters.ToonBufferATexture = SceneTextures->ToonBufferA` | 把纹理引用传给 Uniform 参数 |

**什么是 SceneTextureParameters？**
- SceneTextureParameters = 场景纹理 Uniform 参数结构
- 这是一个「快递包」，CPU 把所有场景纹理打包好发给 GPU
- 着色器可以通过 `View.ToonBufferATexture` 访问

---

## 完整的 ToonBufferA 创建流程

```
┌─────────────────────────────────────────────────────────────┐
│  1. 检查是否需要创建                                         │
│  └─ Bindings.ToonBufferA.Index >= 0 ?                       │
└─────────────────────────────────────────────────────────────┘
                            ↓ 是
┌─────────────────────────────────────────────────────────────┐
│  2. 创建纹理描述符（FRDGTextureDesc）                        │
│  ├─ 大小 = Config.Extent                                     │
│  ├─ 格式 = Bindings.ToonBufferA.Format                       │
│  ├─ 清除值 = Transparent                                     │
│  └─ 标志 = Flags | GFastVRamConfig.ToonBufferA              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  3. 创建纹理（GraphBuilder.CreateTexture）                   │
│  └─ SceneTextures.ToonBufferA = ...                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  4. 加入 GBufferEntries 数组                                 │
│  └─ { TEXT("ToonBufferA"), ToonBufferA, Index }            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  5. 绑定到 MRT（多渲染目标）                                 │
│  └─ RenderTargets[Index] = FTextureRenderTargetBinding(...) │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  6. 设置 Uniform 参数                                         │
│  └─ SceneTextureParameters.ToonBufferATexture = ToonBufferA │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  7. GPU 着色器使用                                           │
│  └─ View.ToonBufferATexture（在 HLSL 中）                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 总结

### 关键点

1. **创建纹理** = 用 GraphBuilder.CreateTexture
2. **FRDGTextureDesc** = 纹理描述符（告诉 GPU 要创建什么样的纹理）
3. **GBufferEntries** = GBuffer 列表，包含 ToonBufferA
4. **SceneTextureParameters** = Uniform 参数快递包，传给着色器
5. **默认值** = SystemTextures.Black（防止未创建时报错）

### 记忆要点

- ✅ SceneTextures.cpp = 建筑工人
- ✅ GraphBuilder = RDG 图构建器
- ✅ FRDGTextureDesc = 建筑图纸
- ✅ GBufferEntries = 房间清单
- ✅ SceneTextureParameters = 快递包
- ✅ 默认值 = SystemTextures.Black

---

**文档版本**: v1.0  
**分析深度**: 源码级（逐行解释）  
**目标读者**: 零基础开发者  
**最后更新**: 2026年4月6日
