# Material.cpp - 材质实现

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Private/Materials/Material.cpp`
- **作用**: UMaterial类的实现
- **MooaToon修改**: 在多处添加Toon支持

## 关键代码分析

### 1. 材质属性编译修改1（第249-250行）

```cpp
if (GetMaterialDomain() != MD_Surface || !IsOpaqueBlendMode(GetBlendMode()) || (GetShadingModels().IsLit() &amp;&amp; !GetShadingModels().HasOnlyShadingModel(MSM_DefaultLit))
    || (GetShadingModels().HasShadingModel(MSM_Toon))) // Mooa Ray Tracing Shadow
{
    Ret = MaterialInterface-&gt;CompileProperty(Compiler, Property);
}
```

#### 零基础解释

这是判断是否编译某个材质属性的代码。

**原来的代码：**
```cpp
if (... || (GetShadingModels().IsLit() &amp;&amp; !GetShadingModels().HasOnlyShadingModel(MSM_DefaultLit)))
```

**MooaToon修改后：**
```cpp
if (... || (GetShadingModels().IsLit() &amp;&amp; !GetShadingModels().HasOnlyShadingModel(MSM_DefaultLit))
    || (GetShadingModels().HasShadingModel(MSM_Toon)))
```

**区别：**
- 原来：不是只有DefaultLit的光照材质才编译
- 现在：Toon材质也要编译

**为什么要加这个？**
- Toon材质需要编译某些属性
- 比如OpacityMask用于光线追踪阴影
- 所以Toon材质也要进入这个分支

### 2. 版本检查（第2942-2943行）

```cpp
// Mooa Material Editor
static_assert(MP_MAX == 40, "New material properties must have DoMaterialAttributeReorder called on them to ensure that any future reordering of property pins is correctly applied.");
```

#### 零基础解释

这是一个静态断言，检查MP_MAX的值。

**static_assert是什么？**
- 编译时断言
- 如果条件不满足，编译报错
- 用于确保代码的正确性

**这里检查什么？**
- 检查MP_MAX是否等于40
- MP_MAX是材质属性的最大枚举值
- MooaToon添加了5个属性，MP_MAX应该是40

**为什么要这个断言？**
- 防止有人在MooaToon之后又添加新属性
- 如果添加了，MP_MAX会变，断言会失败
- 提醒开发者要更新DoMaterialAttributeReorder调用

### 3. 序列化时的属性重排序（第4131-4137行）

```cpp
// Mooa Material Editor
DoMaterialAttributeReorder(&amp;EditorOnly-&gt;MooaEncodedAttribute0, UEVer, RenderObjVer, UE5MainVer);
DoMaterialAttributeReorder(&amp;EditorOnly-&gt;MooaEncodedAttribute1, UEVer, RenderObjVer, UE5MainVer);
DoMaterialAttributeReorder(&amp;EditorOnly-&gt;MooaEncodedAttribute2, UEVer, RenderObjVer, UE5MainVer);
DoMaterialAttributeReorder(&amp;EditorOnly-&gt;MooaEncodedAttribute3, UEVer, RenderObjVer, UE5MainVer);
DoMaterialAttributeReorder(&amp;EditorOnly-&gt;MooaEncodedAttribute4, UEVer, RenderObjVer, UE5MainVer);
// Mooa End
```

#### 零基础解释

这是在序列化时对MooaEncodedAttribute属性进行重排序的代码。

**什么是序列化？**
- 把对象保存到磁盘
- 或者从磁盘加载对象
- 需要处理版本兼容性

**DoMaterialAttributeReorder做什么？**
- 处理材质属性的版本兼容性
- 如果引擎版本变化，重新排序属性
- 保证旧版本材质能正常加载

**为什么5个属性都要调用？**
- MooaEncodedAttribute0-4都是新增的
- 都需要版本兼容性处理
- 所以每个都要调用

### 4. 材质输入描述（第6187-6193行）

```cpp
// Mooa Material Editor
case MP_MooaEncodedAttribute0: SetMaterialInputDescription(EditorOnly-&gt;MooaEncodedAttribute0, false, OutDescription); return true;
case MP_MooaEncodedAttribute1: SetMaterialInputDescription(EditorOnly-&gt;MooaEncodedAttribute1, false, OutDescription); return true;
case MP_MooaEncodedAttribute2: SetMaterialInputDescription(EditorOnly-&gt;MooaEncodedAttribute2, false, OutDescription); return true;
case MP_MooaEncodedAttribute3: SetMaterialInputDescription(EditorOnly-&gt;MooaEncodedAttribute3, false, OutDescription); return true;
case MP_MooaEncodedAttribute4: SetMaterialInputDescription(EditorOnly-&gt;MooaEncodedAttribute4, false, OutDescription); return true;
// Mooa End
```

#### 零基础解释

这是获取材质输入描述的switch-case语句。

**SetMaterialInputDescription做什么？**
- 获取材质输入的描述信息
- 比如：是否连接了节点、连接的什么节点
- 用于材质编辑器UI显示

**为什么要加这5个case？**
- MP_MooaEncodedAttribute0-4是新枚举
- 需要在switch中处理
- 这样才能正确获取它们的描述

### 5. 材质属性编译修改2（第6835-6841行）

```cpp
// Mooa Material Editor
case MP_MooaEncodedAttribute0:	return EditorOnly-&gt;MooaEncodedAttribute0.CompileWithDefault(Compiler, Property);
case MP_MooaEncodedAttribute1:	return EditorOnly-&gt;MooaEncodedAttribute1.CompileWithDefault(Compiler, Property);
case MP_MooaEncodedAttribute2:	return EditorOnly-&gt;MooaEncodedAttribute2.CompileWithDefault(Compiler, Property);
case MP_MooaEncodedAttribute3:	return EditorOnly-&gt;MooaEncodedAttribute3.CompileWithDefault(Compiler, Property);
case MP_MooaEncodedAttribute4:	return EditorOnly-&gt;MooaEncodedAttribute4.CompileWithDefault(Compiler, Property);
//Mooa End
```

#### 零基础解释

这是编译材质属性的switch-case语句。

**CompileWithDefault做什么？**
- 编译材质输入
- 如果没有连接节点，使用默认值
- 生成HLSL代码

**为什么要加这5个case？**
- MP_MooaEncodedAttribute0-4需要编译
- 这样才能生成对应的HLSL代码
- Toon渲染才能使用这些参数

### 6. OpacityMask激活条件（第7484-7485行）

```cpp
case MP_OpacityMask:
    Active = IsMaskedBlendMode(BlendMode)
            || ShadingModels.HasShadingModel(MSM_Toon); // Mooa Ray Tracing Shadow
    break;
