# ToonShadingCommon.ush - GBuffer 编码解码详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Shaders/Private/ToonShadingCommon.ush` |
| **核心功能** | Toon GBuffer 数据结构、编码/解码函数、数学工具函数 |
| **重要性** | ⭐⭐⭐⭐⭐ 核心文件 |

---

## 写给零基础开发者

### 这个文件是做什么的？

**想象一下**：你要把很多信息打包寄给朋友，但是信封大小有限。

这个文件就是负责：
1. **打包（编码）** - 把 Toon 渲染需要的 18 个参数，压缩进 4 个 GBuffer 通道
2. **拆包（解码）** - 从 GBuffer 中读取数据，还原成原来的 18 个参数

---

## 文件结构总览

这个文件分为 7 个主要部分：

| 部分 | 行号 | 内容 |
|------|------|------|
| 1 | 9-71 | 宏定义（世界类型、特性ID等） |
| 2 | 74-141 | 数学工具函数 |
| 3 | 144-207 | 编码/解码辅助函数 |
| 4 | 210-236 | FToonGBufferData 结构体 |
| 5 | 239-288 | 材质属性 ↔ FToonGBufferData |
| 6 | 291-372 | FToonGBufferData ↔ GBuffer |
| 7 | 375-443 | FMooaToonContext 和其他 |

---

## 第一部分：宏定义

### 1.1 世界类型（World Type）

**行号**: 22-44

```hlsl
// 不同的世界类型
#define MOOA_WORLD_TYPE_NONE             0
#define MOOA_WORLD_TYPE_GAME             1
#define MOOA_WORLD_TYPE_Editor           2
#define MOOA_WORLD_TYPE_PIE              3
#define MOOA_WORLD_TYPE_Editor_Preview   4
#define MOOA_WORLD_TYPE_Game_Preview     5
#define MOOA_WORLD_TYPE_Game_RPC         6
#define MOOA_WORLD_TYPE_Inactive         7
```

**这是做什么的？**
- 区分是在游戏中、编辑器中、还是预览窗口中
- 不同场景可能需要不同的渲染逻辑

---

### 1.2 着色特性ID（Shading Feature ID）

**行号**: 48-53

```hlsl
#define MOOA_SHADING_FEATURE_ID_DEFAULT                         0
#define MOOA_SHADING_FEATURE_ID_PBR_SPECULAR                   1
// 需要切线：
#define MOOA_SHADING_FEATURE_ID_KAJIYA_HAIR_SPECULAR           2
#define MOOA_SHADING_FEATURE_ID_DISTANCE_FIELD_FACIAL_SHADOW    3
```

**这是做什么的？**
- 告诉 Toon 着色器用哪种模式渲染
- 4 种模式：

| ID | 模式 | 说明 |
|----|------|------|
| 0 | DEFAULT | 默认模式 |
| 1 | PBR_SPECULAR | PBR 高光模式 |
| 2 | KAJIYA_HAIR_SPECULAR | 头发高光（需要切线） |
| 3 | DISTANCE_FIELD_FACIAL_SHADOW | 面部阴影（需要 SDF） |

---

### 1.3 光线追踪阴影标志

**行号**: 57-62

```hlsl
#define MOOA_RAY_TRACING_SHADOW_FLAG_NONE                        0
#define MOOA_RAY_TRACING_SHADOW_FLAG_FACE                        1
#define MOOA_RAY_TRACING_SHADOW_FLAG_FACE_SCREEN_SPACE_HAIR_SHADOW  2
#define MOOA_RAY_TRACING_SHADOW_FLAG_HAIR                         3
```

---

### 1.4 光源类型

**行号**: 66-70

```hlsl
#define MOOA_LIGHT_TYPE_DIRECTIONAL    0  // 平行光（太阳光）
#define MOOA_LIGHT_TYPE_POINT          1  // 点光源
#define MOOA_LIGHT_TYPE_SPOT           2  // 聚光灯
#define MOOA_LIGHT_TYPE_RECT           3  // 矩形光
```

