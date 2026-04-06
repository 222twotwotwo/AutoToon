# Material.h - 材质类

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Public/Materials/Material.h`
- **作用**: 定义UMaterial类，材质编辑器的核心
- **MooaToon修改**: 添加MooaEncodedAttribute0-4材质输入属性

## 关键代码分析

### 1. MooaEncodedAttribute属性定义（第408-424行）

```cpp
// Mooa Material Editor
UPROPERTY()
FVectorMaterialInput MooaEncodedAttribute0;

UPROPERTY()
FVectorMaterialInput MooaEncodedAttribute1;

UPROPERTY()
FVectorMaterialInput MooaEncodedAttribute2;

UPROPERTY()
FVectorMaterialInput MooaEncodedAttribute3;

UPROPERTY()
FVectorMaterialInput MooaEncodedAttribute4;

// Mooa End
```

#### 零基础解释

这是在UMaterial类中添加的5个MooaEncodedAttribute材质输入属性。

**什么是UMaterial类？**
- UE5材质系统的核心类
- 代表一个材质资源
- 存储材质的所有输入和设置

**什么是FVectorMaterialInput？**
- 材质输入类型
- 可以连接材质表达式节点
- 在材质编辑器中显示为输入引脚
- 最终编译成HLSL代码

**为什么需要这5个属性？**
- Toon渲染需要很多参数
- 5个Float4可以存储20个float值
- 足够编码所有Toon渲染参数

**类比理解：**
```
想象一个表单：
- 原来的表单有很多字段：基础颜色、金属度、粗糙度...
- MooaToon新加了5个字段：
  - MooaEncodedAttribute0
  - MooaEncodedAttribute1
  - MooaEncodedAttribute2
  - MooaEncodedAttribute3
  - MooaEncodedAttribute4
- 这5个字段专门用来填Toon渲染的信息！
```

## 技术细节

### UPROPERTY宏

```cpp
UPROPERTY()
FVectorMaterialInput MooaEncodedAttribute0;
```

**UPROPERTY的作用：**
- 让UE5识别这个属性
- 可以在编辑器中显示
- 可以序列化（保存到磁盘）
- 可以在蓝图中访问

### FVectorMaterialInput vs FVector4

```cpp
// FVectorMaterialInput：材质输入
FVectorMaterialInput MooaEncodedAttribute0;
// 特点：可以连接材质表达式节点

// FVector4：普通向量
FVector4 MooaEncodedAttribute0;
// 特点：只是数据，不能连接节点
```

**为什么用FVectorMaterialInput？**
- 材质编辑器需要连接节点
- FVectorMaterialInput支持表达式连接
- 编译时会生成HLSL代码

### 5个属性的用途

```
MooaEncodedAttribute0-4：
├─ 每个是Float4（4个float）
├─ 5个 × 4 = 20个float
└─ 用来编码Toon渲染的所有参数
```

**具体编码什么？**
- 漫反射Ramp索引和UV偏移
- 高光Ramp索引和UV偏移
- 描边强度
- 面部阴影SDF
- 等等...

## MooaToon集成总结

### 修改内容
1. 在UMaterial类中
2. 添加5个FVectorMaterialInput属性
3. 命名为MooaEncodedAttribute0-4

### 设计意图
- 给Toon材质提供专用的输入引脚
- 在材质编辑器中可见可编辑
- 支持连接材质表达式

## 开发提示

### 如何在材质编辑器中使用？

```
材质编辑器中：
1. 找到MooaEncodedAttribute0-4输入引脚
   ↓
2. 连接材质表达式节点
   ↓
3. 编译材质
   ↓
4. 在Toon渲染中使用
```

### 如何添加自定义材质属性？

参考MooaToon的做法：

```cpp
// 在Material.h中添加
UPROPERTY()
FVectorMaterialInput YourAttribute0;

UPROPERTY()
FVectorMaterialInput YourAttribute1;
```

## 总结

Material.h是UMaterial类的头文件，MooaToon在这里：
1. 添加了5个MooaEncodedAttribute材质输入属性

这个修改展示了：
- 如何扩展UMaterial类
- 如何添加自定义材质输入
- FVectorMaterialInput的用途

关键理解：
- 材质编辑器的输入引脚对应UMaterial的属性
- FVectorMaterialInput支持表达式连接
- 5个属性足够编码所有Toon参数
