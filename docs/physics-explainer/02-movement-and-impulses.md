# 02. Movement and Impulses

In Ecctrl, character movement is completely **physics-driven**. Unlike arcade-style controllers that directly teleport the character by changing its position coordinates, Ecctrl applies continuous **forces and impulses** to a dynamic Rapier rigid body.

This document explains the physics of how keyboard or joystick inputs are translated into aligned forces, how sliding is controlled, how slopes are climbed, and how the capsule stays upright.

---

## 1. Input Translation and Camera-Relative Alignment

When a player presses WASD or uses a virtual joystick, the raw input is represented as digital booleans or analog coordinates. To move the character in a way that feels natural, these inputs must be aligned relative to the camera view and orthogonal to the world gravity field.

### Mathematical Alignment Steps
1. **Camera Direction**: Retrieve the camera's current viewing direction vector ($\mathbf{d}_{\text{cam}}$) using Three.js `camera.getWorldDirection`.
2. **Right Vector**: Compute the camera's right vector ($\mathbf{r}_{\text{cam}}$) via the cross product of the camera direction and camera up vector ($\mathbf{u}_{\text{cam}}$):
   $$\mathbf{r}_{\text{cam}} = \mathbf{d}_{\text{cam}} \times \mathbf{u}_{\text{cam}}$$
3. **Up-Axis Orthogonal Forward Vector**: Calculate the movement forward vector ($\mathbf{f}_{\text{mov}}$) to be strictly orthogonal to the character's reference up axis ($\hat{\mathbf{u}}$):
   $$\mathbf{f}_{\text{mov}} = \hat{\mathbf{u}} \times \mathbf{r}_{\text{cam}}$$
4. **Up-Axis Orthogonal Right Vector**: Recompute the orthogonal rightward vector ($\mathbf{r}_{\text{mov}}$):
   $$\mathbf{r}_{\text{mov}} = \mathbf{f}_{\text{mov}} \times \hat{\mathbf{u}}$$
5. **Construct Input Direction**: Scale these vectors by the user's directional inputs ($x$ and $y$) and normalize the result to obtain the final input direction vector ($\hat{\mathbf{d}}_{\text{input}}$):
   $$\mathbf{v}_{\text{input}} = \mathbf{f}_{\text{mov}} \cdot y + \mathbf{r}_{\text{mov}} \cdot x$$
   $$\hat{\mathbf{d}}_{\text{input}} = \frac{\mathbf{v}_{\text{input}}}{\|\mathbf{v}_{\text{input}}\Vert}$$

This ensures that regardless of the gravity field orientation (e.g. wall walking or walking on a planetoid), the character always moves relative to the screen direction.

---

## 2. Slope Vector Projection

If a character moves horizontally into a slope, they will collide with it and lose momentum unless their velocity is redirected. To solve this, Ecctrl projects the input direction onto the slope plane.

```
       Up Axis (u)
          ^      Normal (n)
          |     ^
          |   /
          |  /  <-- Slope angle (theta)
          | /
----------+-------------> Aligned Slope Direction (movingDirection)
        /
       / <-- Slope Plane
      /
     /
```

1. **Slope Normal**: Retrieve the contacted slope normal vector ($\hat{\mathbf{n}}$) from the shape/ray cast.
2. **Slope Angle in Front**: Calculate the slope angle along the input direction. This is done by projecting the slope normal onto the input vector:
   $$\theta_{\text{front}} = -\arcsin(\hat{\mathbf{n}} \cdot \hat{\mathbf{d}}_{\text{input}})$$
3. **Rotation Axis**: Find the axis of rotation perpendicular to both the input direction and the up-axis:
   $$\mathbf{a}_{\text{rot}} = \hat{\mathbf{d}}_{\text{input}} \times \hat{\mathbf{u}}$$
4. **Aligned Movement Vector**: Rotate the input direction vector around this axis by the slope angle to produce the final aligned moving direction ($\hat{\mathbf{d}}_{\text{moving}}$):
   $$\hat{\mathbf{d}}_{\text{moving}} = \mathcal{R}(\mathbf{a}_{\text{rot}}, \theta_{\text{front}}) \cdot \hat{\mathbf{d}}_{\text{input}}$$

This aligned vector points directly up or down the slope, allowing the character to glide smoothly over slopes without bumping or losing speed.

---

## 3. Sideways Velocity Rejection

When a physical body with inertia turns a corner, it tends to slide sideways (like a car drifting). In platformers or action games, players expect tight, responsive controls with minimal drift. Ecctrl achieves this through **Sideways Velocity Rejection**.

Each frame, the controller decomposes the character's planar relative velocity ($\mathbf{v}_{\text{plane}}$) into two components:
1. **Desired Velocity** ($\mathbf{v}_{\text{des}}$): The velocity component projected along the input direction.
2. **Sideways/Reject Velocity** ($\mathbf{v}_{\text{reject}}$): The orthogonal drift component.