---

## 第二部分：数学工具函数

### 2.1 ToonStep（卡通阶跃函数）

**行号**: 80-88

```hlsl
float ToonStep(
    float Gradient,           // 输入值（通常是 NoL）
    float Feather = 0.0f,      // 羽化值
    float Threshold = 0.5f,     // 阈值
    float FeatherPower = 2.0f,   // 羽化指数
    bool UseSaturate = true,    // 是否限制在 0-1
    bool UseSmoothstep = true   // 是否用 smoothstep
)
{
    Feather = pow(saturate(Feather), FeatherPower);
    float Min = UseSaturate ? saturate(Threshold - Feather) : (Threshold - Feather);
    float Max = UseSaturate ? saturate(Threshold + Feather) : (Threshold + Feather);
    
    return UseSmoothstep
        ? smoothstep(Min, Max, Gradient)
        : InverseLerp(Min, Max, Gradient);
}
```

**这是做什么的？**
- 把平滑的渐变变成卡通风格的阶跃
- 例如：NoL 从 0 到 1 平滑变化 → 变成「暗 → 突然变亮」

**用大白话解释**：
- 想象一个调光开关，从 0 慢慢拧到 100%
- ToonStep 会让它变成：0% → 突然跳到 50% → 突然跳到 100%

---

### 2.2 其他数学工具

**行号**: 90-141

```hlsl
// 重新映射范围
float Remap(float value, float from1, float to1, float from2, float to2)
{
    return (value - from1) / (to1 - from1) * (to2 - from2) + from2;
}

// 快速幂运算
float Pow12(float x)
{
    float x6 = Pow6(x); 
    return x6 * x6;
}

// 快速反余弦（用于角度计算）
float FastACos(float inX)
{
    float res = FastACosPos(inX);
    return (inX >= 0) ? res : PI - res;
}

// 获取曝光
float GetMooaExposure()
{
    return View.PreExposure * View.MooaExposureScale;
}
```

---

## 第三部分：编码/解码辅助函数

### 3.1 为什么需要编码？

**问题**：
- 我们有 18 个 float 要存
- 但 GBuffer 空间有限

**解决方案**：
- 把多个小数字压缩进一个 float
- 例如：把两个 0-1 的 float，各用 8 位，存进一个 float

---

### 3.2 核心编码函数

#### EncodeFloat2ToFloat（两个 float 编码成一个）

**行号**: 183-186

```hlsl
float EncodeFloat2ToFloat(
    float Src1,    // 第一个 float（0-1）
    float Src2,    // 第二个 float（0-1）
    int BitsSrc1 = 8,  // 第一个用几位
    int BitsSrc2 = 8   // 第二个用几位
)
{
    return EncodeUint2ToFloat(
        EncodeFloatToUint(Src1, BitsSrc1), 
        EncodeFloatToUint(Src2, BitsSrc2), 
        BitsSrc1, BitsSrc2);
}
```

**用大白话解释**：
- 想象你有两个 0-255 的数字：100 和 200
- 把它们拼成一个数字：100200
- 然后除以 65535（2^16-1），变成 0-1 范围
- 这样就能存进一个 float 了！

**示例**：
```hlsl
// 把 SpecularColor.r 和 SpecularColor.g 编码进一个 float
EncodedToonBufferA.x = EncodeFloat2ToFloat(
    ToonGBuffer.SpecularColor.r,  // 8 位
    ToonGBuffer.SpecularColor.g   // 8 位
);
```

---

### 3.3 核心解码函数

#### DecodeFloat2FromFloat（从一个 float 解码出两个）

**行号**: 201-207

```hlsl
void DecodeFloat2FromFloat(
    float Src,           // 编码后的 float
    out float Dst1,      // 输出第一个 float
    out float Dst2,      // 输出第二个 float
    int BitsDst1 = 8,    // 第一个用几位
    int BitsDst2 = 8,    // 第二个用几位
    float bias = 0       // 偏置（用于精度修正）
)
{
    uint OutUint1, OutUint2;
    DecodeUint2FromFloat(Src, OutUint1, OutUint2, BitsDst1, BitsDst2, bias);
    Dst1 = DecodeFloatFromUint(OutUint1, BitsDst1);
    Dst2 = DecodeFloatFromUint(OutUint2, BitsDst2);
}
```

