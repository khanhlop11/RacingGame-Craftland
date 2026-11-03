# Technical Handover — 06: Vehicle Model & Physics Dynamics

> **Target Audience:** External Technical Advisor (Independent AI Consultant)  
> **Source Files:** `RacingMovement.fcg`, `KartPhysics.fcg`, `KartPhysicsMath.fcg`, `KartDriftMath.fcg`, `KartSteeringMath.fcg`  
> **Classification:** Vehicle Kinematic Formulation, Mathematical Equations & Tuning Ledger  

---

## 1. Physical State Variables

All vehicle physical state variables are managed inside `RacingMovement.fcg` and synchronized with the engine's `Rigidbody` component:

| Variable Name | Type | Physical Unit | Source Citation | Physical & Gameplay Meaning |
| :--- | :---: | :---: | :--- | :--- |
| `DriveForwardSpeed` | `float` | $\text{units/s}$ ($\text{m/s}$) | [`RacingMovement.fcg:72`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L72) | Authoritative scalar longitudinal velocity along vehicle heading vector. |
| `PhysicsQuat` | `math.Quaternion` | Unit Quaternion | [`RacingMovement.fcg:172`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L172) | Authoritative orientation quaternion of the vehicle chassis. |
| `VelocityQuat` | `math.Quaternion` | Unit Quaternion | [`RacingMovement.fcg:174`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L174) | Direction vector of movement; trails `PhysicsQuat` during drifts. |
| `FacingAngle` | `float` | Degrees ($^\circ$) | [`RacingMovement.fcg:168`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L168) | Current vehicle yaw angle in world space ($0^\circ = +Z$). |
| `SmoothedTurnInput` | `float` | Normalized $[-1, 1]$ | [`RacingMovement.fcg:88`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L88) | Exponentially filtered player steering input. |
| `DriftSlipMemory` | `float` | $\text{units/s}$ | [`RacingMovement.fcg:184`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L184) | Retained lateral inertia persisting through and out of drifts. |
| `BoostPower` | `float` | Normalized $[0, 1]$ | [`RacingMovement.fcg:202`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L202) | Filtered dynamic boost intensity multiplier. |
| `NitroCharge` | `float` | Percent $[0, 100]$ | [`RacingMovement.fcg:102`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L102) | Current fuel available in the nitro tank. |
| `IsGrounded` | `bool` | Boolean | [`RacingMovement.fcg:94`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L94) | True if ground raycasts intersect track surface. |
| `IsDrifting` | `bool` | Boolean | [`RacingMovement.fcg:98`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L98) | True if vehicle is actively sliding laterally. |
| `AirTime` | `float` | Seconds ($\text{s}$) | [`RacingMovement.fcg:214`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L214) | Accumulated duration since leaving ground contact. |

---

## 2. Motion Model & Mathematical Formulation

### 2.1 Velocity Scaling & Arcade Calibration
The physical simulation translates between internal physics units ($\text{units/s}$) and displayed user speeds ($\text{km/h}$) using `SpeedUnitScale = 4.0` ([`RacingMovement.fcg:49`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L49)):
$$\text{Display Speed (km/h)} = v_{\text{phys}} \times \text{SpeedUnitScale} \times 36.0 = v_{\text{phys}} \times 144.0$$
$$v_{\text{phys}} = \frac{\text{Display Speed}}{144.0}$$
* **Top Speed:** $365.0\text{ km/h} \implies v_{\text{phys}} = 2.5347\text{ units/s}$.
* **Nitro Boost Speed:** $405.0\text{ km/h} \implies v_{\text{phys}} = 2.8125\text{ units/s}$.

### 2.2 Longitudinal Acceleration & Deceleration Equations
Longitudinal speed updates via exponential asymptotic Lerp integration in `KartPhysicsMath.fcg:15`:
$$v(t + \Delta t) = v(t) + (v_{\text{target}} - v(t)) \times \left(1.0 - e^{-k_{\text{rate}} \cdot \Delta t}\right)$$
* When accelerating forward ($accelInput = 1.0$):
  $$k_{\text{rate}} = \text{AccelLerpRate} \times \text{launchRamp} \times \text{tractionFactor}$$
  Where $\text{launchRamp} = 1.0 + 1.2 \times \left(1.0 - \frac{v}{v_{\text{top}}}\right)$ provides high initial launch acceleration.
* When braking or reversing ($IsReversing == true$):
  $$v(t + \Delta t) = \text{LerpToSpeed}(v(t), 0.0, \text{BrakeLerpRate}, \Delta t)$$
  With $\text{BrakeLerpRate} = 4.5\text{ s}^{-1}$.

