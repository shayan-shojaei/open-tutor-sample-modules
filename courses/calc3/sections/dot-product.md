# The Dot Product

## Definition

Given two vectors $\mathbf{a} = \langle a_1, a_2, a_3 \rangle$ and $\mathbf{b} = \langle b_1, b_2, b_3 \rangle$ in $\mathbb{R}^3$, their **dot product** is the scalar:

$$\mathbf{a} \cdot \mathbf{b} = a_1 b_1 + a_2 b_2 + a_3 b_3$$

The result is always a **number** (scalar), not a vector.

## Geometric Interpretation

The dot product has an elegant geometric meaning:

$$\mathbf{a} \cdot \mathbf{b} = |\mathbf{a}||\mathbf{b}|\cos\theta$$

where $\theta$ is the angle between the two vectors ($0 \leq \theta \leq \pi$).

This gives us a powerful test for **orthogonality**: two nonzero vectors are perpendicular if and only if $\mathbf{a} \cdot \mathbf{b} = 0$.

## Projections

The **scalar projection** of $\mathbf{b}$ onto $\mathbf{a}$ is:

$$\text{comp}_{\mathbf{a}}\mathbf{b} = \frac{\mathbf{a} \cdot \mathbf{b}}{|\mathbf{a}|}$$

The **vector projection** of $\mathbf{b}$ onto $\mathbf{a}$ is:

$$\text{proj}_{\mathbf{a}}\mathbf{b} = \frac{\mathbf{a} \cdot \mathbf{b}}{|\mathbf{a}|^2}\,\mathbf{a}$$

---

> **Worked Example 1** — Computing a Dot Product
>
> Find $\mathbf{u} \cdot \mathbf{v}$ where $\mathbf{u} = \langle 3, -1, 2 \rangle$ and $\mathbf{v} = \langle 2, 4, -1 \rangle$.
>
> **Solution:**
> $$\mathbf{u} \cdot \mathbf{v} = (3)(2) + (-1)(4) + (2)(-1) = 6 - 4 - 2 = 0$$
>
> Since $\mathbf{u} \cdot \mathbf{v} = 0$, the vectors are **orthogonal**.

---

> **Worked Example 2** — Finding the Angle Between Vectors
>
> Find the angle between $\mathbf{a} = \langle 1, 2, 2 \rangle$ and $\mathbf{b} = \langle 3, 0, 4 \rangle$.
>
> **Solution:**
>
> Step 1 — Compute the dot product:
> $$\mathbf{a} \cdot \mathbf{b} = (1)(3) + (2)(0) + (2)(4) = 3 + 0 + 8 = 11$$
>
> Step 2 — Compute the magnitudes:
> $$|\mathbf{a}| = \sqrt{1^2 + 2^2 + 2^2} = \sqrt{9} = 3, \quad |\mathbf{b}| = \sqrt{9 + 0 + 16} = \sqrt{25} = 5$$
>
> Step 3 — Apply the formula:
> $$\cos\theta = \frac{11}{3 \cdot 5} = \frac{11}{15} \implies \theta = \arccos\!\left(\frac{11}{15}\right) \approx 42.8°$$
