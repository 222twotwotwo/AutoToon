# ReflectionEnvironmentShared.ush - 反射环境共享

## 文件信息
- **路径**: `Engine/Shaders/Private/ReflectionEnvironmentShared.ush`
- **作用**: 反射环境共享函数
- **MooaToon修改**: 在GetSkySHDiffuse中添加Toon间接光照

## 关键代码分析

### 1. GetSkySHDiffuse函数（第86-90行）

```cpp
float3 GetSkySHDiffuse(float3 Normal)
{
	// Mooa Indirect Lighting (Transparent Forward Shading)
#if MATERIAL_SHADINGMODEL_TOON
	Normal *= View.MooaGlobalIlluminationDirectionality;
#endif
	// Mooa End
	float4 NormalVector = float4(Normal, 1.0f);
```

#### 零基础解释

这是在GetSkySHDiffuse函数中添加的Toon间接光照处理。

**什么是GetSkySHDiffuse？**
- 从天空SH辐照度图计算天空漫反射光照
- SH = Spherical Harmonics（球谐函数）
- 用来高效存储环境光照

**这里做了什么？**
- 如果是Toon着色模型
- 把Normal乘以MooaGlobalIlluminationDirectionality
- 然后再传给SH计算

**为什么要乘这个？**
- 控制全局光照的方向性
- 可以让Toon渲染的GI更平或更有方向性
- 通过CVar调整

**类比理解：**
```
想象一个手电筒：
- 普通手电筒：正常的光照方向
- Toon手电筒：MooaGlobalIlluminationDirectionality
- 可以调整光束的集中程度

这段代码就是在调手电筒的光束！
```

## 技术细节

### 球谐函数（SH）是什么？

```
球谐函数（Spherical Harmonics）：
- 数学上的基函数
- 可以高效表示球面函数
- 常用于环境光照
- 3阶SH通常用9个系数
```

**MooaToon的修改：**
```
原来：
Normal → SH计算 → 结果

现在（Toon）：
Normal × MooaGlobalIlluminationDirectionality → SH计算 → 结果
```

## MooaToon集成总结

### 修改内容
1. 在GetSkySHDiffuse函数中
2. 如果是MATERIAL_SHADINGMODEL_TOON
3. 把Normal乘以View.MooaGlobalIlluminationDirectionality

### 设计意图
- 给Toon间接光照提供方向性控制
- 透明前向着色路径使用
- 通过CVar调整

## 开发提示

### 如何在控制台中调整？

```
在UE5控制台中输入：
r.MooaGlobalIlluminationDirectionality 0.5

这样可以调整Toon全局光照的方向性！
```

## 总结

ReflectionEnvironmentShared.ush是反射环境共享函数文件，MooaToon在这里：
1. 在GetSkySHDiffuse中添加Toon间接光照方向性控制

这个文件展示了：
- Toon间接光照需要特殊处理
- 透明前向着色路径的支持
- 通过CVar MooaGlobalIlluminationDirectionality控制

关键理解：
- SH用于高效环境光照
- Toon可以调整GI的方向性
- 只在透明前向着色时生效