$$\mathbf{v}_{\text{des}} = (\mathbf{v}_{\text{plane}} \cdot \hat{\mathbf{d}}_{\text{input}}) \cdot \hat{\mathbf{d}}_{\text{input}}$$
$$\mathbf{v}_{\text{reject}} = \mathbf{v}_{\text{plane}} - \mathbf{v}_{\text{des}}$$

To cancel the drift, the sideways component is scaled by a rejection factor and subtracted from the impulse calculation:
$$\mathbf{I}_{\text{reject\_correction}} = -\mathbf{v}_{\text{reject}} \cdot \text{rejectVelFactor}$$

By default, `rejectVelFactor = 1`, which completely cancels sideways drift on the ground, creating immediate, snappy, and responsive cornering.

---

## 4. The Acceleration, Deceleration, and Movement Impulse

To achieve the target velocity (either `maxWalkVel` or `maxRunVel`), Ecctrl computes the delta velocity needed and applies it as a force-aligned impulse.

The base impulse needed is:
$$\mathbf{I}_{\text{base}} = \left( \hat{\mathbf{d}}_{\text{moving}} \cdot v_{\text{target}} \right) - \mathbf{v}_{\text{plane}}$$

Where $v_{\text{target}}$ is either walking speed or running speed.

The final impulse applied to the rigid body is formulated as:
$$\mathbf{I}_{\text{move}} = \left( \mathbf{I}_{\text{base}} - \mathbf{v}_{\text{reject}} \cdot \text{rejectVelFactor} \right) \cdot \text{multiplier} \cdot dt_{\text{corr}}$$

Where:
- $\text{multiplier} = m_{\text{body}} \cdot \text{accDeltaTime} \cdot \mu_{\text{friction}}$
- $\mu_{\text{friction}}$ is `slideFrictionCoef` when on the ground, or `airDragFactor` when airborne.
- $dt_{\text{corr}}$ is the frame rate correction factor (`60 * world.timestep`).

### Offset Impulse Point
Instead of applying the impulse at the center of mass (which can look static), the impulse is applied at a point offset vertically along the up-axis (`moveImpulsePointOffset`, default `0.5`):
$$\mathbf{p}_{\text{impulse}} = \mathbf{x}_{\text{body}} + \hat{\mathbf{y}}_{\text{local}} \cdot \text{moveImpulsePointOffset}$$

Because the force is applied above the center of mass, it introduces a subtle physical torque, causing the character model to lean slightly into the direction of movement, which looks incredibly lifelike!

---

## 5. Auto-Upright Balancing Torque

A critical challenge of physics-driven character controllers is keeping the character upright. Because the capsule is a dynamic rigid body, any collision or slope contact would normally knock it over. Ecctrl solves this by applying an continuous **Auto-Balance Torque**.

The auto-upright balancing system acts like a **rotational proportional-derivative (PD) controller**.

```
             Up Axis (u)
                ^    /  Local Y Axis (y_local)
                |   /
                |  /  <-- Angular Error (theta)
                | /
                |/
       +--------+--------+
       |     Capsule     |
       +-----------------+
```

### The Balance Formulas
1. **Angular Error Vector** ($\boldsymbol{\theta}_{\text{err}}$): Calculated as the cross product of the character's local vertical axis ($\hat{\mathbf{y}}_{\text{local}}$) and the world/custom reference up axis ($\hat{\mathbf{u}}$):
   $$\boldsymbol{\theta}_{\text{err}} = \hat{\mathbf{y}}_{\text{local}} \times \hat{\mathbf{u}}$$
2. **Proportional Torque**: Pushes the capsule back to alignment:
   $$\mathbf{T}_{\text{spring}} = \boldsymbol{\theta}_{\text{err}} \cdot \text{autoBalanceSpringK}$$
3. **Derivative (Damping) Torque**: Resists angular velocity to prevent wild rocking or spinning:
   $$\mathbf{T}_{\text{damping}} = \boldsymbol{\omega}_{\text{plane}} \cdot \text{autoBalanceDampingC}$$
   Where $\boldsymbol{\omega}_{\text{plane}}$ is the planar angular velocity.
4. **Total Balancing Torque**:
   $$\boldsymbol{\tau}_{\text{balance}} = \left( \mathbf{T}_{\text{spring}} - \mathbf{T}_{\text{damping}} \right) \cdot dt_{\text{corr}}$$

This rotational torque is applied each frame via `body.applyTorqueImpulse()`. It acts as an invisible gyroscope, keeping the capsule perfectly perpendicular to the gravity field under all conditions while remaining reactive to physical impacts.
