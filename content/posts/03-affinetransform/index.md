---
title: "Visualizing CGAffineTransforms"
date: 2020-10-17T13:46:30+05:30
draft: true
tags: [ios, swift]
GHIssueID:
---

Lately I have been watching the excellent "[Essence of Linear Algebra](https://www.youtube.com/playlist?list=PLZHQObOWTQDPD3MizzM2xVFitgF8hE_ab)" video series by [3blue1brown](https://www.3blue1brown.com/). As an iOS developer, I was vaguely familiar with [CGAffineTransform](https://developer.apple.com/documentation/coregraphics/cgaffinetransform) and how it could be used to scale, rotate and shear UIViews. Watching 3b1b's series helped me understand the math behind it. This post is my attempt at explaining, and cementing my own understanding, of the concept.

## Linear Transformations
A linear transformation is, quite simply, transformation of space subject to following restrictions -

- All lines must remain lines
- Origin must not move

Following are examples of transformations which are not linear.

{{<figure src="nonlinear_2d_transform_1.gif" alt="Non linear transformation 1" caption="Non linear transform. Lines get all curvy." position="center" style="border-radius: 8px;">}}
{{<figure src="nonlinear_2d_transform_2.gif" alt="Non linear transformation 2" caption="Non linear transform. Origin moves." position="center" style="border-radius: 8px;">}}
{{<figure src="nonlinear_2d_transform_3.gif" alt="Non linear transformation 3" caption="Non linear transform. Diagonal line gets curvy" position="center" style="border-radius: 8px;">}}

Given these restrictions, how would you go about expressing a linear transformation numerically? To do this, imagine two [special](https://en.wikipedia.org/wiki/Unit_vector) vectors - $\hat{i}$ and $\hat{j}$ lying along postive X and Y axes respectively. In iOS, positive Y axis points downwards rather than upwards, so these vectors plotted in a 2D space look something like this -

{{<figure src="i_j.png" alt="Unit vectors" caption="Unit vectors. Ignore the settings button and slider at the bottom" position="center" style="border-radius: 8px;">}}

Co-ordinates of $\hat{i}$ are (1, 0) and those of $\hat{j}$ are (0, 1). When written down as [row vectors](https://en.wikipedia.org/wiki/Row_and_column_vectors) -

$$
\hat{i}=\begin{bmatrix}1 & 0\end{bmatrix}
$$

$$
\hat{j}=\begin{bmatrix}0 & 1\end{bmatrix}
$$

Now, as long as the transformation is linear, you only need to know where $\hat{i}$ and $\hat{j}$ end up after the transform to describe the linear transformation numerically. Let's consider an example -

{{<figure src="linear_2d_transform_1.gif" alt="Linear transformation 1" caption="A linear transformation" position="center" style="border-radius: 8px;">}}

The crosshairs here denote the position where our unit vectors end up after the transform. $\hat{i}$ moves from (1, 0) to (3, 1) and $\hat{j}$ moves from (0, 1) to (1, 2). So this linear transform can be represented by the matrix $\vec{T}$ -

$$
\vec{T}=\begin{bmatrix}3 & 1\\\\1 & 2\end{bmatrix}
$$

In general, any linear transform can be represented in matrix form as -

$$
\vec{T}=\begin{bmatrix}a & b\\\\c & d\end{bmatrix}
$$

where (a, b) denotes co-ords of $\hat{i}$ and (c, d) denotes the co-ords of $\hat{j}$ after the transform. 

Given a vector $\begin{bmatrix}x & y\end{bmatrix}$ and transfrom $\vec{T}$ (which is a 2x2 matrix), the corresponding vector can be found multiplying vector matrix and the transform matrix.

$$
\begin{bmatrix}\grave{x} & \grave{y}\end{bmatrix} = \begin{bmatrix}x & y\end{bmatrix} \times \begin{bmatrix}a & b\\\\c & d\end{bmatrix}
$$

If all the math stuff scares you, just remember what the matrix represents visually - The new co-ords of unit vectors $\hat{i}$ and $\hat{j}$ after the transform is applied. Let's look at a few more examples visually.

##### [Scale Matrix](https://developer.apple.com/documentation/coregraphics/1455016-cgaffinetransformmakescale)
Scaling is simple. We just need to move $\hat{i}$ and $\hat{j}$ further along their axes by whatever amount we want to scale in each direction.

{{<figure src="scale.gif" alt="Scale transform" caption="Scale by 3 along X axis and by 1.5 along Y axis" position="center" style="border-radius: 8px;">}}

Corresponding matrix -

$$
\vec{T}=\begin{bmatrix}sx & 0\\\\0 & sy\label{A famous equation}\end{bmatrix}
$$

Note that you can specify negative values for sx and sy. What do you think the resulting transform will look like?

##### [Rotation Matrix](https://developer.apple.com/documentation/coregraphics/1455666-cgaffinetransformmakerotation)
Imagine moving $\hat{i}$ from (1, 0) to (0, 1) and $\hat{j}$ from (0, 1) to (-1, 0).

{{<figure src="rotate.gif" alt="Rotate transform" caption="Rotate 90° clockwise" position="center" style="border-radius: 8px;">}}

The effect is same as rotating the image 90° clockwise. For all other angles of rotation, it is simpler to use [CGAffineTransformMakeRotation](https://developer.apple.com/documentation/coregraphics/1455666-cgaffinetransformmakerotation) function which accepts an angle and gives you back the correct transformation matrix. If you wanna understand the math behind rotation angle, read up on [Unit Circle](https://en.wikipedia.org/wiki/Unit_circle)