**用大白话解释**：
- 从 100200 这个数字
- 拆回 100 和 200
- 然后分别除以 255，变回 0-1 范围

---

## 第四部分：FToonGBufferData 结构体

### 4.1 完整的结构体定义

**行号**: 210-236

```hlsl
struct FToonGBufferData
{
    uint ShadingFeatureID;  // 着色特性ID（0-3）
    
    // 漫反射相关
    float3 MainLightShadowColor;    // 主光源阴影颜色（RGB）
    uint DiffuseColorRampIndex;      // 漫反射Ramp索引
    float DiffuseColorRampUVOffset;  // 漫反射Ramp UV偏移
    
    // 高光相关
    float3 SpecularColor;            // 高光颜色（RGB）
    uint SpecularColorRampIndex;     // 高光Ramp索引
    float SpecularColorRampUVOffset; // 高光Ramp UV偏移
    float ReflectionIntensity;        // 反射强度
    float RimLightIntensity;          // 边缘光强度
    float RimLightWidth;              // 边缘光宽度
    
    // 距离场面部阴影
    float FacialShadowSdfLeft;        // 面部阴影SDF（左）
    float FacialShadowSdfRight;       // 面部阴影SDF（右）
    
    // 光线追踪阴影
    uint Stencil;                     // 模板值
    uint RayTracingShadowFlag;        // 光线追踪阴影标志
};
```

### 4.2 字段统计

| 类型 | 数量 | 占用 |
|------|------|------|
| float3 | 2 | 6 floats |
| float | 6 | 6 floats |
| uint | 4 | 4 floats（当作float用） |
| **总计** | | **18 floats** |

**这就是为什么我们需要 5 个 Float4 的材质属性！**
- 18 floats = 4.5 Float4
- 向上取整 = 5 Float4

---

## 第五部分：材质属性 ↔ FToonGBufferData

### 5.1 编码：FToonGBufferData → 材质属性

**行号**: 239-263

```hlsl
void EncodeToonGBufferDataToMaterialAttribute(
    FToonGBufferData ToonGBuffer,
    out float4 MooaEncodedAttribute0,
    out float4 MooaEncodedAttribute1,
    out float4 MooaEncodedAttribute2,
    out float4 MooaEncodedAttribute3,
    out float4 MooaEncodedAttribute4)
{
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // MooaEncodedAttribute0
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    MooaEncodedAttribute0.x   = ToonGBuffer.ShadingFeatureID;
    MooaEncodedAttribute0.yzw = ToonGBuffer.MainLightShadowColor;
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // MooaEncodedAttribute1
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    MooaEncodedAttribute1.x  = ToonGBuffer.DiffuseColorRampIndex;
    MooaEncodedAttribute1.y  = ToonGBuffer.DiffuseColorRampUVOffset;
    MooaEncodedAttribute1.zw = 0;  // 保留
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // MooaEncodedAttribute2
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    MooaEncodedAttribute2.xyz = ToonGBuffer.SpecularColor;
    MooaEncodedAttribute2.w   = ToonGBuffer.SpecularColorRampIndex;
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // MooaEncodedAttribute3
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    MooaEncodedAttribute3.x  = ToonGBuffer.SpecularColorRampUVOffset;
    MooaEncodedAttribute3.y  = ToonGBuffer.ReflectionIntensity;
    MooaEncodedAttribute3.z  = ToonGBuffer.RimLightIntensity;
    MooaEncodedAttribute3.w  = ToonGBuffer.RimLightWidth;
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // MooaEncodedAttribute4
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    MooaEncodedAttribute4.x  = ToonGBuffer.FacialShadowSdfLeft;
    MooaEncodedAttribute4.y  = ToonGBuffer.FacialShadowSdfRight;
    MooaEncodedAttribute4.z  = ToonGBuffer.Stencil;
    MooaEncodedAttribute4.w  = ToonGBuffer.RayTracingShadowFlag;
}
```

