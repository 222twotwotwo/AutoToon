# SceneTypes.h - 材质属性枚举详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Source/Runtime/Engine/Public/SceneTypes.h` |
| **核心功能** | 定义材质属性枚举（EMaterialProperty） |
| **MooaToon修改** | 新增 `MP_MooaEncodedAttribute0-4` |
| **修改位置** | 第 182-188 行 |

---

## 源码分析

### 完整的 EMaterialProperty 枚举（节选）

**文件位置**: `SceneTypes.h:147-194`

```cpp
UENUM(BlueprintType)
enum EMaterialProperty : int
{
    // UE5 原生材质属性（节选）
    MP_EmissiveColor = 0        UMETA(DisplayName = "Emissive"),
    MP_Opacity                   UMETA(DisplayName = "Opacity"),
    MP_OpacityMask               UMETA(DisplayName = "Opacity Mask"),
    MP_BaseColor                 UMETA(DisplayName = "Diffuse"),
    MP_Metallic                  UMETA(DisplayName = "Metallic"),
    MP_Specular                  UMETA(DisplayName = "Specular"),
    MP_Roughness                 UMETA(DisplayName = "Roughness "),
    MP_Anisotropy                UMETA(DisplayName = "Anisotropy"),
    MP_Normal                    UMETA(DisplayName = "Normal"),
    MP_Tangent                   UMETA(DisplayName = "Tangent"),
    MP_SubsurfaceColor           UMETA(DisplayName = "Subsurface"),
    MP_AmbientOcclusion          UMETA(DisplayName = "Ambient Occlusion"),
    MP_Refraction                UMETA(DisplayName = "Refraction"),
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // MooaToon 新增材质属性
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // Mooa Material Editor
    MP_MooaEncodedAttribute0 UMETA(DisplayName = "Mooa Encoded Attribute 0"),
    MP_MooaEncodedAttribute1 UMETA(DisplayName = "Mooa Encoded Attribute 1"),
    MP_MooaEncodedAttribute2 UMETA(DisplayName = "Mooa Encoded Attribute 2"),
    MP_MooaEncodedAttribute3 UMETA(DisplayName = "Mooa Encoded Attribute 3"),
    MP_MooaEncodedAttribute4 UMETA(DisplayName = "Mooa Encoded Attribute 4"),
    // Mooa End
    
    // 系统属性
    MP_MaterialAttributes UMETA(Hidden),
    MP_CustomOutput UMETA(Hidden),
    MP_MAX UMETA(DisplayName = "None"),
};
```

---

## MooaToon 修改详解

### 新增的 5 个材质属性

| 属性 | DisplayName | 索引 | 类型 |
|------|-------------|------|------|
| **MP_MooaEncodedAttribute0** | Mooa Encoded Attribute 0 | 新增 | Float4 |
| **MP_MooaEncodedAttribute1** | Mooa Encoded Attribute 1 | 新增 | Float4 |
| **MP_MooaEncodedAttribute2** | Mooa Encoded Attribute 2 | 新增 | Float4 |
| **MP_MooaEncodedAttribute3** | Mooa Encoded Attribute 3 | 新增 | Float4 |
| **MP_MooaEncodedAttribute4** | Mooa Encoded Attribute 4 | 新增 | Float4 |

---

### 为什么需要 5 个 Float4 属性？

让我们回忆一下 `FToonGBufferData` 的大小：

| FToonGBufferData 字段 | 大小 | 累加 |
|----------------------|------|------|
| ShadingFeatureID | 1 float | 1 |
| MainLightShadowColor | 3 floats | 4 |
| DiffuseColorRampIndex | 1 float | 5 |
| DiffuseColorRampUVOffset | 1 float | 6 |
| SpecularColor | 3 floats | 9 |
| SpecularColorRampIndex | 1 float | 10 |
| SpecularColorRampUVOffset | 1 float | 11 |
| ReflectionIntensity | 1 float | 12 |
| RimLightIntensity | 1 float | 13 |
| RimLightWidth | 1 float | 14 |
| FacialShadowSdfLeft | 1 float | 15 |
| FacialShadowSdfRight | 1 float | 16 |
| Stencil | 1 float | 17 |
| RayTracingShadowFlag | 1 float | **18 floats = 4.5 Float4** |

**答案**：18 个 float = 4.5 个 Float4，所以需要 **5 个 Float4**！

---

## 5 个属性的数据映射

### MooaEncodedAttribute0

| 通道 | 数据 | FToonGBufferData 字段 |
|------|------|----------------------|
| **x** | ShadingFeatureID | `ToonGBuffer.ShadingFeatureID` |
| **y** | MainLightShadowColor.r | `ToonGBuffer.MainLightShadowColor.r` |
| **z** | MainLightShadowColor.g | `ToonGBuffer.MainLightShadowColor.g` |
| **w** | MainLightShadowColor.b | `ToonGBuffer.MainLightShadowColor.b` |

### MooaEncodedAttribute1

| 通道 | 数据 | FToonGBufferData 字段 |
|------|------|----------------------|
| **x** | DiffuseColorRampIndex | `ToonGBuffer.DiffuseColorRampIndex` |
| **y** | DiffuseColorRampUVOffset | `ToonGBuffer.DiffuseColorRampUVOffset` |
| **z** | 0 | 保留 |
| **w** | 0 | 保留 |

### MooaEncodedAttribute2

| 通道 | 数据 | FToonGBufferData 字段 |
|------|------|----------------------|
| **x** | SpecularColor.r | `ToonGBuffer.SpecularColor.r` |
| **y** | SpecularColor.g | `ToonGBuffer.SpecularColor.g` |
| **z** | SpecularColor.b | `ToonGBuffer.SpecularColor.b` |
| **w** | SpecularColorRampIndex | `ToonGBuffer.SpecularColorRampIndex` |

