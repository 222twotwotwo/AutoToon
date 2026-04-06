# SceneRenderTargetParameters.h - GBuffer 定义详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Source/Runtime/Renderer/Public/SceneRenderTargetParameters.h` |
| **核心功能** | ESceneTextureSetupMode 枚举、ToonBufferA 位标志定义 |
| **重要性** | ⭐⭐⭐⭐ |

---

## 写给零基础开发者

### 这个文件是做什么的？

**想象一下**：你是一个仓库管理员，要管理很多储物箱。

这个文件就是那个**仓库清单**，它定义了：
1. 有哪些储物箱（SceneColor、SceneDepth、GBufferA...）
2. ToonBufferA 是第几个储物箱
3. 哪些储物箱是「GBuffer 组」

---

## 文件结构总览（Mooa 部分）

| 部分 | 行号 | 内容 |
|------|------|------|
| 1 | 41-46 | ESceneTextureSetupMode 枚举 - ToonBufferA 和 GBuffers |

---

## 第一部分：ESceneTextureSetupMode 枚举

### 什么是枚举？

**枚举（Enum）= 一个选项列表**

**用大白话解释**：
- 就像餐厅菜单上的选项
- 每个选项都有一个名字和一个编号
- 你可以选一个或多个选项

---

### 位标志（Bit Flags）

**什么是位标志？**

**用大白话解释**：
- 想象一个 32 位的开关板，每个位是一个开关
- `1 << 0` = 第 1 个开关（二进制 0001）
- `1 << 1` = 第 2 个开关（二进制 0010）
- `1 << 2` = 第 3 个开关（二进制 0100）
- ...
- 用 `|`（或运算）可以同时打开多个开关

**示例**：
```cpp
// 打开 GBufferA 和 GBufferB
ESceneTextureSetupMode Setup = GBufferA | GBufferB;
```

---

### ESceneTextureSetupMode 完整定义

**行号**: 29-48

```cpp
enum class ESceneTextureSetupMode : uint32
{
    None            = 0,
    SceneColor      = 1 << 0,
    SceneDepth      = 1 << 1,
    SceneVelocity   = 1 << 2,
    GBufferA        = 1 << 3,
    GBufferB        = 1 << 4,
    GBufferC        = 1 << 5,
    GBufferD        = 1 << 6,
    GBufferE        = 1 << 7,
    GBufferF        = 1 << 8,
    
    // Mooa GBuffer
    ToonBufferA     = 1 << 9,   // ← 新增！第 10 位
    SSAO            = 1 << 10,
    CustomDepth     = 1 << 11,
    
    GBuffers        = GBufferA | GBufferB | GBufferC | GBufferD | GBufferE | GBufferF | ToonBufferA ,
    // Mooa End
    
    All             = SceneColor | SceneDepth | SceneVelocity | GBuffers | SSAO | CustomDepth
};
ENUM_CLASS_FLAGS(ESceneTextureSetupMode);
```

---

### 逐行解释（Mooa 部分）

#### 42 行：ToonBufferA 定义

```cpp
ToonBufferA = 1 << 9,
```

**这是做什么的？**
- 定义 ToonBufferA 位标志
- `1 << 9` = 第 10 位（从 0 开始数）
- 二进制值 = 00000000000000000000001000000000

**为什么是第 9 位？**
- 前面已经用了 0-8 位（GBufferA 到 GBufferF）
- 下一个可用的就是第 9 位

---

#### 45 行：GBuffers 组合

```cpp
GBuffers = GBufferA | GBufferB | GBufferC | GBufferD | GBufferE | GBufferF | ToonBufferA ,
```

**这是做什么的？**
- 把所有 GBuffer 组合在一起
- 包括：GBufferA、GBufferB、GBufferC、GBufferD、GBufferE、GBufferF、**ToonBufferA**
- 用 `|`（或运算）把它们合并

**用大白话解释**：
- GBuffers = 「GBuffer 组」的所有成员
- 以前这个组只有 6 个成员（A-F）
- 现在加入了第 7 个成员：ToonBufferA

---

### 完整的位标志表

| 标志 | 位 | 二进制（32位） | 说明 |
|------|----|----------------|------|
| SceneColor | 0 | ...0001 | 场景颜色 |
| SceneDepth | 1 | ...0010 | 场景深度 |
| SceneVelocity | 2 | ...0100 | 场景速度 |
| GBufferA | 3 | ...1000 | GBuffer A |
| GBufferB | 4 | ...00010000 | GBuffer B |
| GBufferC | 5 | ...00100000 | GBuffer C |
| GBufferD | 6 | ...01000000 | GBuffer D |
| GBufferE | 7 | ...10000000 | GBuffer E |
| GBufferF | 8 | ...000100000000 | GBuffer F |
| **ToonBufferA** | **9** | **...001000000000** | **Toon Buffer A（新增！）** |
| SSAO | 10 | ...010000000000 | SSAO |
| CustomDepth | 11 | ...100000000000 | 自定义深度 |

---

## 第二部分：如何使用？

### 示例 1：只创建 ToonBufferA

```cpp
ESceneTextureSetupMode SetupMode = ESceneTextureSetupMode::ToonBufferA;
```

### 示例 2：创建所有 GBuffer（包括 ToonBufferA）

```cpp
ESceneTextureSetupMode SetupMode = ESceneTextureSetupMode::GBuffers;
// 这等于：
// GBufferA | GBufferB | GBufferC | GBufferD | GBufferE | GBufferF | ToonBufferA
```

### 示例 3：检查是否包含 ToonBufferA

```cpp
if ((SetupMode & ESceneTextureSetupMode::ToonBufferA) != ESceneTextureSetupMode::None)
{
    // 包含 ToonBufferA
}
```

---

## 总结

### 关键点

1. **ESceneTextureSetupMode** = 场景纹理设置模式枚举
2. **ToonBufferA** = 第 9 位（`1 << 9`）
3. **GBuffers** = 所有 GBuffer 的组合，包括 ToonBufferA
4. **位标志** = 用二进制位表示开关，`|` 合并，`&` 检查

### 记忆要点

- ✅ SceneRenderTargetParameters.h = 仓库清单
- ✅ ToonBufferA = 第 10 个储物箱（位 9）
- ✅ GBuffers = 所有 GBuffer 组，包括 ToonBufferA
- ✅ `|` = 合并多个标志
- ✅ `&` = 检查某个标志是否存在

---

**文档版本**: v1.0  
**分析深度**: 源码级（逐行解释）  
**目标读者**: 零基础开发者  
**最后更新**: 2026年4月6日
