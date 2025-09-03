---
title: '[paper review] SINGAPO'
date: 2025-05-18
permalink: /posts/2025/05/blog-post-0001/
tags:
  - research, paper, summary
---

## SINGAPO: Single Image Controlled Generation of Articulated Parts In Objects
| [Project Page](https://3dlg-hcvc.github.io/singapo/) 
| [Arxiv](https://arxiv.org/pdf/2410.16499) 
| [Code](https://github.com/3dlg-hcvc/singapo)
|

* Read ✓✓
* Interests ★★☆☆☆
* Keywords
  * 3d generation
  * articulated object
  * retrieval

### Introduction
A method to generate articulated objects from a single image. Designed a pipeline that produces articulated objects from high-level structure to geometric details in a coarse-to-fine manner, where we use a part connectivity graph and part abstraction as proxies.


### Motivation and contributions
- recent line of works 
  - reconstructs articulated objects from images or videos that capture the object from multi-views or multiple articulated states.
  - aims to generate articulated objects either unconditionally or guided by high-level constraints

- So, in this work, tried to address 
  - use single arbitrary view image as input
  - modular pipeline to allow more control

### Methods
- Input :  single resting state image 
- Output : retrieved and assembled articulated parts
![Alt text](/files/singapo-01.png "pipeline")

### Goal 
generating a set of articulated parts assembling into a 3d object that is geometrically consistent with the input image while being kinematically plausible.
- use the input image to guide the abstract level generation of each part (used diffusion) > this paper focused on this first step.
- fitting part meshes to the generated part arrangement and constructing them into a 3d articulated object (=retrieval)

### Datasets
- used PartNet-Mobility (7 categories) , rendered 20 images from random viewpoints + augmentation = 3063 objects paired with 20 images  for training and 77 objects paired 2 random views for testing 
- ACD dataset to test the generalization capability

### Limitations
- tailored to specific object categories (cuboid shapes)
- mesh retrieval falls short in capturing finer geometric details of  parts as seen in the input
