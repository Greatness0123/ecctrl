# 01. The Floating Spring-Damper System

One of the key reasons the character controller in Ecctrl feels incredibly smooth and never slips through the floor or jitters on stairs is its **Floating Spring-Damper System**.

Instead of letting the character's physical capsule collider drag directly on the ground, the controller **hovers** (floats) slightly above the surface. This document details the mathematics, mechanics, ground detection routines, and clipping prevention logic of this system.

---

## 1. Why Hover? The Concept of a Floating Capsule

In traditional game physics, a character collider is often a simple capsule or cylinder that sits directly on a floor collider. This approach suffers from several issues:
1. **Friction Jitter**: Slidings on flat or uneven surfaces can cause micro-collisions, resulting in jittery motion.
2. **Step/Stair Jitter**: Colliding with stair risers causes the capsule to bump upward violently unless complex step-climbing code is written.
3. **Slope Slide**: Capsules sitting on a slope will naturally slide down unless friction is set to infinity, which prevents movement.

**Ecctrl's solution** is to decouple the hard physics collision from the ground tracking.
- The character's rigid body is kept hovering at a target height above the ground.
- A virtual "spring" pushes the rigid body up to its target float height.
- If the character falls from a great height or is crushed, the physical capsule collider (`CapsuleCollider`) acts as a hard safety barrier to prevent clipping through the floor.

```
       +-----------------------+
       |   Character Capsule   |
       |       Collider        |
       +-----------------------+
                   |
                   | <--- capsuleHalfHeight
                   v
             (Ray Origin)
                   |
                   | <--- Hover Spring (springK & dampingC)
                   |
~~~~~~~~~~~~~~~~~~~X~~~~~~~~~~~~~~~~~~~  <--- Target Float Height (floatHeight)
                   |
                   | <--- Ground Contact
=======================================  <--- Ground Level
```

---

## 2. Ground Detection: ShapeCast vs. RayCast

Ground detection is performed at the beginning of each frame in the `floatCharacter()` callback. Ecctrl supports two modes configured via the `groundDetection` prop:

### A. RayCast Mode (`groundDetection="rayCast"`)
In RayCast mode, a single mathematical line is cast downward along the gravity/up axis:
- **Origin**: Positioned at `currentPos + characterYAxis * rayOriginOffset` (where `rayOriginOffset` defaults to `-capsuleHalfHeight`).
- **Direction**: Directly opposite to the reference up axis (`-referenceUpAxis`).
- **Length**: Configured by `rayLength` (defaults to `capsuleRadius + 1`).

While computationally cheap, a single ray can slip through tiny cracks or seams between colliders, and it doesn't represent the physical width of the character.

### B. ShapeCast Mode (`groundDetection="shapeCast"`) — *Default*
In ShapeCast mode, a 3D sphere/ball shape (using `rayRadius`) is swept downward along the ray direction:
- It uses Rapier's `world.castShape` API.
- Since it sweeps a volume rather than a line, it catches steps, ledges, and slopes accurately, preventing the character from falling off tiny gaps.

### Walkable Slope Filtering & Fallback
Both modes filter out steep slopes to prevent characters from "clinging" or floating on walls:
1. When a ground hit occurs, the angle between the contact normal ($\hat{\mathbf{n}}$) and the reference up-axis ($\hat{\mathbf{u}}$) is calculated:
   $$\theta_{\text{slope}} = \arccos(\hat{\mathbf{n}} \cdot \hat{\mathbf{u}})$$
2. If $\theta_{\text{slope}} \le \text{slopeMaxAngle}$, the hit is considered **walkable** and registered.
3. If the hit is too steep ($\theta_{\text{slope}} > \text{slopeMaxAngle}$), the hit is ignored.
4. To prevent getting stuck on steep slopes, the controller performs a **center ray fallback** (`findWalkableCenterRayHit()`). It scans directly below the character to find a walkable surface. If none is found, the character is flagged as not grounded (`isOnGround.current = false`), causing them to slide off or fall naturally.

---

## 3. The Floating Spring-Damper Equation

Once a walkable ground hit is confirmed, Ecctrl simulates a virtual spring and damper along the up-axis. This is governed by a **one-dimensional damped harmonic oscillator** equation.

### The Physics Formulas
The total floating force ($F_{\text{float}}$) along the reference up-axis is calculated as:
$$F_{\text{float}} = F_{\text{spring}} - F_{\text{damping}}$$

#### 1. Spring Force ($F_{\text{spring}}$)
Based on Hooke's Law, the spring force is proportional to the displacement ($\Delta x$) from the target float height:
$$\Delta x = x_{\text{target}} - x_{\text{actual}}$$
$$F_{\text{spring}} = \Delta x \cdot k$$

