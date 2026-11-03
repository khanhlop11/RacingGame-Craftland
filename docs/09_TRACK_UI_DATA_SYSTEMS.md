# Technical Handover — 09: Track, AI, UI, Persistence & Tooling

> **Target Audience:** External Technical Advisor (Independent AI Consultant)  
> **Source Files:** `RaceManager.fcg`, `LoadUI.fcg`, `KartGuidePath.fcg`, `KartRank.fcg`, `CarGhostSync.fcg`, `ProjectSettings/DataStorage.asset`  
> **Classification:** Track Geometry, Autopilot Navigation, HUD Data Flow, Persistence & Tooling  

---

## 1. Track Representation, AI & Visual Effects

### 1.1 Track Geometry & Spline Representation
* **Track Meshes:** The racing circuit is composed of modular 3D road sections, banked turns, mega-ramps, and loops exported as `.fbx` models with static MeshColliders.
* **Waypoint Spline Node Network:**
  * Defined as an ordered list of 3D points: `WaypointNodes List<math.Vector3>` in [`KartGuidePath.fcg`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartGuidePath.fcg).
  * Vehicle progress is tracked via point-to-segment projection:
    $$T = \frac{(\vec{P}_{\text{car}} - \vec{W}_i) \cdot (\vec{W}_{i+1} - \vec{W}_i)}{\|\vec{W}_{i+1} - \vec{W}_i\|^2}$$
  * When Euclidean distance $\|\vec{P}_{\text{car}} - \vec{W}_i\| < 12.0\text{ m}$, the target advances to $W_{i+1}$ ([`RacingMovement.fcg:3605`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3605)).