### 2.3 Steering & Yaw Dynamics
Player steering commands are processed through speed-sensitive attenuation curves in `KartSteeringMath.fcg`:
$$\omega_{\text{yaw}} = u_{\text{steer}} \times \text{SteerBase} \times k_{\text{speedFalloff}} \times k_{\text{counterSteer}}$$
1. **Speed Falloff Curve:** Above $120\text{ km/h}$, steering authority decays to preserve high-speed control:
   $$k_{\text{speedFalloff}} = \max\left(\text{HighSpeedSteerFloor}, 1.0 - \frac{\text{speed} - 120.0}{365.0 - 120.0} \times (1.0 - \text{HighSpeedSteerFloor})\right)$$
   With $\text{HighSpeedSteerFloor} = 0.68$ (retains 68% steering authority at top speed).
2. **Counter-Steer Sensitivity:** When steering opposite to current drift slip velocity:
   $$k_{\text{counterSteer}} = 1.0 + \text{CounterSteerBoost} = 1.45$$
3. **Quaternion Orientation Integration:**
   $$q_{\text{phys}}(t + \Delta t) = q_{\text{phys}}(t) \times \text{QuaternionFromAxisAngle}(\vec{n}_{\text{ground}}, \omega_{\text{yaw}} \cdot \Delta t)$$

### 2.4 Drift Kinematics & Velocity Alignment
During active drifts (`isDriftActive == true`), vehicle orientation and movement direction decouple:
1. **Chassis Yaw Angle:** Yaw rotates aggressively into the turn ($\text{DriftSteer} = 2.55$).
2. **Velocity Slerp:** The velocity direction vector tracks chassis heading at rate `velAlignSharp`:
   $$q_{\text{vel}}(t + \Delta t) = \text{Slerp}(q_{\text{vel}}(t), q_{\text{phys}}(t), \Delta t \times k_{\text{align}})$$
   Where $k_{\text{align}}$ scales dynamically with forward velocity:
   $$k_{\text{align}} = \text{DriftVelAlignLow} + (\text{DriftVelAlignHigh} - \text{DriftVelAlignLow}) \times \text{clamp}\left(\frac{v - 0.5 v_{\text{top}}}{v_{\text{top}}}, 0, 1\right)$$
   With $\text{DriftVelAlignLow} = 16.0$ and $\text{DriftVelAlignHigh} = 26.0$.
3. **Lateral Slip Velocity:**
   $$v_{\text{lat}} = \vec{v} \cdot \vec{r}_{\text{chassis}}$$
   During drift, $v_{\text{lat}}$ is damped by $\text{DriftGrip} = 0.86$, creating visual sideways skidding without complete loss of control.

### 2.5 Airborne & Jump Dynamics
When `IsGrounded == false`:
1. Ground suspension force drops to zero ($F_{\text{susp}} = 0$).
2. Downward gravitational acceleration is injected via `KartPhysics.ApplyAirGravity`:
   $$\vec{F}_{\text{down}} = -m \cdot (\text{Gravity} + \text{ExtraAirGravity}) \cdot \vec{u}_{Y} = -m \cdot (20.0 + 5.0) \cdot \vec{u}_{Y} = -25.0 \cdot m \cdot \vec{u}_{Y}$$
3. Velocity alignment sharpness drops to $\text{VelAlignAir} = 0.5$, allowing vehicle momentum to fly realistically along the launch vector without snapping to mid-air steering inputs.

---

## 3. Comprehensive Tuning Constants Ledger