### 5.2 详细映射表

| 材质属性 | 通道 | FToonGBufferData 字段 |
|---------|------|----------------------|
| **MooaEncodedAttribute0** | x | ShadingFeatureID |
| | yzw | MainLightShadowColor |
| **MooaEncodedAttribute1** | x | DiffuseColorRampIndex |
| | y | DiffuseColorRampUVOffset |
| **MooaEncodedAttribute2** | xyz | SpecularColor |
| | w | SpecularColorRampIndex |
| **MooaEncodedAttribute3** | x | SpecularColorRampUVOffset |
| | y | ReflectionIntensity |
| | z | RimLightIntensity |
| | w | RimLightWidth |
| **MooaEncodedAttribute4** | x | FacialShadowSdfLeft |
| | y | FacialShadowSdfRight |
| | z | Stencil |
| | w | RayTracingShadowFlag |

---

### 5.3 解码：材质属性 → FToonGBufferData

**行号**: 265-288

```hlsl
FToonGBufferData DecodeToonGBufferDataFromMaterialAttribute(
    float4 MooaEncodedAttribute0,
    float4 MooaEncodedAttribute1,
    float4 MooaEncodedAttribute2,
    float4 MooaEncodedAttribute3,
    float4 MooaEncodedAttribute4)
{
    FToonGBufferData ToonGBuffer;
    
    // MooaEncodedAttribute0
    ToonGBuffer.ShadingFeatureID      = MooaEncodedAttribute0.x;
    ToonGBuffer.MainLightShadowColor   = MooaEncodedAttribute0.yzw;
    
    // MooaEncodedAttribute1
    ToonGBuffer.DiffuseColorRampIndex   = MooaEncodedAttribute1.x;
    ToonGBuffer.DiffuseColorRampUVOffset = MooaEncodedAttribute1.y;
    
    // MooaEncodedAttribute2
    ToonGBuffer.SpecularColor           = MooaEncodedAttribute2.xyz;
    ToonGBuffer.SpecularColorRampIndex  = MooaEncodedAttribute2.w;
    
    // MooaEncodedAttribute3
    ToonGBuffer.SpecularColorRampUVOffset = MooaEncodedAttribute3.x;
    ToonGBuffer.ReflectionIntensity        = MooaEncodedAttribute3.y;
    ToonGBuffer.RimLightIntensity          = MooaEncodedAttribute3.z;
    ToonGBuffer.RimLightWidth              = MooaEncodedAttribute3.w;
    
    // MooaEncodedAttribute4
    ToonGBuffer.FacialShadowSdfLeft        = MooaEncodedAttribute4.x;
    ToonGBuffer.FacialShadowSdfRight       = MooaEncodedAttribute4.y;
    ToonGBuffer.Stencil                     = MooaEncodedAttribute4.z;
    ToonGBuffer.RayTracingShadowFlag       = MooaEncodedAttribute4.w;
    
    return ToonGBuffer;
}
```

---

## 第六部分：FToonGBufferData ↔ GBuffer

### 6.1 GBuffer 存储布局（重要！）

**行号**: 293-308

这段注释详细说明了数据是如何存进 GBuffer 的：

```
ToonBufferA (RGBA Float 16)
  x: SpecularColor.r(8)      SpecularColor.g(8)
  y: SpecularColor.b(8)      SpecularColorRampUVOffset(8)
  z: DiffuseColorRampUVOffset(8)  RimLightIntensity(4)  RimLightWidth(4)
  w: DiffuseColorRampIndex(6)  SpecularColorRampIndex(5)  Stencil(5)

CustomData (GBufferD) (RGBA Float 8)
  x: MainLightShadowColor.r(8)
  y: MainLightShadowColor.g(8)
  z: MainLightShadowColor.b(8)
  w: ReflectionIntensity(4)  ShadingFeatureID(2)  RayTracingShadowFlag(2)

Metallic:    FacialShadowSdfLeft(8)
Anisotropy:  FacialShadowSdfRight(8)
```

