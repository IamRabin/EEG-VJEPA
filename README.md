---
license: mit
---

## **Model Name:** EEG-VJEPA

### **Description:** 
EEG-VJEPA is a self-supervised learning model for EEG signal analysis. It treats EEG signals as spatiotemporal sequences, leveraging both spatial and temporal information. This model uses a Vision Transformer (ViT) as the backbone to capture long-range dependencies, enabling robust representation learning from EEG data.


### **Performance**

**(ViT-M-4x30x4)** <br />
**Accuracy (Frozen Eval):** 83.0% <br />
**F1 Score (Frozen Eval):** 82.4% <br />
**AUROC (Frozen Eval):** 87.7%

**(ViT-B-4x30x2)**  <br />
**Accuracy (Frozen Eval):** 81.2%  <br />
**F1 Score (Frozen Eval):** 81.0%  <br />
**AUROC (Frozen Eval):** 87.9%

### **Usage**
This model can be used / fine-tuned for various EEG analysis tasks, including fine-tuning for abnormal EEG classification. It is particularly effective for scenarios where spatial and temporal dependencies are critical. The code repository can be found at https://github.com/amir-hojjati/eeg-vjepa. The base model's encoder can be loaded with the pre-trained weights provided here and with the hyperparameters specificed in the paper for each model.

### **Training Configuration**
**Optimizer:** AdamW  <br />
**Learning Rate:** 0.0002 (initial), 0.000625 (reference), 1e-6 (final)  <br />
**Weight Decay:** CosineWDSchedule (0.04, 0.4)  <br />
**Gradient Clipping:** Max Norm 10.0  <br />
**Pretrained on:** 5 Nvidia V100-32GB GPUs for up to 400 epochs

### **Pretraining Details**
**Pre-training Dataset:** NMT + TUH

**Batch Size:** 10

**Encoder Size** <br />
(ViT-M-4x30x4): Medium (ViT-M) with 21M parameters <br />
(ViT-B-4x30x2): Base (ViT-B) with 85M parameters

**Predictor Depth** <br />
(ViT-M-4x30x4): 6 <br />
(ViT-B-4x30x2): 12 

**Sampling Rate** <br />
(ViT-M-4x30x4): 3 <br />
(ViT-B-4x30x2): 2

**# Frames** <br />
(ViT-M-4x30x4): 32 <br />
(ViT-B-4x30x2): 16

**# Clips** <br />
(ViT-M-4x30x4): 1 <br />
(ViT-B-4x30x2): 3

**Tubelet Size** <br />
(ViT-M-4x30x4): 4 <br />
(ViT-B-4x30x2): 2

**Augmentation** <br />
(ViT-M-4x30x4): Spatial (AR + Scale) <br />
(ViT-B-4x30x2): Spatial (AR + Scale)

**Random Resize Aspect Ratio** <br />
(ViT-M-4x30x4): (0.75, 1.35) <br />
(ViT-B-4x30x2): (0.75, 1.35)

**Random Resize Scale:** <br />
(ViT-M-4x30x4): (0.3, 1.0) <br />
(ViT-B-4x30x2): (0.3, 1.0)


---


### Pre-trained Weights
The pre-trained weights for the following models (best performing models) in the Weights directory:
- eeg_vjepa_ViT-B_4×30×2
- eeg_vjepa_ViT-M_4×30×4

---

### Setup

Run:
```bash
conda create -n eeg-vjepa python=3.9 pip
conda activate eeg-vjepa
python setup.py install
```

### Quick start to load the pre-trained weights

```python
import torch
import src.models.vision_transformer as vit
import yaml
import argparse

# Set up argument parser for config file
parser = argparse.ArgumentParser()
parser.add_argument('--config',
                    default='/path/to/configs/pretrain/config.yaml',
                    help='Path to config file')
args = parser.parse_args()

# Load config
with open(args.config, 'r') as f:
    config = yaml.safe_load(f)
    
model_name = args.get('model').get('model_name',
                                   'vit_small')
cfgs_data = args.get('data')
crop_size = cfgs_data.get('crop_size', (19, 500))
patch_size = cfgs_data.get('patch_size', (4, 30))
if isinstance(patch_size, list):
    patch_size = tuple(patch_size)
num_frames = cfgs_data.get('num_frames', 32)
tubelet_size = cfgs_data.get('tubelet_size', 4)
    
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
# Initialize encoder
encoder = vit.__dict__[model_name](
    img_size=crop_size,
    patch_size=patch_size, 
    num_frames=num_frames,
    tubelet_size=tubelet_size,
    uniform_power=False,
    use_sdpa=False
).to(device)

# Load pretrained weights
checkpoint = torch.load("path/to/weights.pth.tar",
                        map_location=device)
pretrained_dict = checkpoint['target_encoder']
pretrained_dict = {k.replace('module.', ''): v for k, v in pretrained_dict.items()}
pretrained_dict = {k.replace('backbone.', ''): v for k, v in pretrained_dict.items()}
encoder.load_state_dict(pretrained_dict, strict=False)

# Set to eval mode for inference
encoder.eval()
for p in encoder.parameters():
    p.requires_grad = False

# Data loader loop
...
```


### Local pre-training

```bash
python -m app.main \
  --fname configs/pretrain/config.yaml \
  --devices cuda:0 cuda:1 cuda:2
```


### Eval and Fine-tuning

```bash
python -m evals.main \
  --fname configs/eval/config.yaml \
  --devices cuda:0 cuda:1 cuda:2
```

---

