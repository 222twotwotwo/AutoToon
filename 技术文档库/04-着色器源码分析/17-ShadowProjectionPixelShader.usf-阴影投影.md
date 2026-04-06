# ShadowProjectionPixelShader.usf - 阴影投影像素着色器

## 文件信息
- **路径**: `Engine/Shaders/Private/ShadowProjectionPixelShader.usf`
- **作用**: 阴影投影像素着色器
- **MooaToon修改**: 在2处添加Toon阴影偏移支持

## 关键代码分析

### 1. 方向光/聚光灯阴影偏移（第271-289行）

```cpp
// Mooa Shadow: Shadow Map, Directional/Spot Light
#if SHADING_PATH_DEFERRED &amp;&amp; !FORWARD_SHADING &amp;&amp; !SUBPIXEL_SHADOW &amp;&amp; !SUBSTRATE_ENABLED
{
	FGBufferData GBufferData = GetGBufferData(ScreenUV);
	if (GBufferData.ShadingModelID == SHADINGMODELID_TOON &amp;&amp; View.MooaShadowBias &gt; 0)
	{
		BRANCH if (bIsDirectional)
		{
			Settings.TransitionScale = max(0, SoftTransitionScale.z / max(1, View.MooaShadowBias));
		}
		else
		{
			// Spot Light
			Settings.TransitionScale = max(0, SoftTransitionScale.z - (View.MooaShadowBias * 0.75f));
		}
	}
}
#endif
// Mooa End
```

#### 零基础解释

这是方向光和聚光灯的Toon阴影偏移处理。

**什么是Shadow Bias（阴影偏移）？**
- 阴影偏移是用来解决阴影痤疮（Shadow Acne）的
- 阴影痤疮就是阴影里出现的条纹状伪影
- 把阴影稍微往光照方向推一点

**Toon阴影为什么需要特殊处理？**
- Toon渲染可能有特殊的阴影需求
- 卡通风格的阴影可能需要不同的偏移
- 通过CVar `MooaShadowBias` 控制

**代码流程：**
```
1. 检查是否是延迟着色、不是前向、不是亚像素、不是Substrate
   ↓
2. 获取GBuffer数据
   ↓
3. 检查是否是Toon着色模型且MooaShadowBias &gt; 0
   ↓
4. 如果是方向光：
   Settings.TransitionScale = SoftTransitionScale.z / MooaShadowBias
   ↓
5. 如果是聚光灯：
   Settings.TransitionScale = SoftTransitionScale.z - (MooaShadowBias * 0.75f)
```

**类比理解：**
```
想象戴眼镜：
- 普通眼镜：正常的阴影偏移
- Toon专用眼镜：MooaShadowBias
- 根据眼镜度数调整阴影位置
```

### 2. 点光源/矩形光阴影偏移（第537-546行）

```cpp
// Mooa Shadow: Shadow Map, Point/Rect Light
float DepthBias = PointLightDepthBias.x;
#if !SUBSTRATE_ENABLED
{
	FGBufferData GBufferData = GetGBufferData(ScreenUV);
	if (GBufferData.ShadingModelID == SHADINGMODELID_TOON)
		DepthBias += View.MooaShadowBias * 0.01f;
}
#endif
// Mooa End

float Shadow = CubemapHardwarePCF(TranslateWorldPosition, LightPositionAndInvRadius.xyz, LightPositionAndInvRadius.w, DepthBias /*Mooa Shadow*/, SlopeBias, PointLightDepthBias.z);
```

#### 零基础解释

这是点光源和矩形光的Toon阴影偏移处理。

**这里做了什么？**
- 给DepthBias加上MooaShadowBias * 0.01f
- 只在不是Substrate时才做
- 只对Toon着色模型有效

**为什么乘以0.01？**
- 缩放因子
- 让MooaShadowBias的影响小一些
- 避免偏移过大

**代码流程：**
```
1. 初始化DepthBias = PointLightDepthBias.x
   ↓
2. 如果不是Substrate：
   ↓
3. 获取GBuffer数据
   ↓
4. 如果是Toon着色模型：
   DepthBias += MooaShadowBias * 0.01f
   ↓
5. 把DepthBias传给CubemapHardwarePCF
```

## 技术细节

### 阴影偏移的作用

```
阴影偏移解决的问题：
- 阴影痤疮（Shadow Acne）：阴影里的条纹
- 彼得·潘（Peter Panning）：物体和阴影分离

MooaToon给了额外的控制！
```

### 四种光源类型的处理

| 光源类型 | 处理方式 |
|---------|---------|
| **方向光** | TransitionScale = SoftTransitionScale.z / MooaShadowBias |
| **聚光灯** | TransitionScale = SoftTransitionScale.z - (MooaShadowBias * 0.75f) |
| **点光源** | DepthBias += MooaShadowBias * 0.01f |
| **矩形光** | 同点光源 |

## MooaToon集成总结

### 修改内容
1. 方向光/聚光灯：调整TransitionScale
2. 点光源/矩形光：调整DepthBias

### 设计意图
- 给Toon阴影提供特殊的偏移控制
- 解决Toon渲染的阴影问题
- 通过CVar MooaShadowBias调节

## 开发提示

### 如何在控制台中调整？

```
在UE5控制台中输入：
r.MooaShadowBias 0.5

这样可以调整Toon阴影的偏移！
```

## 总结

ShadowProjectionPixelShader.usf是阴影投影像素着色器，MooaToon在这里有**2处**修改：
1. 方向光/聚光灯：调整TransitionScale
2. 点光源/矩形光：调整DepthBias

这个文件展示了：
- Toon阴影需要特殊的偏移处理
- 支持四种光源类型
- 通过CVar MooaShadowBias控制

关键理解：
- 阴影偏移解决阴影痤疮
- Toon渲染有特殊的阴影需求
- 四种光源都有对应的处理
