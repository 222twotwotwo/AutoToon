
# Definitions.usf - 着色器宏定义详细分析

## 写给零基础开发者

大家好！这是 MooaToon 引擎中非常基础的一个文件 —— `Definitions.usf`。

### 这个文件是做什么的？

`Definitions.usf` 是 UE5 着色器的**宏定义文件**，它定义了：
- 各种编译宏（如 `MATERIAL_SHADINGMODEL_TOON`）
- 着色器中常用的常量

### MooaToon 在这个文件里改了什么？

MooaToon 在这个文件里新增了 `MATERIAL_SHADINGMODEL_TOON` 宏的默认值定义！

---

## 文件信息表

| 项目 | 说明 |
|------|------|
| **文件路径** | `Engine/Shaders/Private/Definitions.usf` |
| **主要功能** | 着色器宏定义 |
| **重要性** | ⭐⭐⭐ 基础文件，定义编译宏 |
| **代码行数** | 约 200+ 行 |
| **MooaToon 修改** | 1 处 |

---

## MooaToon 修改

```hlsl
// Mooa Toon Shading Model
#ifndef MATERIAL_SHADINGMODEL_TOON
#define MATERIAL_SHADINGMODEL_TOON						0
#endif
// Mooa End
```

**说明：**
- 如果 `MATERIAL_SHADINGMODEL_TOON` 宏没有被定义（`#ifndef`）
- 就定义它为 0（`#define ... 0`）
- 这样可以避免编译错误

**写给零基础：**

为什么要这样做？

1. **HLSLMaterialTranslator.cpp** 在编译材质时会定义这个宏
   - 如果材质用的是 Toon 着色模型，就定义为 1
   - 否则定义为 0

2. 但如果某个着色器文件没有经过 HLSLMaterialTranslator 编译
   - 这个宏就不存在
   - 直接使用会导致编译错误

3. 所以在 Definitions.usf 中给它一个默认值 0
   - 这样即使宏没被定义，也能正常编译
   - 只是不会执行 Toon 相关的代码

---

## 宏的使用示例

### 在着色器中使用

```hlsl
#if MATERIAL_SHADINGMODEL_TOON
    // 只有当材质是 Toon 着色模型时，这段代码才会编译
    // 执行 Toon 相关的逻辑
#else
    // 否则执行默认逻辑
#endif
```

### 在 HLSLMaterialTranslator.cpp 中定义

```cpp
if (ShadingModel == MSM_Toon)
{
    Compiler.AddMacro(TEXT("MATERIAL_SHADINGMODEL_TOON"), TEXT("1"));
}
else
{
    Compiler.AddMacro(TEXT("MATERIAL_SHADINGMODEL_TOON"), TEXT("0"));
}
```

---

## 总结

### 关键点回顾

1. **默认宏定义** - `MATERIAL_SHADINGMODEL_TOON` 默认值为 0
2. **避免编译错误** - 确保宏始终存在

### 记忆要点

- **核心作用**：给编译宏提供默认值，避免编译错误
- **定义位置**：Definitions.usf
- **默认值**：0

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `HLSLMaterialTranslator.cpp` | 定义 MATERIAL_SHADINGMODEL_TOON 宏 |
| `MaterialTemplate.ush` | 使用 MATERIAL_SHADINGMODEL_TOON 宏 |
| `BasePassPixelShader.usf` | 使用 MATERIAL_SHADINGMODEL_TOON 宏 |

---

祝你学习顺利！🚀

