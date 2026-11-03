# Technical Handover — 08: Input Controls & Chase Camera System

> **Target Audience:** External Technical Advisor (Independent AI Consultant)  
> **Source Files:** `RacingMovement.fcg`, `KartInputActions.fcg`, `KartCameraRuntime.fcg`, `KartCameraMath.fcg`, `ProjectSettings/Camera.asset`  
> **Classification:** Control Signal Latency, Smoothing Cascades, Dual-Pivot Camera Math & Shake Conflicts  

---

## 1. Input Controls & Steering Pipeline

### 1.1 Input Sources & Sampling Mechanisms
Player inputs arrive via two distinct mechanisms:
1. **Touch UI Buttons (Mobile):**
   * Handled by event listener scripts: `TurnLeft.fcg`, `TurnRight.fcg`, `BoostButton.fcg`, `BrakeButton.fcg`.
   * Listen to `ui.OnTouchDown` and `ui.OnTouchUp` events.
   * Directly mutate boolean state flags on `RacingMovementInstance`: `IsTurningLeft`, `IsTurningRight`, `IsBraking`, `IsBoosting`.
2. **Keyboard & Gamepad (PC / Controller):**
   * Configured in `ProjectSettings/InputSystem.asset` and processed in `KartInputActions.fcg`.
   * Sampled every frame inside `RacingMovement.fcg::OnUpdate`:
     * Steer Left: `A` or `LeftArrow` $\implies IsTurningLeft$
     * Steer Right: `D` or `RightArrow` $\implies IsTurningRight$
     * Accelerate: Automatic throttle when race is active (`accelInput = 1.0` at [`L4640`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L4640)).
     * Brake / Drift: `S`, `DownArrow`, or `Space` $\implies IsBraking$
     * Nitro Boost: `LeftShift` or `W` $\implies IsBoosting$
     * Reset / Respawn: `R` $\implies ResetVehicle()$

### 1.2 The Double Exponential Smoothing Cascade (Input Lag Root Cause)
The user reported: *"Driving controls feel unsmooth."*  
**This issue is CONFIRMED BY CODE through a cascaded double exponential filter:**

```text
[Raw Button Press] (0.0 or 1.0)
       │
       ▼
[Filter 1: Input Smoothing] (RacingMovement.fcg:4514-4524)
   SmoothedTurnInput = SmoothedTurnInput + (raw - SmoothedTurnInput) * (dt * turnSharp)
   • turnSharp = 5.5 (or 3.6 when reversing direction)
   • Time constant τ₁ = 1 / 5.5 ≈ 0.182 seconds
       │
       ▼
[Filter 2: Steering Ramp] (RacingMovement.fcg:2588, CalcSteerRate)
   SteerRamp = SteerRamp + (turnInput - SteerRamp) * (dt * SteerRampSharp)
   • SteerRampSharp = 9.0
   • Time constant τ₂ = 1 / 9.0 ≈ 0.111 seconds
       │
       ▼
[Output to Yaw Integration]: PhysicsQuat (RacingMovement.fcg:4710)
```

#### Mathematical Latency Breakdown
Because two low-pass exponential filters are cascaded in series, the step response to a steering press suffers severe cumulative latency:
$$t_{90\%} \approx 2.3 \times \tau_1 + 2.3 \times \tau_2 = 2.3 \times (0.182\text{ s}) + 2.3 \times (0.111\text{ s}) \approx 0.419\text{ s} + 0.255\text{ s} \approx \mathbf{0.674\text{ seconds}}$$
* **Observable Symptom:** When a player presses or releases a turn button, the vehicle takes over **$670\text{ ms}$ (more than half a second)** to reach full steering deflection! 
* When rapidly counter-steering to correct a slide, `turnSharp = 3.6` increases $\tau_1$ to $0.278\text{s}$, producing an agonizing **$890\text{ ms}$ latency**. This makes the vehicle feel sluggish, unresponsive, and disconnected from player touch.
* **Framerate Independence:** Because `dt = 0.033` is hardcoded regardless of real frametime, running at 60 FPS or 120 FPS does not reduce this latency in real-world time.

---

## 2. Chase Camera Architecture & Mathematics

### 2.1 Dual-Pivot Look-At Implementation
The chase camera operates on a dual-pivot look-at architecture implemented in [`RacingMovement.fcg:5650-5735`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L5650-L5735):

