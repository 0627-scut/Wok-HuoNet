# Wok-HuoNet: An End-to-End Lightweight Instance Segmentation Framework for Real-Time Huohou Assessment in Chinese Stir-Frying

This folder provides the model configuration files for **Wok-HuoNet**, a lightweight YOLOv11n-seg-based instance segmentation architecture proposed for real-time doneness (*Huohou*) assessment in Chinese stir-frying. Full methodological details, data, and experiments are reported in the manuscript submitted to *Engineering Applications of Artificial Intelligence* (EAAI).

The released YAML files describe the network topology (pruned backbone, neck, and detection/segmentation heads), the channel-pruning setting, and the three proposed enhancement modules. They are sufficient to understand the architecture and to reproduce the ablation variants reported in the manuscript.

## Three Proposed Modules

Wok-HuoNet introduces three lightweight, plug-and-play modules on top of a channel-pruned YOLOv11n-seg baseline:

| Abbr. | Code identifier | Inserted at | Role |
|-------|-----------------|-------------|------|
| **DCA** | `DCA_Conv` | backbone, before SPPF | **Doneness Context Attention** convolutional module. Extracts multi-scale context at 1×1, 2×2, 4×4, 8×8, and fuses it with an environmental-noise branch to produce an attention weight that strengthens doneness-related channels and suppresses oil-fume/occlusion interference |
| **MSE** | `C2fMSE` | neck (replaces every `C3k2`) | **Multi-Scale Edge-aware** CSP module. Three branches — a 3×3 semantic bottleneck, an edge-aware dual-layer convolution for ingredient boundaries, and a multi-scale branch — sharpen segmentation edges under occlusion at the native C3k2 cost |
| **MSHR** | `MultiScaleHeadRefine_v2` | before the Segment head | **Multi-Scale Head Refinement**. Applies three depthwise convolutions (3×3, 7×7, 9×9) to the neck output to recover the fine-grained texture details lost to pruning and downsampling |

The full model (**DCA + MSE + MSHR** on the 768-channel pruned baseline) is `Wok-HuoNet.yaml`. It reaches **mAP@0.5 = 0.752 at 7.3 GFLOPs** — 0.6 percentage points higher than the unpruned YOLOv11n-seg baseline while using **24.74% fewer FLOPs and 32.95% fewer parameters**.


## Configuration Files

All configuration files listed below reside under `ultralytics/cfg/models/Wok-HuoNet/`. Tables show only filenames for brevity.

### A. Proposed Full Model

| File | Max Channels | Modules | Conv @18/21 | Description |
|------|:---:|---|:---:|---|
| `Wok-HuoNet.yaml` | **768** | DCA + MSE + MSHR | Conv | **Final proposed model** — pruned YOLOv11n-seg (768) + three enhancement modules |

### B. Channel-Pruning Variants

The native YOLOv11n-seg uses 1024-dimensional channels in the deep backbone and neck. *Max Channels* below denotes the largest base channel width specified in the backbone (the P5/SPPF stage), before the Ultralytics `scales` width multiplier is applied. The proposed setting (**768**) gives the optimal accuracy–efficiency trade-off, reducing FLOPs by 25.77% (9.7 → 7.2 GFLOPs) at only a 0.9 percentage-point mAP cost. The 640 (over-pruned) and 1024 (unpruned) variants are released for comparison.

| File | Max Channels | Modules | Description |
|------|:---:|---|---|
| `Wok-HuoNet-c640.yaml` | 640 | DCA + MSE + MSHR | Over-pruned variant (lower bound of the pruning sweep) |
| `Wok-HuoNet-c1024.yaml` | 1024 | DCA + MSE + MSHR | Unpruned variant (original YOLOv11n-seg width, upper bound) |

### C. Structural Variant (Down-sampling Convolution)

