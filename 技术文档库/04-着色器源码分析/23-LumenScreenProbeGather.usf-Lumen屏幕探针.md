# LumenScreenProbeGather.usf - Lumen屏幕探针收集

## 文件信息
- **路径**: `Engine/Shaders/Private/Lumen/LumenScreenProbeGather.usf`
- **作用**: Lumen屏幕探针收集着色器
- **MooaToon修改**: 添加Toon Lumen法线扁平化

## 关键代码分析

### 1. Toon Lumen法线扁平化（第1200-1209行）

```cpp
// Mooa Indirect Lighting
// https://zhuanlan.zhihu.com/p/25839790454
{
	float3 CanmeraVector = GetCameraVectorFromWorldPosition(WorldPosition);
	BRANCH if (Material.ShadingID == SHADINGMODELID_TOON)
	{
		Material.WorldNormal = normalize(lerp(WorldNormal, -CanmeraVector, saturate(View.MooaGlobalIlluminationLumenNormalFlatten)));
	}
}
// Mooa End
```

#### 零基础解释

这是Toon Lumen法线扁平化的处理。

**什么是Lumen？**
- UE5的动态全局光照系统
- Lumen = Lightmass + UE5
- 实时全局光照

**什么是法线扁平化？**
- 把法线变得更平
- 让GI更均匀
- 适合卡通风格

**这里做了什么？**
- 如果是Toon着色模型
- 把WorldNormal在WorldNormal和-CanmeraVector之间插值
- 插值因子是MooaGlobalIlluminationLumenNormalFlatten

**lerp函数：**
```hlsl
lerp(A, B, Alpha) = A * (1 - Alpha) + B * Alpha

如果Alpha = 0：结果 = A
如果Alpha = 1：结果 = B
如果Alpha = 0.5：结果 = (A + B) / 2
```

**CanmeraVector是什么？**
- 从世界位置到相机的向量
- -CanmeraVector就是指向相机的反方向

**类比理解：**
```
想象熨衣服：
- 原来的法线：皱巴巴的衣服
- 熨平：lerp(原来的, 平的, 熨平程度)
- MooaGlobalIlluminationLumenNormalFlatten：熨平程度

这段代码就是在熨法线！
```

## 技术细节

### 为什么要扁平化法线？

```
卡通风格的GI：
- 不需要太真实的光照变化
- 希望更平、更均匀
- 扁平化法线可以达到这个效果

真实感渲染：法线变化丰富 → GI变化丰富
卡通风格：法线扁平化 → GI更平
```

### 代码流程

```
1. 计算CanmeraVector（从世界位置到相机）
   ↓
2. 如果是Toon着色模型：
   ↓
3. Material.WorldNormal = lerp(
       WorldNormal,           // 原来的法线
       -CanmeraVector,        // 指向相机反方向
       MooaGlobalIlluminationLumenNormalFlatten  // 扁平化程度
   )
```

## MooaToon集成总结

### 修改内容
1. 如果是Toon着色模型
2. 把WorldNormal在WorldNormal和-CanmeraVector之间插值
3. 插值因子是MooaGlobalIlluminationLumenNormalFlatten

### 设计意图
- 给Toon Lumen提供法线扁平化控制
- 让GI更平、更均匀
- 适合卡通风格
- 通过CVar调整

## 开发提示

### 如何调整Lumen法线扁平化？

```
在UE5控制台中输入：
r.MooaGlobalIlluminationLumenNormalFlatten 0.5

这样可以调整Toon Lumen的法线扁平化程度！
```

## 总结

LumenScreenProbeGather.usf是Lumen屏幕探针收集着色器，MooaToon在这里：
1. 添加Toon Lumen法线扁平化

这个文件展示了：
- Toon Lumen需要法线扁平化
- 通过lerp在WorldNormal和-CanmeraVector之间插值
- 通过CVar MooaGlobalIlluminationLumenNormalFlatten控制

关键理解：
- Lumen是UE5的实时GI
- 法线扁平化让GI更平
- 适合卡通风格
- 知乎链接：https://zhuanlan.zhihu.com/p/25839790454
