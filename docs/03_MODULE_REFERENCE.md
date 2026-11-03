# Technical Handover — 03: Complete Module Reference

> **Target Audience:** External Technical Advisor (Independent AI Consultant)  
> **Source Directory:** `Assets/Script/FCG/` and `Assets/Script/FCG/Kart/`  
> **Classification:** Comprehensive API Reference, State Storage & Call Graphs  

---

## 1. Primary Game Controllers

### 1.1 `RacingMovement.fcg` (5,857 lines)
* **Purpose:** The monolithic client vehicle controller. Orchestrates input processing, vehicle physics calculations, ground adhesion, drift mechanics, nitro boost levels, collision responses, audio playback, chase camera logic, visual transforms, and HUD updates.
* **Public / Key Lifecycle API:**
  * `func OnStart()`: Initializes vehicle state, caches component references (Rigidbody, Camera, Audio, Transform), resets physics variables, and connects input listeners ([`L3400-3510`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3400-L3510)).
  * `func OnUpdate()`: Main per-frame simulation loop ([`L3516-5854`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3516-L5854)).
  * `func ResetVehicle(pos math.Vector3, rot math.Quaternion)`: Resets position, zeroes all linear and angular velocities, clears drift state, and snaps camera ([`L3350-3390`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3350-L3390)).
  * `func ActivateNitro(tier int)`: Initiates nitro consumption for specified tier (1, 2, or 3) ([`L2450-2510`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L2450-L2510)).
  * `func TriggerWallCrash(hitPoint math.Vector3, hitNormal math.Vector3, impactSpeed float)`: Applies rebound impulse and speed penalty ([`L4390-4450`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L4390-L4450)).
* **Internal State Fields:**
  * `DriveForwardSpeed float`: Current longitudinal speed in physics units/s ([`L72`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L72)).
  * `SmoothedTurnInput float`: Exponentially smoothed steering value $[-1.0, 1.0]$ ([`L88`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L88)).
  * `IsGrounded bool`: True when vertical ground raycasts detect track surface ([`L94`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L94)).
  * `IsDrifting bool`: True when vehicle is in active lateral slide ([`L98`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L98)).
  * `NitroAmount float`: Current nitro tank gauge $[0.0, 100.0]$ ([`L102`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L102)).
  * `UsePredictiveWallCcd bool = false`: Flag controlling continuous collision detection ([`L117`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L117)).
* **Callers & Callees:**
  * *Callers:* Engine Runtime (`OnStart`, `OnUpdate`), UI Buttons (`TurnLeft`, `TurnRight`, `BoostButton`, `BrakeButton`).
  * *Callees:* `KartPhysics`, `KartCameraRuntime`, `KartAudio`, `KartVisual`, `LoadUI`, `RaceManager`.

### 1.2 `RaceManager.fcg` (1,140 lines)
* **Purpose:** Client-side race coordinator managing countdown sequence, checkpoint progression, lap completion, race finish triggers, and split times.
* **Public API:**
  * `func StartCountdown()`: Initiates 3-2-1-GO race start sequence ([`L180-220`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L180-L220)).
  * `func CheckpointTriggered(index int, player entity)`: Validates sequential checkpoint passing ([`L290-340`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L290-L340)).
  * `func CompleteLap()`: Increments lap counter, checks win condition, dispatches lap split ([`L410-460`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L410-L460)).
  * `func FinishRace()`: Locks player controls, shows podium UI, dispatches results to server ([`L520-580`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L520-L580)).
* **Internal State Fields:**
  * `CurrentCheckpoint int`: Index of the next required checkpoint ([`L112`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L112)).
  * `CurrentLap int`: Current active lap number (1 to TotalLaps) ([`L114`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L114)).
  * `TotalLaps int = 3`: Target laps for race completion ([`L116`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L116)).
  * `RaceState int`: Enum representing 0: Waiting, 1: Countdown, 2: Racing, 3: Finished ([`L120`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L120)).
* **Callers & Callees:**
  * *Callers:* `KartCheckpoint`, `RacingMovement`, Server RPCs.
  * *Callees:* `LoadUI`, `RacingMovement`, `RacingMovementServer`.

