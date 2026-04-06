# BasePassCommon.ush - BasePass公共定义

## 文件信息
- **路径**: `Engine/Shaders/Private/BasePassCommon.ush`
- **作用**: 定义BasePass（基础通道）使用的宏、结构体和公共代码
- **MooaToon修改**: 将Toon着色模型加入WRITES_CUSTOMDATA_TO_GBUFFER宏

## 关键代码分析

### 1. WRITES_CUSTOMDATA_TO_GBUFFER宏（第44-45行）

```hlsl
// Mooa Toon Shading Model
#define WRITES_CUSTOMDATA_TO_GBUFFER		(USES_GBUFFER && (MATERIAL_SHADINGMODEL_SUBSURFACE || MATERIAL_SHADINGMODEL_PREINTEGRATED_SKIN || MATERIAL_SHADINGMODEL_SUBSURFACE_PROFILE || MATERIAL_SHADINGMODEL_CLEAR_COAT || MATERIAL_SHADINGMODEL_TWOSIDED_FOLIAGE || MATERIAL_SHADINGMODEL_HAIR || MATERIAL_SHADINGMODEL_CLOTH || MATERIAL_SHADINGMODEL_EYE || MATERIAL_SHADINGMODEL_TOON))
```

#### 零基础解释

这个宏决定了哪些着色模型需要写入CustomData到GBuffer。

**什么是CustomData？**
- CustomData是GBuffer中的一个通道
- 用于存储着色模型专用的数据
- 不是所有着色模型都需要

**为什么需要这个宏？**
- 节省带宽：不需要的话就不写
- 优化性能：不写就不用读
- 条件编译： shader可以根据这个宏决定是否包含相关代码

**MooaToon做了什么？**
- 在宏的最后加了 `|| MATERIAL_SHADINGMODEL_TOON`
- 告诉引擎：Toon着色模型也需要写CustomData

**类比理解：**
```
想象GBuffer是一个包裹：
- 基础包裹：GBufferA, GBufferB, GBufferC（必需）
- 附加包裹：CustomData（可选）

WRITES_CUSTOMDATA_TO_GBUFFER就是一张清单：
"这些人需要附加包裹：
- 次表面散射
- 皮肤
- 清漆
- ...
- Toon（MooaToon新加的）"
```

### 2. USES_GBUFFER宏（第41行）

```hlsl
#define USES_GBUFFER	(FEATURE_LEVEL >= FEATURE_LEVEL_SM4 && (MATERIALBLENDING_SOLID || MATERIALBLENDING_MASKED) && !FORWARD_SHADING)
```

#### 零基础解释

这个宏决定了是否使用GBuffer（延迟渲染）。

**三个条件：**
1. **FEATURE_LEVEL >= FEATURE_LEVEL_SM4**
   - 显卡功能级别足够
   - SM4 = DirectX 10级别

2. **(MATERIALBLENDING_SOLID || MATERIALBLENDING_MASKED)**
   - 材质是不透明或遮罩的
   - 半透明材质不用GBuffer

3. **!FORWARD_SHADING**
   - 不是前向渲染
   - 前向渲染也不用GBuffer

**总结：**
- 只有不透明/遮罩的、延迟渲染的材质才用GBuffer
- Toon着色模型通常用于不透明角色，所以符合条件

## 技术细节

### WRITES_CUSTOMDATA_TO_GBUFFER的完整列表

让我们看一下所有需要CustomData的着色模型：

| 着色模型 | 用途 | CustomData存什么 |
|---------|------|-----------------|
| **Subsurface** | 次表面散射 | 散射参数 |
| **PreintegratedSkin** | 预积分皮肤 | 皮肤参数 |
| **SubsurfaceProfile** | 次表面剖面 | 剖面ID |
| **ClearCoat** | 清漆 | 清漆粗糙度 |
| **TwoSidedFoliage** | 双面植物 | 植物参数 |
| **Hair** | 头发 | 头发参数 |
| **Cloth** | 布料 | 布料参数 |
| **Eye** | 眼睛 | 眼睛参数 |
| **Toon** | 卡通 | Toon编码数据 |

### Toon如何使用CustomData？

在ToonShadingCommon.ush中：

```hlsl
void EncodeToonGBufferDataToMRT(
    FToonGBufferData ToonGBuffer,
    out float4 EncodedToonBufferA,
    out float4 CustomData,  // ← 这个就是CustomData
    inout float Metallic
)
{
    // 把一部分Toon数据编码到CustomData
    CustomData.x = ...;
    CustomData.y = ...;
    CustomData.z = ...;
    CustomData.w = ...;
}
```

**为什么分开存？**
- ToonBufferA：存主要的Toon数据
- CustomData：存额外的Toon数据
- 因为一个通道存不下所有数据

### 条件编译的好处

```hlsl
#if WRITES_CUSTOMDATA_TO_GBUFFER
    // 这段代码只在需要CustomData时编译
    Out.CustomData = ...;
#endif
```

**好处：**
1. **代码变小**：不需要的代码不编译
2. **性能更好**：不需要的计算不执行
3. **维护简单**：相关代码集中在一起

## MooaToon集成总结

### 修改内容
1. 在WRITES_CUSTOMDATA_TO_GBUFFER宏中
2. 添加MATERIAL_SHADINGMODEL_TOON
3. 让Toon着色模型也能写入CustomData

### 设计意图
- 最小修改：只加一个条件
- 复用系统：使用现有的CustomData机制
- 保持一致：和其他着色模型一样处理

## 开发提示

### 如何给你的着色模型添加CustomData支持？

步骤1：在BasePassCommon.ush中修改宏
```hlsl
#define WRITES_CUSTOMDATA_TO_GBUFFER (... || MATERIAL_SHADINGMODEL_YOURS)
```

步骤2：在你的着色器中使用CustomData
```hlsl
// 编码时
void EncodeYourDataToMRT(..., out float4 CustomData)
{
    CustomData.x = ...;
    CustomData.y = ...;
}

// 解码时
void DecodeYourDataFromMRT(float4 CustomData, ...)
{
    ... = CustomData.x;
    ... = CustomData.y;
}
```

### 如何判断是否在使用GBuffer？

```hlsl
#if USES_GBUFFER
    // 延迟渲染路径
    ...
#else
    // 前向渲染路径
    ...
#endif
```

### CustomData的容量

CustomData是一个float4：
- 4个通道，每个32位浮点数
- 总共128位存储空间
- 可以存很多数据！

## 总结

BasePassCommon.ush是BasePass的配置文件，MooaToon在这里：
1. 把Toon加入了CustomData写入列表

这个修改很小，但很重要：
- 没有它，Toon的CustomData就不会被写入
- Toon渲染就会出错
- 这就是"引擎修改"的典型例子：找到正确的地方，加一点点代码

关键理解：
- 引擎有很多宏控制各种功能
- 修改这些宏可以开启/关闭功能
- 最小修改原则：尽量不改核心逻辑，只改配置
