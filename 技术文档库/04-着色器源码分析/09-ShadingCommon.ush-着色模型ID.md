
# ShadingCommon.ush - 着色公共函数详细分析

## 写给零基础开发者

大家好！这是 MooaToon 引擎中一个重要的文件 —— `ShadingCommon.ush`。

### 这个文件是做什么的？

`ShadingCommon.ush` 是 UE5 **着色公共函数**库，它定义了：
- `SHADINGMODELID_*` 常量（着色模型 ID）
- `GetShadingModelColor()` 函数（调试用，给不同着色模型显示不同颜色）

### MooaToon 在这个文件里改了什么？

MooaToon 在这个文件里做了以下修改：
1. 新增 `SHADINGMODELID_TOON` 常量（值为 13）
2. 在 `GetShadingModelColor()` 函数中新增 Toon 着色模型的颜色

---

## 文件信息表

| 项目 | 说明 |
|------|------|
| **文件路径** | `Engine/Shaders/Private/ShadingCommon.ush` |
| **主要功能** | 着色模型 ID 定义、调试颜色 |
| **重要性** | ⭐⭐⭐ 重要文件，定义着色模型 ID |
| **代码行数** | 约 100+ 行 |
| **MooaToon 修改** | 2 处 |

---

## MooaToon 修改总览

| 位置 | 功能 | 代码行数 |
|------|------|---------|
| **修改 1** | 定义 SHADINGMODELID_TOON 常量 | 33 |
| **修改 2** | GetShadingModelColor() 中新增 Toon 颜色 | 66, 84 |

---

## 逐块详细解释

---

### 修改 1：定义 SHADINGMODELID_TOON 常量

```hlsl
#define SHADINGMODELID_TOON					13		// Mooa Toon Shading Model
#define SHADINGMODELID_NUM					14
```

| 常量 | 值 | 说明 |
|------|-----|------|
| `SHADINGMODELID_TOON` | 13 | Toon 着色模型 ID |
| `SHADINGMODELID_NUM` | 14 | 着色模型总数（从 0 到 13，共 14 个） |

**写给零基础：**
- 这个 ID 必须和 C++ 代码中的 `EMaterialShadingModel::MSM_Toon` 一致
- C++ 中 `MSM_Toon` 的值也是 13
- 这样 C++ 和 HLSL 才能对应上

---

### 修改 2：GetShadingModelColor() 中新增 Toon 颜色

这个函数用于**调试**，在编辑器中显示不同着色模型的颜色。

#### 第一处（if-else 版本）

```hlsl
else if (ShadingModelID == SHADINGMODELID_TOON) return float3(0.75f, 0.1f, 0.1f);// Mooa Toon Shading Model
```

#### 第二处（switch 版本）

```hlsl
case SHADINGMODELID_TOON: return float3(0.75f, 0.2f, 0.0f);// Mooa Toon Shading Model
```

| 颜色 | RGB 值 | 视觉效果 |
|------|--------|---------|
| if-else 版本 | (0.75, 0.1, 0.1) | 深红色 |
| switch 版本 | (0.75, 0.2, 0.0) | 深橙色 |

**写给零基础：**
- 为什么有两个版本？
  - 一个是 `#if` 分支里的 if-else 版本
  - 一个是 `#else` 分支里的 switch 版本
  - 取决于编译器设置，会用其中一个
- 这个颜色只在编辑器调试时显示，不影响最终渲染

---

## 着色模型 ID 对照

| ID | 名称 | 说明 |
|----|------|------|
| 0 | DEFAULT_LIT | 默认光照 |
| 1 | SUBSURFACE | 次表面散射 |
| 2 | PREINTEGRATED_SKIN | 预积分皮肤 |
| 3 | CLEAR_COAT | 清漆 |
| 4 | SUBSURFACE_PROFILE | 次表面轮廓 |
| 5 | TWO_SIDED_FOLIAGE | 双面植物 |
| 6 | HAIR | 头发 |
| 7 | CLOTH | 布料 |
| 8 | EYE | 眼睛 |
| 9 | SINGLELAYERWATER | 单层水 |
| 10 | THIN_TRANSLUCENT | 薄半透明 |
| 11 | FOLIAGE | 植物 |
| 12 | SHADINGMODELID_NUM-1 | （预留） |
| **13** | **TOON** | **Toon 着色模型（新增）** |

---

## 总结

### 关键点回顾

1. **SHADINGMODELID_TOON = 13** - 着色模型 ID
2. **GetShadingModelColor()** - 调试颜色（深红色/深橙色）
3. **SHADINGMODELID_NUM = 14** - 总数更新

### 记忆要点

- **核心作用**：定义 Toon 着色模型的 ID 和调试颜色
- **ID 值**：13
- **调试颜色**：(0.75, 0.1, 0.1) 或 (0.75, 0.2, 0.0)

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `EngineTypes.h` | EMaterialShadingModel::MSM_Toon 定义（C++） |
| `ShadingModels.ush` | IntegrateBxDF() 中使用 SHADINGMODELID_TOON |

---

祝你学习顺利！🚀

