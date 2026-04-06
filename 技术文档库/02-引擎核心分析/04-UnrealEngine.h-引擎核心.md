# UnrealEngine.h - UE引擎核心

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Public/UnrealEngine.h`
- **作用**: UE引擎核心头文件
- **MooaToon修改**: 添加SkeletalMeshMooaOutlineDistanceScale

## 关键代码分析

### 1. FEngineShowFlags结构体（第413-415行）

```cpp
float SkeletalMeshOverlayDistanceScale;
// Mooa Outline Material
float SkeletalMeshMooaOutlineDistanceScale;
// Mooa End
```

#### 零基础解释

这是在FEngineShowFlags结构体中添加的Mooa描边距离缩放。

**什么是FEngineShowFlags？**
- 引擎显示标志结构体
- 存储各种渲染相关的设置
- 比如视图距离缩放、LOD距离缩放等

**SkeletalMeshOverlayDistanceScale：**
- 骨骼网格体覆盖层距离缩放
- 控制覆盖层（比如衣服）的LOD距离

**SkeletalMeshMooaOutlineDistanceScale：**
- MooaToon新增的
- 骨骼网格体Mooa描边距离缩放
- 控制Toon描边的LOD距离

**为什么需要这个？**
- Toon描边可能需要特殊的距离缩放
- 比如远处描边可以细一些
- 近处描边可以粗一些

**类比理解：**
```
想象眼镜的度数：
- 普通眼镜：SkeletalMeshOverlayDistanceScale
- Toon专用眼镜：SkeletalMeshMooaOutlineDistanceScale（新加的）
- 控制看Toon描边的清晰度
```

## 技术细节

### FEngineShowFlags的用途

```
FEngineShowFlags存储：
- 视图距离缩放
- LOD距离缩放
- 各种渲染设置
- Mooa描边距离缩放（新加的）
```

## MooaToon集成总结

### 修改内容
1. 在FEngineShowFlags结构体中
2. 添加SkeletalMeshMooaOutlineDistanceScale

### 设计意图
- 给Toon描边提供独立的距离缩放
- 控制描边的LOD距离

## 总结

UnrealEngine.h是UE引擎核心头文件，MooaToon在这里：
1. 在FEngineShowFlags中添加SkeletalMeshMooaOutlineDistanceScale

这个修改展示了：
- 如何扩展FEngineShowFlags
- Toon描边需要特殊的距离控制

关键理解：
- FEngineShowFlags是渲染设置的核心
- Toon描边需要独立的距离缩放