### 1.3 `LoadUI.fcg` (1,561 lines)
* **Purpose:** Master UI controller handling screen HUD elements: analog speedometer needle rotation, digital speed text, nitro bar fill, lap counter, position rank badge, minimap player blips, and finish screen overlays.
* **Public API:**
  * `func UpdateSpeedometer(speedKmh float)`: Rotates needle and updates digital speed display ([`L210-260`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/LoadUI.fcg#L210-L260)).
  * `func UpdateNitroGauge(current float, max float)`: Adjusts progress fill of nitro meter ([`L290-330`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/LoadUI.fcg#L290-L330)).
  * `func UpdateLapDisplay(currentLap int, totalLaps int)`: Refreshes HUD lap indicator ([`L360-390`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/LoadUI.fcg#L360-L390)).
  * `func UpdateMinimapPosition(worldPos math.Vector3, worldYaw float)`: Transforms world coordinates to 2D minimap canvas coordinates ([`L450-510`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/LoadUI.fcg#L450-L510)).
* **Internal State Fields:**
  * `NeedleStartAngle float = -120.0`: Speedometer zero-speed angle in degrees.
  * `NeedleEndAngle float = 120.0`: Speedometer maximum-speed angle in degrees.
  * `MaxDisplaySpeed float = 400.0`: Speed value corresponding to full needle deflection.
* **Callers & Callees:**
  * *Callers:* `RacingMovement`, `RaceManager`, `KartRank`.
  * *Callees:* Craftland UI Engine API (`ui.SetRotation`, `ui.SetText`, `ui.SetFillAmount`).

### 1.4 `RacingMovementServer.fcg` (365 lines)
* **Purpose:** Authoritative server-side race manager. Tracks connected players, validates checkpoint progression to prevent cheating, and relays compressed opponent transforms to all clients.
* **Public API:**
  * `func OnPlayerJoined(player entity)`: Registers player in room list, assigns spawn grid slot ([`L45-80`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovementServer.fcg#L45-L80)).
  * `func OnClientCheckpointReport(player entity, cpIndex int, timestamp float)`: Validates checkpoint order ([`L120-160`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovementServer.fcg#L120-L160)).
  * `func BroadcastOpponentStates()`: Periodically dispatches positions and rotations of all racers to peers ([`L210-270`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovementServer.fcg#L210-L270)).
* **Internal State Fields:**
  * `PlayerRecords List<PlayerRaceRecord>`: Per-player lap, checkpoint, and timing state.
  * `ServerTickRate float = 0.05`: 20Hz network replication broadcast interval.
* **Callers & Callees:**
  * *Callers:* Engine Server Events, Client RPC Events.
  * *Callees:* `network.BroadcastEventToClient`.

---

## 2. Modular Subsystem Scripts (`Assets/Script/FCG/Kart/`)

### 2.1 Physics & Movement Group
* **`KartPhysics.fcg` (98 lines):**
  * *Purpose:* Adapts calculated vehicle velocities into the engine's `Rigidbody` component.
  * *Key Function:* `ApplyTargetVelocity(targetVel math.Vector3, assist float)` ([`L11-18`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartPhysics.fcg#L11-L18)) computes `(targetVel - currentVel) * assist` and injects via `ForceType.Velocity`.
* **`KartPhysicsMath.fcg` (127 lines):**
  * *Purpose:* Pure mathematical equations for longitudinal vehicle acceleration, aerodynamic drag, rolling resistance, and braking deceleration curves.
  * *Key Function:* `CalcForwardSpeed(currentSpeed float, throttle float, brake float, dt float, config PhysicsConfig) float`.
* **`KartGroundProbe.fcg` (109 lines):**
  * *Purpose:* Executes raycasts downward from four chassis corners (Front-Left, Front-Right, Rear-Left, Rear-Right) to compute average surface distance, ground normal, and airborne status.
* **`KartGroundingMath.fcg` (76 lines):**
  * *Purpose:* Calculates target chassis pitch and roll angles matching the ground surface normal vector.

### 2.2 Drift & Handling Group
* **`KartDriftState.fcg` (62 lines):**
  * *Purpose:* Finite state machine tracking drift status: `None`, `Initiating`, `Active`, `Exiting`, `Failed`.
* **`KartDriftMath.fcg` (119 lines):**
  * *Purpose:* Derives drift slip angle, dynamic lateral slip velocity, and counter-steer yaw rate from steering input and current vehicle velocity.
* **`KartSteeringRuntime.fcg` (83 lines):**
  * *Purpose:* Applies deadzone thresholds, input filtering, and speed-sensitive steering reduction curves.
* **`KartSteeringMath.fcg` (85 lines):**
  * *Purpose:* Computes non-linear steering sensitivity falloff as speed increases:
    $$k_{\text{steer}} = \frac{1.0}{1.0 + \left(\frac{\text{speed}}{\text{SteerDropSpeed}}\right)^2}$$

### 2.3 Camera & Visual Group
* **`KartCameraRuntime.fcg` (232 lines):**
  * *Purpose:* Independent runtime module implementing chase camera tracking, FOV updates, and position smoothing.
* **`KartCameraMath.fcg` (120 lines):**
  * *Purpose:* Contains pure math functions for camera spring-damping, look-at target offsets, and noise-based camera rumble math.
* **`KartVisual.fcg` (160 lines):**
  * *Purpose:* Updates visual mesh bone transforms: spinning front and rear wheels, steering front hubs, tilting chassis on turn/drift.
* **`KartVisualMath.fcg` (196 lines):**
  * *Purpose:* Computes visual roll/pitch offsets and suspension spring compression angles.
* **`LocalRotation.fcg` (200 lines):**
  * *Purpose:* Standalone utility providing local-space coordinate rotation matrices for sub-mesh entities.
* **`SpringCamera.fcg` (115 lines):**
  * *Purpose:* Standalone harmonic oscillator camera implementation for experimental testing.

### 2.4 Collision & Navigation Group
* **`KartCCD.fcg` (112 lines):**
  * *Purpose:* Continuous Collision Detection utility performing sphere/capsule raycast sweeps along the movement velocity vector. (Currently disabled in main loop).
* **`KartGuidePath.fcg` (199 lines):**
  * *Purpose:* Reads track waypoint nodes, computes distance to line, and calculates AI steer correction angles.
* **`KartCheckpoint.fcg` (46 lines):**
  * *Purpose:* Modular trigger script attached to checkpoint collider volumes; dispatches trigger event to `RaceManager`.
* **`KartRank.fcg` (209 lines):**
  * *Purpose:* Compares race progress (lap $\times 10000$ + checkpoint $\times 100$ + distance to next checkpoint) across all active racers to determine leaderboard positions 1st through 8th.

### 2.5 Audio, UI & Effects Group
* **`KartAudio.fcg` (230 lines):**
  * *Purpose:* Adjusts engine loop audio pitch from idle ($0.8\times$) to redline ($2.2\times$), starts tire screech loops during drift, and plays crash impacts.
* **`KartNitro.fcg` (105 lines):**
  * *Purpose:* Manages 3-tier nitro tank logic: Tier 1 (Yellow, 1.2x boost), Tier 2 (Orange, 1.4x boost), Tier 3 (Super Cyan, 1.8x boost).
* **`KartSpeedometer.fcg` (162 lines):**
  * *Purpose:* Formats digital speed strings and computes needle angular deflection.
* **`KartSkidmark.fcg` (96 lines):**
  * *Purpose:* Spawns and chains procedural quad ribbon meshes at tire contact points when `IsDrifting == true`.

### 2.6 Utility & Inactive Stubs
* **`KartMath.fcg` (93 lines):**
  * *Purpose:* General math utility providing vector clamping, `ExpDecay`, `Lerp`, `Slerp`, and angle wrapping.
* **`KartLifecycle.fcg` (40 lines):**
  * *Purpose:* Manages player vehicle spawning, respawning, and destruction sequences.
* **`KartCarFactory.fcg` (17 lines):**
  * *Purpose:* Vehicle prefab instantiation helper stub.
* **`KartNetworkSync.fcg` (13 lines):**
  * *Purpose:* **STUB / INACTIVE**. 13-line placeholder for network replication.
* **`CarGhostSync.fcg` (41 lines):**
  * *Purpose:* Records telemetry frames (Position, Rotation, Speed) to replay buffer for time trial ghost visualization.

---

## 3. Input Button Handlers (`Assets/Script/FCG/`)

All touch buttons are client-side UI scripts listening for touch down/up events and dispatching boolean state changes to `RacingMovement.fcg`:

* **`TurnLeft.fcg` (24 lines):** Sets `TurnLeftPressed = true/false`.
* **`TurnRight.fcg` (24 lines):** Sets `TurnRightPressed = true/false`.
* **`BoostButton.fcg` (39 lines):** Dispatches nitro activation event to `RacingMovement::ActivateNitro()`.
* **`BrakeButton.fcg` (43 lines):** Sets `BrakePressed = true/false`. Triggers drift if steering is active.
* **`ReverseButton.fcg` (21 lines):** Toggles reverse gear state (`IsReversing`).
* **`ResetButton.fcg` (18 lines):** Calls `RacingMovement::ResetVehicle()` to return to last valid track checkpoint.
* **`KartInputActions.fcg` (111 lines):** Bridges keyboard (W, A, S, D, Space, Shift, R) and gamepad inputs to vehicle action flags.

