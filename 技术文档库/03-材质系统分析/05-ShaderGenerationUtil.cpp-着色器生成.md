# ShaderGenerationUtil.cpp - 着色器生成工具

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Private/ShaderCompiler/ShaderGenerationUtil.cpp`
- **作用**: 着色器编译和生成的工具函数
- **MooaToon修改**: 在4处添加Toon支持

## 关键代码分析

### 1. 着色器编译宏定义（第135行）

```cpp
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_CLEAR_COAT);
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_TWOSIDED_FOLIAGE);
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_HAIR);
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_CLOTH);
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_EYE);
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_SINGLELAYERWATER);
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_THIN_TRANSLUCENT);
FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_TOON); // Mooa Toon Shading Model
```

#### 零基础解释

这是获取着色器编译宏定义的代码。

**FETCH_COMPILE_BOOL宏做什么？**
- 从材质中获取编译布尔值
- 设置到着色器编译环境中
- 对应HLSL中的#define

**MATERIAL_SHADINGMODEL_TOON宏：**
- 在HLSL中可以用#if MATERIAL_SHADINGMODEL_TOON
- 判断当前材质是否是Toon着色模型
- 条件编译Toon相关代码

**类比理解：**
```
想象编译开关：
- MATERIAL_SHADINGMODEL_DEFAULTLIT：默认光照开关
- MATERIAL_SHADINGMODEL_CLEAR_COAT：清漆开关
- MATERIAL_SHADINGMODEL_TOON：Toon开关（新加的）

这个代码就是把Toon开关也加上！
```

### 2. GBuffer槽名字符串（第525-528行）

```cpp
case GBS_SeparatedMainDirLight:
    return TEXT("SeparatedMainDirLight");
// Mooa GBuffer
case GBS_MooaToonDataA:
    return TEXT("MooaToonContext.EncodedToonBufferA");
// Mooa End
default:
    break;
```

#### 零基础解释

这是获取GBuffer槽名字符串的switch-case语句。

**这个函数做什么？**
- 把GBuffer槽枚举转换成字符串
- 用于生成HLSL代码
- 比如：GBS_MooaToonDataA → "MooaToonContext.EncodedToonBufferA"

**为什么需要这个？**
- 生成HLSL代码时需要变量名
- 这个函数提供对应的字符串

### 3. GBuffer槽设置（第1757-1762行）

```cpp
// Mooa GBuffer
case MSM_Toon:
    SetSharedGBufferSlots(Slots);
    Slots[GBS_CustomData] = true;
    Slots[GBS_MooaToonDataA] = true;
    break;
// Mooa End
```

#### 零基础解释

这是设置Toon材质使用哪些GBuffer槽的代码。

**SetSharedGBufferSlots做什么？**
- 设置共享的GBuffer槽
- 比如：基础颜色、法线等

**Slots[GBS_CustomData] = true：**
- Toon材质使用CustomData槽
- 存储Toon数据的一部分

**Slots[GBS_MooaToonDataA] = true：**
- Toon材质使用MooaToonDataA槽
- 这是ToonBufferA对应的槽

**Toon材质用哪些GBuffer槽？**
1. 共享槽：基础颜色、法线等
2. CustomData：Toon数据的一部分
3. MooaToonDataA：ToonBufferA

### 4. GBuffer槽使用情况（第1909-1914行）

```cpp
// Mooa GBuffer
if (Mat.MATERIAL_SHADINGMODEL_TOON)
{
    SetStandardGBufferSlots(Slots, bWriteEmissive, bHasTangent, bHasVelocity, bWritesVelocity, bHasStaticLighting, false);
    Slots[GBS_CustomData] = EGBufferSlotUsage::Written;
}
// Mooa End
```

#### 零基础解释

这是设置Toon材质GBuffer槽使用情况的代码。

**SetStandardGBufferSlots做什么？**
- 设置标准的GBuffer槽
- 参数控制是否写入某些槽

**Slots[GBS_CustomData] = EGBufferSlotUsage::Written：**
- CustomData槽是Written状态
- 表示Toon材质会写入CustomData

**EGBufferSlotUsage枚举：**
```cpp
enum class EGBufferSlotUsage : uint8
{
    Unused,   // 未使用
    Used,     // 使用（只读）
    Written   // 写入
};
```

## 技术细节

### 着色器编译流程

```
1. 从材质中获取编译宏
   ↓
2. 设置到编译环境
   ↓
3. 根据着色模型选择GBuffer槽
   ↓
4. 生成HLSL代码
   ↓
5. 编译着色器
```

**MooaToon在这个流程中：**
1. 添加MATERIAL_SHADINGMODEL_TOON宏
2. 添加GBS_MooaToonDataA槽
3. 设置Toon材质的GBuffer槽
4. 设置Toon材质的GBuffer使用情况

## MooaToon集成总结

### 修改内容
1. 着色器编译宏：添加MATERIAL_SHADINGMODEL_TOON
2. GBuffer槽名：添加GBS_MooaToonDataA
3. GBuffer槽设置：添加MSM_Toon case
4. GBuffer槽使用：添加Toon材质处理

### 设计意图
- 让Toon材质参与着色器编译
- Toon材质使用正确的GBuffer槽
- 生成正确的HLSL代码

## 开发提示

### 如何添加自定义着色模型？

参考MooaToon的做法，需要修改：

1. **EngineTypes.h**：添加枚举
2. **Definitions.usf**：添加宏
3. **ShaderGenerationUtil.cpp**：
   - FETCH_COMPILE_BOOL
   - GBuffer槽名
   - GBuffer槽设置
   - GBuffer槽使用情况
4. **其他文件**...

## 总结

ShaderGenerationUtil.cpp是着色器生成工具文件，MooaToon在这里有**4处**修改：
1. 着色器编译宏：添加MATERIAL_SHADINGMODEL_TOON
2. GBuffer槽名：添加GBS_MooaToonDataA → "MooaToonContext.EncodedToonBufferA"
3. GBuffer槽设置：MSM_Toon使用CustomData和MooaToonDataA
4. GBuffer槽使用情况：Toon材质写入CustomData

这个文件展示了：
- 着色器编译的关键步骤
- 如何添加自定义着色模型的编译支持
- GBuffer槽的配置方式

关键理解：
- ShaderGenerationUtil.cpp是着色器编译的核心
- 添加新着色模型需要修改很多地方
- GBuffer槽配置很重要