| Parameter Constant | Value | Unit | File & Line Citation | Gameplay & Architectural Meaning |
| :--- | :---: | :---: | :--- | :--- |
| `SpeedUnitScale` | `4.0` | Scalar | [`RacingMovement.fcg:49`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L49) | Arcade speed multiplier ($1\text{ unit/s} = 144\text{ display km/h}$). |
| `TopSpeed` | `365.0` | $\text{km/h}$ | [`RacingMovement.fcg:51`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L51) | Maximum straight-line driving speed. |
| `BoostTopSpeed` | `405.0` | $\text{km/h}$ | [`RacingMovement.fcg:133`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L133) | Maximum speed achievable during nitro boost. |
| `ReverseSpeed` | `30.0` | $\text{km/h}$ | [`RacingMovement.fcg:55`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L55) | Maximum speed when in reverse gear. |
| `AccelLerpRate` | `0.18` | $\text{s}^{-1}$ | [`RacingMovement.fcg:52`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L52) | Exponential acceleration convergence rate. |
| `BoostLerpRate` | `1.2` | $\text{s}^{-1}$ | [`RacingMovement.fcg:53`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L53) | Acceleration rate when nitro boost is activated. |
| `BrakeLerpRate` | `4.5` | $\text{s}^{-1}$ | [`RacingMovement.fcg:54`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L54) | Deceleration rate when applying brakes. |
| `ReverseLerpRate` | `1.0` | $\text{s}^{-1}$ | [`RacingMovement.fcg:56`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L56) | Acceleration rate when reversing. |
| `Grip` | `0.94` | Ratio | [`RacingMovement.fcg:57`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L57) | Lateral tire friction grip during normal grip driving. |
| `Steer` | `1.85` | $\text{rad/s}$ | [`RacingMovement.fcg:58`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L58) | Base steering yaw rate authority for normal cornering. |
| `DriftSteer` | `2.55` | $\text{rad/s}$ | [`RacingMovement.fcg:59`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L59) | Enhanced steering yaw rate authority during drifts. |
| `TurnSpeedRatio` | `0.95` | Ratio | [`RacingMovement.fcg:62`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L62) | Top speed speed cap factor when cornering normally (95%). |
| `DriftSustainRatio` | `0.88` | Ratio | [`RacingMovement.fcg:63`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L63) | Minimum retained forward speed fraction during drift (88%). |
| `DriftBrakeMult` | `1.6` | Scalar | [`RacingMovement.fcg:64`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L64) | Drift deceleration rate multiplier. |
| `DriftEnterSharp` | `7.5` | $\text{s}^{-1}$ | [`RacingMovement.fcg:71`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L71) | Transition speed into drift slip angle. |
| `DriftExitSharp` | `5.2` | $\text{s}^{-1}$ | [`RacingMovement.fcg:72`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L72) | Recovery speed when exiting drift to straight driving. |
| `DriftGrip` | `0.86` | Ratio | [`RacingMovement.fcg:138`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L138) | Reduced tire grip coefficient during active drift. |
| `VelAlignGrounded` | `16.0` | $\text{s}^{-1}$ | [`RacingMovement.fcg:82`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L82) | Slerp rate of velocity vector to heading on ground. |
| `VelAlignDrift` | `18.0` | $\text{s}^{-1}$ | [`RacingMovement.fcg:83`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L83) | Slerp rate of velocity vector during active drift. |
| `VelAlignAir` | `0.5` | $\text{s}^{-1}$ | [`RacingMovement.fcg:84`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L84) | Slerp rate of velocity vector when airborne. |
| `HighSpeedSteerStart` | `120.0` | $\text{km/h}$ | [`RacingMovement.fcg:90`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L90) | Speed threshold where steering sensitivity begins to decay. |
| `HighSpeedSteerFloor` | `0.68` | Ratio | [`RacingMovement.fcg:91`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L91) | Minimum retained steering authority at top speed (68%). |
| `SteerRampSharp` | `9.0` | $\text{s}^{-1}$ | [`RacingMovement.fcg:92`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L92) | Exponential ramp sharpness for steering input. |
| `Gravity` | `20.0` | $\text{m/s}^2$ | [`RacingMovement.fcg:99`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L99) | Downward acceleration applied when airborne. |
| `ExtraAirGravity` | `5.0` | $\text{m/s}^2$ | [`RacingMovement.fcg:100`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L100) | Supplementary gravity to snap airborne karts downward. |
| `GroundGravity` | `38.0` | $\text{m/s}^2$ | [`RacingMovement.fcg:101`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L101) | Downward force pinning vehicle to track surface normal. |
| `DriftDownforceBonus` | `42.0` | $\text{m/s}^2$ | [`RacingMovement.fcg:102`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L102) | Additional normal downforce applied during drifts. |
| `SuspSpring` | `60.0` | $\text{N/m}$ | [`RacingMovement.fcg:107`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L107) | Suspension spring stiffness coefficient. |
| `SuspDamping` | `55.0` | $\text{N}\cdot\text{s/m}$ | [`RacingMovement.fcg:108`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L108) | Suspension shock absorber damping coefficient. |
| `NitroChargeRate` | `95.0` | $\text{units/s}$ | [`RacingMovement.fcg:134`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L134) | Charge accumulation rate during drifts and stunt jumps. |
| `NitroDrainRate` | `7.0` | $\text{units/s}$ | [`RacingMovement.fcg:135`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L135) | Consumption rate during active nitro boost (~14.2s tank). |

