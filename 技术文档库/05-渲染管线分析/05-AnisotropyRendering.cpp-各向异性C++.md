# AnisotropyRendering.cpp - 各向异性渲染（C++）

## 文件信息
- **路径**: `Engine/Source/Runtime/Renderer/Private/AnisotropyRendering.cpp`
- **作用**: 各向异性渲染通道的C++端管理
- **MooaToon修改**: 让Toon着色模型也使用各向异性通道

## 关键代码分析

### 1. 编译时判断是否支持各向异性材质（第31-36行）

```cpp
return 
    FDataDrivenShaderPlatformInfo::GetSupportsAnisotropicMaterials(Platform) &&
    !Substrate::IsSubstrateEnabled() && // Substrate renders anisotropy surface natively, without extra pass.
    MaterialParameters.bHasAnisotropyConnected &&
    !IsTranslucentBlendMode(MaterialParameters) &&
    MaterialParameters.ShadingModels.HasAnyShadingModel({ MSM_DefaultLit, MSM_ClearCoat, MSM_Toon }); // Mooa Anisotropy
```

#### 零基础解释

这段代码在编译时判断：是否支持各向异性材质。

**什么是各向异性通道？**
- 各向异性（Anisotropy）是一种材质效果
- 比如：头发、拉丝金属，高光会沿着某个方向拉伸
- 各向异性通道专门渲染这种效果

**原来的判断条件：**
```
支持各向异性吗？
├─ 平台支持？
├─ 不是Substrate？
├─ 连接了各向异性节点？
├─ 不是半透明？
└─ 着色模型是DefaultLit或ClearCoat？
```

**修改后的判断条件：**
```
支持各向异性吗？
├─ 平台支持？
├─ 不是Substrate？
├─ 连接了各向异性节点？
├─ 不是半透明？
└─ 着色模型是DefaultLit、ClearCoat或Toon？ ← MooaToon新加的
```

**为什么Toon需要各向异性通道？**
- Toon需要把一些数据编码到Anisotropy参数里
- 即使Toon不用各向异性效果，也需要这个通道
- 所以把Toon也加入了名单

**类比理解：**
```
想象一个专用通道：
- 原来只给"DefaultLit"和"ClearCoat"用
- 现在"Toon"也需要用这个通道（即使不是用来做各向异性）
- 所以把Toon也加入了允许名单
```

### 2. 运行时判断是否绘制各向异性通道（第138-141行）

```cpp
static bool ShouldDraw(const FMaterial& Material, bool bMaterialUsesAnisotropy)
{
    const bool bIsNotTranslucent = IsOpaqueOrMaskedBlendMode(Material);
    return (bMaterialUsesAnisotropy && bIsNotTranslucent && Material.GetShadingModels().HasAnyShadingModel({ MSM_DefaultLit, MSM_ClearCoat, MSM_Toon })); // Mooa Anisotropy
}
```

#### 零基础解释

这段代码在运行时判断：是否绘制各向异性通道。

**和编译时的区别：**
- 编译时：决定是否编译相关shader
- 运行时：决定是否绘制这个通道

**判断条件和编译时一样：**
- 使用各向异性，且
- 不是半透明，且
- 着色模型是DefaultLit、ClearCoat或Toon ← MooaToon新加的

**为什么两处都要改？**
- 编译时：如果不改，Toon的各向异性shader不会被编译
- 运行时：如果不改，即使编译了也不会被绘制
- 两处都要改，才能正常工作

## 技术细节

### 各向异性通道的完整流程

```
编译阶段：
1. 判断是否支持各向异性材质（第31-36行）
   ├─ 平台支持？
   ├─ 不是Substrate？
   ├─ 连接了各向异性节点？
   ├─ 不是半透明？
   └─ 着色模型是DefaultLit、ClearCoat或Toon？
   └─ 是 → 编译各向异性shader
   └─ 否 → 不编译

运行阶段：
2. 判断是否绘制各向异性通道（第138-141行）
   ├─ 使用各向异性？
   ├─ 不是半透明？
   └─ 着色模型是DefaultLit、ClearCoat或Toon？
   └─ 是 → 绘制各向异性通道
   └─ 否 → 不绘制

3. 如果绘制
   ├─ 调用AnisotropyPassShader.usf
   ├─ Toon编码数据到Anisotropy
   └─ 输出GBufferF
```

### HasAnyShadingModel函数

```cpp
MaterialParameters.ShadingModels.HasAnyShadingModel({ MSM_DefaultLit, MSM_ClearCoat, MSM_Toon })
```

**这个函数做什么？**
- 检查是否有多个着色模型中的任意一个
- 传入一个数组，返回true/false

**和HasShadingModel的区别：**
```cpp
// 检查单个
HasShadingModel(MSM_Toon)

// 检查多个中的任意一个
HasAnyShadingModel({ MSM_A, MSM_B, MSM_C })
```

### Substrate的注释

```cpp
!Substrate::IsSubstrateEnabled() && // Substrate renders anisotropy surface natively, without extra pass.
```

**什么是Substrate？**
- Substrate是UE5的新材质系统
- 它可以原生渲染各向异性，不需要额外通道
- 所以Substrate启用时，这个通道就不用了

## MooaToon集成总结

### 修改内容
1. 在编译时判断中添加Toon着色模型
2. 在运行时判断中添加Toon着色模型

### 设计意图
- 让Toon材质也能使用各向异性通道
- 即使Toon不用各向异性效果，也需要这个通道来存数据
- 最小修改：只加两个条件判断

## 开发提示

### 如何让你的着色模型也使用各向异性通道？

参考MooaToon的做法：

```cpp
// 1. 在编译时判断中添加
MaterialParameters.ShadingModels.HasAnyShadingModel({ MSM_DefaultLit, MSM_ClearCoat, MSM_Yours });

// 2. 在运行时判断中添加
Material.GetShadingModels().HasAnyShadingModel({ MSM_DefaultLit, MSM_ClearCoat, MSM_Yours });
```

### 为什么要改两处？

和光线追踪一样的道理：
- 只改编译时：shader会编译，但不会被绘制
- 只改运行时：想绘制，但shader没编译
- 两处都改：完美工作 ✓

### 各向异性通道的其他用途

MooaToon展示了一个有趣的用法：
- 不一定用各向异性通道来做各向异性效果
- 可以"借用"这个通道来存其他数据
- 只要在shader中正确编码/解码就行

## 总结

AnisotropyRendering.cpp是各向异性渲染的C++管理文件，MooaToon在这里：
1. 在编译时判断中添加Toon
2. 在运行时判断中添加Toon

这个修改展示了：
- 如何让自定义着色模型使用现有的渲染通道
- 编译时和运行时判断的区别
- 如何"借用"通道来做其他用途

关键理解：
- 渲染通道不一定只能做原本的用途
- 可以灵活使用现有资源
- Toon用各向异性通道来存额外数据
