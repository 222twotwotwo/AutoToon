# 方案一：神经Ramp预测

## 方案概述

神经Ramp预测是三种方案中最简单、最轻量的方案。它保持MooaToon原有管线完全不变，只修改Ramp采样部分，用神经网络预测Ramp索引和UV偏移。

---

## 核心思路

```
┌─────────────────────────────────────────────────────────────┐
│                    风格图像输入                              │
│                    (Style Image)                             │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  风格编码器 (Style Encoder)                  │
│              (CNN / ViT / StyleGAN Encoder)                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼ 风格特征 (Style Feature)
         ┌───────────┴───────────┐
         │                       │
         ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│  神经Ramp预测器  │    │  神经Ramp预测器  │
│  (漫反射)        │    │  (高光)          │
└────────┬─────────┘    └────────┬─────────┘
         │                         │
         ▼                         ▼
┌──────────────────┐    ┌──────────────────┐
│  Ramp索引        │    │  Ramp索引        │
│  UV偏移          │    │  UV偏移          │
└────────┬─────────┘    └────────┬─────────┘
         │                         │
         └──────────┬──────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              MooaToon原有渲染管线 (不变)                     │
│              (ToonBxDF, Ramp采样等)                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 详细设计

### 1. 风格编码器 (Style Encoder)

#### 网络架构

```python
import torch
import torch.nn as nn
import torchvision.models as models

class StyleEncoder(nn.Module):
    def __init__(self, style_dim=512):
        super().__init__()
        
        # 使用预训练的ResNet作为 backbone
        self.backbone = models.resnet18(pretrained=True)
        
        # 移除最后的分类层
        self.backbone = nn.Sequential(*list(self.backbone.children())[:-1])
        
        # 风格投影层
        self.style_proj = nn.Sequential(
            nn.Linear(512, 1024),
            nn.ReLU(),
            nn.Linear(1024, style_dim)
        )
    
    def forward(self, x):
        # x: [B, 3, H, W]
        features = self.backbone(x)  # [B, 512, 1, 1]
        features = features.flatten(1)  # [B, 512]
        style = self.style_proj(features)  # [B, style_dim]
        return style
```

#### 变体：使用 Vision Transformer (ViT)

```python
from transformers import ViTModel

class StyleEncoderViT(nn.Module):
    def __init__(self, style_dim=512):
        super().__init__()
        
        self.vit = ViTModel.from_pretrained('google/vit-base-patch16-224')
        self.style_proj = nn.Sequential(
            nn.Linear(768, 1024),
            nn.ReLU(),
            nn.Linear(1024, style_dim)
        )
    
    def forward(self, x):
        outputs = self.vit(x)
        cls_token = outputs.last_hidden_state[:, 0, :]  #<[BOS_never_used_51bce0c785ca2f68081bfa7d91973934]> token
        style = self.style_proj(cls_token)
        return style
```

#### 变体：使用 StyleGAN Encoder

```python
class StyleEncoderStyleGAN(nn.Module):
    def __init__(self, style_dim=512):
        super().__init__()
        
        # StyleGAN-like encoder
        self.conv_layers = nn.Sequential(
            nn.Conv2d(3, 64, 3, padding=1),
            nn.ReLU(),
            nn.AvgPool2d(2),
            nn.Conv2d(64, 128, 3, padding=1),
            nn.ReLU(),
            nn.AvgPool2d(2),
            nn.Conv2d(128, 256, 3, padding=1),
            nn.ReLU(),
            nn.AvgPool2d(2),
            nn.Conv2d(256, 512, 3, padding=1),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d(1)
        )
        
        self.style_proj = nn.Linear(512, style_dim)
    
    def forward(self, x):
        features = self.conv_layers(x)
        features = features.flatten(1)
        style = self.style_proj(features)
        return style
