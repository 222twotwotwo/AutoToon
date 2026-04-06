# RayTracingMaterialHitShaders.usf - 光线追踪材质命中着色器

## 文件信息
- **路径**: `Engine/Shaders/Private/RayTracing/RayTracingMaterialHitShaders.usf`
- **作用**: 实现光线追踪的材质命中着色器
- **MooaToon修改**: 添加Toon的光线追踪阴影支持，包括自阴影和投射阴影控制

## 关键代码分析

### 1. 调试宏定义（第5-10行）

```hlsl
// Mooa Debug
// #define MATERIAL_SHADINGMODEL_TOON 1
// #define USE_MATERIAL_ANY_HIT_SHADER 1
// #define USE_MATERIAL_CLOSEST_HIT_SHADER 1
// #define SUBSTRATE_ENABLED 0
// Mooa End
```

#### 零基础解释

这些是调试用的宏，被注释掉了。

**什么是调试宏？**
- 宏就像开关
- 定义后可以强制某些行为
- 用于调试和测试

**这些宏的作用：**
- `MATERIAL_SHADINGMODEL_TOON 1`：强制启用Toon着色模型
- `USE_MATERIAL_ANY_HIT_SHADER 1`：强制使用任意命中着色器
- `USE_MATERIAL_CLOSEST_HIT_SHADER 1`：强制使用最近命中着色器
- `SUBSTRATE_ENABLED 0`：禁用Substrate

**为什么注释掉？**
- 这些是调试用的
- 正常情况下不需要
- 需要时取消注释即可

### 2. 检测Toon着色模型（第673-680行）

```hlsl
// Mooa Ray Tracing Shadow
bool IsHitToonShadingModel = false;
#if !SUBSTRATE_ENABLED 
    #if MATERIAL_SHADINGMODEL_TOON
        IsHitToonShadingModel = true;
    #endif
#endif
// Mooa End
```

#### 零基础解释

检测当前命中的是不是Toon着色模型。

**什么是任意命中着色器（Any Hit Shader）？**
- 光线追踪时，光线可能穿过多个物体
- 任意命中着色器在每次命中时调用
- 可以决定是否继续追踪，或者忽略这个命中

**IsHitToonShadingModel的作用：**
- 标记当前命中的是不是Toon材质
- 后面会用这个标记做特殊处理

**条件编译：**
- `!SUBSTRATE_ENABLED`：不用Substrate时才启用
- `MATERIAL_SHADINGMODEL_TOON`：是Toon着色模型时才启用

### 3. 修改混合模式判断（第683、691、752行）

```hlsl
#if MATERIALBLENDING_SOLID && !MATERIAL_SHADINGMODEL_TOON // Mooa Ray Tracing Shadow
    // ...
#endif

#if MATERIALBLENDING_MASKED || MATERIAL_SHADINGMODEL_TOON // Mooa Ray Tracing Shadow
    // ...
#endif

#if MATERIALBLENDING_MASKED || MATERIAL_SHADINGMODEL_TOON // Mooa Ray Tracing Shadow
    // ...
#endif
```

#### 零基础解释

修改了混合模式的判断条件，把Toon单独处理。

**原来的逻辑：**
- 不透明（Solid）材质：继续追踪
- 遮罩（Masked）材质：执行遮罩测试

**修改后的逻辑：**
- 不透明 + 非Toon：继续追踪
- 遮罩 或 Toon：执行遮罩测试

**为什么这样改？**
- Toon材质可能需要特殊的阴影处理
- 即使是不透明的Toon材质，也要执行遮罩测试逻辑
- 因为后面有Toon专用的阴影控制

### 4. Toon光线追踪阴影处理（第760-786行）

```hlsl
// Mooa Ray Tracing Shadow
#if !SUBSTRATE_ENABLED && MATERIAL_SHADINGMODEL_TOON
    FMooaCustomPayload HitMooaCustomPayload = (FMooaCustomPayload)0;
    HitMooaCustomPayload.MooaCustomData = uint(GetMaterialCustomData0(MaterialParameters));

    // if RayOrigin and HitPoint both Toon Material
    if (PackedPayload.GetShadingModelID() == SHADINGMODELID_TOON && IsHitToonShadingModel)
    {
        // Disable Self Shadow
        if (HitMooaCustomPayload.IsDisableSelfShadow() &&
            PackedPayload.MooaCustomPayload.GetStencil() == HitMooaCustomPayload.GetStencil())
            IgnoreHit();
        
        // Disable Cast Shadow on Toon Material
        if (HitMooaCustomPayload.IsDisableCastShadowOnToon() &&
            PackedPayload.MooaCustomPayload.GetStencil() != HitMooaCustomPayload.GetStencil())
            IgnoreHit();
    }
#endif // !SUBSTRATE_ENABLED && MATERIAL_SHADINGMODEL_TOON
// Mooa End
```

#### 零基础解释

这是Toon光线追踪阴影的核心逻辑，控制是否忽略某些命中。

**什么是IgnoreHit()？**
- IgnoreHit()告诉光线追踪："忽略这次命中"
- 光线会继续前进，好像没碰到这个物体
- 用来实现特殊的阴影效果

**两个功能：**