1. **Camera Position Calculation:**
   $$\vec{P}_{\text{cam}} = \vec{P}_{\text{car}} + R(q_{\text{cam}}) \cdot \vec{O}_{\text{local}} + R(q_{\text{cam}}) \cdot \vec{O}_{\text{lookahead}}$$
   Where:
   $$\vec{O}_{\text{local}} = \begin{pmatrix} 0.30 \cdot \text{CamDriftSide} + \text{bob}_X \\ \text{BaseCamHeight} + \text{drop} + \text{driftDrop} + \text{nitroDrop} + \text{bob}_Y \\ \text{camBackZ} + \text{bob}_Z \end{pmatrix}$$
   * $\text{BaseCamHeight} = 1.40\text{ m}$.
   * $\text{camBackZ} = -6.60\text{ m} - \text{fovInertia} - \text{CamLagBack} - \text{boostPull} - \text{nitroBackPull}$ (clamped between $-6.60\text{ m}$ and $-7.90\text{ m}$).
2. **Dual-Pivot Target Node (Vehicle Hood Elevation):**
   Instead of looking at the vehicle origin (which positions the car at the bottom of the screen), the camera targets a raised point forward along vehicle heading:
   $$\vec{T}_{\text{lookAt}} = \vec{P}_{\text{car}} + \vec{f}_{\text{cam}} \cdot (4.50\text{ m} + \text{lookAhead}) + \vec{u}_{\text{cam}} \cdot 0.65\text{ m}$$
3. **Aim Vector & LookRotation:**
   $$\vec{d}_{\text{aim}} = \frac{\vec{T}_{\text{lookAt}} - \vec{P}_{\text{cam}}}{\|\vec{T}_{\text{lookAt}} - \vec{P}_{\text{cam}}\|}$$
   $$q_{\text{look}} = \text{math.LookRotation}(\vec{d}_{\text{aim}}, \vec{u}_{\text{cam}})$$
4. **Final Euler Composition (With Roll & Shakes):**
   $$\text{Euler}_{\text{final}} = \begin{pmatrix} \text{Euler}_{\text{look}}.X + \text{BaseCamPitch} + \text{shake}_X \\ \text{Euler}_{\text{look}}.Y + \text{shake}_Y + \text{CamDriftYawInward} \\ \text{Euler}_{\text{look}}.Z + \text{totalRollDeg} + \text{shake}_Z \end{pmatrix}$$
   Where $\text{BaseCamPitch} = 3.5^\circ$ tilts the camera down to elevate the car toward screen center.

---

## 3. Camera Jitter, FOV Mismatch & Shake Conflicts

The user reported: *"The chase camera feels unsmooth."*  
**This issue is CONFIRMED BY CODE through three competing systems:**

### 3.1 Defect 1: Conflicting Dual Shake Systems
* **Script Euler Noise Bob:** [`RacingMovement.fcg:5445-5460, 5630-5653`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L5445-L5460) executes complex high-frequency sinusoidal noise math on Euler angles ($26\text{ Hz}, 29\text{ Hz}, 33\text{ Hz}, 51\text{ Hz}, 67\text{ Hz}$).
* **Native GPU Camera Shake:** Concurrently, [`RacingMovement.fcg:5737, 5743`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L5737) triggers engine native shakes (`camera.PlayCustomCameraShake`) on nitro ignition and wall impacts.
* **Result:** The native camera shake displaces camera projection matrices while the script concurrently oscillates world Euler angles. The two systems fight each other at asynchronous frequencies, producing high-frequency screen shudder and visual stutter.

### 3.2 Defect 2: Angular Velocity Damper Overload
* **Code Proof:** [`KartCameraRuntime.fcg:104-114`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartCameraRuntime.fcg#L104-L114):
  ```fcg
  var camAngleDeltaDeg = math.QuaternionAngle(CamQuat, PrevCamQuat)
  CamAngularVelocity = camAngleDeltaDeg / dt
  camRotSharp = CalcCamRotSharp(CamRotationSharp, CamCatchUpSharp, cameraBoostActive, CamAngularVelocity, CamAngularVelocityCap)
  ```
* **Failure Mechanism:** During sharp turns or drift initiation, `CamAngularVelocity` spikes. `CalcCamRotSharp` dynamically penalizes `camRotSharp`, dropping it down to $2.0$.
* This causes the camera to fall far behind the turn. When angular velocity slows slightly, `camRotSharp` jumps back up to $14.0$, snapping the camera violently forward to catch up. The camera repeatedly cycles between lagging and snapping, producing rhythmic rotational jitter.

### 3.3 Defect 3: Configuration vs Script FOV Mismatch
* **Project Configuration:** `ProjectSettings/Camera.asset:21` configures `CameraFov: 120`.
* **Runtime Script Logic:** [`RacingMovement.fcg:5407`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L5407) hard-caps dynamic FOV at $76.0^\circ$:
  `if targetFOV > 76.0 { targetFOV = 76.0 }`
* **Result:** During scene initialization or before `OnUpdate` executes, the camera renders with extreme $120^\circ$ fisheye distortion, flattening vehicle proportions. When the script kicks in, FOV snaps from $120^\circ$ to $60^\circ - 76^\circ$, causing an abrupt zoom pop.

