# LightSceneInfo.cpp - 光源场景信息

## 文件信息
- **路径**: `Engine/Source/Runtime/Renderer/Private/LightSceneInfo.cpp`
- **作用**: 光源场景信息的实现
- **MooaToon修改**: 添加Mooa阴影光照通道检查

## 关键代码分析

### 1. 阴影光照通道检查（第478-481行）

```cpp
if (!(LightSceneInfo-&gt;Proxy-&gt;GetLightingChannelMask() &amp; PrimitiveSceneProxy-&gt;GetMooaCastShadowsToLightingChannels())) // Mooa Shadow Channel Mask
{
    return false;
}
```

#### 零基础解释

这是检查光源和物体的阴影光照通道是否匹配的代码。

**这段代码在哪里？**
- 在判断光源是否应该投射阴影给物体的函数中
- 如果返回false，就不投射阴影
- 如果返回true，就投射阴影

**代码逻辑：**
```cpp
if (!( 光源的光照通道 &amp; 物体的Mooa阴影光照通道 ))
{
    return false;  // 不投射阴影
}
```

**逐句解释：**

**1. LightSceneInfo-&gt;Proxy-&gt;GetLightingChannelMask()**
- 获取光源的光照通道掩码
- 比如：0b101（通道1和3启用）

**2. PrimitiveSceneProxy-&gt;GetMooaCastShadowsToLightingChannels()**
- 获取物体的Mooa阴影光照通道
- 比如：0b110（通道2和3启用）

**3. & 操作符（按位与）**
- 两个掩码的交集
- 0b101 &amp; 0b110 = 0b100（只有通道3匹配）

**4. ! 操作符（逻辑非）**
- 如果结果为0（没有匹配的通道）
- !0 = true，进入if块

**5. return false**
- 不投射阴影

**举个例子：**

```
场景：
- 光源A：通道1启用（0b001）
- 光源B：通道2启用（0b010）
- 物体X：只向通道1投阴影（0b001）
- 物体Y：只向通道2投阴影（0b010）

结果：
- 光源A → 物体X：匹配，投阴影
- 光源A → 物体Y：不匹配，不投阴影
- 光源B → 物体X：不匹配，不投阴影
- 光源B → 物体Y：匹配，投阴影
```

**类比理解：**
```
想象入场券：
- 光源：有几张票（通道1、2、3）
- 物体：只接受某些票
- 检查：光源的票和物体接受的票有没有交集
- 如果有：允许入场（投阴影）
- 如果没有：拒绝入场（不投阴影）
```

## 技术细节

### 阴影投射的完整判断流程

```
判断是否投射阴影：
1. 光源是否影响物体的包围盒？
   ↓
2. 光源是否只从电影物体投射阴影？
   ↓
3. Mooa阴影光照通道是否匹配？ ← MooaToon新加的
   ↓
4. 所有检查通过，投射阴影
```

**MooaToon的修改在第3步！**

### 位运算详解

```cpp
// 按位与（&）
0b101  // 光源通道
&amp; 0b110  // 物体Mooa通道
= 0b100  // 结果：只有通道3匹配

// 如果结果不为0：有匹配的通道
if (Result != 0)
{
    // 投射阴影
}

// 如果结果为0：没有匹配的通道
if (Result == 0)
{
    // 不投射阴影
}
```

**MooaToon的代码用的是!：**
```cpp
if (!(A &amp; B))  // 等价于 if ((A &amp; B) == 0)
{
    return false;
}
```

### 为什么需要这个功能？

**场景举例：**

```
场景：
- 角色A（Toon）：只向通道1投阴影
- 角色B（Toon）：只向通道2投阴影
- 光源1（通道1）：照亮角色A
- 光源2（通道2）：照亮角色B

效果：
- 角色A的阴影只在光源1中出现
- 角色B的阴影只在光源2中出现
- 更精细的阴影控制
```

## MooaToon集成总结

### 修改内容
1. 在阴影投射判断函数中
2. 添加Mooa阴影光照通道检查
3. 如果通道不匹配，不投射阴影

### 设计意图
- 给Toon物体提供精细的阴影控制
- 支持按通道控制阴影投射
- 与UE5光照通道系统配合

## 开发提示

### 如何使用Mooa阴影光照通道？

```cpp
// 设置物体的Mooa阴影光照通道
UPrimitiveComponent* Component = ...;
if (Component &amp;&amp; Component-&gt;SceneProxy)
{
    FLightingChannels Channels;
    Channels.bChannel0 = true;  // 通道1
    Channels.bChannel1 = false; // 通道2
    Channels.bChannel2 = true;  // 通道3
    
    Component-&gt;SceneProxy-&gt;SetMooaCastShadowsToLightingChannels_GameThread(Channels);
}

// 设置光源的光照通道
ULightComponent* Light = ...;
Light-&gt;LightingChannels.bChannel0 = true;
Light-&gt;LightingChannels.bChannel1 = false;
Light-&gt;LightingChannels.bChannel2 = true;
```

### 位运算辅助函数

```cpp
// 检查通道是否启用
bool IsChannelEnabled(uint8 Mask, int Channel)
{
    return (Mask &amp; (1 &lt;&lt; Channel)) != 0;
}

// 启用通道
uint8 EnableChannel(uint8 Mask, int Channel)
{
    return Mask | (1 &lt;&lt; Channel);
}

// 禁用通道
uint8 DisableChannel(uint8 Mask, int Channel)
{
    return Mask &amp; ~(1 &lt;&lt; Channel);
}
```

## 总结

LightSceneInfo.cpp是光源场景信息的实现文件，MooaToon在这里：
1. 添加Mooa阴影光照通道检查
2. 如果光源和物体的通道不匹配，不投射阴影
3. 提供精细的阴影控制

这个修改展示了：
- 如何扩展UE5的阴影系统
- 如何用光照通道控制阴影
- Toon渲染的精细控制需求

关键理解：
- Toon需要特殊的阴影控制
- MooaToon用光照通道实现
- 可以按通道控制阴影投射
