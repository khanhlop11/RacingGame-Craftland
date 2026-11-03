# Technical Handover — 04: Comprehensive Feature Catalogue

> **Target Audience:** External Technical Advisor (Independent AI Consultant)  
> **Source Project:** Max Speed  
> **Classification:** Full Gameplay Feature Discovery & Status Matrix  

---

## 1. Feature Status Summary Table

| Category | Feature Name | Status | Primary Source Files |
| :--- | :--- | :---: | :--- |
| **Race Rules** | Lap & Checkpoint Tracking | **COMPLETE** | `RaceManager.fcg:290`, `KartCheckpoint.fcg:20` |
| **Race Rules** | 3-2-1-GO Race Countdown | **COMPLETE** | `RaceManager.fcg:180`, `LoadUI.fcg:540` |
| **Race Rules** | Dynamic Race Ranking (1st–8th) | **COMPLETE** | `KartRank.fcg:60`, `LoadUI.fcg:410` |
| **Race Rules** | Respawn / Reset to Track | **PARTIAL** | `RacingMovement.fcg:3350`, `ResetButton.fcg:10` |
| **Vehicle Systems**| Longitudinal Acceleration & Brakes | **COMPLETE** | `RacingMovement.fcg:4630`, `KartPhysicsMath.fcg:40` |
| **Vehicle Systems**| Speed-Sensitive Steering | **COMPLETE** | `RacingMovement.fcg:2580`, `KartSteeringMath.fcg:25` |
| **Vehicle Systems**| Inward Drift Kinematics | **COMPLETE** | `RacingMovement.fcg:2650`, `KartDriftMath.fcg:35` |
| **Vehicle Systems**| 3-Tier Nitro Boost System | **COMPLETE** | `RacingMovement.fcg:2450`, `KartNitro.fcg:20` |
| **Vehicle Systems**| Vehicle Customization & Garage | **NOT FOUND** | N/A |
| **Track & Physics**| Continuous Collision Detection (CCD) | **BROKEN** | `RacingMovement.fcg:117`, `KartCCD.fcg:30` |
| **Track & Physics**| Wall Scraping & Bounce Response | **PARTIAL** | `RacingMovement.fcg:4320`, `KartPhysics.fcg:60` |
| **Track & Physics**| 4-Point Ground Suspension Probe | **COMPLETE** | `RacingMovement.fcg:3980`, `KartGroundProbe.fcg:25` |
| **Track & Physics**| Airborne Ramp Jumps | **COMPLETE** | `RacingMovement.fcg:4080`, `KartPhysicsMath.fcg:80` |
| **Track & Physics**| Speed Boost Pads | **COMPLETE** | `RacingMovement.fcg:4210` |
| **AI & Autopilot** | AI Autopilot Waypoint Following | **COMPLETE** | `RacingMovement.fcg:4530`, `KartGuidePath.fcg:50` |
| **AI & Autopilot** | Dynamic AI Opponent Rubber-Banding | **PARTIAL** | `KartGuidePath.fcg:120`, `RacingMovementServer.fcg:180` |
| **Replay & Ghosts**| Time Trial Ghost Recording & Playback | **COMPLETE** | `CarGhostSync.fcg:15`, `RacingMovement.fcg:3558` |
| **Multiplayer**    | Client Physics Authority Replication | **COMPLETE** | `RacingMovementServer.fcg:210`, `RacingMovement.fcg:3310`|
| **Multiplayer**    | Network Sync Stub Module | **DEAD CODE** | `KartNetworkSync.fcg:1-13` |
| **Camera**         | Chase Camera Dynamic Rig | **PARTIAL** | `RacingMovement.fcg:5370`, `KartCameraRuntime.fcg:80`|
| **Camera**         | Screen Rumble & Camera Shake | **BROKEN** | `RacingMovement.fcg:5445`, `ProjectSettings/Camera.asset`|
| **UI & HUD**       | Analog Needle & Digital Speedometer | **COMPLETE** | `LoadUI.fcg:210`, `KartSpeedometer.fcg:40` |
| **UI & HUD**       | Nitro Gauge Bar | **COMPLETE** | `LoadUI.fcg:290` |
| **UI & HUD**       | Minimap with Dynamic Track Blips | **COMPLETE** | `LoadUI.fcg:450`, `ProjectSettings/MinimapV2.asset` |
| **Audio**          | Engine RPM Pitch Modulation | **COMPLETE** | `KartAudio.fcg:45`, `RacingMovement.fcg:5200` |
| **Audio**          | Drift Screech & Nitro Blast SFX | **COMPLETE** | `KartAudio.fcg:110` |
| **VFX**            | Nitro Exhaust Plumes & Skidmarks | **COMPLETE** | `KartVisual.fcg:80`, `KartSkidmark.fcg:30` |
| **Persistence**    | Player Bests & Cloud Storage | **PARTIAL** | `ProjectSettings/DataStorage.asset:10` |
| **Testing / Debug**| Debug Overlay & Dev Variables | **COMPLETE** | `ProjectSettings/GameDebugVAR.gdvar:1-111` |
| **Weapons/Combat** | Kart Weapons / Projectiles / Items | **NOT FOUND** | N/A |
| **Economy/IAP**    | Currency, Shop, In-App Purchases | **NOT FOUND** | N/A |