**用大白话解释**：
- 想象你有一个 4 抽屉的柜子（ToonBufferA）
- 每个抽屉可以放 2-3 个小盒子
- 另外还要借用邻居家的 3 个抽屉（CustomData、Metallic、Anisotropy）

---

### 6.2 编码：FToonGBufferData → GBuffer

**行号**: 309-335

```hlsl
void EncodeToonGBufferDataToMRT(
    FToonGBufferData ToonGBuffer,
    out float4 EncodedToonBufferA,
    out float4 CustomData,
    inout float Metallic)
{
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // ToonBufferA.x
    // SpecularColor.r(8) + SpecularColor.g(8)
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    EncodedToonBufferA.x = EncodeFloat2ToFloat(
        ToonGBuffer.SpecularColor.r, 
        ToonGBuffer.SpecularColor.g
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // ToonBufferA.y
    // SpecularColor.b(8) + SpecularColorRampUVOffset(8)
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    EncodedToonBufferA.y = EncodeFloat2ToFloat(
        ToonGBuffer.SpecularColor.b, 
        ToonGBuffer.SpecularColorRampUVOffset
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // ToonBufferA.z
    // DiffuseColorRampUVOffset(8) + [RimLightIntensity(4) + RimLightWidth(4)]
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    EncodedToonBufferA.z = EncodeFloat2ToFloat(
        ToonGBuffer.DiffuseColorRampUVOffset,
        EncodeFloat2ToFloat(
            ToonGBuffer.RimLightIntensity, 
            ToonGBuffer.RimLightWidth, 
            4, 4  // 各用 4 位
        ),
        8, 8
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // ToonBufferA.w
    // [DiffuseColorRampIndex(6) + SpecularColorRampIndex(5)] + Stencil(5)
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    EncodedToonBufferA.w = EncodeUint2ToFloat(
        EncodeUint2ToUint(
            ToonGBuffer.DiffuseColorRampIndex, 
            ToonGBuffer.SpecularColorRampIndex, 
            6, 5
        ),
        ToonGBuffer.Stencil,
        11, 5
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // CustomData (GBufferD)
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    CustomData.xyz = saturate(ToonGBuffer.MainLightShadowColor.rgb);
    CustomData.w = EncodeUint2ToFloat(
        EncodeFloatToUint(ToonGBuffer.ReflectionIntensity, 4),
        EncodeUint2ToUint(
            ToonGBuffer.ShadingFeatureID, 
            ToonGBuffer.RayTracingShadowFlag, 
            2, 2
        ),
        4, 4
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // Metallic（面部阴影模式时借用）
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    BRANCH if (ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_DISTANCE_FIELD_FACIAL_SHADOW)
    {
        Metallic = ToonGBuffer.FacialShadowSdfLeft;
    }
}
```

---

### 6.3 解码：GBuffer → FToonGBufferData

**行号**: 346-372

