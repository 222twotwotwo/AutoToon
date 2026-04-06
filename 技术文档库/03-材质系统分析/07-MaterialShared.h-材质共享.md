# MaterialShared.h - 材质共享

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Public/MaterialShared.h`
- **作用**: 材质共享的工具函数
- **MooaToon修改**: 在IsSubsurfaceShadingModel中添加MSM_Toon

## 关键代码分析

### 1. IsSubsurfaceShadingModel函数（第138-145行）

```cpp
inline bool IsSubsurfaceShadingModel(FMaterialShadingModelField ShadingModel)
{
	return ShadingModel.HasShadingModel(MSM_Subsurface) || ShadingModel.HasShadingModel(MSM_PreintegratedSkin) ||
		ShadingModel.HasShadingModel(MSM_SubsurfaceProfile) || ShadingModel.HasShadingModel(MSM_TwoSidedFoliage) ||
		ShadingModel.HasShadingModel(MSM_Cloth) || ShadingModel.HasShadingModel(MSM_Eye)
		// Mooa
		|| ShadingModel.HasShadingModel(MSM_Toon);
}
```

#### 零基础解释

这是判断着色模型是否是次表面散射（Subsurface）的函数。

**原来的代码：**
```cpp
return ShadingModel.HasShadingModel(MSM_Subsurface) || 
       ShadingModel.HasShadingModel(MSM_PreintegratedSkin) ||
       ShadingModel.HasShadingModel(MSM_SubsurfaceProfile) || 
       ShadingModel.HasShadingModel(MSM_TwoSidedFoliage) ||
       ShadingModel.HasShadingModel(MSM_Cloth) || 
       ShadingModel.HasShadingModel(MSM_Eye);
```

**MooaToon修改后：**
```cpp
return ... || ShadingModel.HasShadingModel(MSM_Toon);
```

**区别：**
- 原来：判断是否是次表面相关着色模型
- 现在：Toon也被认为是次表面相关

**什么是次表面散射（Subsurface Scattering）？**
- 光线进入物体表面
- 在内部散射
- 从另一个位置出来
- 比如皮肤、蜡、牛奶

**为什么Toon要加入？**
- Toon渲染可能需要类似的处理
- 或者Toon材质也使用次表面相关的代码路径
- 所以把Toon也加入这个判断

**类比理解：**
```
想象一个俱乐部：
- 原来的会员：皮肤、蜡、植物、布料、眼睛
- MooaToon修改后：Toon也加入了俱乐部

这个函数就是检查是不是俱乐部会员！
```

## 技术细节

### 次表面散射着色模型列表

```cpp
MSM_Subsurface           // 次表面
MSM_PreintegratedSkin    // 预积分皮肤
MSM_SubsurfaceProfile    // 次表面轮廓
MSM_TwoSidedFoliage      // 双面植物
MSM_Cloth                // 布料
MSM_Eye                  // 眼睛
MSM_Toon                 // Toon（新加的）
```

**这些着色模型的共同点：**
- 都需要特殊的光照处理
- 都涉及光线在物体内部的传播
- 都不是简单的表面反射

## MooaToon集成总结

### 修改内容
1. 在IsSubsurfaceShadingModel函数中
2. 添加MSM_Toon到判断条件

### 设计意图
- Toon材质可能使用次表面相关的代码路径
- 保持与其他着色模型的一致性

## 总结

MaterialShared.h是材质共享的工具函数头文件，MooaToon在这里：
1. 在IsSubsurfaceShadingModel中添加MSM_Toon

这个修改展示了：
- Toon可能使用次表面相关的代码路径
- 如何扩展工具函数

关键理解：
- Toon不仅是个简单的着色模型
- 可能涉及更复杂的光照处理
