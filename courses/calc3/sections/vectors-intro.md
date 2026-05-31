# Vectors in 3D Space

## What is a Vector?

A **vector** in $\mathbb{R}^3$ is a quantity with both magnitude and direction, written as:

$$\mathbf{v} = \langle v_1, v_2, v_3 \rangle$$

where $v_1$, $v_2$, $v_3$ are the **components** along the $x$-, $y$-, and $z$-axes.

Contrast with a **scalar** (e.g., temperature, mass), which has only magnitude.

## Basic Operations

**Vector addition** is component-wise:
$$\mathbf{a} + \mathbf{b} = \langle a_1 + b_1,\; a_2 + b_2,\; a_3 + b_3 \rangle$$

**Scalar multiplication** scales each component:
$$c\,\mathbf{a} = \langle c a_1,\; c a_2,\; c a_3 \rangle$$

## Magnitude

The **magnitude** (or length) of $\mathbf{v} = \langle v_1, v_2, v_3 \rangle$ is:

$$|\mathbf{v}| = \sqrt{v_1^2 + v_2^2 + v_3^2}$$

## Unit Vectors

A **unit vector** has magnitude 1. To convert any nonzero vector to a unit vector:

$$\hat{\mathbf{v}} = \frac{\mathbf{v}}{|\mathbf{v}|}$$

The standard unit vectors are $\mathbf{i} = \langle 1,0,0\rangle$, $\mathbf{j} = \langle 0,1,0\rangle$, $\mathbf{k} = \langle 0,0,1\rangle$.

Any vector can be written as $\mathbf{v} = v_1\,\mathbf{i} + v_2\,\mathbf{j} + v_3\,\mathbf{k}$.

---

> **Worked Example**
>
> Let $\mathbf{a} = \langle 1, -2, 2 \rangle$.
>
> **(a)** Find $|\mathbf{a}|$.
> $$|\mathbf{a}| = \sqrt{1^2 + (-2)^2 + 2^2} = \sqrt{1 + 4 + 4} = \sqrt{9} = 3$$
>
> **(b)** Find the unit vector $\hat{\mathbf{a}}$.
> $$\hat{\mathbf{a}} = \frac{1}{3}\langle 1, -2, 2 \rangle = \left\langle \frac{1}{3},\, -\frac{2}{3},\, \frac{2}{3} \right\rangle$$
>
> *Check:* $|\hat{\mathbf{a}}| = \sqrt{\frac{1}{9} + \frac{4}{9} + \frac{4}{9}} = \sqrt{\frac{9}{9}} = 1$ ✓