```hlsl
FToonGBufferData DecodeToonGBufferDataFromMRT(
    float4 EncodedToonBufferA,
    float4 CustomData,
    inout float Metallic,
    inout float EncodedAnisotropy)
{
    FToonGBufferData ToonGBuffer = (FToonGBufferData)0;
    uint Out0, Out1, Out2;
    float Out3;
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // ToonBufferA.x → SpecularColor.r + SpecularColor.g
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    DecodeFloat2FromFloat(
        EncodedToonBufferA.x, 
        ToonGBuffer.SpecularColor.r, 
        ToonGBuffer.SpecularColor.g, 
        8, 8, 0.5f  // 0.5 是偏置，用于精度修正
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // ToonBufferA.y → SpecularColor.b + SpecularColorRampUVOffset
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    DecodeFloat2FromFloat(
        EncodedToonBufferA.y, 
        ToonGBuffer.SpecularColor.b, 
        ToonGBuffer.SpecularColorRampUVOffset, 
        8, 8, 0.5f
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // ToonBufferA.z → DiffuseColorRampUVOffset + [RimLightIntensity + RimLightWidth]
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    DecodeFloat2FromFloat(
        EncodedToonBufferA.z, 
        ToonGBuffer.DiffuseColorRampUVOffset, 
        Out3, 
        8, 8, 0.5f
    );
    DecodeFloat2FromFloat(
        Out3, 
        ToonGBuffer.RimLightIntensity, 
        ToonGBuffer.RimLightWidth, 
        4, 4, 0.5f
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // ToonBufferA.w → [DiffuseColorRampIndex + SpecularColorRampIndex] + Stencil
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    DecodeUint2FromFloat(
        EncodedToonBufferA.w, 
        Out0, 
        ToonGBuffer.Stencil, 
        11, 5, 0.5f
    );
    DecodeUint2FromUint(
        Out0, 
        ToonGBuffer.DiffuseColorRampIndex, 
        ToonGBuffer.SpecularColorRampIndex, 
        6, 5
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // CustomData → MainLightShadowColor + ReflectionIntensity + ...
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    ToonGBuffer.MainLightShadowColor = saturate(CustomData.xyz);
    DecodeUint2FromFloat(CustomData.w, Out0, Out1, 4, 4, 0.5f);
    ToonGBuffer.ReflectionIntensity = DecodeFloatFromUint(Out0, 4);
    DecodeUint2FromUint(
        Out1, 
        ToonGBuffer.ShadingFeatureID, 
        ToonGBuffer.RayTracingShadowFlag, 
        2, 2
    );
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // Metallic/Anisotropy → FacialShadowSdf（面部阴影模式时）
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    BRANCH if (ToonGBuffer.ShadingFeatureID == MOOA_SHADING_FEATURE_ID_DISTANCE_FIELD_FACIAL_SHADOW)
    {
        ToonGBuffer.FacialShadowSdfLeft = Metallic;
        ToonGBuffer.FacialShadowSdfRight = EncodedAnisotropy;
        Metallic = 0.0f;
        EncodedAnisotropy = 0.5f;
    }
    
    return ToonGBuffer;
}
```

---

## 第七部分：其他结构

### 7.1 FMooaToonContext

**行号**: 375-396

```hlsl
struct FMooaToonContext
{
    FToonGBufferData ToonGBuffer;
    
    float4 EncodedToonBufferA;  // BasePass 输出到 ToonBufferA
    
    uint LightType;
    float3 LightColor;
    bool IsEditorPreviewWorldType;
    float Exposure;
    
    uint2 PixelPos;  // SVPos.xy
};
```

---

## 完整数据流总结

```
材质编辑器
    ↓ (MooaEncodedAttribute0-4)
DecodeToonGBufferDataFromMaterialAttribute()
    ↓
FToonGBufferData
    ↓
EncodeToonGBufferDataToMRT()
    ↓
GBuffer (ToonBufferA + CustomData + Metallic + Anisotropy)
    ↓
DecodeToonGBufferDataFromMRT()
    ↓
FToonGBufferData
    ↓
ToonBxDF()
    ↓
最终渲染
```

---

## 总结

### 关键点

1. **FToonGBufferData** 有 18 个字段，需要 5 个 Float4 存储
2. **ToonBufferA** 是新增的 RGBA Float16 纹理
3. 还要**借用** CustomData、Metallic、Anisotropy 通道
4. 编码/解码函数负责数据的打包和拆包

### 记忆要点

- ✅ 18 个字段 = 5 个 Float4
- ✅ ToonBufferA + 3 个借用通道 = 完整存储
- ✅ Encode... 函数 = 打包
- ✅ Decode... 函数 = 拆包

---

**文档版本**: v1.0  
**分析深度**: 源码级（逐行解释）  
**目标读者**: 零基础开发者  
**最后更新**: 2026年4月6日
