# MaterialAttributeDefinitionMap.cpp - 材质属性注册详解

## 文件信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `Engine/Source/Runtime/Engine/Private/Materials/MaterialAttributeDefinitionMap.cpp` |
| **核心功能** | 注册材质属性的元数据 |
| **MooaToon修改** | 注册 MP_MooaEncodedAttribute0-4 |
| **修改位置** | 第 279-285 行 |

---

## 源码分析

### InitializeAttributeMap 函数

**文件位置**: `MaterialAttributeDefinitionMap.cpp:258-330`

这个函数在引擎启动时调用，用来初始化所有材质属性的定义。

```cpp
void FMaterialAttributeDefinitionMap::InitializeAttributeMap()
{
    check(!bIsInitialized);
    bIsInitialized = true;
    const bool bHideAttribute = true;
    LLM_SCOPE(ELLMTag::Materials);

    // All types plus default/missing attribute
    AttributeMap.Empty(MP_MAX + 1);

    // Basic attributes (UE5 原生)
    Add(FGuid(0x69B8D336, 0x16ED4D49, 0x9AA49729, 0x2F050F7A), 
        TEXT("BaseColor"), MP_BaseColor, MCT_Float3, FVector4(0,0,0,0), SF_Pixel);
    
    Add(FGuid(0x57C3A161, 0x7F064296, 0xB00B24A5, 0xA496F34C), 
        TEXT("Metallic"), MP_Metallic, MCT_Float, FVector4(0,0,0,0), SF_Pixel);
    
    // ... 更多原生属性 ...

    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // MooaToon 新增材质属性注册
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // Mooa Material Editor
    Add(FGuid(0xC1A0EA03, 0x15C74A55, 0xB852BAD5, 0x9A20789A), 
        TEXT("MooaEncodedAttribute0"), MP_MooaEncodedAttribute0, 
        MCT_Float4, FVector4(0,0,0,0), SF_Pixel);
    
    Add(FGuid(0xC6A0AB14, 0x15D75A65, 0xB132ABD4, 0x0B20789C), 
        TEXT("MooaEncodedAttribute1"), MP_MooaEncodedAttribute1, 
        MCT_Float4, FVector4(0,0,0,0), SF_Pixel);
    
    Add(FGuid(0xC7A1EA12, 0x23C54A54, 0xA851BCD1, 0x5A2C784C), 
        TEXT("MooaEncodedAttribute2"), MP_MooaEncodedAttribute2, 
        MCT_Float4, FVector4(0,0,0,0), SF_Pixel);
    
    Add(FGuid(0xC1C0BC01, 0x1AC73C53, 0xB152BAB5, 0x9A21389B), 
        TEXT("MooaEncodedAttribute3"), MP_MooaEncodedAttribute3, 
        MCT_Float4, FVector4(0,0,0,0), SF_Pixel);
    
    Add(FGuid(0xC320DA20, 0x15D75E62, 0xD252BCD7, 0x9A22489D), 
        TEXT("MooaEncodedAttribute4"), MP_MooaEncodedAttribute4, 
        MCT_Float4, FVector4(0,0,0,0), SF_Pixel);
    // Mooa End

    // Advanced attributes (继续原生属性)
    // ...
}
```

---

## Add 函数参数详解

让我们看一下 Add 函数的签名：

```cpp
void Add(
    const FGuid& AttributeID,      // [1] 属性的唯一GUID
    const FString& AttributeName,   // [2] 属性名称
    EMaterialProperty Property,      // [3] 枚举值
    EMaterialValueType ValueType,    // [4] 值类型
    const FVector4& DefaultValue,    // [5] 默认值
    EShaderFrequency ShaderFrequency,// [6] 着色频率
    int32 TexCoordIndex,             // [7] 纹理坐标索引（可选）
    bool bIsHidden,                  // [8] 是否隐藏
    MaterialAttributeBlendFunction BlendFunction // [9] 混合函数（可选）
)
```

---

## MooaToon 属性注册详解

### MP_MooaEncodedAttribute0

| 参数 | 值 | 说明 |
|------|-----|------|
| **AttributeID** | `C1A0EA03-15C74A55-B852BAD5-9A20789A` | 唯一标识符 |
| **AttributeName** | `MooaEncodedAttribute0` | 显示名称 |
| **Property** | `MP_MooaEncodedAttribute0` | 枚举值 |
| **ValueType** | `MCT_Float4` | 4个浮点数 |
| **DefaultValue** | `(0, 0, 0, 0)` | 默认全0 |
| **ShaderFrequency** | `SF_Pixel` | 像素着色器 |

---

### 完整的 5 个属性注册