### MooaEncodedAttribute3

| 通道 | 数据 | FToonGBufferData 字段 |
|------|------|----------------------|
| **x** | SpecularColorRampUVOffset | `ToonGBuffer.SpecularColorRampUVOffset` |
| **y** | ReflectionIntensity | `ToonGBuffer.ReflectionIntensity` |
| **z** | RimLightIntensity | `ToonGBuffer.RimLightIntensity` |
| **w** | RimLightWidth | `ToonGBuffer.RimLightWidth` |

### MooaEncodedAttribute4

| 通道 | 数据 | FToonGBufferData 字段 |
|------|------|----------------------|
| **x** | FacialShadowSdfLeft | `ToonGBuffer.FacialShadowSdfLeft` |
| **y** | FacialShadowSdfRight | `ToonGBuffer.FacialShadowSdfRight` |
| **z** | Stencil | `ToonGBuffer.Stencil` |
| **w** | RayTracingShadowFlag | `ToonGBuffer.RayTracingShadowFlag` |

---

## 这个文件在整个系统中的位置

```
SceneTypes.h (定义枚举)
    ↓
MaterialAttributeDefinitionMap.cpp (注册属性元数据)
    ↓
HLSLMaterialTranslator.cpp (编译属性)
    ↓
MaterialTemplate.ush (GetMaterialMooaEncodedAttributeX)
    ↓
BasePassPixelShader.usf (读取属性并编码)
```

---

## 实际使用示例

### 在材质编辑器中使用

1. 打开材质编辑器
2. 在右键菜单搜索「Mooa Encoded Attribute」
3. 你会看到 5 个节点：
   - Mooa Encoded Attribute 0
   - Mooa Encoded Attribute 1
   - ...
   - Mooa Encoded Attribute 4
4. 连接你的逻辑到这些节点

### 在 C++ 代码中使用

```cpp
// 在 HLSLMaterialTranslator.cpp 中
if (MaterialShadingModels.HasShadingModel(MSM_Toon))
{
    Chunk[MP_MooaEncodedAttribute0] = 
        Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute0, this);
    Chunk[MP_MooaEncodedAttribute1] = 
        Material->CompilePropertyAndSetMaterialProperty(MP_MooaEncodedAttribute1, this);
    // ... 其他属性
}
```

### 在着色器中使用

```hlsl
// 在 MaterialTemplate.ush 中
float4 GetMaterialMooaEncodedAttribute0(FPixelMaterialInputs PixelMaterialInputs)
{
    return PixelMaterialInputs.MooaEncodedAttribute0;
}

// 在 BasePassPixelShader.usf 中
EncodeToonGBufferDataToMRT(
    GBuffer.MooaToonContext.ToonGBuffer,
    GetMaterialMooaEncodedAttribute0(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute1(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute2(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute3(PixelMaterialInputs),
    GetMaterialMooaEncodedAttribute4(PixelMaterialInputs));
```

---

## 相关文件索引

| 文件 | 作用 |
|------|------|
| `MaterialAttributeDefinitionMap.cpp:280-284` | 注册属性元数据 |
| `HLSLMaterialTranslator.cpp:1632-1639` | 编译属性 |
| `MaterialTemplate.ush:3686-3708` | GetMaterialMooaEncodedAttributeX 函数 |
| `BasePassPixelShader.usf:1083-1087` | 读取并编码属性 |
| `ToonShadingCommon.ush:242-286` | 解码属性到 FToonGBufferData |

---

## HybriToon 扩展建议

### 方案A：复用现有的 5 个属性（推荐）

**优点**：
- 不需要修改引擎
- 风险最低
- 完全兼容

**方法**：
- 通过 ShadingFeatureID 区分风格
- 复用现有通道存储神经特征
- 不需要新增属性

---

### 方案B：新增 MooaEncodedAttribute5-9

**如果方案A空间不够，可以新增**：

```cpp
// 在 SceneTypes.h 中新增
MP_MooaEncodedAttribute5 UMETA(DisplayName = "Mooa Encoded Attribute 5"),
MP_MooaEncodedAttribute6 UMETA(DisplayName = "Mooa Encoded Attribute 6"),
MP_MooaEncodedAttribute7 UMETA(DisplayName = "Mooa Encoded Attribute 7"),
MP_MooaEncodedAttribute8 UMETA(DisplayName = "Mooa Encoded Attribute 8"),
MP_MooaEncodedAttribute9 UMETA(DisplayName = "Mooa Encoded Attribute 9"),
```

**同时需要修改**：
- `MaterialAttributeDefinitionMap.cpp` - 注册新属性
- `HLSLMaterialTranslator.cpp` - 编译新属性
- `MaterialTemplate.ush` - 新增 GetMaterialMooaEncodedAttribute5-9
- `BasePassPixelShader.usf` - 读取新属性
- `ToonShadingCommon.ush` - 解码新属性

---

## 总结

### 关键点

1. **MP_MooaEncodedAttribute0-4** 是 MooaToon 新增的 5 个 Float4 材质属性
2. 它们是**艺术家（TA）和渲染管线之间的接口**
3. 5 个 Float4 = 20 个 float，可以存下完整的 FToonGBufferData（18 个 float）
4. HybriToon 建议优先复用现有属性

### 记忆要点

- ✅ 5 个 Float4，不是 1 个
- ✅ 这是与材质编辑器的接口
- ✅ 数据流向：材质编辑器 → 属性 → FToonGBufferData → ToonBufferA
- ✅ HybriToon 从这里注入神经特征

---

**文档版本**: v1.0  
**分析深度**: 源码级  
**最后更新**: 2026年4月6日