### 1.2 AI Autopilot Navigation & Rubber-Banding
* **Autonomous Steering:** Inside [`KartGuidePath.fcg:45-90`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartGuidePath.fcg#L45-L90), the autopilot calculates a lookahead target point $12.0\text{m}$ along the track spline ahead of the car's current projected position.
* **Target Yaw Angle:**
  $$\psi_{\text{des}} = \text{math.ATan2}(T_{\text{lookAhead}}.X - P_{\text{car}}.X, T_{\text{lookAhead}}.Z - P_{\text{car}}.Z)$$
  $$\Delta \psi = \text{NormalizeAngle}(\psi_{\text{des}} - \psi_{\text{car}})$$
* **AI Steer Injection:** `AiTurnRate = math.Clamp(deltaPsi * 4.5, -SteerMax, SteerMax)` is injected directly into `PhysicsQuat` ([`RacingMovement.fcg:4705`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L4705)), completely eliminating understeer for AI bots.
* **Rubber-Banding:** Handled in [`RacingMovementServer.fcg:180-205`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovementServer.fcg#L180-L205). If an AI racer trails the player by more than $50\text{m}$, its `AiDesiredSpeedKmh` increases by $+15\%$. If it leads by more than $30\text{m}$, speed is capped at $92\%$ of player top speed.

### 1.3 Particle Effects & Visual Systems
* **Exhaust Nitro Jets (`Nitro_Jet_Exhaust.particles`):** Dual particle systems anchored to rear exhaust bones. Emits high-velocity cyan flame sprites when `IsBoosting == true`.
* **Procedural Skidmarks (`KartSkidmark.fcg:1-96`):** When `IsDrifting == true` and rear wheel sensors are grounded, dynamic ribbon line strips are spawned between consecutive wheel contact positions.
* **Wall Friction Sparks (`Wall_Scrape_Sparks.particles`):** Emitted at raycast contact point $(ActiveWallNormX, Y, ActiveWallNormZ)$ during scraping.

---

## 2. UI Architecture & Screen State Machine

### 2.1 Screen State Machine
The game UI flow is governed by `RaceManager.fcg` and displayed by `LoadUI.fcg`:

```text
[0: LOBBY / ROOM WAITING]
   │ • Intro Flycam circles starting grid (IsIntroCamActive == true)
   │ • Player vehicle controls locked (IsExternallyLocked == true)
   ▼
[1: RACE COUNTDOWN (3 - 2 - 1 - GO)]
   │ • Plays Countdown_Beep.wav and displays animated 3D numbers
   │ • Engine revs audio active; controls remain locked
   ▼
[2: RACING HUD ACTIVE]
   │ • Controls unlocked (IsRaceStarted = true, IsExternallyLocked = false)
   │ • Active HUD: Speedometer needle, digital km/h, Nitro gauge, Lap counter, Minimap
   ▼
[3: RACE FINISH / PODIUM]
   │ • Crosses final lap finish line
   │ • Decelerates vehicle automatically via FinishSmoothFactor
   │ • Displays Results Podium, personal best split time, and return to lobby button
```

### 2.2 HUD Elements & Data Feeds

| HUD UI Element | Update Function | Data Source in Code | Visual Behavior |
| :--- | :--- | :--- | :--- |
| **Speedometer Needle** | `LoadUI::UpdateSpeedometer` | `RacingMovement::CurrentSpeedKmh` | Rotates from $-120^\circ$ ($0\text{ km/h}$) to $+120^\circ$ ($400\text{ km/h}$). |
| **Digital Speed Text** | `LoadUI::UpdateSpeedometer` | `math.Round(CurrentSpeedKmh)` | Displays whole integer text `"XXX km/h"`. |
| **Nitro Gauge Bar** | `LoadUI::UpdateNitroGauge` | `RacingMovement::NitroCharge` | Sets horizontal fill percentage from $0.0$ to $1.0$. |
| **Lap Counter** | `LoadUI::UpdateLapDisplay` | `RaceManager::CurrentLap` | Displays `"LAP X / 3"`. |
| **Rank Badge** | `LoadUI::UpdateRank` | `KartRank::PlayerRank` | Displays `"1st"`, `"2nd"`, etc., updated at 5Hz. |
| **Minimap Blips** | `LoadUI::UpdateMinimap` | `UIKartPos`, `UIKartYaw` | Maps 3D world coordinates to 2D UI minimap image canvas. |

---

## 3. Data Storage, Persistence & Network Protocol

### 3.1 Persistence & DataStorage Schema
Player data persistence is defined in `ProjectSettings/DataStorage.asset`:
* **Module Database Entity:** `T_43_YU_WS_MODULE_DATABASE` (ID: `K8WSZ7M6`).
* **Table Schema (`TimerRank` Sheet):**
  * Key: `PlayerID` (`String`).
  * Column 1: `Time` (`String`): Formatted string of best lap time (`"MM:SS.mmm"`).
  * Column 2: `Attempts` (`Int`): Total number of race attempts recorded.
* **Saving Mechanism:** Cloud database write triggered in [`RaceManager.fcg:530`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RaceManager.fcg#L530) upon crossing the finish line.

### 3.2 The Integer-Scaled Network Serialization Protocol
Craftland Studio compiler contains a critical architectural constraint:
* **The Constraint:** Attempting to pass `entity`, `object`, or `List<object>` across `DispatchEventToEntity` causes compiler crashes (`Important.txt:36`).
* **The Solution:** Telemetry is serialized exclusively into `List<int>` with $1000\times$ fixed-point scaling:
  * $X_{\text{net}} = \text{math.Round}(P_X \times 1000.0) + 1,000,000,000$
  * $Y_{\text{net}} = \text{math.Round}(P_Y \times 1000.0) + 1,000,000,000$
  * $Z_{\text{net}} = \text{math.Round}(P_Z \times 1000.0) + 1,000,000,000$
  * $\text{Yaw}_{\text{net}} = \text{math.Round}(\text{Yaw} \times 100.0)$
* **Client Unpacking:** Receiver strips the $1,000,000,000$ offset and divides by $1000.0$ to reconstruct exact floating-point world coordinates without network packet failure.

---

## 4. Rendering & Performance Hotspots

### 4.1 Scene Setup & Lighting
* **Lighting Model:** Real-time directional sun light with cascaded shadow maps, supplemented by an ambient skybox cubemap.
* **Materials:** Vehicle body uses PBR specular/roughness shaders with metallic flake reflections. Track barriers use emissive materials for nighttime cyberpunk aesthetic.
* **Draw Calls & Mesh Complexity:** Circuit model comprises ~65,000 triangles and ~180 draw calls on mobile hardware.

### 4.2 Critical Performance Hotspots in Hot Loop
1. **Raycast Overhead:** Every frame executes 7 synchronous raycasts: 4 ground suspension probes + 1 forward crash probe + 2 side scrape whiskers ([`RacingMovement.fcg:3980-4370`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3980-L4370)).
2. **String Allocations in UI:** `convert.ToString(math.Round(CurrentSpeedKmh))` inside `OnUpdate` allocates heap strings every frame, generating garbage collector pressure.
3. **Unpooled Skidmark Spawning:** In [`RacingMovement.fcg:4550`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L4550), `start CreateSkidLineAsync` continuously instantiates new entity line segments during drift rather than recycling an object pool.

### 4.3 Profiling & FPS Measurement
* In-engine frame counting is implemented in [`RacingMovement.fcg:3519-3528`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3519-L3528):
  * `FpsFrameCount` increments per tick.
  * When `secNow != FpsLastSec`, `MeasuredFrameRate` calculates real frames rendered per second.
  * Developers can inspect `MeasuredFrameRate` via `ProjectSettings/GameDebugVAR.gdvar`.

---

## 5. Build, Tests & Git History

### 5.1 Compilation & Build Tooling
* **Compiler Path:** `UGCLanguage/release/fccompile_external.exe`
* **Compilation Command:**
  ```powershell
  fccompile_external.exe -i "c:\Max Speed23-07\maxspeed_published\Max Speed\Assets"
  ```
* **Linting & Diagnostics:** Provided via Craftland Studio Language Server (`.craftland/agents/skills/fc-lint-fix/`).

### 5.2 Recent Git Commit History
```text
b4a384a Backup: Save all current FCG files before reference database overhaul
c48ffed backup: toan bo FCG scripts voi Reverse Flycam WP51 va World Waypoints on dinh
a77e2b4 Phuc hoi toan bo tai nguyen du an tu ban1 va giu code 9718aea
9718aea checkpoint: trang thai on dinh truoc khi prompt lon
7081df5 chore: cap nhat chi theo doi cac file fcg
43ef4df ban1
```