| 属性 | GUID | ValueType | DefaultValue |
|------|------|-----------|--------------|
| MP_MooaEncodedAttribute0 | C1A0EA03-... | MCT_Float4 | (0,0,0,0) |
| MP_MooaEncodedAttribute1 | C6A0AB14-... | MCT_Float4 | (0,0,0,0) |
| MP_MooaEncodedAttribute2 | C7A1EA12-... | MCT_Float4 | (0,0,0,0) |
| MP_MooaEncodedAttribute3 | C1C0BC01-... | MCT_Float4 | (0,0,0,0) |
| MP_MooaEncodedAttribute4 | C320DA20-... | MCT_Float4 | (0,0,0,0) |

**共同点**：
- 都是 `MCT_Float4`（4个浮点数）
- 默认值都是 `(0, 0, 0, 0)`
- 都是 `SF_Pixel`（像素着色器频率）

---

## EMaterialValueType 枚举

**值类型定义**：

| 值 | 说明 |
|----|------|
| `MCT_Float` | 单个浮点数 |
| `MCT_Float2` | 2个浮点数 |
| `MCT_Float3` | 3个浮点数（通常是颜色/向量） |
| `MCT_Float4` | 4个浮点数 |
| `MCT_ShadingModel` | 着色模型 |
| ... | ... |

---

## EShaderFrequency 枚举

**着色频率定义**：

| 值 | 说明 |
|----|------|
| `SF_Vertex` | 顶点着色器 |
| `SF_Pixel` | 像素着色器 |
| ... | ... |

**注意**：MooaToon 的属性都是 `SF_Pixel`，因为它们在像素着色器中使用。

---

## FGuid 的作用

**为什么需要 GUID？**

1. **唯一性**：每个属性都有全球唯一的标识符
2. **序列化**：保存材质时用 GUID 而不是名字
3. **稳定性**：即使改了名字，GUID 不变，老材质仍然能用

**MooaToon 的 5 个 GUID**：
```
MooaEncodedAttribute0: C1A0EA03-15C74A55-B852BAD5-9A20789A
MooaEncodedAttribute1: C6A0AB14-15D75A65-B132ABD4-0B20789C
MooaEncodedAttribute2: C7A1EA12-23C54A54-A851BCD1-5A2C784C
MooaEncodedAttribute3: C1C0BC01-1AC73C53-B152BAB5-9A21389B
MooaEncodedAttribute4: C320DA20-15D75E62-D252BCD7-9A22489D
```

**⚠️ 重要提示**：如果 HybriToon 要新增属性，必须生成新的唯一 GUID，不要改现有的！

---

## 这个文件在整个系统中的位置

```
MaterialAttributeDefinitionMap.cpp (注册属性)
    ↓
材质编辑器 (显示属性节点)
    ↓
HLSLMaterialTranslator.cpp (编译属性)
    ↓
着色器 (使用属性)
```

---

## HybriToon 扩展建议

### 如果要新增 MP_MooaEncodedAttribute5-9

**步骤1：在 SceneTypes.h 中新增枚举**
```cpp
MP_MooaEncodedAttribute5 UMETA(DisplayName = "Mooa Encoded Attribute 5"),
MP_MooaEncodedAttribute6 UMETA(DisplayName = "Mooa Encoded Attribute 6"),
// ...
```

**步骤2：在 MaterialAttributeDefinitionMap.cpp 中注册**
```cpp
// 生成新的 GUID！不要用现有的！
Add(FGuid(/* 新生成的GUID */), 
    TEXT("MooaEncodedAttribute5"), MP_MooaEncodedAttribute5, 
    MCT_Float4, FVector4(0,0,0,0), SF_Pixel);
```

**步骤3：在其他相关文件中添加支持**
- `HLSLMaterialTranslator.cpp`
- `MaterialTemplate.ush`
- `BasePassPixelShader.usf`
- ...

---

## 如何生成新的 GUID？

### 方法1：用 Visual Studio
1. 打开 Visual Studio
2. 工具 → 创建 GUID
3. 选择「Registry Format」
4. 复制，去掉大括号，格式化为 4 段

### 方法2：用在线工具
搜索「GUID generator」

### 方法3：用 Python
```python
import uuid
print(uuid.uuid4())
# 输出类似: 12345678-1234-5678-1234-567812345678
```

---

## 总结

### 关键点

1. **这个文件注册材质属性的元数据**
2. **每个属性需要一个唯一的 GUID**
3. **MooaToon 的 5 个属性都是 MCT_Float4，SF_Pixel**
4. **新增属性时要生成新的 GUID，不要改现有的**

### 记忆要点

- ✅ Add 函数 = 注册属性
- ✅ GUID 要唯一，不能重复
- ✅ MooaEncodedAttribute0-4 = 5 个 Float4
- ✅ HybriToon 新增属性要先生成新 GUID

---

**文档版本**: v1.0  
**分析深度**: 源码级  
**最后更新**: 2026年4月6日
