---
title: "The math behind CGAffineTransforms"
date: 2020-10-17T13:46:30+05:30
draft: true
tags: [ios, swift]
GHIssueID:
---

Lately I have been watching the excellent "[Essence of Linear Algebra](https://www.youtube.com/playlist?list=PLZHQObOWTQDPD3MizzM2xVFitgF8hE_ab)" video series by [3blue1brown](https://www.3blue1brown.com/). As an iOS developer, I was vaguely familiar with [CGAffineTransform](https://developer.apple.com/documentation/coregraphics/cgaffinetransform) and how it could be used to scale, rotate and shear UIViews. Watching 3b1b's series helped me understand the math behind it. This post is my attempt at explaining, and cementing my own understanding, of the concept.

## A quick look at Linear Transformations
Before diving into affine transformations, let's look at a subset of it called linear transformations. Ignoring the word linear, a "transformation" is, quite simply, transformation of space. You can think of transformation as a function (not in the programming sense, but in mathematical sense) that, given any sample space as input, transforms all points in the input space to some resulting output space. This is best understood visually. If your input space is a two dimensional grid, these GIFs represent some transformations of that 2D space.

{{< image src="nonlinear_2d_transform_1.gif" alt="Non Linear 2D Transform" position="center" style="border-radius: 8px;" >}}