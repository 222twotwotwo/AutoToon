# GBufferHelpers.ush - GBuffer编码解码辅助函数

## 文件信息
- **路径**: `Engine/Shaders/Private/GBufferHelpers.ush`
- **作用**: 提供GBuffer数据的编码、解码、量化等辅助函数
- **MooaToon修改**: 在GBufferPostDecode函数中集成Toon解码逻辑

## 关键代码分析

### 1. GBufferPostDecode函数 - Toon解码集成（第385-426行）

```hlsl
void GBufferPostDecode(inout FGBufferData Ret, bool bChecker, bool bGetNormalizedNormal)
{
    Ret.CustomData = HasCustomGBufferData(Ret.ShadingModelID) ? Ret.CustomData : half(0.0f);

    // Mooa GBuffer Decode
    BRANCH if (Ret.ShadingModelID == SHADINGMODELID_TOON)
    {
        Ret.MooaToonContext.ToonGBuffer = DecodeToonGBufferDataFromMRT(
            Ret.MooaToonContext.EncodedToonBufferA, 
            Ret.CustomData, 
            Ret.Metallic, 
            Ret.Anisotropy
        );
        InitMooaToonContext(Ret);
    }
    // Mooa End

    // ... 其他解码逻辑

    // Mooa GBuffer Decode
    BRANCH if (Ret.ShadingModelID == SHADINGMODELID_TOON)
    {
        Ret.GBufferAO = Ret.GenericAO;
        Ret.DiffuseIndirectSampleOcclusion = 0x0;
        Ret.IndirectIrradiance = 1;
    }
    // Mooa End
}
```

#### 零基础解释

这个函数在GBuffer解码后调用，做一些后处理。MooaToon在这里加了两次Toon相关的逻辑。

**第一次Toon解码（第393-398行）：**

想象你收到一个包裹（EncodedToonBufferA），需要打开它：

```
包裹（EncodedToonBufferA）→ 拆包函数（DecodeToonGBufferDataFromMRT）→ 内容（ToonGBuffer）
```

**步骤分解：**
1. 检查是不是Toon着色模型
2. 如果是，调用拆包函数
3. 把拆出来的东西放进ToonGBuffer
4. 初始化Toon上下文

**为什么用BRANCH？**
- BRANCH告诉编译器：这个if分支可能不会执行
- 只有Toon材质才会走这个分支
- 其他材质可以跳过，节省性能

**第二次Toon处理（第419-425行）：**

这是对AO（环境光遮蔽）和间接光照的特殊处理：

```hlsl
Ret.GBufferAO = Ret.GenericAO;              // 直接用GenericAO
Ret.DiffuseIndirectSampleOcclusion = 0x0;   // 清除采样遮蔽
Ret.IndirectIrradiance = 1;                  // 间接光照设为1
```

**为什么这样做？**
- 卡通渲染不适用UE5默认的间接光照系统
- 我们用自己的方式处理光照
- 所以把这些参数重置为简单值

### 2. 量化函数族（第18-258行）

这个文件包含大量量化函数，比如：

```hlsl
uint EncodeQuantize6(float Value, float QuantizationBias)
{
    return min(uint(saturate(Value) * 63.0f + .5f + QuantizationBias), 63u);
}

float DecodeQuantize6(uint Value)
{
    return float(Value) / 63.0f;
}
```

#### 零基础解释

**什么是量化？**
- 把连续的值变成离散的值
- 就像把尺子上的刻度变粗
- 目的是节省空间

**举例：EncodeQuantize6**
- 输入：0.0到1.0之间的浮点数
- 输出：0到63之间的整数（6位）
- 精度损失：1/63 ≈ 1.6%

**为什么需要量化？**
- GBuffer的存储空间有限
- 每个通道只有8位（0-255）
- 需要把多个数据塞进一个通道

**类比理解：**
```
想象你有一个行李箱（8位通道），可以装256个东西
你要装：
- A数据：需要6位（0-63）
- B数据：需要2位（0-3）

打包：
A × 4 + B = 最终值（0-255）

拆包：
A = 最终值 / 4（取整）
B = 最终值 % 4（取余）
```

## 技术细节

### GBufferPostDecode的调用时机

```
渲染流程：
1. BasePass → 编码GBuffer（包括ToonBufferA）
2. 延迟着色 → 读取GBuffer
3. DecodeGBufferData() → 解码基础数据
4. GBufferPostDecode() → 后处理解码 ← MooaToon在这里
5. 光照计算 → 使用解码后的数据
```

### 为什么有两处Toon处理？

**第一处（第393-398行）：**
- 位置：在GBuffer基础解码之后
- 作用：解码Toon专用数据
- 必须：没有这个，Toon数据就用不了

**第二处（第419-425行）：**
- 位置：在AO和间接光照处理之后
- 作用：覆盖默认的AO/间接光照
- 原因：卡通渲染有自己的光照逻辑

### BRANCH vs FLATTEN

```hlsl
BRANCH if (条件) { ... }   // 分支执行，不满足就跳过
FLATTEN if (条件) { ... }  // 两边都执行，用条件选择结果
```

**为什么用BRANCH？**
- Toon材质可能只占场景的一小部分
- 大多数材质不需要执行Toon解码
- BRANCH可以跳过不必要的计算

## MooaToon集成总结

### 修改内容
1. 在GBufferPostDecode函数中
2. 添加Toon数据解码逻辑
3. 添加Toon AO/间接光照重置逻辑

### 设计意图
- 在统一的GBuffer后处理中处理Toon
- 保持代码结构清晰
- 性能优化：只在需要时执行

## 开发提示

### 如何添加自定义着色模型的解码？

参考MooaToon的做法：

```hlsl
void GBufferPostDecode(inout FGBufferData Ret, ...)
{
    // 你的着色模型解码
    BRANCH if (Ret.ShadingModelID == SHADINGMODELID_YOURS)
    {
        Ret.YourContext.Data = DecodeYourDataFromMRT(...);
        InitYourContext(Ret);
    }

    // ... 其他逻辑

    // 你的着色模型特殊处理
    BRANCH if (Ret.ShadingModelID == SHADINGMODELID_YOURS)
    {
        // 覆盖某些参数
    }
}
```

### 量化技巧

如果你需要把多个数据塞进一个通道：

```hlsl
// 打包：两个4位数据 → 一个8位数据
uint PackTwo4Bit(uint A, uint B)
{
    return (A & 0xF) | ((B & 0xF) << 4);
}

// 拆包
void UnpackTwo4Bit(uint Packed, out uint A, out uint B)
{
    A = Packed & 0xF;
    B = (Packed >> 4) & 0xF;
}
```

## 总结

GBufferHelpers.ush是GBuffer处理的工具库，MooaToon在这里：
1. 集成了Toon数据的解码
2. 重置了Toon的AO和间接光照

这个文件展示了如何：
- 在现有的渲染流程中插入自定义逻辑
- 使用分支优化性能
- 处理数据的编码和解码

关键理解：
- GBuffer解码不是一步完成的
- 有基础解码，还有后处理解码
- 自定义着色模型可以在后处理阶段插入