```

#### 零基础解释

这是判断OpacityMask属性是否激活的代码。

**原来的代码：**
```cpp
Active = IsMaskedBlendMode(BlendMode);
```

**MooaToon修改后：**
```cpp
Active = IsMaskedBlendMode(BlendMode)
        || ShadingModels.HasShadingModel(MSM_Toon);
```

**区别：**
- 原来：只有Masked混合模式才激活
- 现在：Toon材质也要激活

**为什么Toon材质需要OpacityMask？**
- 用于光线追踪阴影
- Toon材质可能需要自阴影控制
- 所以即使不是Masked混合模式也要激活

### 7. Anisotropy激活条件（第7496-7497行）

```cpp
case MP_Anisotropy:
    // Mooa Material Editor
    Active = ShadingModels.HasAnyShadingModel({ MSM_DefaultLit, MSM_ClearCoat, MSM_Toon }) &amp;&amp; (!bIsTranslucentBlendMode || !bIsVolumetricTranslucencyLightingMode);
    break;
```

#### 零基础解释

这是判断Anisotropy属性是否激活的代码。

**原来的代码：**
```cpp
Active = ShadingModels.HasAnyShadingModel({ MSM_DefaultLit, MSM_ClearCoat }) &amp;&amp; ...;
```

**MooaToon修改后：**
```cpp
Active = ShadingModels.HasAnyShadingModel({ MSM_DefaultLit, MSM_ClearCoat, MSM_Toon }) &amp;&amp; ...;
```

**区别：**
- 原来：只支持DefaultLit和ClearCoat
- 现在：还支持Toon

**为什么Toon需要Anisotropy？**
- Toon渲染可能需要各向异性效果
- 比如头发的各向异性高光
- 所以Toon材质也要激活Anisotropy

## 技术细节

### 为什么有这么多修改？

MooaToon在Material.cpp中有**7处**修改，因为：
1. 添加了新的材质属性
2. 需要在多个地方处理这些属性
3. Toon材质需要特殊处理一些属性

### 修改的分类

| 修改类型 | 位置 | 数量 |
|---------|------|------|
| **新增属性处理** | 序列化、描述、编译 | 4处 |
| **Toon材质特殊处理** | 属性编译、激活条件 | 3处 |

## MooaToon集成总结

### 修改内容
1. 材质属性编译：添加MSM_Toon支持
2. 静态断言：检查MP_MAX
3. 序列化：属性重排序
4. 输入描述：5个case
5. 属性编译：5个case
6. OpacityMask激活：添加MSM_Toon
7. Anisotropy激活：添加MSM_Toon

### 设计意图
- 完整支持MooaEncodedAttribute0-4
- Toon材质特殊处理某些属性
- 保证版本兼容性

## 开发提示

### 如何添加新材质属性？

参考MooaToon的做法，需要修改：

1. **Material.h**：添加UPROPERTY
2. **SceneTypes.h**：添加枚举
3. **MaterialAttributeDefinitionMap.cpp**：注册属性
4. **Material.cpp**：
   - 序列化时重排序
   - 获取描述的switch-case
   - 编译的switch-case
   - 其他需要的地方

## 总结

Material.cpp是UMaterial类的实现文件，MooaToon在这里有**7处**修改：
1. 材质属性编译添加Toon支持
2. 静态断言检查MP_MAX
3. 序列化时属性重排序（5个）
4. 材质输入描述（5个case）
5. 材质属性编译（5个case）
6. OpacityMask激活条件添加Toon
7. Anisotropy激活条件添加Toon

这个文件展示了：
- 添加新材质属性需要修改很多地方
- Toon材质需要特殊处理某些属性
- 版本兼容性很重要

关键理解：
- Material.cpp是材质系统的核心
- 添加新属性需要全面修改
- Toon不是简单的着色模型，需要深度集成
