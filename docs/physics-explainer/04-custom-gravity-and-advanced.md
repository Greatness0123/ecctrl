# 04. Custom Gravity and Advanced Systems

Beyond character movement and grounding, Ecctrl features a suite of advanced, high-performance systems for custom-gravity worlds, physical vehicles, stabilization-driven drones, and frame-rate independent time control.

This document details the mechanics and equations behind these systems.

---

## 1. Position-Based Custom Gravity Fields

Standard physics engines apply a uniform global gravity vector (such as $(0, -9.81, 0)$) to all bodies. For games with spherical worlds, gravity tunnels, or wall-walking, gravity must vary dynamically depending on where an object is located in space.

### The Custom Gravity Pipeline
To prevent the global gravity force and the custom gravity field from stacking, Rapier's global gravity is configured to `[0, 0, 0]` in custom gravity scenes:
```tsx
<Physics gravity={[0, 0, 0]}>
```

1. **Gravity Callback**: A custom function is stored in the `useCustomGravity` Zustand store. This function accepts a 3D position vector and returns the gravity vector at that position:
   $$\mathbf{g} = \mathbf{f}_{\text{gravity}}(\mathbf{x}_{\text{body}})$$
2. **Applying the Force**: In the physics update step, Ecctrl evaluates the gravity vector at the rigid body's coordinate and applies a corresponding physical impulse:
   $$\mathbf{I}_{\text{gravity}} = \mathbf{g}(\mathbf{x}_{\text{body}}) \cdot m_{\text{body}} \cdot \text{scale}_{\text{gravity}} \cdot dt$$
3. **Dynamic Up-Axis Alignment**:
   The controller aligns its reference up-axis to directly oppose the gravity vector:
   $$\hat{\mathbf{u}} = -\frac{\mathbf{g}}{\|\mathbf{g}\|}$$
   The character's camera and look-at controls use this dynamic up-axis, allowing the player to walk upside down, around spherical planets, or up walls seamlessly.

---

## 2. Advanced Vehicle System: ShapeCast Wheels

Ecctrl contains a complete **torque-driven, suspension-aligned vehicle controller** (`EcctrlVehicle` coupled with `ShapeCastWheel`). Rather than hardcoding the car's speed, the car accelerates because wheels apply physical torque and friction impulses to the road.

```
       +------------------------------------+
       |            Car Chassis             |
       +------------------------------------+
              |                      |
              | <-- Suspension       | <-- Suspension
             (O) ShapeCast Wheel    (O) ShapeCast Wheel
```

### Key Components

#### 1. Wheel Shape Casting and Suspension
Each wheel component performs a cylinder or sphere ShapeCast downward from the wheel hub along the suspension direction.
- **Suspension Travel Displacement**: $\Delta x = \text{rayLength} - \text{hitDistance}$
- **Suspension Force** ($F_{\text{suspension}}$): Modeled using a spring-damper equation:
  $$F_{\text{suspension}} = \Delta x \cdot \text{springK} - v_{\text{suspension}} \cdot \text{dampingC}$$
  This force is applied to the car chassis at the wheel's local position, supporting the weight of the vehicle and absorbing bumps.

#### 2. Longitudinal and Lateral Tire Slip
Tire grip is not static; it depends on the sliding speed of the rubber against the asphalt (called **slip**).
- **Longitudinal Slip Ratio** ($\lambda$): The ratio of wheel spin speed to the actual forward speed of the car chassis:
  $$\lambda = \frac{\omega_{\text{wheel}} \cdot r_{\text{wheel}} - v_{\text{forward}}}{\max(|v_{\text{forward}}|, 1e-6)}$$
- **Lateral Slip Angle** ($\alpha$): The angle between the direction the wheel is pointing and the actual velocity vector of the wheel contact point:
  $$\alpha = \arctan\left(\frac{v_{\text{sideways}}}{v_{\text{forward}}}\right)$$

