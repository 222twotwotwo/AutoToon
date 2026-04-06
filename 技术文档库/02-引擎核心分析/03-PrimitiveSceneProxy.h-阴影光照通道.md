# PrimitiveSceneProxy.h - 图元场景代理

## 文件信息
- **路径**: `Engine/Source/Runtime/Engine/Public/PrimitiveSceneProxy.h`
- **作用**: 定义图元场景代理类，用于渲染线程
- **MooaToon修改**: 添加Mooa阴影光照通道功能

## 关键代码分析

### 1. SetMooaCastShadowsToLightingChannels_GameThread函数声明（第282-287行）

```cpp
// Mooa Shadow Channel Mask
/**
 * Updates the MooaCastShadowsToLightingChannels for the primitive proxy.
 */
void SetMooaCastShadowsToLightingChannels_GameThread(FLightingChannels LightingChannels);
// Mooa End
```

#### 零基础解释

这是设置Mooa阴影光照通道的游戏线程函数。

**什么是PrimitiveSceneProxy？**
- 图元场景代理
- 游戏线程和渲染线程之间的桥梁
- 存储渲染需要的数据

**什么是光照通道（Lighting Channels）？**
- UE5的光照系统
- 可以让光源只影响特定的物体
- 比如：光源1只影响通道1的物体

**MooaCastShadowsToLightingChannels是什么？**
- MooaToon新增的功能
- 控制Toon物体向哪些通道投射阴影
- 可以让Toon物体只向特定通道投阴影

**_GameThread后缀：**
- 表示这个函数在游戏线程调用
- 游戏线程不能直接修改渲染线程数据
- 需要用这种方式安全地传递

**类比理解：**
```
想象一个舞台：
- 物体在舞台上
- 灯光照亮物体
- 物体投下阴影

普通光照通道：
- 控制哪个灯照亮哪个物体

Mooa阴影光照通道（新增）：
- 控制哪个物体向哪些通道投阴影
- 更精细的阴影控制
```

### 2. GetMooaCastShadowsToLightingChannels函数（第748-750行）

```cpp
// Mooa Shadow Channel Mask
inline uint8 GetMooaCastShadowsToLightingChannels() const { return MooaCastShadowsToLightingChannels; }
// Mooa End
```

#### 零基础解释

这是获取Mooa阴影光照通道的内联函数。

**inline函数：**
- 内联函数
- 编译器可能直接展开代码
- 更快的函数调用

**这个函数做什么？**
- 返回MooaCastShadowsToLightingChannels成员变量
- 供渲染线程读取
- const表示不修改成员

### 3. MooaCastShadowsToLightingChannels成员变量（第1529-1531行）

```cpp
// Mooa Shadow Channel Mask
uint8 MooaCastShadowsToLightingChannels = 0b111;
// Mooa End
```

#### 零基础解释

这是Mooa阴影光照通道的成员变量。

**uint8类型：**
- 8位无符号整数
- 可以存0-255的值
- 足够存3个光照通道

**默认值0b111：**
- 二进制：111
- 十进制：7
- 表示所有3个通道都启用
- 默认向所有通道投阴影

**位掩码（Bit Mask）：**
- 每一位代表一个通道
- 位为1表示启用
- 位为0表示禁用

```
通道1：位0（最右边）
通道2：位1
通道3：位2

0b111 = 所有通道启用
0b001 = 只通道1启用
0b010 = 只通道2启用
0b100 = 只通道3启用
```

## 技术细节

### 光照通道的工作原理

```
普通光照通道（LightingChannelMask）：
├─ 控制：光源影响哪些物体
└─ 位置：LightSceneInfo.cpp

Mooa阴影光照通道（MooaCastShadowsToLightingChannels）：
├─ 控制：物体向哪些通道投阴影
└─ 位置：PrimitiveSceneProxy.h（MooaToon新增）
```

**两者的区别：**

| 功能 | 普通光照通道 | Mooa阴影光照通道 |
|-----|------------|-----------------|
| **控制方向** | 光源→物体 | 物体→阴影通道 |
| **控制内容** | 光照 | 阴影投射 |
| **谁控制** | 光源 | 物体 |

### 为什么需要Mooa阴影光照通道？

**场景举例：**
```
场景：
- 角色A（Toon）
- 角色B（Toon）
- 光源1（通道1）
- 光源2（通道2）

需求：
- 角色A只向通道1投阴影
- 角色B只向通道2投阴影

Mooa阴影光照通道可以实现！
```

### 游戏线程到渲染线程的传递

```
游戏线程：
1. 调用SetMooaCastShadowsToLightingChannels_GameThread()
   ↓
2. 把请求加入渲染命令队列
   ↓
3. 渲染线程执行命令
   ↓
4. 更新MooaCastShadowsToLightingChannels变量
   ↓
5. 渲染时使用新值
```

## MooaToon集成总结

### 修改内容
1. 添加SetMooaCastShadowsToLightingChannels_GameThread函数
2. 添加GetMooaCastShadowsToLightingChannels函数
3. 添加MooaCastShadowsToLightingChannels成员变量

### 设计意图
- 给Toon物体提供精细的阴影控制
- 支持按通道控制阴影投射
- 与UE5光照通道系统配合

## 开发提示

### 如何使用Mooa阴影光照通道？

```cpp
// 在游戏线程中
UPrimitiveComponent* Component = ...;
if (Component &amp;&amp; Component-&gt;SceneProxy)
{
    // 设置只向通道1和2投阴影
    FLightingChannels Channels;
    Channels.bChannel0 = true;
    Channels.bChannel1 = true;
    Channels.bChannel2 = false;
    
    Component-&gt;SceneProxy-&gt;SetMooaCastShadowsToLightingChannels_GameThread(Channels);
}
```

### 如何在渲染线程读取？

```cpp
// 在渲染线程中
const FPrimitiveSceneProxy* Proxy = ...;
uint8 ShadowChannels = Proxy-&gt;GetMooaCastShadowsToLightingChannels();

// 检查通道1
if (ShadowChannels &amp; (1 &lt;&lt; 0))
{
    // 通道1启用
}
```

### FLightingChannels和uint8的转换

```cpp
// FLightingChannels → uint8
FLightingChannels Channels;
Channels.bChannel0 = true;
Channels.bChannel1 = true;
Channels.bChannel2 = false;

uint8 ChannelMask = 0;
if (Channels.bChannel0) ChannelMask |= (1 &lt;&lt; 0);
if (Channels.bChannel1) ChannelMask |= (1 &lt;&lt; 1);
if (Channels.bChannel2) ChannelMask |= (1 &lt;&lt; 2);
```

## 总结

PrimitiveSceneProxy.h是图元场景代理的头文件，MooaToon在这里：
1. 添加了Mooa阴影光照通道功能
2. 提供设置和获取函数
3. 默认值0b111（所有通道启用）

这个修改展示了：
- 如何扩展UE5的光照系统
- 如何添加精细的阴影控制
- 游戏线程和渲染线程的安全交互

关键理解：
- Toon渲染需要特殊的阴影控制
- MooaToon扩展了光照通道系统
- 可以按通道控制阴影投射