**功能1：禁用自阴影（Disable Self Shadow）**
```
条件：
- 禁用自阴影标志开启
- 光线起点和命中点的Stencil相同

动作：
- IgnoreHit() → 忽略这次命中
- 结果：物体不会给自己投阴影
```

**为什么需要这个？**
- 卡通渲染中，角色可能不需要自阴影
- 比如：头发不应该在脸上投阴影
- 用Stencil来标识同一个角色

**功能2：禁用Toon之间的投射阴影（Disable Cast Shadow on Toon）**
```
条件：
- 禁用Toon之间投射阴影标志开启
- 光线起点和命中点的Stencil不同

动作：
- IgnoreHit() → 忽略这次命中
- 结果：Toon物体不会给其他Toon物体投阴影
```

**为什么需要这个？**
- 卡通渲染中，角色之间可能不需要互相投阴影
- 比如：角色A不应该在角色B身上投阴影
- 保持画面干净

**类比理解：**
```
想象拍照时的阴影控制：
- 禁用自阴影：角色的左手不给右手投阴影
- 禁用Toon之间投射阴影：角色A不给角色B投阴影

就像有选择地"擦除"某些阴影
让画面更符合卡通风格
```

## 技术细节

### 光线追踪阴影的工作流程

```
光线追踪阴影流程：

1. 发射阴影光线
   ↓
2. 碰到物体 → 调用任意命中着色器
   ↓
3. Toon特殊处理（MooaToon）
   ├─ 检查是不是Toon到Toon
   ├─ 检查是否禁用自阴影
   ├─ 检查是否禁用Toon之间投射
   └─ 决定是否IgnoreHit()
   ↓
4. 如果没有IgnoreHit() → 这个物体挡住了光线，有阴影
   如果IgnoreHit() → 光线继续，没有阴影
```

### FMooaCustomPayload结构体

这个结构体应该在某个头文件中定义，大概是这样：

```hlsl
struct FMooaCustomPayload
{
    uint MooaCustomData;
    
    bool IsDisableSelfShadow()
    {
        return (MooaCustomData & FLAG_DISABLE_SELF_SHADOW) != 0;
    }
    
    bool IsDisableCastShadowOnToon()
    {
        return (MooaCustomData & FLAG_DISABLE_CAST_SHADOW_ON_TOON) != 0;
    }
    
    uint GetStencil()
    {
        return (MooaCustomData >> STENCIL_SHIFT) & STENCIL_MASK;
    }
};
```

**数据编码：**
- 把多个标志塞进一个uint里
- 用位操作来读取

### 为什么用Stencil？

**Stencil的作用：**
- 标识不同的物体/角色
- 同一个角色的Stencil值相同
- 不同角色的Stencil值不同

**使用场景：**
- 自阴影：检查Stencil是否相同
- Toon之间阴影：检查Stencil是否不同

## MooaToon集成总结

### 修改内容
1. 添加调试宏（注释掉）
2. 检测Toon着色模型
3. 修改混合模式判断
4. 实现Toon光线追踪阴影控制
   - 禁用自阴影
   - 禁用Toon之间投射阴影

### 设计意图
- 给卡通渲染提供灵活的阴影控制
- 支持艺术家调整阴影效果
- 保持与现有光线追踪系统的集成

## 开发提示

### 如何实现自定义的光线追踪阴影控制？

参考MooaToon的做法：

```hlsl
// 1. 在任意命中着色器中
if (IsYourShadingModel())
{
    // 2. 读取自定义数据
    FYourCustomPayload Payload = ...;
    
    // 3. 判断条件
    if (ShouldIgnoreHit(Payload))
    {
        // 4. 忽略命中
        IgnoreHit();
    }
}
```

### 如何编码多个标志到一个uint？

```hlsl
// 定义标志位
#define FLAG_A (1 << 0)
#define FLAG_B (1 << 1)
#define FLAG_C (1 << 2)

// 编码
uint PackedData = 0;
PackedData |= (EnableA ? FLAG_A : 0);
PackedData |= (EnableB ? FLAG_B : 0);
PackedData |= (EnableC ? FLAG_C : 0);

// 解码
bool EnableA = (PackedData & FLAG_A) != 0;
bool EnableB = (PackedData & FLAG_B) != 0;
bool EnableC = (PackedData & FLAG_C) != 0;
```

### IgnoreHit()的注意事项

**IgnoreHit()只在任意命中着色器中有效！**
- 最近命中着色器中不能用
- 只有任意命中着色器可以忽略命中

**IgnoreHit()不是万能的：**
- 它只能让光线继续
- 不能改变光线的方向
- 不能添加新的命中

## 总结

RayTracingMaterialHitShaders.usf是光线追踪命中着色器，MooaToon在这里：
1. 添加了Toon着色模型的检测
2. 修改了混合模式的判断
3. 实现了Toon专用的光线追踪阴影控制
   - 禁用自阴影
   - 禁用Toon之间投射阴影

这个修改展示了：
- 如何在光线追踪中实现自定义阴影逻辑
- 如何用IgnoreHit()控制阴影
- 如何用位操作编码多个标志

关键理解：
- 光线追踪提供了很大的灵活性
- 任意命中着色器可以控制阴影
- 卡通渲染常常需要特殊的阴影处理
