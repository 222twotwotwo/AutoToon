# MaterialExpressionMakeMaterialAttributes.h - 材质属性节点

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Public/Materials/MaterialExpressionMakeMaterialAttributes.h`
- **作用**: 定义MakeMaterialAttributes材质表达式节点
- **MooaToon修改**: 添加MooaEncodedAttribute0-4输入引脚

## 关键代码分析

### 1. MooaEncodedAttribute输入引脚（第77-93行）

```cpp
UPROPERTY()
FExpressionInput Displacement;

// Mooa Material Editor
UPROPERTY()
FExpressionInput MooaEncodedAttribute0;

UPROPERTY()
FExpressionInput MooaEncodedAttribute1;

UPROPERTY()
FExpressionInput MooaEncodedAttribute2;

UPROPERTY()
FExpressionInput MooaEncodedAttribute3;

UPROPERTY()
FExpressionInput MooaEncodedAttribute4;

// Mooa End
```

#### 零基础解释

这是在UMaterialExpressionMakeMaterialAttributes类中添加的5个MooaEncodedAttribute输入引脚。

**什么是UMaterialExpressionMakeMaterialAttributes？**
- "Make Material Attributes"材质表达式节点
- 在材质编辑器中可以看到
- 用来构建材质属性

**FExpressionInput是什么？**
- 材质表达式输入引脚
- 可以连接其他材质表达式节点
- 在材质编辑器中显示为输入引脚

**为什么要加这5个引脚？**
- 材质编辑器中需要连接MooaEncodedAttribute
- 用户可以在节点图中连接这些引脚
- 编译时会生成对应的HLSL代码

**类比理解：**
```
想象一个接线板：
- 原来有很多插孔：基础颜色、金属度、粗糙度...
- MooaToon新加了5个插孔：
  - MooaEncodedAttribute0
  - MooaEncodedAttribute1
  - MooaEncodedAttribute2
  - MooaEncodedAttribute3
  - MooaEncodedAttribute4
- 你可以把线插到这些插孔里！
```

## 技术细节

### 材质编辑器中的样子

```
在材质编辑器中：
Make Material Attributes节点
├─ Base Color
├─ Metallic
├─ Roughness
├─ ...
├─ Mooa Encoded Attribute 0 ← 新加的
├─ Mooa Encoded Attribute 1 ← 新加的
├─ Mooa Encoded Attribute 2 ← 新加的
├─ Mooa Encoded Attribute 3 ← 新加的
└─ Mooa Encoded Attribute 4 ← 新加的
```

**用户可以：**
- 连接节点到这些引脚
- 或者留空（使用默认值）
- 编译材质时会生成代码

## MooaToon集成总结

### 修改内容
1. 在UMaterialExpressionMakeMaterialAttributes类中
2. 添加5个FExpressionInput属性
3. 命名为MooaEncodedAttribute0-4

### 设计意图
- 在材质编辑器中显示MooaEncodedAttribute输入引脚
- 用户可以连接材质表达式
- 支持可视化编辑Toon参数

## 开发提示

### 如何在材质编辑器中使用？

```
材质编辑器中：
1. 找到Make Material Attributes节点
   ↓
2. 展开Mooa Encoded Attribute引脚
   ↓
3. 连接你的材质表达式
   ↓
4. 编译材质
```

## 总结

MaterialExpressionMakeMaterialAttributes.h是MakeMaterialAttributes材质表达式的头文件，MooaToon在这里：
1. 添加了5个MooaEncodedAttribute输入引脚

这个修改展示了：
- 如何扩展材质编辑器节点
- 如何添加自定义输入引脚
- 用户可以在节点图中编辑Toon参数

关键理解：
- 材质编辑器的引脚对应C++的FExpressionInput
- 用户可以可视化编辑Toon参数
- 编译时会生成HLSL代码
