# RayTracingCommon.ush - 光线追踪共享

## 文件信息
- **路径**: `Engine/Shaders/Private/RayTracing/RayTracingCommon.ush`
- **作用**: 光线追踪共享函数
- **MooaToon修改**: 在4处添加Toon光线追踪支持

## 关键代码分析

### 1. 包含ToonShadingCommon.ush（第15-17行）

```cpp
// Mooa Ray Tracing
#include "/Engine/Private/ToonShadingCommon.ush"
// Mooa End
```

#### 零基础解释

这是包含ToonShadingCommon.ush头文件。

**为什么要包含？**
- 需要ToonShadingCommon.ush里的定义
- 比如FMooaCustomPayload结构体
- 比如Toon相关的函数

### 2. FPackedMaterialClosestHitPayload添加MooaCustomPayload（第813-815行）

```cpp
// Mooa Ray Tracing Shadow
FMooaCustomPayload MooaCustomPayload;				 // 4 bytes
// Mooa End
```

#### 零基础解释

这是在FPackedMaterialClosestHitPayload中添加MooaCustomPayload。

**什么是FPackedMaterialClosestHitPayload？**
- 光线追踪最近命中载荷
- 存储命中时的材质信息
- 打包后的数据结构

**FMooaCustomPayload是什么？**
- MooaToon自定义载荷
- 4字节大小
- 存储Toon相关的阴影控制数据

**类比理解：**
```
想象一个快递包裹：
- 原来的包裹：FPackedMaterialClosestHitPayload
- 新加的小盒子：MooaCustomPayload（4字节）
- 里面装着Toon阴影控制数据
```

### 3. SetShadingModelID函数（第925-927行）

```cpp
// Mooa Ray Tracing Shadow
void SetShadingModelID(uint ShadingModelID) {IorAndShadingModelIDAndBlendingModeAndPrimitiveLightingChannelMask |= (ShadingModelID &amp; 0xF) &lt;&lt; 16; }
// Mooa End
```

#### 零基础解释

这是SetShadingModelID函数的实现。

**这个函数做了什么？**
- 设置着色模型ID
- 把ShadingModelID打包到IorAndShadingModelIDAndBlendingModeAndPrimitiveLightingChannelMask
- 用位操作：(ShadingModelID &amp; 0xF) &lt;&lt; 16

**为什么需要这个？**
- Toon光线追踪需要设置着色模型ID
- 在MooaTraceVisibilityRay中会用到

### 4. MooaTraceVisibilityRay函数（第1168-1194行）

```cpp
// Mooa Ray Tracing Shadow
#if !SUBSTRATE_ENABLED
FMinimalPayload MooaTraceVisibilityRay(
	in RaytracingAccelerationStructure TLAS,
	in uint RayFlags,
	in uint InstanceInclusionMask,
	in FRayDesc Ray,
	inout uint ShadingModelID,
	inout FMooaCustomPayload MooaCustomPayload)
{
	FPackedMaterialClosestHitPayload PackedPayload = (FPackedMaterialClosestHitPayload)0;
		
	PackedPayload.SetShadingModelID(ShadingModelID);
	PackedPayload.MooaCustomPayload = MooaCustomPayload;

	TraceVisibilityRayPacked(PackedPayload, TLAS, RayFlags, InstanceInclusionMask, Ray);

	FMinimalPayload MinimalPayload = (FMinimalPayload)0;

	MinimalPayload.HitT = PackedPayload.HitT;
	MooaCustomPayload = PackedPayload.MooaCustomPayload;
	ShadingModelID = PackedPayload.GetShadingModelID();

	return MinimalPayload;
}
#endif
// Mooa End
```

#### 零基础解释

这是MooaTraceVisibilityRay函数，专门用于Toon光线追踪。

**这个函数做了什么？**
1. 创建PackedPayload
2. 设置ShadingModelID
3. 设置MooaCustomPayload
4. 调用TraceVisibilityRayPacked
5. 返回MinimalPayload
6. 输出MooaCustomPayload和ShadingModelID

**为什么需要这个函数？**
- 普通的TraceVisibilityRay不传递MooaCustomPayload
- Toon光线追踪需要传递MooaCustomPayload
- 所以专门写了MooaTraceVisibilityRay

**参数说明：**
- `TLAS`：顶层加速结构
- `RayFlags`：光线标志
- `InstanceInclusionMask`：实例包含掩码
- `Ray`：光线描述
- `ShadingModelID`（inout）：输入输出着色模型ID
- `MooaCustomPayload`（inout）：输入输出Mooa自定义载荷

**类比理解：**
```
想象寄快递：
- 普通快递：TraceVisibilityRay
- Toon专用快递：MooaTraceVisibilityRay
- 要寄的东西：ShadingModelID + MooaCustomPayload
- 收件人：TraceVisibilityRayPacked
- 返回：MinimalPayload + 更新后的ShadingModelID和MooaCustomPayload
```

## 技术细节

### FMooaCustomPayload的大小

```cpp
FMooaCustomPayload MooaCustomPayload; // 4 bytes
```

**4字节能存什么？**
- 1个uint32
- 或者4个uint8
- 或者其他4字节的数据结构

**FMooaCustomPayload在ToonShadingCommon.ush中定义。**

## MooaToon集成总结

### 修改内容
1. 包含ToonShadingCommon.ush
2. FPackedMaterialClosestHitPayload添加MooaCustomPayload
3. 添加SetShadingModelID函数
4. 添加MooaTraceVisibilityRay函数

### 设计意图
- 给Toon光线追踪提供自定义载荷
- 支持Toon阴影控制
- 专门的MooaTraceVisibilityRay函数

## 总结

RayTracingCommon.ush是光线追踪共享文件，MooaToon在这里有**4处**修改：
1. 包含ToonShadingCommon.ush
2. FPackedMaterialClosestHitPayload添加MooaCustomPayload（4字节）
3. 添加SetShadingModelID函数
4. 添加MooaTraceVisibilityRay函数

这个文件展示了：
- Toon光线追踪需要自定义载荷
- 专门的MooaTraceVisibilityRay函数
- 与RayTracingMaterialHitShaders.usf配合使用

关键理解：
- FMooaCustomPayload是4字节
- MooaTraceVisibilityRay专门用于Toon
- 与RayTracingMaterialHitShaders.usf配合
