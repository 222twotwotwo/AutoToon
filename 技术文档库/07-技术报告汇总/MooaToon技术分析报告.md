# MooaToon 技术分析报告

## 目录
1. [项目概述](#项目概述)
2. [核心技术原理](#核心技术原理)
3. [代码架构分析](#代码架构分析)
4. [材质参数系统](#材质参数系统)
5. [深度魔改与神经渲染评估](#深度魔改与神经渲染评估)
6. [HybriToon项目集成方案](#hybritoon项目集成方案)

---

## 项目概述

### MooaToon简介

**MooaToon** 是由 Jason Ma 开发的 UE5 专业三渲二渲染插件，旨在彻底解决 UE5 卡通渲染的痛点。它结合了 UE 原生的光照特性和强大的材质系统，释放美术师的潜力。

- **官方网站**: https://mooatoon.com/
- **GitHub**: https://github.com/JasonMa0012/MooaToon
- **版本**: 支持 UE5.4-5.7+

### 核心特性

#### 1. 光照系统
- **Lumen 全局光照**: 可自由控制 GI 强度和混合
- **反射控制**: 可自由控制反射强度
- **多类型阴影**: 虚拟阴影贴图、光线追踪阴影
- **高级阴影功能**:
  - 支持忽略任意部分的自阴影
  - 可控的头发阴影宽度

#### 2. 半透明渲染
- **前向着色**: 支持光照半透明
- **抖动透明度**: Dithered Opacity
- **抖动半透明阴影**: Dithered Translucency Shadow

#### 3. 材质灵活性
通过材质层（Material Layer）可自由组合和创建包含以下特性的材质：

- **日式动画风格**: 主要纯色块、清晰光影，常用于还原动画和手绘效果
- **美式卡通风格**: 通常带有 GI，阴影更柔和
- **自定义参数**:
  - 可自定义基础色、阴影色、高光色
  - 可自定义光影范围（待支持 Ramp Map）
  - 可自定义高光范围，支持各向异性高光
  - 基于 Kajiya-Kay 的动态风格化头发高光
  - 基于每光源屏幕空间深度测试的边缘光
  - 基于球形映射顶点法线、法线贴图或其他方法的面部阴影

#### 4. 描边系统
通过单个叠加材质实现描边：

- **传统背面描边**: Back Face Outline
- **屏幕空间深度法线卷积正面描边**
- **输出速度**: 配合 TSR 抗锯齿使用

#### 5. 法线平滑工具
- **一键烘焙工具**: 用于平滑法线
- **Houdini 示例文件**: 用于处理法线和顶点颜色

#### 6. 电影级后期效果
- **正确的自动曝光和手动曝光**
- **全局控制的曝光补偿**
- **全局控制的饱和度、对比度等调整**
- **LookDev 工具**

#### 7. 其他特性
- **Morph Targets 法线强度**

---

## 核心技术原理

### 1. 平滑法线技术

#### 问题背景
三渲二渲染中，硬边会导致阴影断裂，严重影响卡通效果的视觉质量。传统的法线计算方法无法满足卡通渲染对平滑光影过渡的需求。

#### 解决方案：SmoothNormalCommand

**文件位置**: `MooaToonScripts/Source/MooaToonEditorScripts/Private/SmoothNormalCommand.cpp`

**核心思想**:
1. 重新计算网格的法线
2. 按位置聚合相同顶点的法线
3. 计算曲率并进行加权
4. 将平滑法线变换到切线空间
5. 存储到 UV 通道供材质使用

#### 详细实现步骤

```cpp
// 步骤1: 重新计算法线
TArray<FVector3f> RecomputedNormals;
MeshUtilities.CalculateNormals(Positions, VertexIndices, UV0s, SmoothingGroupIndices, TangentOptions, RecomputedNormals);

// 步骤2: 构建位置-法线映射
TMap<FVector3f, FVector3f> PosNormalMap;
for (每个顶点) {
    PosNormalMap[位置] += 法线;
}

// 步骤3: 计算曲率
// 计算一个顶点与周围顶点的法线差异的平均值作为曲率
// 越接近平面，曲率越接近零；越接近边缘，曲率越接近1
TArray<float> Curvatures;
UMooaToonEditorBPLibrary::MooaGetMeshNormalDiffCurvatures(Mesh, LOD, Curvatures);

// 步骤4: 平均法线并变换到切线空间
for (每个Wedge) {
    FVector3f SmoothNormal = PosNormalMap[位置].GetSafeNormal();
    
    // 构建切线空间矩阵
    FMatrix TangentToNormal(TangentX, TangentY, TangentZ, FVector(0,0,0));
    
    // 变换到切线空间
    FVector SmoothNormalTangentSpace = TangentToNormal.InverseTransformVector(SmoothNormal);
    
    // 曲率加权
    SmoothNormalTangentSpace *= Curvatures[WedgeIndex];
    
    // 存储到UV通道
    UV2 = (UV2.X, SmoothNormalTangentSpace.X);
    UV3 = (SmoothNormalTangentSpace.Y, SmoothNormalTangentSpace.Z);
}
```

#### 曲率计算原理

**文件位置**: `MooaToonScripts/Source/MooaToonEditorScripts/Private/MooaToonEditorBPLibrary.cpp:253-376`

```cpp
// 计算一个顶点与周围顶点的法线差异的平均值作为曲率
for (每个面) {
    for (每个顶点) {
        float NormalDiff = 0;
        int AdjacentCount = 0;
        
        for (每个相邻面) {
            for (每个相邻顶点) {
                if (相邻点 != 当前点) {
                    // 计算法线点积并映射到[0,1]
                    float Dot = AdjacentNormal.Dot(CurrNormal) * 0.5 + 0.5;
                    // 法线差异 = 1 - 点积
                    NormalDiff += 1 - Dot;
                    AdjacentCount++;
                }
            }
        }
        
        // 归一化并开方增强对比度
        NormalDiff /= AdjacentCount;
        Curvatures[WedgeIndex] = Clamp(Pow(NormalDiff, 0.5f), 1e-3, 1);
    }
}
```

**曲率作用**:
- 在材质中，曲率可以用于控制描边粗细（边缘曲率高，描边更粗）
- 可以用于控制阴影柔和度
- 可以用于增强手绘感

### 2. 数据访问层设计

**文件位置**: `MooaToonScripts/Source/MooaToonEditorScripts/Public/MooaToonEditorBPLibrary.h`

MooaToon 提供了完整的网格数据读写 API，支持静态网格和骨骼网格：

#### 核心概念说明

```
Positions (Point)：组成Mesh的很多独立的只有位置属性的Vertex
    类似Houdini中的Point概念
    每一个Vertex可以被VertexIndices引用多次以构成一个个Triangle

Wedge：附加了位置、Normal、UV等属性的Vertex
    类似Houdini中的Vertex概念
    两个不同的Wedge可以拥有相同的位置和不同的Normal或UV（硬边）

VertexIndices：键为WedgeIndex，值为VertexIndex
    每三个为一个Triangle依次排列
    Wedge的数量通常远多于Vertex
```

#### API 接口

```cpp
// 获取网格数据
UFUNCTION(BlueprintCallable)
static void MooaGetMeshData(
    UObject* InStaticOrSkeletalMesh,
    int InLOD,
    TArray<FVector3f>& OutPositions,
    TArray<int>& OutVertexIndices,
    TArray<FVector3f>& OutNormals,
    TArray<FVector3f>& OutTangents,
    TArray<FVector3f>& OutBinormals,
    TArray<FColor>& OutColors,
    TArray<FVector2f>& OutUV0s,
    TArray<FVector2f>& OutUV1s,
    TArray<FVector2f>& OutUV2s,
    TArray<FVector2f>& OutUV3s
);

// 设置网格数据
UFUNCTION(BlueprintCallable)
static void MooaSetMeshData(...);

// 获取曲率
UFUNCTION(BlueprintCallable)
static void MooaGetMeshNormalDiffCurvatures(...);

// 重新计算MikkT空间切线
UFUNCTION(BlueprintCallable)
static void MooaRecomputeMikkTSpaceTangentBinormal(...);

// 多线程最近点查找（BVH加速）
UFUNCTION(BlueprintCallable)
static void MooaFindNearestPointsOnDynamicMesh(...);
```

---

## 代码架构分析

### 目录结构

```
MooaToon-Engine/
├── Engine/
│   └── Plugin/
│       ├── MooaToonScripts/          # 核心插件代码
│       │   ├── Source/
│       │   │   ├── MooaToonScripts/          # 运行时模块
│       │   │   │   ├── Private/
│       │   │   │   │   └── MooaToonScripts.cpp
│       │   │   │   ├── Public/
│       │   │   │   │   └── MooaToonScripts.h
│       │   │   │   └── MooaToonScripts.Build.cs
│       │   │   └── MooaToonEditorScripts/    # 编辑器模块
│       │   │       ├── Private/
│       │   │       │   ├── SmoothNormalCommand.cpp      # 平滑法线核心
│       │   │       │   ├── MooaToonEditorBPLibrary.cpp # 蓝图库
│       │   │       │   └── MooaToonEditorScripts.cpp
│       │   │       ├── Public/
│       │   │       │   ├── SmoothNormalCommand.h
│       │   │       │   ├── MooaToonEditorBPLibrary.h
│       │   │       │   └── MooaToonEditorScripts.h
│       │   │       └── MooaToonEditorScripts.Build.cs
│       │   ├── Binaries/
│       │   └── MooaToonScripts.uplugin
│       └── MooaToonThirdparty/        # 第三方依赖
│           ├── VRM4U/                   # VRM支持
│           ├── KawaiiPhysics/           # 物理模拟
│           └── HoudiniEngineForUnreal/ # Houdini集成
```

### 模块架构

#### 1. MooaToonScripts（运行时模块）
- **类型**: Runtime
- **用途**: 游戏运行时所需的核心功能
- **当前状态**: 基础框架，功能相对简单

#### 2. MooaToonEditorScripts（编辑器模块）
- **类型**: Editor
- **用途**: 编辑器工具、数据处理、蓝图节点
- **核心功能**:
  - `SmoothNormalCommand`: 平滑法线命令
  - `MooaToonEditorBPLibrary`: 蓝图函数库

### 插件配置

**文件位置**: `MooaToonScripts/MooaToonScripts.uplugin`

```json
{
    "FileVersion": 3,
    "Version": 1,
    "VersionName": "0.1.0",
    "FriendlyName": "MooaToonScripts",
    "Description": "The Ultimate Solution for Cinematic Toon Rendering in UE5",
    "Category": "Rendering",
    "CreatedBy": "Jason Ma",
    "CreatedByURL": "https://github.com/JasonMa0012/MooaToon",
    "DocsURL": "https://mooatoon.com/docs/getting-started",
    "CanContainContent": false,
    "Modules": [
        {
            "Name": "MooaToonScripts",
            "Type": "Runtime",
            "LoadingPhase": "Default"
        },
        {
            "Name": "MooaToonEditorScripts",
            "Type": "Editor",
            "LoadingPhase": "Default"
        }
    ],
    "Plugins": [
        {
            "Name": "GeometryScripting",
            "Enabled": true
        }
    ]
}
```

---

## 材质参数系统

基于 MooaToon 的功能特性和参考 MToon 实现，设计完整的材质参数系统。

### 参数向量结构设计

参考实现: `HybriToon/material_params.py`

#### 总体架构

总维度: **53维**

```
MooaToonMaterialParams
├── ColorParams (12维)
│   ├── base_color (3)
│   ├── shadow_color (3)
│   ├── highlight_color (3)
│   └── outline_color (3)
├── ShadingParams (6维)
│   ├── shadow_threshold (1)
│   ├── shadow_softness (1)
│   ├── shadow_offset (1)
│   ├── gi_intensity (1)
│   ├── gi_blend (1)
│   └── reflection_intensity (1)
├── HighlightParams (6维)
│   ├── highlight_intensity (1)
│   ├── highlight_range (1)
│   ├── highlight_softness (1)
│   ├── highlight_anisotropy (1)
│   ├── highlight_anisotropy_dir (1)
│   └── highlight_shift (1)
├── HairParams (6维)
│   ├── hair_shadow_width (1)
│   ├── hair_highlight_intensity (1)
│   ├── hair_highlight_shift (1)
│   ├── hair_highlight_width (1)
│   ├── kajiya_kay_shift (1)
│   └── kajiya_kay_secondary (1)
├── OutlineParams (6维)
│   ├── outline_width (1)
│   ├── outline_width_curve (1)
│   ├── outline_depth_offset (1)
│   ├── outline_screen_space (1)
│   ├── outline_backface (1)
│   └── outline_frontface (1)
├── RimLightParams (4维)
│   ├── rimlight_intensity (1)
│   ├── rimlight_range (1)
│   ├── rimlight_softness (1)
│   └── rimlight_depth_test (1)
├── FaceShadowParams (4维)
│   ├── faceshadow_intensity (1)
│   ├── faceshadow_blur (1)
│   ├── faceshadow_normalmap_weight (1)
│   └── faceshadow_spherical_weight (1)
└── PostProcessParams (9维)
    ├── exposure_compensation (1)
    ├── saturation (1)
    ├── contrast (1)
    ├── gamma (1)
    ├── temperature (1)
    ├── tint (1)
    ├── vignette_intensity (1)
    ├── vignette_radius (1)
    └── morph_normal_intensity (1)
```

### 风格预设

#### 1. 日式动画风格（Japanese Anime）
- 特点: 高对比度、纯色块、清晰阴影边缘
- 关键参数:
  - shadow_softness: 0.1（硬阴影）
  - shadow_threshold: 0.5（中间阈值）
  - gi_intensity: 0.0（关闭GI）

#### 2. 美式卡通风格（American Cartoon）
- 特点: 柔和阴影、带GI、更自然的过渡
- 关键参数:
  - shadow_softness: 0.5（柔和阴影）
  - gi_intensity: 0.8（开启GI）
  - reflection_intensity: 0.5

#### 3. 水彩风格（Watercolor）
- 特点: 非常柔和、低对比度、温暖色调
- 关键参数:
  - shadow_softness: 0.9（极软阴影）
  - saturation: 0.8（降低饱和度）
  - temperature: 0.3（暖色调）

#### 4. 赛博朋克风格（Cyberpunk）
- 特点: 高饱和、霓虹色调、强高光
- 关键参数:
  - highlight_intensity: 1.5（强高光）
  - saturation: 1.5（高饱和）
  - outline_width: 1.2（粗描边）

---

## 深度魔改与神经渲染评估

### 1. MooaToon 魔改可行性分析

#### 1.1 优势
- **开源架构**: 核心代码完全开源，可自由修改
- **模块化设计**: 分离运行时和编辑器模块，便于扩展
- **完整 API**: 提供数据读写接口，便于集成
- **材质层架构**: Material Layer 支持灵活组合
- **UE5 原生**: 深度集成 Lumen、Nanite 等新技术

#### 1.2 限制
- **需要修改引擎**: MooaToon 本质上需要修改引擎代码（不是纯插件）
- **材质系统黑盒**: 核心材质逻辑可能在引擎修改部分
- **文档有限**: 技术细节文档较少

### 2. 神经渲染集成方案

#### 方案一：参数控制（推荐，低耦合）
**实现难度**: ⭐⭐ (简单)
**性能**: ⭐⭐⭐⭐⭐ (优秀)
**质量**: ⭐⭐⭐ (良好)

**架构**:
```
参考图 → 风格编码器 (PyTorch) → 参数向量 → ONNX → UE5 NNE → MooaToon材质
```

**优点**:
- 不修改 MooaToon 底层
- 推理速度快
- 可解释性强
- 易于调试

**缺点**:
- 受限于现有参数空间
- 无法实现像素级风格化

#### 方案二：特征图注入（中等耦合）
**实现难度**: ⭐⭐⭐ (中等)
**性能**: ⭐⭐⭐ (良好)
**质量**: ⭐⭐⭐⭐ (很好)

**架构**:
```
参考图 → CNN特征提取 → 特征图 → 注入MooaToon材质 → 风格化渲染
```

**修改点**:
- 在 MooaToon 材质中添加特征图输入
- 修改光照计算，融合神经特征

**优点**:
- 更灵活的风格控制
- 可实现局部风格变化

**缺点**:
- 需要修改材质
- 显存占用增加

#### 方案三：端到端神经渲染（深度魔改）
**实现难度**: ⭐⭐⭐⭐⭐ (困难)
**性能**: ⭐⭐ (一般)
**质量**: ⭐⭐⭐⭐⭐ (最佳)

**架构**:
```
3D场景 → 几何缓冲 → 神经渲染器 → 最终图像
          ↓
     MooaToon特征
```

**修改点**:
- 替换或增强 MooaToon 的着色器
- 集成神经渲染管线（如 Instant NGP、3DGS）
- 可能需要修改引擎渲染管线

**优点**:
- 最高质量的风格化
- 可实现手绘质感
- 突破传统渲染限制

**缺点**:
- 开发复杂度极高
- 性能开销大
- 需要大量训练数据

### 3. 推荐路线图

#### Phase 1: 参数控制（1-2个月）
- 目标: 快速验证概念
- 实现: 风格编码器 → 参数向量
- 交付: 可工作的原型

#### Phase 2: 特征融合（2-3个月）
- 目标: 提升质量
- 实现: 特征图注入材质
- 交付: 高质量风格化

#### Phase 3: 混合渲染（3-6个月）
- 目标: 手绘质感
- 实现: Mesh + 3DGS 混合
- 交付: 电影级质量

---

## HybriToon项目集成方案

### 1. 技术架构

```
┌─────────────────────────────────────────────────────────────┐
│                        HybriToon                             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │   资产层     │    │    AI层      │    │   引擎层     │ │
│  │              │    │              │    │              │ │
│  │ • Blender    │    │ • Python 3.12│    │ • UE5.5.4   │ │
│  │ • VRM4U      │───▶│ • PyTorch 2.7│───▶│ • MooaToon   │ │
│  │ 导入         │    │ • ONNX导出   │    │ • NNE推理    │ │
│  │              │    │              │    │              │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 2. 开发环境配置

#### 2.1 Python环境

**使用 Anaconda PyTorch 环境**: 是的，强烈推荐使用！

**requirements.txt**:
```
torch>=2.7.0
torchvision>=0.22.0
onnx>=1.16.0
onnxruntime>=1.18.0
opencv-python>=4.9.0
numpy>=1.26.0
pillow>=10.3.0
scikit-learn>=1.4.0
matplotlib>=3.8.0
tqdm>=4.66.0
```

#### 2.2 环境激活

```powershell
# 激活Anaconda环境
conda activate your_pytorch_env

# 验证安装
python -c "import torch; print(f'PyTorch: {torch.__version__}')"
python -c "import onnx; print(f'ONNX: {onnx.__version__}')"
```

### 3. 详细开发计划

#### Day 1-7: 第一周 - 基础搭建
- ✅ Day 1: 研究 MooaToon 材质参数，设计参数向量结构
- Day 2: 搭建 PyTorch 训练框架，准备数据集
- Day 3: 实现简单的风格编码器（CNN）
- Day 4: 实现 ONNX 导出逻辑
- Day 5: 研究 UE5 NNE 插件
- Day 6: 实现 UE5 端参数应用
- Day 7: 集成测试，端到端跑通

#### Day 8-14: 第二周 - 模型优化
- Day 8: 收集风格-参数配对数据集
- Day 9: 训练风格编码器，调优超参数
- Day 10: 实现数据增强，提升泛化能力
- Day 11: 添加风格插值功能
- Day 12: 性能优化（模型量化、剪枝）
- Day 13: 批量测试多种风格
- Day 14: 第二周总结与演示

#### Day 15-21: 第三周 - UE集成与优化
- Day 15: 实现 UE5 编辑器工具（风格选择面板）
- Day 16: 实现实时参数调节与预览
- Day 17: 集成 VRM 支持，测试 VRoid 模型
- Day 18: 优化材质参数应用逻辑
- Day 19: 添加风格预设保存/加载
- Day 20: 性能测试与优化
- Day 21: 第三周总结，制作演示视频

### 4. 大创项目建议

#### 项目名称
**HybriToon: AI 增强的实时三渲二渲染系统**

#### 项目亮点
1. **技术创新**: 将神经渲染与传统三渲二结合
2. **实用价值**: 美术师可以一键风格化，大幅提升效率
3. **完整度**: 从训练到推理的完整 Pipeline
4. **演示效果**: 风格转换叙事游戏，视觉冲击力强

#### 演示内容建议
制作一个简短的风格转换叙事游戏：
- 场景: 现代城市 → 日式动画 → 水彩画 → 赛博朋克
- 角色: VRM 角色在不同风格中互动
- 交互: 玩家可以随时切换风格

---

## 总结

### MooaToon 核心价值
1. **解决硬边问题**: 平滑法线技术是三渲二的关键
2. **UE5 原生集成**: 充分利用 Lumen、Nanite 等新技术
3. **灵活材质系统**: Material Layer 支持自由组合
4. **完整工具链**: 从法线处理到后期效果的完整解决方案

### HybriToon 机会
1. **低耦合起步**: 先做参数控制，快速验证
2. **渐进式增强**: 逐步加入神经特征，提升质量
3. **3DGS 未来**: 长期目标是 Mesh + 3DGS 混合渲染
4. **大创优势**: 技术难度适中，演示效果出色

### 关键文件速查
| 功能 | 文件路径 |
|------|---------|
| 平滑法线核心 | `MooaToonScripts/Source/MooaToonEditorScripts/Private/SmoothNormalCommand.cpp` |
| 蓝图函数库 | `MooaToonScripts/Source/MooaToonEditorScripts/Private/MooaToonEditorBPLibrary.cpp` |
| API 头文件 | `MooaToonScripts/Source/MooaToonEditorScripts/Public/MooaToonEditorBPLibrary.h` |
| 参数向量设计 | `HybriToon/material_params.py` |

---

## 参考资料
- MooaToon GitHub: https://github.com/JasonMa0012/MooaToon
- MooaToon 文档: https://mooatoon.com/docs/
- VRM4U: https://github.com/ruyo/VRM4U
- MToon: https://github.com/Santarh/MToon

---

**报告生成时间**: 2026年4月
**报告版本**: v1.0
