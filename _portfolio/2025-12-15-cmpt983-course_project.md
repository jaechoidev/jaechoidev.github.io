---
title: "CMPT 983 Generative Modeling Course Project"
excerpt: "π0.5-IMLE-LoRA: Fast Action Sequence Generation via Implicit Maximum Likelihood Estimation"
collection: portfolio, Project
header:
  teaser: 983.png
---

## Project Overview

- **Course**: CMPT 983 - Generative Modeling (Fall 2025)
- **Topic**: Vision-Language-Action (VLA) Model Optimization
- **Number of Team members**: Team of 2

### Abstract

We study action sequence generation in Vision-Language-Action (VLA) models, focusing on the action expert component. Specifically, we replace the flow-matching-based action expert in π0.5 with a conditional generative model trained using rejection sampling implicit maximum likelihood estimation (RS-IMLE). This modification removes the need for iterative denoising during inference and enables single-pass action sequence generation.

### Technologies Used
- **Framework**: PyTorch, LeRobot
- **Models**: π0.5, PaliGemma-2B (VLM), Gemma-300M (Action Expert), U-Net(Action generator)
- **Techniques**: RS-IMLE, LoRA, Flow Matching
- **Benchmark**: LIBERO (40 robot manipulation tasks over 4 task suites)

## Method

### π0.5 Architecture
The π0.5 model consists of two major components:
1. **Vision Language Model (VLM)**: PaliGemma-2B that generates conditional latent embeddings encoding current task and state observation
2. **Action Expert**: Gemma-300M that takes the latent embedding and generates action chunk sequences using flow matching

### Our Approach: RS-IMLE Action Expert
We replaced the flow-matching action expert with a model trained using **Rejection Sampling IMLE (RS-IMLE)**:
- Enables **single-pass action generation** (vs. 10 iterative denoising steps)
- Reduces inference latency by ~70%
- Uses likelihood-free training that ensures training data coverage

### LoRA Fine-tuning
Applied Low-Rank Adaptation (LoRA) to efficiently fine-tune the VLM:
- Target modules: Attention layers (q, k, v, o projections) and FFN layers
- Rank r=32, scaling factor α=2r, dropout=0.05
- Two-stage training: First train action expert (6k steps), then joint fine-tuning with LoRA (6k steps)

## My Contributions

- **U-Net Action Expert Implementation**: Trained and evaluated π0.5-IMLE-U-Net variants with different pooling strategies (average pooling, attention pooling, type encoding + attention pooling)
- **π0.5-IMLE-LoRA Implementation**: Implemented LoRA fine-tuning for the VLM backbone
- **Benchmark & Profiling**: Measured inference time and evaluation time across all model variants
- **Report Writing**: Co-authored the final project report

## Results

### Main Results

| Variant | Success Rate | Inference Time | Training Steps |
|---------|--------------|----------------|----------------|
| π0.5 (baseline) | **94%** | 131.13ms | - |
| π0.5-one-step-flow-matching | **94.25%** | **28.6ms** | 6k |
| π0.5-LoRA-RSIMLE | 86% | 39.62ms | 12k (2-stage) |

### U-Net Action Expert Results (My Experiments)

| Variant | Success Rate (LIBERO Spatial) | Success Rate (Full LIBERO) |
|---------|-------------------------------|----------------------------|
| U-Net (avg pool) | 99% | 0% |
| U-Net (attn pool) | 98% | 6.5% |
| U-Net (type enc+attn pool) | 98% | 19.75% |

### Key Findings

1. **RS-IMLE reduces inference time by ~70%** but incurs 8-14% performance drop
2. **One-step flow matching achieves best trade-off**: maintains 94%+ success rate with lowest inference time (28.6ms)
3. **Objective alignment matters**: Flow-matching fine-tuning preserves pretrained action manifold better than RS-IMLE
4. **U-Net information bottleneck**: Pooling-based conditioning works for simple tasks (LIBERO Spatial) but fails on diverse tasks due to compressing rich VLM sequences into single vectors

## Discussion

### RS-IMLE vs Flow Matching Trade-off

Our experiments reveal a trade-off between inference efficiency and task performance when fine-tuning the action expert with RS-IMLE objective. While π0.5-LoRA-RSIMLE substantially improves inference speed relative to the baseline, it incurs an absolute performance drop of 8-14% in success rate across the LIBERO benchmark.

### Importance of Objective Alignment

The comparison with one-step flow matching highlights the importance of **objective alignment** with the pretrained action expert. Fine-tuning the flow-matching model to operate in a single inference step preserves both task performance and inference efficiency, likely because the pretrained checkpoint already encodes a velocity field consistent with the flow-matching objective.

In contrast, RS-IMLE optimizes a fundamentally different objective that emphasizes sample coverage rather than trajectory likelihood. This **objective mismatch** causes the model to deviate from the pretrained action manifold, resulting in reduced task success.

### Inference Speedup Analysis

Although RS-IMLE reduces the number of forward passes during inference, the observed speedup is substantially smaller than the theoretical 10x reduction implied by decreasing the number of denoising steps from 10 to 1. This suggests that non-negligible overheads such as action chunking and model I/O contribute significantly to the overall action generation latency.

### U-Net Information Bottleneck

The U-Net experiments revealed a critical insight: the **information bottleneck** created by pooling VLM outputs into a single conditioning vector discards spatial relationships, fine-grained language grounding, and token-level semantics. This explains why π0.5 uses a transformer-based action expert with cross-attention instead of U-Net, as it can attend selectively to different parts of the conditioning sequence for each action generation step.

