# ReflectionEnvironmentPixelShader.usf - 反射环境像素着色器

## 文件信息
- **路径**: `Engine/Shaders/Private/ReflectionEnvironmentPixelShader.usf`
- **作用**: 反射环境像素着色器
- **MooaToon修改**: 添加Toon反射强度控制

## 关键代码分析

### 1. Toon反射强度控制（第562-568行）

```cpp
// Mooa Reflection
BRANCH if (GBuffer.ShadingModelID == SHADINGMODELID_TOON)
{
	OutColor.xyz *= View.MooaGlobalIlluminationIntensity;
	SpecularColor *= Pow2(GBuffer.MooaToonContext.ToonGBuffer.ReflectionIntensity);
}
// Mooa End
```

#### 零基础解释

这是Toon反射强度的控制代码。

**这里做了什么？**
1. 把OutColor乘以MooaGlobalIlluminationIntensity
2. 把SpecularColor乘以ReflectionIntensity的平方

**为什么这么做？**
- 控制Toon渲染的全局光照强度
- 控制Toon材质的反射强度
- ReflectionIntensity是从FToonGBufferData来的

**代码流程：**
```
1. 检查是否是Toon着色模型
   ↓
2. OutColor.xyz *= MooaGlobalIlluminationIntensity
   ↓
3. SpecularColor *= Pow2(ReflectionIntensity)
```

**Pow2是什么？**
- 平方函数
- Pow2(x) = x * x
- 让反射强度的变化更明显

**类比理解：**
```
想象调音量：
- 主音量：MooaGlobalIlluminationIntensity
- 反射音量：ReflectionIntensity（还要平方）
- 两个都调，效果更明显！
```

## 技术细节

### 反射强度的来源

```
ReflectionIntensity来自哪里？
1. 材质中设置（通过MooaEncodedAttribute）
   ↓
2. BasePass中编码到GBuffer
   ↓
3. GBuffer.MooaToonContext.ToonGBuffer.ReflectionIntensity
   ↓
4. 在这里使用
```

### 为什么用平方？

```
不用平方：
ReflectionIntensity = 0.5 → 0.5倍

用平方：
ReflectionIntensity = 0.5 → 0.25倍
ReflectionIntensity = 0.8 → 0.64倍
ReflectionIntensity = 1.0 → 1.0倍

平方让小的值更小，大的值保持不变！
```

## MooaToon集成总结

### 修改内容
1. 如果是Toon着色模型
2. OutColor乘以MooaGlobalIlluminationIntensity
3. SpecularColor乘以ReflectionIntensity的平方

### 设计意图
- 给Toon反射提供强度控制
- 全局光照强度和材质反射强度结合
- 通过CVar和材质参数双重控制

## 开发提示

### 如何调整反射强度？

**1. 通过CVar调整全局：**
```
在UE5控制台中输入：
r.MooaGlobalIlluminationIntensity 0.5
```

**2. 通过材质参数调整局部：**
```
在材质中：
设置ReflectionIntensity（通过MooaEncodedAttribute）
```

## 总结

ReflectionEnvironmentPixelShader.usf是反射环境像素着色器，MooaToon在这里：
1. 添加Toon反射强度控制

这个文件展示了：
- Toon反射需要强度控制
- 全局CVar和材质参数结合
- 使用平方让变化更明显

关键理解：
- OutColor控制全局光照
- SpecularColor控制反射
- 平方让小值更小
