# MooaToon 开发前置知识 - 图形学与TA基础

## 目录
1. [写给零基础开发者](#写给零基础开发者)
2. [基础图形学知识](#基础图形学知识)
3. [UE5渲染管线基础](#ue5渲染管线基础)
4. [卡通渲染技术基础](#卡通渲染技术基础)
5. [TA（技术美术）工作流基础](#ta技术美术工作流基础)
6. [MooaToon快速入门](#mooatoon快速入门)
7. [学习资源推荐](#学习资源推荐)

---

## 写给零基础开发者

### 什么是HybriToon项目？

**一句话概括**：我们要做一个「AI加持的卡通渲染引擎」，让艺术家只需要给一张参考图，AI就能自动调整出想要的Toon风格。

### 你需要知道的（按优先级）

| 优先级 | 领域 | 核心概念 | 学习时间 |
|--------|------|---------|---------|
| ⭐⭐⭐ | UE5基础 | 材质编辑器、蓝图、渲染管线 | 1-2周 |
| ⭐⭐⭐ | 卡通渲染 | Ramp纹理、描边、阴影 | 1周 |
| ⭐⭐ | 图形学 | GBuffer、延迟渲染、着色器 | 2-3周 |
| ⭐⭐ | TA基础 | 材质制作、法线贴图、光照 | 1-2周 |
| ⭐ | 深度学习 | CNN、风格迁移、ONNX | 3-4周 |

**好消息**：你不需要全部精通！我们会从「能跑起来」开始，逐步深入。

---

## 基础图形学知识

### 1. 渲染管线是什么？

**想象你在画画**：
1. **几何阶段** - 打草稿，确定物体位置
2. **光栅化** - 把线条变成像素
3. **像素着色** - 给像素上色
4. **输出** - 最终画面

**在3D渲染中**：
```
3D模型 → 顶点着色器 → 光栅化 → 像素着色器 → 屏幕
```

---

### 2. GBuffer是什么？（非常重要！）

**GBuffer = Geometry Buffer = 几何缓冲区**

**想象一下**：你要给照片修图，但你不仅想要最终的照片，还想要：
- 每个像素的颜色
- 每个像素的法线（朝向）
- 每个像素的粗糙度
- 每个像素的金属度
- ...等等

**这就是GBuffer！** 它把渲染所需的所有信息都存下来，供后续使用。

#### UE5中的传统GBuffer

| GBuffer通道 | 存储内容 | 用途 |
|------------|---------|------|
| **GBufferA** | BaseColor（基础颜色） | 物体颜色 |
| **GBufferB** | Normal（法线） | 表面朝向 |
| **GBufferC** | Metallic（金属度）+ Roughness（粗糙度） | PBR材质参数 |
| **GBufferD** | CustomData（自定义数据） | 特殊效果 |

#### MooaToon扩展的GBuffer

MooaToon新增了 **ToonBufferA**，专门用来存Toon渲染的数据！

```
传统GBuffer (A-D) + ToonBufferA = MooaToon GBuffer
```

---

### 3. 延迟渲染 vs 前向渲染

#### 前向渲染（Forward Rendering）

**简单理解**：每个物体，每个光源，都算一遍光照

**优点**：简单，适合少量光源
**缺点**：光源多了慢，O(物体数 × 光源数)

#### 延迟渲染（Deferred Rendering）

**MooaToon用的就是这个！**

**简单理解**：
1. 先把所有物体的GBuffer渲染出来（不管光照）
2. 再对GBuffer做光照计算

**优点**：光源多了也快，O(屏幕像素数 × 光源数)
**缺点**：内存占用大，不透明物体

**为什么MooaToon用延迟渲染？**
- Toon渲染需要很多自定义数据（Ramp索引、描边强度等）
- 延迟渲染可以把这些数据存在GBuffer里

---

### 4. 着色器（Shader）是什么？

**着色器 = 告诉GPU怎么渲染的小程序**

#### 两种主要着色器

| 着色器类型 | 作用 | 输入 | 输出 |
|-----------|------|------|------|
| **顶点着色器** | 处理顶点，变换位置 | 顶点位置、法线等 | 变换后的顶点 |
| **像素着色器** | 处理像素，计算颜色 | 像素位置、UV等 | 最终颜色 |

#### HLSL语言（Unreal用的着色器语言）

**简单示例**：
```hlsl
// 像素着色器
float4 MyPixelShader(float2 UV : TEXCOORD0) : SV_Target
{
    // 读取纹理
    float4 Color = Texture.Sample(Sampler, UV);
    
    // 变亮一点
    Color.rgb *= 1.2;
    
    // 输出
    return Color;
}
```

---

## UE5渲染管线基础

### 1. UE5的渲染流程

```
BasePass (基础通道)
    ↓
ShadowPass (阴影通道)
    ↓
DeferredLighting (延迟光照)
    ↓
PostProcessing (后期处理)
    ↓
最终画面
```

#### BasePass（基础通道）- MooaToon的核心

**做什么**：渲染GBuffer，包括ToonBufferA

**关键文件**：`Engine/Shaders/Private/BasePassPixelShader.usf`

**MooaToon在这里做什么**：
```hlsl
// 把MooaEncodedAttribute编码到ToonBufferA
EncodeToonGBufferDataToMRT(
    GBuffer.MooaToonContext.ToonGBuffer,
    GetMaterialMooaEncodedAttribute0(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute1(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute2(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute3(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute4(PixelMaterialInputs));
```

---

#### DeferredLighting（延迟光照）- Toon着色在这里

**做什么**：读取GBuffer，计算光照

**关键文件**：`Engine/Shaders/Private/ToonShadingModel.ush`

**MooaToon在这里做什么**：
```hlsl
// Toon BxDF - 核心的Toon着色逻辑
FDirectLighting ToonBxDF(...)
{
    // 1. 采样漫反射Ramp
    half4 DiffuseColorRamp = SampleGlobalRamp(...);
    
    // 2. 计算高光
    half3 Specular = ...;
    
    // 3. 计算描边
    half Outline = ...;
    
    // 输出
    return FDirectLighting(Diffuse, Specular);
}
```

---

### 2. UE5材质系统

#### 材质编辑器

**想象成一个可视化的编程环境**，用节点连线来写材质：

| 节点类型 | 作用 |
|---------|------|
| **Constant** | 常量（数字、颜色） |
| **Texture Sample** | 采样纹理 |
| **Multiply** | 乘法 |
| **Add** | 加法 |
| **Clamp** | 限制范围 |

#### MooaToon新增的材质属性

在材质编辑器中，你会看到5个新属性：

| 属性 | 类型 | 作用 |
|------|------|------|
| **Mooa Encoded Attribute 0** | Float4 | 着色特性ID + 阴影颜色 |
| **Mooa Encoded Attribute 1** | Float4 | 漫反射Ramp索引 + UV偏移 |
| **Mooa Encoded Attribute 2** | Float4 | 高光颜色 + 高光Ramp索引 |
| **Mooa Encoded Attribute 3** | Float4 | 高光Ramp UV偏移 + 反射 + 边缘光 |
| **Mooa Encoded Attribute 4** | Float4 | 面部阴影SDF + 模板 + 光线追踪 |

**注意**：你不需要一开始就理解所有这些，先知道它们是用来传递Toon参数的就行！

---

### 3. 控制台变量（CVar）

**CVar = Console Variable = 控制台变量**

**简单理解**：可以在运行时调整的参数

**MooaToon有30+个CVar**，例如：
```
r.Mooa.DiffuseColorRamp.EnablePostRampShadow 1
r.Mooa.RimLight.Intensity 0.5
r.Mooa.DebugValue 0
```

**怎么用**：在UE5控制台按 `~` 键，输入命令即可

---

## 卡通渲染技术基础

### 1. 什么是卡通渲染（Toon Shading）？

**卡通渲染 = 让3D渲染看起来像2D卡通画**

#### PBR vs Toon对比

| 特性 | PBR（真实感渲染） | Toon（卡通渲染） |
|------|------------------|-----------------|
| 光照过渡 | 平滑渐变 | 分层（阶跃） |
| 阴影 | 柔和 | 硬边 |
| 高光 | 物理准确 | 风格化 |
| 描边 | 无 | 有 |

---

### 2. Ramp纹理（核心！）

**Ramp纹理 = 渐变纹理 = 颜色查找表**

**简单理解**：
- 给光照强度（0-1），查Ramp纹理，得到颜色
- 这样可以把平滑的光照变成阶跃的卡通风格

#### Ramp纹理长什么样

```
[黑色] → [深灰色] → [浅灰色] → [白色]
  0.0      0.3        0.6        1.0
```

#### MooaToon中的Ramp

MooaToon用 **Ramp图集**（Atlas），把多个Ramp存在一张纹理里：

```
┌─────────────────────────────────┐
│  Ramp 0  │  Ramp 1  │  Ramp 2  │  ...
├─────────────────────────────────┤
│  Ramp 4  │  Ramp 5  │  Ramp 6  │  ...
└─────────────────────────────────┘
```

**怎么用**：
- `DiffuseColorRampIndex` - 选第几个Ramp
- `NoL` - 光照强度（0-1）
- 查Ramp纹理，得到颜色

---

### 3. 卡通渲染三要素

#### 要素1：漫反射Ramp

**传统PBR漫反射**：
```hlsl
// 平滑过渡
half3 Diffuse = BaseColor * NoL;
```

**Toon漫反射**：
```hlsl
// 阶跃过渡（查Ramp）
half4 RampColor = SampleGlobalRamp(RampAtlas, NoL, RampIndex);
half3 Diffuse = BaseColor * RampColor;
```

---

#### 要素2：风格化高光

**传统PBR高光**：
- 基于物理
- Cook-Torrance BRDF

**Toon高光**：
- 可以是Ramp（和漫反射一样）
- 可以是Kajiya-Kay（头发高光）
- 可以是任意风格化形状

---

#### 要素3：描边（Outline）

**常见描边方法**：

| 方法 | 优点 | 缺点 |
|------|------|------|
| **后处理描边** | 简单，不需要修改模型 | 可能不准确 |
| **背面挤出** | 准确 | 需要双面材质 |
| **屏幕空间深度/法线** | 准确，不需要修改模型 | 计算量大 |

**MooaToon用的是**：屏幕空间深度测试描边

---

### 4. MooaToon的特色功能

| 功能 | 说明 |
|------|------|
| **全局Ramp图集** | 所有材质共享一套Ramp |
| **各向异性高光** | Kajiya-Kay头发模型 |
| **屏幕空间描边** | 基于深度测试 |
| **距离场面部阴影** | SDF面部阴影 |
| **屏幕空间头发阴影** | 头发自阴影 |

---

## TA（技术美术）工作流基础

### 1. 什么是TA？

**TA = Technical Artist = 技术美术**

**一句话**：既懂艺术，又懂技术，在艺术家和程序员之间搭桥

---

### 2. TA的核心技能

#### 技能1：材质制作

**关键概念**：
- **Base Color** - 基础颜色
- **Normal Map** - 法线贴图（表面细节）
- **Roughness** - 粗糙度（0=光滑，1=粗糙）
- **Metallic** - 金属度（0=非金属，1=金属）

**MooaToon材质额外需要**：
- MooaEncodedAttribute0-4
- Toon着色模型

---

#### 技能2：纹理制作

**常用软件**：
- Photoshop / Substance Painter
- Clip Studio Paint（卡通风格）

**Ramp纹理制作要点**：
- 宽度256-1024，高度16-64
- 阶跃分明，不要太平滑
- 可以有多个颜色层级

---

#### 技能3：光照设置

**Toon光照要点**：
- 主光源方向要明显（方便画阴影）
- 环境光不要太强（保持对比度）
- 可以用 **r.Mooa.GlobalIlluminationDirectionality** 调整

---

### 3. MooaToon材质制作流程

```
1. 导入模型
    ↓
2. 创建材质
    ├─ 着色模型选Toon
    ├─ 连接BaseColor、Normal等
    └─ 连接MooaEncodedAttribute0-4
    ↓
3. 调整参数
    ├─ Ramp索引
    ├─ 描边强度
    ├─ 边缘光
    └─ 等等
    ↓
4. 测试渲染
    ↓
5. 调整CVar
```

---

## MooaToon快速入门

### 1. 环境准备

#### 需要的软件

| 软件 | 用途 |
|------|------|
| **UE 5.3-5.5** | 引擎（必须） |
| **Visual Studio 2022** | 编译引擎（如果你要改引擎） |
| **Git** | 版本控制 |
| **Python 3.10+** | AI部分（HybriToon需要） |

---

### 2. 获取MooaToon

```bash
# 克隆MooaToon仓库
git clone https://github.com/JasonMa0012/MooaToon.git

# 或者用你已经有的版本
# D:\MooaToon\MooaToon-Engine
```

---

### 3. 第一个MooaToon材质

#### 步骤1：创建材质

1. 在UE5内容浏览器右键
2. Material → 命名为「MyFirstToonMaterial」
3. 双击打开材质编辑器

#### 步骤2：设置Toon着色模型

1. 在材质详情面板
2. 找到「Material」→「Shading Model」
3. 选择「Toon」

#### 步骤3：连接基础属性

```
纹理采样（BaseColor）→ BaseColor节点
纹理采样（Normal）→ Normal节点
常量（0.5）→ Roughness节点
常量（0.0）→ Metallic节点
```

#### 步骤4：连接MooaEncodedAttribute（可选，先用默认值）

你可以先不连，用默认值测试，或者：
```
常量（0.0, 0.0, 0.0, 0.0）→ MooaEncodedAttribute0
常量（0.0, 0.0, 0.0, 0.0）→ MooaEncodedAttribute1
常量（0.0, 0.0, 0.0, 0.0）→ MooaEncodedAttribute2
常量（0.0, 0.0, 0.0, 0.0）→ MooaEncodedAttribute3
常量（0.0, 0.0, 0.0, 0.0）→ MooaEncodedAttribute4
```

#### 步骤5：保存并应用

1. 点击「Save」
2. 拖到模型上
3. 看效果！

---

### 4. 常用CVar调试

| CVar命令 | 作用 |
|----------|------|
| `r.Mooa.DebugValue 1` | 开启调试模式 |
| `r.Mooa.RimLight.Intensity 0.5` | 调整边缘光强度 |
| `r.Mooa.DiffuseColorRamp.EnablePostRampShadow 1` | 开启Ramp后阴影 |

---

## 学习资源推荐

### 1. UE5基础

| 资源 | 类型 |
|------|------|
| [UE5官方文档](https://docs.unrealengine.com/5.0/) | 文档 |
| [UE5材质基础教程（B站）](https://www.bilibili.com/) | 视频 |
| [虚幻引擎官方油管频道](https://www.youtube.com/@UnrealEngine) | 视频 |

---

### 2. 图形学基础

| 资源 | 类型 |
|------|------|
| [Learn OpenGL](https://learnopengl.com/) | 教程 |
| [GAMES101-现代计算机图形学入门](https://www.bilibili.com/video/BV1X7411F744/) | 视频 |
| [Real-Time Rendering 4th](https://www.realtimerendering.com/) | 书籍 |

---

### 3. 卡通渲染

| 资源 | 类型 |
|------|------|
| [MooaToon 文档](https://mooatoon.com/docs/) | 文档 |
| [卡通渲染技术指南](https://www.patreon.com/c/_/posts) | 文章 |
| [UE5 Toon Shader教程（YouTube）](https://www.youtube.com/) | 视频 |

---

### 4. TA基础

| 资源 | 类型 |
|------|------|
| [TA之路](https://www.zhihu.com/column/ta-road) | 文章 |
| [Substance Painter官方教程](https://www.adobe.com/products/substance3d-painter.html) | 视频 |
| [美术的程序思维](https://www.bilibili.com/) | 视频 |

---

## 下一步学习路径

### 第1周：UE5基础
- 熟悉UE5界面
- 学会材质编辑器基础
- 做一个简单的PBR材质

### 第2周：MooaToon入门
- 跑通MooaToon示例
- 做第一个Toon材质
- 调整CVar看效果

### 第3-4周：图形学基础
- 理解渲染管线
- 理解GBuffer
- 理解延迟渲染

### 第5-6周：卡通渲染深入
- 理解Ramp纹理
- 理解ToonBxDF
- 看MooaToon着色器代码

### 第7-8周：HybriToon准备
- Python基础
- 深度学习基础
- ONNX基础

---

## 常见问题

### Q1: 我完全不会编程，能做这个项目吗？

**A**: 可以！项目分工：
- **TA方向**：不需要写太多代码，专注于材质和艺术
- **程序方向**：需要写C++/HLSL/Python
- **AI方向**：需要写Python

### Q2: 我应该先学什么？

**A**: 按这个顺序：
1. UE5材质编辑器（能做出材质就行）
2. 跑通MooaToon示例
3. 看MooaToon文档
4. 再深入图形学

### Q3: 看不懂代码怎么办？

**A**: 正常！
1. 先看报告1-4，建立全局概念
2. 再看关键文件（注释都有中文）
3. 搜关键词（比如「Ramp」「Outline」）
4. 看不懂就跳过，先看能看懂的

---

## 总结

### 你现在应该知道的

1. **GBuffer** - 存渲染数据的缓冲区，MooaToon扩展了ToonBufferA
2. **Ramp纹理** - 卡通渲染核心，把平滑光照变阶跃
3. **MooaEncodedAttribute0-4** - 5个材质属性，用来传Toon参数
4. **延迟渲染** - MooaToon用的渲染方式

### 下一步行动

1. 打开UE5，跑MooaToon示例
2. 做第一个Toon材质
3. 调整CVar，看效果变化
4. 看技术报告1-4，建立全局概念

---

**祝你学习顺利！有问题随时问！** 🎨

---

**文档生成时间**: 2026年4月6日
**文档版本**: v1.0
**目标读者**: 零基础开发者