---

## 2. Detailed Feature Catalogues

### 2.1 3-Tier Nitro Boost System
* **Player Experience:** Pressing or holding the boost button triggers a supersonic thrust burst. Exhaust pipes ignite with luminous flame jets, the camera FOV widens, engine audio screams, and vehicle speed rapidly surges past standard top speed.
* **Status:** **COMPLETE** ([`RacingMovement.fcg:2450-2550`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L2450-L2550), [`KartNitro.fcg:1-105`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartNitro.fcg#L1-L105)).
* **State Owned:** `NitroAmount float` ($0.0$ to $100.0$), `NitroTier int` ($0, 1, 2, 3$), `IsNitroActive bool`.
* **Trigger Conditions:** Player clicks UI `BoostButton` or presses `Shift/Space`. Requires `NitroAmount > 15.0`.
* **Key Constants:**
  * Base Top Speed: $360.0\text{ display km/h}$ ($2.50\text{ physics units/s}$).
  * Tier 1 Multiplier: $1.15\times$ ($414\text{ km/h}$, yellow flame).
  * Tier 2 Multiplier: $1.30\times$ ($468\text{ km/h}$, orange flame).
  * Tier 3 Multiplier: $1.50\times$ ($540\text{ km/h}$, cyan Mach diamond flame).
  * Consumption Rate: $25.0\text{ units/second}$ (4.0s full tank duration).
  * Drift Charge Rate: $+12.0\text{ units/second}$ during active drift.
* **Known Issues:** Activating nitro immediately zeroes `lateralSlip` and `DriftSlipMemory` ([`RacingMovement.fcg:2698-2701`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L2698-L2701)), which artificially snaps car inertia and eliminates realistic vehicle weight feel.

### 2.2 Drift & Handling Kinematics
* **Player Experience:** Tapping brake while turning kicks the rear chassis outward into a slide. The vehicle tilts into the apex, tire smoke and sparks billow from the rear tires, rubber skidmark ribbons paint the road, and the nitro bar rapidly charges.
* **Status:** **COMPLETE** ([`RacingMovement.fcg:2650-2790`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L2650-L2790), [`KartDriftMath.fcg:1-119`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartDriftMath.fcg#L1-L119)).
* **State Owned:** `IsDrifting bool`, `DriftDirection float` ($-1.0$ left, $+1.0$ right), `DriftAngle float`, `DriftChargeAccumulator float`.
* **Trigger Conditions:** `BrakePressed == true` and `abs(SmoothedTurnInput) > 0.3` and `IsGrounded == true`.
* **Key Constants:**
  * `MinDriftSpeed = 80.0\text{ display km/h}` ($0.55\text{ units/s}$).
  * `MaxDriftYawAngle = 42.0^\circ`.
  * `DriftFrictionMultiplier = 0.88` (reduced forward drag to maintain slide speed).
* **Known Issues:** The vehicle suffers from outward centrifugal slip on sharp hairpins because it lacks an inward centripetal velocity assist vector (as seen in Ace Racer, where inward assist pulls the car along corner apexes).

### 2.3 Wall Collision & Continuous Collision Detection (CCD)
* **Player Experience:** When striking track guardrails or bridge barriers, the car is supposed to scrape smoothly along the wall with friction sparks or bounce backward on severe head-on impacts.
* **Status:** **BROKEN / INCOMPLETE** ([`RacingMovement.fcg:117-118, 4320-4495`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L117-L118)).
* **Mechanism:**
  * In `RacingMovement.fcg:117-118`: `UsePredictiveWallCcd bool = false` and `UseCustomWallScrapeResponse bool = false`.
  * The continuous capsule sweep routine in `KartCCD.fcg` is **completely bypassed**.
  * Wall detection falls back to a single center raycast at height $0.8\text{m}$.
* **Symptoms:** At high speeds ($>300\text{ km/h}$), the vehicle advances more than $0.08\text{m}$ to $0.15\text{m}$ per frame, punching completely through thin walls and mesh barriers before the single raycast or discrete physics collider can register contact.

### 2.4 Chase Camera System
* **Player Experience:** Third-person chase camera positioned behind the vehicle, adjusting pitch on slopes, trailing on turns, widening FOV on nitro boost, and shaking on rough impacts.
* **Status:** **PARTIAL / JITTERY** ([`RacingMovement.fcg:5370-5540`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L5370-L5540), [`KartCameraRuntime.fcg:1-232`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartCameraRuntime.fcg#L1-L232)).
* **State Owned:** `CamPos math.Vector3`, `CamQuat math.Quaternion`, `CamFOV float`, `CamShakeOffset math.Vector3`.
* **Key Constants:**
  * Default Camera Distance: $3.2\text{m}$ behind, $1.4\text{m}$ height.
  * Engine Default FOV: $120.0^\circ$ ([`ProjectSettings/Camera.asset:21`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/ProjectSettings/Camera.asset#L21)) vs Script Dynamic FOV: $55.0^\circ - 76.0^\circ$ ([`RacingMovement.fcg:5407`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L5407)).
* **Known Issues:** Severe conflict between script sinusoidal Euler shake math and native GPU camera shakes (`camera.PlayCustomCameraShake`), compounded by angular velocity damper fighting steering slerp, creating jerky camera stutter during turns.

### 2.5 AI Autopilot & Waypoint Navigation
* **Player Experience:** When autopilot mode is toggled, or for AI bot opponents, the car navigates track centerlines autonomously, cornering and avoiding course boundaries.
* **Status:** **COMPLETE** ([`KartGuidePath.fcg:1-199`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartGuidePath.fcg#L1-L199), [`RacingMovement.fcg:4530-4560`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L4530-L4560)).
* **State Owned:** `WaypointNodes List<math.Vector3>`, `CurrentWaypointIdx int`, `AutopilotSteerTarget float`.
* **Mechanism:** Samples line segment between `WaypointNodes[i]` and `WaypointNodes[i+1]`. Computes perpendicular deviation distance and projects lookahead point $12.0\text{m}$ forward to calculate desired steering angle.

### 2.6 Ghost Telemetry & Replay System
* **Player Experience:** Players can race against a translucent "ghost" car representing their best lap time.
* **Status:** **COMPLETE** ([`CarGhostSync.fcg:1-41`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/CarGhostSync.fcg#L1-L41), [`RacingMovement.fcg:3558-3595`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3558-L3595)).
* **Mechanism:** Records timestamp, position, quaternion, and speed into an in-memory telemetry array at 10Hz. On replay, interpolates between telemetry frames and applies coordinates to the ghost entity.

### 2.7 Multiplayer Replication & Network Protocol
* **Player Experience:** 8-player peer-to-peer race room with synchronized start countdown, real-time leaderboard positions, and visual opponent movement.
* **Status:** **COMPLETE (Client Authority Architecture)** ([`RacingMovementServer.fcg:1-365`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovementServer.fcg#L1-L365)).
* **Mechanism:** Server cannot run physics; each client simulates their own car and broadcasts integer-encoded telemetry packages ($1000\times$ scaled coordinates) to the server every $50\text{ms}$ ($20\text{Hz}$). The server validates checkpoint timestamps and broadcasts opponent positions.
* **Dead Code:** `KartNetworkSync.fcg` (13 lines) is an abandoned stub file.

