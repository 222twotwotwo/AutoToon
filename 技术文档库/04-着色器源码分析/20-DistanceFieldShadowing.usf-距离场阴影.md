# DistanceFieldShadowing.usf - 距离场阴影

## 文件信息
- **路径**: `Engine/Shaders/Private/DistanceFieldShadowing.usf`
- **作用**: 距离场阴影着色器
- **MooaToon修改**: 添加Toon距离场阴影偏移

## 关键代码分析

### 1. 距离场阴影偏移（第713-718行）

```cpp
// Mooa Shadow: Shadow Map, Distance Field Directional Shadow
BRANCH if (GetGBufferData(ScreenUV).ShadingModelID == SHADINGMODELID_TOON)
{
	RayStartOffset += View.MooaShadowBias;
}
// Mooa End
```

#### 零基础解释

这是距离场阴影的Toon阴影偏移处理。

**什么是距离场阴影（Distance Field Shadowing）？**
- 使用距离场（Distance Field）生成的阴影
- 比传统阴影贴图更软、更真实
- 适合大面积的软阴影
- 常用于方向光

**这里做了什么？**
- 如果是Toon着色模型
- 把RayStartOffset加上MooaShadowBias
- 这样光线起始点会偏移

**为什么要偏移？**
- 解决阴影痤疮
- Toon渲染可能需要特殊的偏移
- 通过CVar MooaShadowBias控制

**类比理解：**
```
想象跳远：
- 普通跳远：正常的起跳点
- Toon跳远：RayStartOffset += MooaShadowBias
- 往前多跳一点

这段代码就是在调起跳点！
```

## 技术细节

### 距离场阴影工作原理

```
距离场阴影流程：
1. 从像素位置出发
   ↓
2. 沿着光照方向发射光线
   ↓
3. 光线起始点：OpaqueTranslatedWorldPosition + LightDirection * RayStartOffset
   ↓
4. 光线终点：OpaqueTranslatedWorldPosition + LightDirection * TraceDistance
   ↓
5. 追踪距离场
   ↓
6. 得到阴影结果
```

**MooaToon的修改：**
```
原来：
RayStartOffset = 原值

现在（Toon）：
RayStartOffset = 原值 + MooaShadowBias
```

## MooaToon集成总结

### 修改内容
1. 如果是Toon着色模型
2. RayStartOffset += MooaShadowBias

### 设计意图
- 给Toon距离场阴影提供偏移控制
- 解决Toon渲染的距离场阴影问题
- 通过CVar MooaShadowBias调节

## 开发提示

### 如何调整距离场阴影偏移？

```
在UE5控制台中输入：
r.MooaShadowBias 0.5

这样可以调整Toon距离场阴影的偏移！
```

## 总结

DistanceFieldShadowing.usf是距离场阴影着色器，MooaToon在这里：
1. 添加Toon距离场阴影偏移

这个文件展示了：
- Toon距离场阴影需要特殊偏移
- 通过CVar MooaShadowBias控制
- 和ShadowProjectionPixelShader.usf用同一个CVar

关键理解：
- 距离场阴影是另一种阴影技术
- Toon在多种阴影技术中都有修改
- MooaShadowBias是统一的控制参数
