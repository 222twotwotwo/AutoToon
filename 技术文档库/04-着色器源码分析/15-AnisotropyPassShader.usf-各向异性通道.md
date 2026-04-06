# AnisotropyPassShader.usf - 各向异性通道

## 文件信息
- **路径**: `Engine/Shaders/Private/AnisotropyPassShader.usf`
- **作用**: 实现各向异性渲染通道，编码GBufferF
- **MooaToon修改**: 在各向异性通道中集成Toon数据编码

## 关键代码分析

### 1. Toon数据编码到各向异性通道（第135-146行）

```hlsl
// Mooa Anisotropy
#if MATERIAL_SHADINGMODEL_TOON
FToonGBufferData ToonGBuffer = DecodeToonGBufferDataFromMaterialAttribute(
    GetMaterialMooaEncodedAttribute0(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute1(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute2(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute3(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute4(PixelMaterialInputs));

EncodeToonGBufferDataToAnisotropy(ToonGBuffer, Anisotropy);
#endif
// Mooa End
```

#### 零基础解释

这段代码在各向异性通道中编码Toon数据到Anisotropy参数中。

**什么是各向异性通道？**
- 各向异性（Anisotropy）是一种材质特性
- 比如头发、拉丝金属，高光会沿着某个方向拉伸
- 各向异性通道专门处理这种效果
- 输出GBufferF，存储切线和各向异性值

**为什么Toon要用到各向异性通道？**
- Toon需要存储更多数据
- GBuffer空间不够用了
- 所以"借用"各向异性通道的部分空间
- 把Toon数据编码到Anisotropy参数里

**代码逻辑：**
```
1. 从材质属性解码ToonGBuffer
   ↓
2. 把ToonGBuffer编码到Anisotropy
   ↓
3. 正常编码GBufferF（切线+Anisotropy）
```

**类比理解：**
```
想象你要寄一个包裹：
- 包裹A（ToonBufferA）：已经装满了
- 包裹B（CustomData）：也装满了
- 包裹C（Anisotropy）：还有点空间

所以你把一些小东西塞进包裹C的空隙里
这样就不用额外买包裹D了
```

## 技术细节

### 各向异性通道的工作流程

```
AnisotropyPassShader.usf的工作：

1. 计算材质参数
   ├─ Anisotropy（各向异性值）
   └─ WorldTangent（世界切线）

2. Toon特殊处理（MooaToon）
   ├─ 解码ToonGBuffer
   └─ 编码到Anisotropy

3. 编码GBufferF
   └─ 存储切线和Anisotropy
```

### EncodeToonGBufferDataToAnisotropy函数

这个函数应该在ToonShadingCommon.ush中定义，大概是这样：

```hlsl
void EncodeToonGBufferDataToAnisotropy(
    FToonGBufferData ToonGBuffer,
    inout float Anisotropy
)
{
    // 把Toon数据的一部分编码到Anisotropy中
    // Anisotropy原本是[-1, 1]，可以用来存一些数据
    // 或者用Anisotropy的位来存其他信息
}
```

**为什么用Anisotropy？**
- 卡通渲染通常不需要各向异性效果
- 所以Anisotropy参数对Toon来说是"空闲"的
- 可以拿来存Toon的其他数据

### 条件编译的使用

```hlsl
#if MATERIAL_SHADINGMODEL_TOON
    // 只在Toon着色模型时编译这段代码
#endif
```

**好处：**
- 非Toon材质不会有这段代码
-  shader更小，性能更好
- 逻辑清晰

## MooaToon集成总结

### 修改内容
1. 在各向异性通道中
2. 添加Toon数据解码
3. 添加Toon数据到Anisotropy的编码

### 设计意图
- 充分利用现有GBuffer通道
- 不新增额外的渲染目标
- 最小化对引擎的修改

## 开发提示

### 如何"借用"其他通道的空间？

参考MooaToon的做法：

```hlsl
// 1. 检查是不是你的着色模型
#if MATERIAL_SHADINGMODEL_YOURS

// 2. 解码你的数据
FYourData YourData = DecodeYourDataFromMaterialAttribute(...);

// 3. 编码到"空闲"参数
EncodeYourDataToSomething(YourData, SomeParameter);

#endif
```

### 如何判断哪些参数可以"借用"？

| 参数 | 是否可借用 | 原因 |
|-----|-----------|------|
| **Anisotropy** | ✅ | 卡通渲染通常不需要 |
| **Metallic** | ❌ | 可能会用到 |
| **Roughness** | ❌ | 可能会用到 |
| **CustomData** | ✅ | 如果不需要的话 |

### 记住：不要破坏原有功能！

重要：
- 只在你的着色模型时修改
- 其他着色模型不受影响
- 使用条件编译隔离

## 总结

AnisotropyPassShader.usf是各向异性通道的实现，MooaToon在这里：
1. 解码Toon数据
2. 编码到Anisotropy参数

这个修改展示了：
- 如何充分利用现有资源
- 如何在不新增渲染目标的情况下存更多数据
- 如何使用条件编译隔离代码

关键理解：
- GBuffer空间是宝贵的
- 可以"借用"不需要的参数
- 但要小心，不要破坏其他功能
