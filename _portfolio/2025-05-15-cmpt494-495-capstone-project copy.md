---
title: "CMPT 494/495 Capstone Project"
excerpt: "Interactive 3D Scene Viewer and Editor - PAPR Implementation in Nerfstudio"
collection: portfolio, capstone
---

## Project Overview

- **Supervisor**: Ke Li (keli@sfu.ca)
- **Duration**: May - December 2025
- **Team members**: Team of two (Summer) + one more member (Fall)

### References
- [PAPR Paper Project Page](https://github.com/zvict/papr)
- [Nerfstudio Documentation](https://docs.nerf.studio)

### Technologies Used
- **ML Framework**: PyTorch, Nerfstudio
- **GPU Computing**: CUDA, WebGPU
- **Languages**: Python, TypeScript
- **Libraries**: webgpu-torch, Viser (3D visualization)

### My Contributions

#### Summer 2025 (CMPT 494)
Worked with Cameron on porting PAPR (Point-based Adaptive Point Rendering) into Nerfstudio framework.

**Key Contributions:**
- **Porting PAPR paper's codebase to work NeRFStudio**
- **Config Optimization**: Explored hyperparameter settings to improve training results
- **GPU Memory Debugging**: Resolved memory issues occurring during training
- **Scheduler Re-initialization Bug Fix**: Fixed NaN value issues through fast-forward scheduling and learning rate tuning
- **.ply File Export**: Implemented functionality to export trained models as point cloud format
- **Unit Test Development**: Wrote unit tests for core components like PAPRField

**Training Results Visualization**
Training progress at 240k and 245k iterations showing Ground Truth vs Predicted, Depth Map, Point Cloud, and Training Losses:

![Training Results at 240k iterations](/images/capstone-training-240k.png)

![Training Results at 245k iterations](/images/capstone-training-245k.png)


#### Fall 2025 (CMPT 495)
Led the **Client-side Rendering with WebGPU-torch** implementation.

**Goal:** Enable PAPR model rendering directly in the browser without Nerfstudio backend to eliminate network latency

**Key Contributions:**
- **PAPR Model TypeScript Porting**: Re-implemented PyTorch-based PAPR model in TypeScript using webgpu-torch library
- **Missing Kernel Implementation**: Implemented tensor operations missing in webgpu-torch (clamp, permute, softmax, transpose, conv2d, slice, sliceCopy, etc)
- **Debugging Large Tensor Support**: Extended 1D dispatch to 2D dispatch to handle tensors with 16M+ elements
- **Performance Analysis**:
  - Analyzed kernel call overhead through Sin Benchmark
  - Measured execution time distribution using Chrome DevTools GPU profiling
  - Identified that 68.9% of render time was synchronization overhead (23,000+ WebGPU events for 512x512 render)

**Results:**
- Successfully rendered in browser for single test case
- 2.5x~4x slower render time compared to PyTorch (Assuming it's due to WebGPU synchronization overhead)
- Documented limitations and improvement directions for webgpu-torch library

| Resolution | Patches | Render Time | Kernel Calls |
|------------|---------|-------------|--------------|
| 64×64      | 1       | 0.38s       | 567          |
| 256×256    | 16      | 2.74s       | 5,502        |
| 512×512    | 64      | 11.35s      | 21,366       |


### Project Architecture

The project consists of multiple interconnected repositories:

```
3d-editor (main)
├── nerfstudio/
│   ├── models/papr_model.py          # Main PAPR model
│   ├── fields/papr_field.py          # Point-based field with attention
│   ├── pipelines/papr_pipeline.py    # Training pipeline
│   └── viewer/custom/                # Point cloud editor
├── papr-cuda-topk/                   # CUDA kernel optimization
└── viser/                            # Custom Viser fork
    └── webgpu-torch/                 # WebGPU-torch fork (my focus)
```

