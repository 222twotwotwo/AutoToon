
# MaterialTemplate.ush - 材质模板函数详细分析

## 写给零基础开发者

大家好！这是 MooaToon 引擎中非常重要的一个文件 —— `MaterialTemplate.ush`。

### 这个文件是做什么的？

`MaterialTemplate.ush` 是 UE5 的**材质模板文件**，它定义了大量从材质中获取参数的函数，比如：
- `GetMaterialBaseColor()` - 获取基础色
- `GetMaterialRoughness()` - 获取粗糙度
- `GetMaterialNormal()` - 获取法线

### MooaToon 在这个文件里改了什么？

MooaToon 在这个文件里新增了 5 个函数：
- `GetMaterialMooaEncodedAttribute0()`
- `GetMaterialMooaEncodedAttribute1()`
- `GetMaterialMooaEncodedAttribute2()`
- `GetMaterialMooaEncodedAttribute3()`
- `GetMaterialMooaEncodedAttribute4()`

这些函数用于在着色器中读取我们在材质编辑器里设置的 MooaEncodedAttribute 参数！

---

## 文件信息表

| 项目 | 说明 |
|------|------|
| **文件路径** | `Engine/Shaders/Private/MaterialTemplate.ush` |
| **主要功能** | 定义材质参数获取函数 |
| **重要性** | ⭐⭐⭐⭐ 重要文件，连接材质编辑器和着色器 |
| **代码行数** | 约 4000+ 行（非常大的文件） |
| **MooaToon 修改** | 3 处 |

---

## MooaToon 修改总览

| 位置 | 功能 | 代码行数 |
|------|------|---------|
| **修改 1** | GetMaterialMooaEncodedAttribute0-4 函数 | 3685-3710 |
| **修改 2** | GetMaterialMaskInputRaw 支持 Toon | 3773 |
| **修改 3** | GetMaterialTangent 各向异性支持 | 4048-4069 |

---

## 逐块详细解释

---

### 修改 1：GetMaterialMooaEncodedAttribute0-4 函数

这是最重要的修改！新增了 5 个函数来读取 MooaEncodedAttribute！

```hlsl
// Mooa GBuffer
float4 GetMaterialMooaEncodedAttribute0(FPixelMaterialInputs PixelMaterialInputs)
{
	return PixelMaterialInputs.MooaEncodedAttribute0;
}

float4 GetMaterialMooaEncodedAttribute1(FPixelMaterialInputs PixelMaterialInputs)
{
	return PixelMaterialInputs.MooaEncodedAttribute1;
}

float4 GetMaterialMooaEncodedAttribute2(FPixelMaterialInputs PixelMaterialInputs)
{
	return PixelMaterialInputs.MooaEncodedAttribute2;
}

float4 GetMaterialMooaEncodedAttribute3(FPixelMaterialInputs PixelMaterialInputs)
{
	return PixelMaterialInputs.MooaEncodedAttribute3;
}

float4 GetMaterialMooaEncodedAttribute4(FPixelMaterialInputs PixelMaterialInputs)
{
	return PixelMaterialInputs.MooaEncodedAttribute4;
}
// Mooa End
```

| 函数 | 说明 | 返回值类型 |
|------|------|-----------|
| `GetMaterialMooaEncodedAttribute0()` | 获取 MooaEncodedAttribute0 | float4 |
| `GetMaterialMooaEncodedAttribute1()` | 获取 MooaEncodedAttribute1 | float4 |
| `GetMaterialMooaEncodedAttribute2()` | 获取 MooaEncodedAttribute2 | float4 |
| `GetMaterialMooaEncodedAttribute3()` | 获取 MooaEncodedAttribute3 | float4 |
| `GetMaterialMooaEncodedAttribute4()` | 获取 MooaEncodedAttribute4 | float4 |

**写给零基础：**

这些函数的作用非常简单：
- 输入：`FPixelMaterialInputs`（包含所有材质输入的结构体）
- 输出：对应的 `MooaEncodedAttributeX` 值

它们就像"翻译官"一样，把材质编辑器里的参数传递给着色器代码！

---

### 修改 2：GetMaterialMaskInputRaw 支持 Toon