```

---

### 2. 神经Ramp预测器 (Neural Ramp Predictor)

#### 网络架构

```python
class NeuralRampPredictor(nn.Module):
    def __init__(self, style_dim=512, num_ramps=32):
        super().__init__()
        
        self.num_ramps = num_ramps
        
        # 共享的MLP
        self.shared_mlp = nn.Sequential(
            nn.Linear(style_dim, 1024),
            nn.LayerNorm(1024),
            nn.ReLU(),
            nn.Linear(1024, 512),
            nn.LayerNorm(512),
            nn.ReLU()
        )
        
        # 漫反射Ramp预测头
        self.diffuse_head = nn.Sequential(
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Linear(256, num_ramps + 1)  # num_ramps + UV偏移
        )
        
        # 高光Ramp预测头
        self.specular_head = nn.Sequential(
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Linear(256, num_ramps + 1)  # num_ramps + UV偏移
        )
    
    def forward(self, style_feature):
        # style_feature: [B, style_dim]
        shared = self.shared_mlp(style_feature)  # [B, 512]
        
        # 漫反射预测
        diffuse_out = self.diffuse_head(shared)  # [B, num_ramps + 1]
        diffuse_ramp_logits = diffuse_out[:, :-1]  # [B, num_ramps]
        diffuse_uv_offset = diffuse_out[:, -1:]  # [B, 1]
        
        # 高光预测
        specular_out = self.specular_head(shared)  # [B, num_ramps + 1]
        specular_ramp_logits = specular_out[:, :-1]  # [B, num_ramps]
        specular_uv_offset = specular_out[:, -1:]  # [B, 1]
        
        # 用Gumbel-Softmax采样Ramp索引（可微分）
        diffuse_ramp_idx = F.gumbel_softmax(diffuse_ramp_logits, tau=1.0, hard=True)
        specular_ramp_idx = F.gumbel_softmax(specular_ramp_logits, tau=1.0, hard=True)
        
        return {
            'diffuse_ramp_idx': diffuse_ramp_idx,
            'diffuse_uv_offset': diffuse_uv_offset,
            'specular_ramp_idx': specular_ramp_idx,
            'specular_uv_offset': specular_uv_offset
        }
```

---

### 3. 完整的端到端模型

```python
class HybriToonRampPredictor(nn.Module):
    def __init__(self, style_dim=512, num_ramps=32):
        super().__init__()
        
        self.style_encoder = StyleEncoder(style_dim)
        self.ramp_predictor = NeuralRampPredictor(style_dim, num_ramps)
    
    def forward(self, style_image):
        style_feature = self.style_encoder(style_image)
        ramp_predictions = self.ramp_predictor(style_feature)
        return ramp_predictions
```

---

## 训练策略

### 1. 数据集准备

需要收集以下数据：
- 风格图像（动漫、插画等）
- 对应的渲染结果（或配对的真实图像）
- Ramp纹理图集

### 2. 损失函数

```python
import torch.nn.functional as F

def compute_loss(
    ramp_predictions,
    target_diffuse_ramp_idx,
    target_diffuse_uv_offset,
    target_specular_ramp_idx,
    target_specular_uv_offset,
    rendered_image,
    target_image
):
    # Ramp索引损失（交叉熵）
    diffuse_ramp_loss = F.cross_entropy(
        ramp_predictions['diffuse_ramp_idx'],
        target_diffuse_ramp_idx
    )
    specular_ramp_loss = F.cross_entropy(
        ramp_predictions['specular_ramp_idx'],
        target_specular_ramp_idx
    )
    
    # UV偏移损失（L1）
    diffuse_uv_loss = F.l1_loss(
        ramp_predictions['diffuse_uv_offset'],
        target_diffuse_uv_offset
    )
    specular_uv_loss = F.l1_loss(
        ramp_predictions['specular_uv_offset'],
        target_specular_uv_offset
    )
    
    # 图像重建损失（感知损失）
    perceptual_loss = compute_perceptual_loss(rendered_image, target_image)
    
    # 总损失
    total_loss = (
        diffuse_ramp_loss +
        specular_ramp_loss +
        0.1 * diffuse_uv_loss +
        0.1 * specular_uv_loss +
        1.0 * perceptual_loss
    )
    
    return total_loss
```

### 3. 感知损失 (Perceptual Loss)

```python
from torchvision.models import vgg19

class PerceptualLoss(nn.Module):
    def __init__(self):
        super().__init__()
        vgg = vgg19(pretrained=True).features
        self.layers = nn.ModuleList([
            vgg[:4],   # relu1_2
            vgg[4:9],  # relu2_2
            vgg[9:18], # relu3_4
            vgg[18:27] # relu4_4
        ])
        
        for param in self.parameters():
            param.requires_grad = False
    
    def forward(self, x, y):
        loss = 0.0
        for layer in self.layers:
            x = layer(x)
            y = layer(y)
            loss += F.l1_loss(x, y)
        return loss
```

---

## UE5集成

### 1. 导出ONNX模型

```python
# 导出模型
model = HybriToonRampPredictor()
model.eval()

dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(
    model,
    dummy_input,
    "hybri_toon_ramp_predictor.onnx",
    export_params=True,
    opset_version=12,
    do_constant_folding=True,
    input_names=['style_image'],
    output_names=['diffuse_ramp_idx', 'diffuse_uv_offset', 'specular_ramp_idx', 'specular_uv_offset']
)
```

### 2. UE5 C++集成

```cpp
// 在SceneRendering.cpp中添加
#include "Microsoft/ONNXRuntimeModule.h"

class FHybriToonRampPredictor
{
public:
    FHybriToonRampPredictor()
    {
        // 加载ONNX模型
        OnnxRuntimeModule = FModuleManager::LoadModuleChecked&lt;FONNXRuntimeModule&gt;("ONNXRuntime");
        Model = OnnxRuntimeModule.CreateModel(TEXT("hybri_toon_ramp_predictor.onnx"));
    }
    
    void PredictRamp(
        const TArray&lt;float&gt;&amp; StyleImage,
        int32&amp; OutDiffuseRampIdx,
        float&amp; OutDiffuseUVOffset,
        int32&amp; OutSpecularRampIdx,
        float&amp; OutSpecularUVOffset
    )
    {
        // 准备输入
        TArray&lt;FONNXInputTensor&gt; Inputs;
        Inputs.Add(FONNXInputTensor(TEXT("style_image"), StyleImage.GetData(), {1, 3, 224, 224}));
        
        // 推理
        TArray&lt;FONNXOutputTensor&gt; Outputs = Model-&gt;Run(Inputs);
        
        // 获取输出
        OutDiffuseRampIdx = Outputs[0].GetData&lt;int32&gt;()[0];
        OutDiffuseUVOffset = Outputs[1].GetData&lt;float&gt;()[0];
        OutSpecularRampIdx = Outputs[2].GetData&lt;int32&gt;()[0];
        OutSpecularUVOffset = Outputs[3].GetData&lt;float&gt;()[0];
    }
    
private:
    FONNXRuntimeModule* OnnxRuntimeModule;
    TSharedPtr&lt;FONNXModel&gt; Model;
};
```

### 3. 修改ToonShadingModel.ush

```hlsl
// 在ToonShadingModel.ush中添加
#if HYBRITOON_ENABLED
    // 从神经预测器获取Ramp参数
    uint DiffuseRampIdx = HybriToon.DiffuseRampIdx;
    float DiffuseUVOffset = HybriToon.DiffuseUVOffset;
    uint SpecularRampIdx = HybriToon.SpecularRampIdx;
    float SpecularUVOffset = HybriToon.SpecularUVOffset;
#else
    // 原有逻辑
    uint DiffuseRampIdx = ToonGBuffer.DiffuseColorRampIndex;
    float DiffuseUVOffset = ToonGBuffer.DiffuseColorRampUVOffset;
    uint SpecularRampIdx = ToonGBuffer.SpecularColorRampIndex;
    float SpecularUVOffset = ToonGBuffer.SpecularColorRampUVOffset;
#endif
```

---

## 优缺点总结

### 优点 ✅

1. **修改量最小**
   - 只修改Ramp采样部分
   - 保持MooaToon原有管线完全不变

2. **性能影响最低**
   - 风格编码器可以预计算
   - Ramp预测器很小，推理很快
   - 可以离线推理，实时应用

3. **容易调试和维护**
   - 逻辑简单清晰
   - 问题容易定位
   - 可以独立测试每个模块

4. **保持MooaToon所有功能**
   - 描边、头发阴影、面部阴影等都保留
   - 可以混合使用神经预测和手动设置

### 缺点 ❌

1. **效果受限于Ramp纹理**
   - 无法超出Ramp纹理的范围
   - 需要高质量的Ramp图集

2. **无法完全端到端优化**
   - 优化目标是预测Ramp参数，不是最终图像
   - 可能不是全局最优

---

## 推荐使用场景

1. **快速原型验证**
   - 想要快速验证AI风格的效果
   - 不需要完美的效果

2. **性能敏感场景**
   - 游戏、VR等实时应用
   - 对帧率要求高

3. **保持现有管线**
   - 已经有成熟的MooaToon管线
   - 不想做大的改动

---

## 下一步

方案一实现并验证后，可以考虑：
- 升级到方案二（神经光照注入）
- 收集更多数据，提升效果
- 优化性能，降低推理延迟

---

**祝你实施顺利！** 🚀