| File | Max Channels | Modules | Conv @18/21 | Description |
|------|:---:|---|:---:|---|
| `Wok-HuoNet-DW.yaml` | 768 | DCA + MSE + MSHR | **DWConv** | The two neck down-sampling convolutions (layers 18 and 21) use depthwise convolution (`DWConv`) instead of the standard `Conv`|

### D. Module Ablation Study

All ablation configs use max channel = 768 and standard `Conv` (rather than the `DWConv` of Section C) at the two neck down-sampling positions. Each row removes one or more proposed modules to isolate its contribution. Together they form a representative subset of the full ablation table reported in the manuscript.

| File | Modules kept | Removed | Ablation role |
|------|---|---|---|
| `Wok-HuoNet-noMSE.yaml` | DCA + MSHR | MSE | Effect of the MSE (edge-aware) module |
| `Wok-HuoNet-noDCA.yaml` | MSE + MSHR | DCA | Effect of the DCA module |
| `Wok-HuoNet-MSHRonly.yaml` | MSHR only | DCA + MSE | MSHR alone |
| `Wok-HuoNet-MSEonly.yaml` | MSE only | DCA + MSHR | MSE alone |

> The configs above are a representative subset released for reproducibility. The single-module baselines and the pairwise combinations released here let each module's marginal contribution be read directly against the pruned-768 baseline and the full model. The complete ablation matrix is reported in the manuscript's ablation table.

## Usage

The configurations are compatible with the Ultralytics Python API. Train the full model with the same hyperparameter setup used in the paper:

```python
from ultralytics import YOLO

model = YOLO("path/to/Wok-HuoNet.yaml")
model.train(
    data="path/to/data.yaml",
    epochs=150,
    imgsz=640,
    device=0,
    optimizer="SGD",
    lr0=0.01,
    momentum=0.9,
    batch=8,
    lrf=0.01,
    warmup_epochs=5,
    weight_decay=0.008,
    warmup_bias_lr=0.0001,
    patience=20,
    hsv_h=0, hsv_s=0.1, hsv_v=0.2,
    erasing=0.3,
    mosaic=0.5, mixup=0,
    close_mosaic=0,
    amp=False,
)

# To run an ablation variant, simply swap the config path, e.g.:
# model = YOLO("path/to/Wok-HuoNet-noMSE.yaml")
```

## Code and Data Availability

The configuration files for Wok-HuoNet are provided in this repository. The full implementation (custom block and task-pipeline source code) and the dataset used in this study will be released upon acceptance of the manuscript. For collaboration, peer review, or replication purposes, the source code and dataset are available from the corresponding author upon reasonable request.

> **Corresponding author:** Prof. Dr. Jun-Hu Cheng
> School of Food Science and Engineering, South China University of Technology, Guangzhou 510641, China
> Email: chengjunhu1229@163.com  ·  ORCID: [0000-0003-3928-1770](https://orcid.org/0000-0003-3928-1770)

## Citation

If you find this work useful, please cite:

```bibtex
@article{WokHuoNet2026,
  title   = {Wok-HuoNet: An End-to-End Lightweight Instance Segmentation Framework for Real-Time Huohou Assessment in Chinese Stir-Frying},
  author  = {Wang, Han and Lin, Yuandong and Zhang, Hengrui and Zeng, Xin-An and Wu, Jingzhu and Jia, Yuze and Tang, Xiangwei and Cheng, Jun-Hu},
  journal = {Engineering Applications of Artificial Intelligence},
  year    = {2026},
  note    = {Under review}
}
```

## License

This work builds upon the [Ultralytics AGPL-3.0](https://github.com/ultralytics/ultralytics/blob/main/LICENSE) framework. The additional modules and configurations introduced by Wok-HuoNet are released under the same license. For commercial licensing inquiries, please contact the corresponding author.

## Acknowledgements

This work is based on the [Ultralytics YOLO](https://github.com/ultralytics/ultralytics) codebase. We thank the Ultralytics team and the open-source community for their foundational contributions.