```hlsl
#if MATERIALBLENDING_MASKED || MATERIAL_SHADINGMODEL_TOON // Mooa Ray Tracing Shadow
// Returns the material mask value generated from the material expressions.
// Use GetMaterialMask() to get the value altered depending on the material blend mode.
half GetMaterialMaskInputRaw(FPixelMaterialInputs PixelMaterialInputs)
{
	return PixelMaterialInputs.OpacityMask;
```

**修改说明：**
- 原来的条件是：`#if MATERIALBLENDING_MASKED`
- MooaToon 修改为：`#if MATERIALBLENDING_MASKED || MATERIAL_SHADINGMODEL_TOON`

| 条件 | 说明 |
|------|------|
| `MATERIALBLENDING_MASKED` | 遮罩混合模式（半透明切边） |
| `MATERIAL_SHADINGMODEL_TOON` | Toon 着色模型 |

**写给零基础：**
- 为什么要加这个？
  - 因为 MooaToon 需要支持光线追踪阴影
  - 即使不是 Masked 混合模式，Toon 着色模型也需要这个函数

---

### 修改 3：GetMaterialTangent 各向异性支持

```hlsl
// Mooa Anisotropy
// MooaToon uses 0 as the default value for Tangent and converts it to World Tangent before output.
// Non-zero values are output as-is.
BRANCH if (all(Tangent == 0))
#if MATERIAL_TANGENTSPACENORMAL &amp;&amp; !MATERIAL_SHADINGMODEL_TOON
		Tangent = float3(1, 0, 0);
#else
		// ... 省略其他代码 ...
#endif
// Mooa End
```

**修改说明：**
- MooaToon 对切线（Tangent）的处理做了特殊优化
- 当切线为 (0,0,0) 时，Toon 着色模型不会默认赋值为 (1,0,0)
- 而是会继续执行后面的代码，根据法线和副切线计算切线

| 条件 | 行为 |
|------|------|
| `MATERIAL_TANGENTSPACENORMAL &amp;&amp; !MATERIAL_SHADINGMODEL_TOON` | 切线为 0 时，默认赋值 (1,0,0) |
| `MATERIAL_SHADINGMODEL_TOON` | 切线为 0 时，不默认赋值，继续计算 |

**写给零基础：**
- 什么是切线（Tangent）？
  - 切线是用于法线贴图、各向异性高光的方向向量
- 为什么 Toon 要特殊处理？
  - 因为 MooaToon 支持各向异性高光（用于头发）
  - 需要更灵活的切线处理方式

---

## 数据流图

```
材质编辑器
    ↓
MP_MooaEncodedAttribute0-4（材质属性）
    ↓
FPixelMaterialInputs.MooaEncodedAttribute0-4（像素着色器输入）
    ↓
GetMaterialMooaEncodedAttribute0-4()（材质模板函数）
    ↓
BasePassPixelShader.usf 调用这些函数
    ↓
解码得到 FToonGBufferData
    ↓
编码到 ToonBufferA
```

---

## 使用示例

### 在 BasePassPixelShader.usf 中使用

```hlsl
// 在 BasePassPixelShader.usf 中调用这些函数
GBuffer.MooaToonContext.ToonGBuffer = DecodeToonGBufferDataFromMaterialAttribute(
    GetMaterialMooaEncodedAttribute0(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute1(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute2(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute3(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute4(PixelMaterialInputs));
```

**写给零基础：**
- 这就是 BasePass 中如何获取 MooaEncodedAttribute 的
- 调用 5 个 Get 函数，把结果传给 `DecodeToonGBufferDataFromMaterialAttribute`
- 解码后得到完整的 `FToonGBufferData`

---

## 总结

### 关键点回顾

1. **5 个 Get 函数** - GetMaterialMooaEncodedAttribute0-4
2. **GetMaterialMaskInputRaw** - 支持 Toon 着色模型的光线追踪阴影
3. **GetMaterialTangent** - 各向异性切线特殊处理

### 记忆要点

- **核心作用**：连接材质编辑器和着色器
- **5 个函数**：GetMaterialMooaEncodedAttribute0-4
- **返回类型**：都是 float4
- **输入参数**：都是 FPixelMaterialInputs

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `ToonShadingCommon.ush` | DecodeToonGBufferDataFromMaterialAttribute 定义 |
| `BasePassPixelShader.usf` | 调用这些函数 |
| `HLSLMaterialTranslator.cpp` | 编译这些函数 |

---

祝你学习顺利！🚀

