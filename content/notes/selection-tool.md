---
title: "Selection Tool in Maya"
date: 2022-03-27T21:08:07-07:00
draft: true
toc: false
images:
tags:
  - untagged
---

I've been working on the selection tool which can select objects by an proxy object. The proxy can be a cube or sphere, and same logic is implemented to the camera distance selection as well.
The main concept came from a blog post about frustum culling. It refers something like basic collision logic. To do this, I needed normal vector and a point for each planes.
Every objects in the scene will be checked if the farthest bbox point is on the same direction of plane normal or not. It is calculated fast because it does one calculation per object for every planes.
However, it has limitation for sure because we only can use bounding box, not original shape of the object.

For the more accurate approach, we have to find one of the farthest points of the object's vertices (or it could be a closest point). If we can find the farthest point, we can use the same logic.
If we change it to use closestPointOnMesh, we need two points, and compare them to see if the objects are overwrapped. 

The selection logic itself was not very hard to implement because the blog post has a lot of information and it was clear to understand. Instead, I had to spend more time on building UI and 
testing performances for each features. This selection tool probably used to distinguish between rendered area and the other, so if the set is huge, the selecting elements itself could be heavy
in Maya. I have implemented most of the features that were requested, but some of them are still need to be figured out for better performance.

Source codes on github is not a studio version, but I rebuilt most of the features as same. I hope it could be useful for someone.



Reference
- Frustum Culling in Maya (https://gizmosandgames.com/2017/05/21/frustum-culling-in-maya/)
