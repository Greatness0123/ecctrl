# 03. Reactive Animation

Unlike many traditional game engines where character animations drive the physical movement of the character in the world (known as **Root Motion**), Ecctrl uses a purely **Reactive Animation System**.

The physics engine is the absolute source of truth. The physical capsule moves, slides, falls, and jumps according to rigid body dynamics, and the skeletal animation system simply "reacts" and matches the character's physical state.

---

## 1. The Myth of Direct Animation-Driven Movement

In root-motion setups, animation clips dictate the displacement of the character mesh, and the physics collider is forced to follow. While this ensures foot-sliding is minimized, it makes physics interactions (like hitting a wall, sliding down a slope, or riding a moving elevator) extremely difficult to synchronize.

**Ecctrl's paradigm is the opposite**:
- **Source of Truth**: The `RigidBody` capsule handles all collisions and velocity.
- **Mesh Attachment**: The visual Three.js mesh group is nested directly inside the `RigidBody` component. It is carried along by the physics simulation.
- **Animation Reaction**: We monitor the real-time physical properties of the capsule (speed, vertical velocity, grounded status) and select the corresponding animation clip to match.

---

## 2. Reactive Animation Coupling (The State Resolver)

The component `EcctrlAnimationStateController` reads the `EcctrlHandle` ref on every frame. It acts as a bridge between the physical state of the capsule and the visual animation state.

```
+--------------------------------------------------+
|               Rapier RigidBody                   |
|  (linvel, isOnGround, isFalling, runActive)      |
+--------------------------------------------------+
                         |
                         v  [useFrame reads handle]
+--------------------------------------------------+
|      EcctrlAnimationStateController              |
|  (evaluates conditions & resolves state string)   |
+--------------------------------------------------+
                         |
                         v  [publishes to store]
+--------------------------------------------------+
|           useEcctrlAnimationStore                |
|  ("IDLE" | "WALK" | "RUN" | "JUMP_START", etc.)  |
+--------------------------------------------------+
                         |
                         v  [listens to store]
+--------------------------------------------------+
|             Character Model Mesh                 |
|  (crossfades skeletal clips inside Three.js)     |
+--------------------------------------------------+
```

### The State Decision Matrix
The default resolver, `resolveEcctrlAnimationState()`, runs the following logic to decide which animation state string to publish to the Zustand store:

```typescript
export const resolveEcctrlAnimationState = (ctx: EcctrlAnimationStateContext): EcctrlAnimationState => {
  const { isOnGround, wasOnGround, isFalling, isMoving, runActive, jumpActive } = ctx;

  // 1. If jump was just triggered while grounded
  if (jumpActive && wasOnGround) return "JUMP_START";

  // 2. If we just landed on the ground from the air
  if (isOnGround && !wasOnGround) return "JUMP_LAND";

  // 3. If we are grounded
  if (isOnGround) {
    if (isMoving) {
      return runActive ? "RUN" : "WALK";
    }
    return "IDLE";
  }

  // 4. If we are in the air
  return isFalling ? "JUMP_FALL" : "JUMP_IDLE";
};
```

---

## 3. How Skeletal Animations are Loaded and Blended

Once the Zustand store updates the active state, the `AnimatedCharacterModel` component receives the new state and plays the corresponding clip using `@react-three/drei` and Three.js animation APIs.

### The Animation Asset (`AnimationLibrary.glb`)
The skeletal animation data is **not** procedural code. The keyframe tracks are baked inside an external GLB asset:
- **Location**: `public/AnimationLibrary.glb`
- This file is preloaded using `useGLTF.preload("/AnimationLibrary.glb")`.
- It contains the bone hierarchy (skeleton) and clips:
  - `Idle_Loop`
  - `Walk_Loop`
  - `Jog_Fwd_Loop` (RUN)
  - `Jump_Start`
  - `Jump_Loop` (JUMP_IDLE and JUMP_FALL)
  - `Jump_Land`

### Cross-Fading and State Blending
To prevent sudden, unnatural snapping between different states (e.g., from running directly to idle), Three.js `AnimationMixer` cross-fades between clips over a short duration.

#### Loopable Transitions (e.g., Run, Walk, Idle)
For continuous animations, the action is reset and faded in over `0.2` seconds:
```javascript
const nextActionName = statusToActionMap[actionStore];
const nextAction = actions[nextActionName];

nextAction
  .reset()
  .crossFadeFrom(actions[prevActionName], 0.2)
  .play();
```

#### One-Time Actions (e.g., Jump Start, Jump Land)
For actions that should only play once, the animation is clamped at the end and is prevented from looping:
```javascript
nextAction
  .reset()
  .crossFadeFrom(actions[prevActionName], 0.1)
  .setLoop(THREE.LoopOnce, 1)
  .play();

nextAction.clampWhenFinished = true;
```

---

## 4. Key Advantages of Reactive Animations

1. **Perfect Physics Interaction**: The model's collisions are always accurate. It will never clip into walls or doors because the collider has physical size, while the mesh is merely a visual follower.
2. **Network Sync Simplicity**: In multiplayer games, you only need to sync the character's rigid body position, rotation, and input states. Every client can run the local state resolver to play the correct animations automatically, saving massive amounts of bandwidth.
3. **Procedural Adjustments**: Because the bones are driven by an animation mixer, you can easily overlay procedural modifications (such as Inverse Kinematics for feet alignment on slopes, or look-at head angles) on top of the physics-driven movement.