#### 3. Curve Lookup Tables (LUTs) for Tire Grip
Tire rubber behaves nonlinearly: grip peaks at a specific slip ratio (around 10%–20% slip) and then falls off as the tire begins to skid.

Ecctrl bakes these physical curves into **Lookup Tables (LUTs)** at startup (`bakeCurveLUT`) for maximum runtime efficiency.
- During a frame, the controller samples the LUT using the current slip:
  $$\mu_{\text{longitudinal}} = \text{evaluateCurve}(\lambda, \text{lngSlipCurve})$$
  $$\mu_{\text{lateral}} = \text{evaluateCurve}(\alpha, \text{latSlipCurve})$$
- The final friction force applied is proportional to the suspension load times the grip coefficient:
  $$F_{\text{grip}} = F_{\text{suspension}} \cdot \mu_{\text{grip}}$$

This creates realistic tire physics, allowing for wheel spins during hard acceleration, sliding during handbrake turns, and realistic understeer or oversteer when cornering!

---

## 3. Propeller Drones: Flight Mixing Brain

The drone system uses a **thrust-and-torque vector mixing algorithm** to fly. Instead of moving the drone with direct linear speed, each of the four propellers (`ThrustPropeller`) is a physical motor that exerts thrust and reaction torque.

### 1. Forces Involved
- **Thrust Force**: Push upward along the local vertical axis of the propeller:
  $$\mathbf{F}_{\text{thrust}} = \text{throttle} \cdot \text{maxThrust} \cdot \hat{\mathbf{z}}_{\text{prop}}$$
- **Reaction Torque** ($\boldsymbol{\tau}_{\text{reaction}}$): Newton's Third Law for spinning propellers. Spinning a propeller introduces an opposite rotational torque on the drone chassis:
  $$\boldsymbol{\tau}_{\text{reaction}} = \text{throttle} \cdot \text{maxThrust} \cdot \text{torqueRatio} \cdot \hat{\mathbf{z}}_{\text{prop}} \cdot (-1)^{\text{invertTorque}}$$

### 2. Thrust Mixing Matrix
To fly, roll, pitch, or yaw, the drone controller must distribute throttle to each of the motors. This is done by a mixing matrix:
$$\text{throttle}_{i} = \text{throttle}_{\text{vertical}} + (\mathbf{l} \times \mathbf{r})_{i} \cdot \text{demand}_{\text{attitude}} + \text{demand}_{\text{yaw}} \cdot (-1)^{\text{invertTorque}_{i}}$$

Where:
- $\text{throttle}_{\text{vertical}}$ controls height.
- $\mathbf{l} \times \mathbf{r}$ is the leverage torque depending on the propeller's 3D offset relative to the center of mass.
- $\text{demand}_{\text{attitude}}$ coordinates pitch and roll.
- $\text{demand}_{\text{yaw}}$ coordinates rotational spinning.

The drone's internal Proportional-Derivative (PD) controller monitors its current roll, pitch, and yaw, and dynamically adjusts motor outputs to keep the drone perfectly stable.

---

## 4. Deterministic Time Control

In modern physics engines, variable frame rates ($dt$) can cause physics simulations to behave differently (e.g., characters jumping higher on fast PCs). Furthermore, creating bullet-time (slow-motion) effects by modifying the engine's delta time can lead to spring instabilities.

Ecctrl overcomes this via the `TimeControl` component.

1. **Pausing Default Loop**: Rapier's automatic physics update loop is paused:
   ```tsx
   <Physics paused>
   ```
2. **Manual stepping**: `TimeControl` steps the world manually during the R3F `useFrame` callback:
   ```typescript
   world.step(clampedDelta * timeScale);
   ```
3. **Deterministic Timestep**:
   - `clampedDelta` is capped at a maximum (e.g., `1 / 30`) to prevent massive coordinate jumps on sudden frame rate drops.
   - `timeScale` can be varied smoothly (e.g., from `1.0` to `0.1` for bullet time) without affecting the internal physics integration formulas, ensuring stable springs and perfect collision resolving under all speeds.