Where:
- $x_{\text{target}}$ is the target floating distance `groundFloatingDistance` (e.g., `rayRadius + floatHeight` for ShapeCast).
- $x_{\text{actual}}$ is the hit distance `groundHitDistance` (time of impact from the cast).
- $k$ is the spring stiffness coefficient (`springK`, defaults to `80`).

#### 2. Damping Force ($F_{\text{damping}}$)
To prevent infinite oscillation (bouncing), we must damp the system based on velocity. The damping force is proportional to the character's relative vertical velocity:
$$v_{\text{rel}} = \mathbf{v}_{\text{relative}} \cdot \hat{\mathbf{u}}$$
$$F_{\text{damping}} = v_{\text{rel}} \cdot c$$

Where:
- $\mathbf{v}_{\text{relative}}$ is the character's velocity relative to the ground (subtracting platform velocity if on a moving platform).
- $\hat{\mathbf{u}}$ is the unit up-axis vector.
- $c$ is the damping coefficient (`dampingC`, defaults to `6`).

#### 3. Converting Force to Impulse
Since Rapier processes impulses ($I = F \cdot dt$), the floating force is converted to a vector impulse ($\mathbf{I}_{\text{float}}$) and applied to the rigid body:
$$\mathbf{I}_{\text{float}} = (F_{\text{spring}} - F_{\text{damping}}) \cdot dt \cdot \hat{\mathbf{u}}$$

Where $dt$ is the physics engine timestep (`world.timestep`).

### Implementation Code (from `Ecctrl.tsx`)
```typescript
springDistVec.current.copy(referenceUpAxis).multiplyScalar(groundFloatingDistance.current - groundHitDistance.current)
dampingVelVec.current.copy(relativeVel.current).projectOnVector(referenceUpAxis);

floatingForce.current.subVectors(
  springDistVec.current.multiplyScalar(springK),
  dampingVelVec.current.multiplyScalar(dampingC)
)

floatingImpulse.current.copy(floatingForce.current).multiplyScalar(world.timestep);

// During jump startup, skip downward adhesion that can cancel the jump.
if (jumpActive.current && floatingImpulse.current.dot(referenceUpAxis) < 0) {
  floatingImpulse.current.set(0, 0, 0)
}

if (!body.isSleeping()) {
  body.applyImpulse(floatingImpulse.current, false);
}
```

---

## 4. Platform Force Interaction & Newton's Third Law

When a character stands on a dynamic rigid body (like a box, a vehicle, or a physics-driven bridge), they should exert weight onto that object. This is a direct implementation of **Newton's Third Law: Action and Reaction**.

### Apply Counter Mass (`applyCounterMass`)
If `applyCounterMass` is enabled, Ecctrl calculates the combined force of gravity and the floating support impulse, and applies an equal and opposite downward impulse to the standing body at the exact `standingPoint`:

1. **Floating Force Reaction**: The magnitude of the upward spring-damper impulse:
   $$I_{\text{float\_mag}} = \max(-\mathbf{I}_{\text{float}} \cdot \hat{\mathbf{g}}, 0)$$
2. **Gravity Weight**: The static weight of the character over the timestep:
   $$I_{\text{weight}} = m_{\text{character}} \cdot \|\mathbf{g}\| \cdot dt$$
3. **Mass Ratio Falloff**: To prevent heavy characters from violently flipping lightweight dynamic objects, a curve (`massRatioFallOffCurve`) scales down the counter impulse if the platform is too light:
   $$\text{ratio} = \text{clamp}\left(\frac{m_{\text{platform}}}{m_{\text{character}}}, 0, 1\right)$$
   $$\text{scale} = \text{evaluateCurve}(\text{ratio})$$

The final counter impulse is applied to the platform:
$$\mathbf{I}_{\text{counter}} = \mathbf{\hat{g}} \cdot \max(I_{\text{float\_mag}}, I_{\text{weight}}) \cdot \text{scale}$$

This guarantees that boxes slide down slopes when stood on, and physical planks bend realistically under the player's weight.

---

## 5. Clipping Prevention & Ground Tolerance

Why doesn't the model clip or pass through the ground even when subjected to extreme external forces?

1. **Dual-Layer Defense**:
   - **Layer 1 (Soft Spring)**: The spring-damper keeps the body floating under normal conditions.
   - **Layer 2 (Hard Collider)**: If an external force (e.g., falling from a high skybox or a heavy object falling on the player) overcomes the spring force, the physical `CapsuleCollider` makes hard contact with the ground collider. Because they are both solid rigid bodies in Rapier, they cannot intersect.
2. **Ray Hit Forgiveness (`rayHitForgiveness`)**:
   - In fast-moving environments (like elevators or bumpy terrain), the ground distance might briefly exceed the ideal floating height.
   - `rayHitForgiveness` (default `0.28`) provides a safety buffer. If the ground is within `groundFloatingDistance + rayHitForgiveness`, the character is still marked as `isOnGround = true`, maintaining traction and preventing jittery state changes.
