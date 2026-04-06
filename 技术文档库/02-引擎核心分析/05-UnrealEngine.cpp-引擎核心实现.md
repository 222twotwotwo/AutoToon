# UnrealEngine.cpp - UE引擎核心实现

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Private/UnrealEngine.cpp`
- **作用**: UE引擎核心实现
- **MooaToon修改**: 在3处添加Toon描边支持

## 关键代码分析

### 1. 构造函数初始化（第788-790行）

```cpp
, StaticMeshLODDistanceScale(1.0f)
, SkeletalMeshOverlayDistanceScale(1.0f)
// Mooa Outline Material
, SkeletalMeshMooaOutlineDistanceScale(1.0f)
// Mooa End
, MaxAnisotropy(-1)
```

#### 零基础解释

这是在构造函数中初始化SkeletalMeshMooaOutlineDistanceScale。

**初始化列表：**
- C++构造函数的初始化方式
- 在冒号后面初始化成员变量
- 比在函数体内赋值更高效

**默认值1.0f：**
- 默认距离缩放是1.0
- 表示不缩放
- 用户可以通过CVar修改

### 2. 比较运算符（第806-808行）

```cpp
StaticMeshLODDistanceScale == Other.StaticMeshLODDistanceScale &amp;&amp;
// Mooa Outline Material
SkeletalMeshMooaOutlineDistanceScale == Other.SkeletalMeshMooaOutlineDistanceScale &amp;&amp;
// Mooa End
SkeletalMeshOverlayDistanceScale == Other.SkeletalMeshOverlayDistanceScale;
```

#### 零基础解释

这是在比较运算符中添加SkeletalMeshMooaOutlineDistanceScale的比较。

**为什么要在比较运算符中添加？**
- 判断两个FEngineShowFlags是否相等
- 需要比较所有成员变量
- 包括新增的SkeletalMeshMooaOutlineDistanceScale

**如果不添加会怎样？**
- 两个对象即使SkeletalMeshMooaOutlineDistanceScale不同
- 比较结果也会是相等
- 导致错误的判断

### 3. CVar更新（第882-886行）

```cpp
{
    static const auto SkeletalMeshMooaOutlineDistanceScale = ConsoleMan.FindTConsoleVariableDataFloat(TEXT("r.ViewDistanceScale.SkeletalMeshMooaOutline"));
    LocalScalabilityCVars.SkeletalMeshMooaOutlineDistanceScale = FMath::Max(SkeletalMeshMooaOutlineDistanceScale-&gt;GetValueOnGameThread(), 0.0f);
    LocalScalabilityCVars.SkeletalMeshMooaOutlineDistanceScale *= LocalScalabilityCVars.ViewDistanceScale;
}
```

#### 零基础解释

这是从CVar更新SkeletalMeshMooaOutlineDistanceScale的代码。

**CVar是什么？**
- Console Variable（控制台变量）
- 可以在控制台中修改
- 实时调整渲染参数

**r.ViewDistanceScale.SkeletalMeshMooaOutline：**
- MooaToon新增的CVar
- 控制骨骼网格体Mooa描边的距离缩放
- 可以在控制台中输入修改

**代码流程：**
```
1. 找到CVar
   ↓
2. 获取CVar的值（游戏线程）
   ↓
3. 确保不小于0.0
   ↓
4. 乘以ViewDistanceScale
   ↓
5. 保存到LocalScalabilityCVars
```

**类比理解：**
```
想象调音量：
- 主音量：ViewDistanceScale
- 描边音量：r.ViewDistanceScale.SkeletalMeshMooaOutline
- 最终音量：描边音量 × 主音量

这段代码就是在调描边音量！
```

## 技术细节

### CVar的使用

```cpp
// 1. 定义CVar（在其他地方）
static TAutoConsoleVariable<float> CVarSkeletalMeshMooaOutlineDistanceScale(
    TEXT("r.ViewDistanceScale.SkeletalMeshMooaOutline"),
    1.0f,
    TEXT("Scale factor for skeletal mesh Mooa outline distance.\n")
    TEXT("Default: 1.0"),
    ECVF_RenderThreadSafe);

// 2. 使用CVar
static const auto CVar = ConsoleMan.FindTConsoleVariableDataFloat(
    TEXT("r.ViewDistanceScale.SkeletalMeshMooaOutline"));
float Value = CVar-&gt;GetValueOnGameThread();
```

### 为什么乘以ViewDistanceScale？

```
最终缩放 = 描边缩放 × 视图距离缩放

这样：
- 如果视图距离缩放是0.5
- 描边缩放是1.0
- 最终缩放是0.5
- 描边距离也会减半
```

**好处：**
- 统一受视图距离缩放影响
- 不需要单独调整描边缩放
- 保持一致性

## MooaToon集成总结

### 修改内容
1. 构造函数：初始化SkeletalMeshMooaOutlineDistanceScale为1.0f
2. 比较运算符：添加SkeletalMeshMooaOutlineDistanceScale比较
3. CVar更新：从r.ViewDistanceScale.SkeletalMeshMooaOutline更新

### 设计意图
- 给Toon描边提供独立的距离缩放CVar
- 通过控制台实时调整
- 与视图距离缩放配合

## 开发提示

### 如何在控制台中使用？

```
在UE5控制台中输入：
r.ViewDistanceScale.SkeletalMeshMooaOutline 2.0

这样描边距离会变成2倍！
```

### 如何添加自定义CVar？

参考MooaToon的做法：

```cpp
// 1. 在头文件中添加成员变量
float YourDistanceScale;

// 2. 在构造函数中初始化
, YourDistanceScale(1.0f)

// 3. 在比较运算符中添加
YourDistanceScale == Other.YourDistanceScale &amp;&amp;

// 4. 在CVar更新中添加
static const auto YourCVar = ConsoleMan.FindTConsoleVariableDataFloat(TEXT("r.YourCVar"));
LocalScalabilityCVars.YourDistanceScale = FMath::Max(YourCVar-&gt;GetValueOnGameThread(), 0.0f);
LocalScalabilityCVars.YourDistanceScale *= LocalScalabilityCVars.ViewDistanceScale;
```

## 总结

UnrealEngine.cpp是UE引擎核心实现文件，MooaToon在这里有**3处**修改：
1. 构造函数：初始化SkeletalMeshMooaOutlineDistanceScale为1.0f
2. 比较运算符：添加SkeletalMeshMooaOutlineDistanceScale比较
3. CVar更新：从r.ViewDistanceScale.SkeletalMeshMooaOutline更新

这个文件展示了：
- 如何添加自定义CVar
- 如何在构造函数中初始化
- 如何在比较运算符中添加
- 如何从CVar更新值

关键理解：
- CVar是UE5调整参数的好方式
- Toon描边需要独立的距离控制
- 要在多个地方同步修改
