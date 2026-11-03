# Technical Handover — 01: Overview & Project Inventory

> **Target Audience:** External Technical Advisor (Independent AI Consultant)  
> **Source Project:** Max Speed (`c:\Max Speed23-07\maxspeed_published\Max Speed`)  
> **Language / Compiler:** FCG (Free Fire Craftland Scripting Language, Compiler v1.14–v1.17.1)  
> **Classification:** High-Speed Arcade Racing Game (Custom Client Physics & Camera)  

---

## 1. System Overview

### 1.1 What the Game Is
**Max Speed** is a high-speed arcade racing game built inside **Craftland Studio** (Garena Free Fire UGC platform). Players pilot supercars across custom-modeled stunt tracks with looping geometry, jumps, and dynamic obstacle courses. 
While Craftland Studio provides base entity management, asset loading, and network synchronization, **the vehicle dynamics, steering feel, drift kinematics, ground adherence, collision scraping, race timing, checkpoint tracking, and chase camera were constructed from scratch in FCG code**.

### 1.2 Technology Stack & Execution Environment
* **Scripting Language:** FCG (`.fcg` / `.fcc`), an object-oriented, statically typed language compiled via `fccompile_external.exe` (or `fccompile.exe`). Syntax resembles C# and Go (`var name Type = val`, `func Name(params) ReturnType`).
* **Runtime Engine:** Garena Craftland Studio engine (proprietary C++ / Unity-derived runtime).
* **Rendering API:** Direct3D 11 / OpenGL ES 3.0 / Vulkan (abstracted under Craftland Studio engine).
* **External Dependencies:** No external third-party package managers (`npm`, `nuget`, `pip`). Scripting relies solely on Craftland engine modules: `math`, `physics`, `camera`, `input`, `ui`, `network`, `time`, `game`, `audio`.
* **Execution Architecture:**
  * `[platform_client]`: Executes all vehicle simulation, input sampling, physics impulse injection, raycasts, collision resolution, local particles, audio, and chase camera math. The engine forbids accessing `entity<Rigidbody>` on `[platform_server]`.
  * `[platform_server]`: Manages race state, lap verification, checkpoint progression validation, room countdown, and client RPC relays (`RacingMovementServer.fcg`).

### 1.3 Target Hardware & Framerate Assumptions
* **Target Platforms:** Mobile devices (Android, iOS) and PC Client (Windows).
* **Target Framerates:** 30 FPS (low-end mobile), 60 FPS (standard mobile/PC), 120 FPS (high-end PC).
* **Critical Design Flaw in Timing:** The codebase hardcodes `dt = 0.033` (30.3 FPS assumption) for core movement updates in [`RacingMovement.fcg:3548`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3548) despite measuring `realDt` from `time.GetDeltaTime()`. At 60 FPS or 120 FPS, the simulation advances in accelerated time, distorting vehicle dynamics.

### 1.4 How to Build and Run
1. **Editor Execution:** Launch Craftland Studio, open project `main.gproj`, load scene, and press **Play (F5)**.
2. **Command-Line Compilation:**
   ```powershell
   & "C:\Users\Admin\AppData\Local\Programs\Craftland Studio\resources\LocalData\Utilities\UGCLanguage\release\fccompile_external.exe" -i "c:\Max Speed23-07\maxspeed_published\Max Speed\Assets"
   ```
3. **Build Artifacts:** Compiled bytecode is cached in `Temp/UGCLanguage/editorGen/` and `Temp/UGCLanguage/fccache/`.

### 1.5 Coordinate Systems & Physical Conventions
* **World Space:** Left-handed Cartesian coordinate system (Unity convention).
  * $+X$: World Right (East)
  * $+Y$: World Up
  * $+Z$: World Forward (North)
* **Rotations:**
  * Euler Angles: Measured in degrees ($^\circ$). Order: Pitch ($X$), Yaw ($Y$), Roll ($Z$).
  * Forward Heading ($Yaw = 0^\circ$): Faces directly along World $+Z$ (`math.Vector3(0, 0, 1)`).
  * Quaternions: Implemented via `math.Quaternion(x, y, z, w)` and converted via `math.EulerToQuaternion()`.
* **Units of Distance & Speed Scaling:**
  * Geometry Units: $1.0\text{ unit} \approx 1.0\text{ meter}$.
  * Physical Velocity: Measured in units/second ($\text{m/s}$).
  * **The 40x Speed Inflation Disconnect:** The speedometer displays speed scaled by `SpeedUnitScale = 4.0` ([`RacingMovement.fcg:49`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L49)):
    $$\text{Display Speed (km/h)} = \text{speedMagnitude} \times 4.0 \times 36.0 = \text{speedMagnitude} \times 144.0$$
    Standard physical conversion ($1.0\text{ m/s} = 3.6\text{ km/h}$) is inflated by **40.0x**. Thus, when the UI reads $360\text{ km/h}$, the actual in-engine velocity is only $2.5\text{ m/s}$ ($2.5\text{ units/frame-sec}$)!

---

## 2. Directory Tree & Line Counts

```text
Max Speed/
├── Assets/
│   ├── Animation/              # Character & Kart Skeletal Animations (.ac, .fbx)
│   ├── Audio/                  # Engine loops, drift screech, nitro whoosh, UI sounds (.wav, .mp3)
│   ├── BuiltInAC/              # Craftland character avatar controllers (.ac)
│   ├── Materials/              # Vehicle paints, track asphalt, neon emissives (.mat)
│   ├── Models/                 # 3D Meshes: Supercar body, wheels, tracks, obstacles (.fbx)
│   ├── Particles/              # Particle systems: Nitro jets, tire smoke, sparks (.particles)
│   ├── Prefabs/                # Entity templates: Vehicle root, checkpoints, pickups (.prefab)
│   ├── Script/
│   │   ├── ECA/                # Craftland visual scripting (.eca)
│   │   │   └── HideUiPlayer.eca            # Event-Condition-Action UI toggler (871 lines)
│   │   └── FCG/                # Source code in FCG language
│   │       ├── Kart/           # Modularized subsystem scripts (23 files, ~3,200 lines)
│   │       ├── BoostButton.fcg             # Mobile boost input handler (39 lines)
│   │       ├── BrakeButton.fcg             # Mobile brake/reverse input handler (43 lines)
│   │       ├── CameraSetup.fcg             # Scene camera initialization (54 lines)
│   │       ├── CarGhostSync.fcg            # Ghost car replay telemetry (41 lines)
│   │       ├── clientCamera.fcg            # Client camera stub (13 lines)
│   │       ├── KartInputActions.fcg        # Input mappings (111 lines)
│   │       ├── KartRank.fcg                # Race leaderboard sorting (209 lines)
│   │       ├── LoadUI.fcg                  # HUD, Speedometer, Nitro Bar, Minimap (1,561 lines)
│   │       ├── LocalRotation.fcg           # Wheel rotation utility (200 lines)
│   │       ├── RaceEvents.fcg              # Custom event definitions (4 lines)
│   │       ├── RaceHUD.fcg                 # HUD overlay helper (29 lines)
│   │       ├── RaceManager.fcg             # Race lifecycle, checkpoints, laps (1,140 lines)
│   │       ├── RacingMovement.fcg          # Monolithic vehicle controller & loop (5,857 lines)
│   │       ├── RacingMovementServer.fcg    # Server race authority & relay (365 lines)
│   │       ├── ResetButton.fcg             # Track reset UI button (18 lines)
│   │       ├── ReverseButton.fcg           # Reverse toggle UI button (21 lines)
│   │       ├── SpringCamera.fcg            # Camera spring-damper experiment (115 lines)
│   │       ├── TurnLeft.fcg                # Mobile steer left button (24 lines)
│   │       └── TurnRight.fcg               # Mobile steer right button (24 lines)
│   ├── Textures/               # Albedo, normal, specular, UI icons (.png, .jpg)
│   └── UI/                     # UI Layout definitions & sprites
├── ProjectSettings/            # Engine configuration assets (.asset, .cproj, .bp)
│   ├── Camera.asset            # Engine FOV, clipping planes (41 lines)
│   ├── DataStorage.asset       # Server cloud database schemas (58 lines)
│   ├── InputSystem.asset       # Virtual key bindings (54 lines)
│   ├── Physics.asset           # Engine gravity, layer collision matrix (445 lines)
│   └── PlayerSetting.asset     # Player spawn, camera presets (135 lines)
├── AGENTS.md                   # AI constitution & developer rules (69 lines)
├── fcconfig.json               # FCG compiler configuration (10 lines)
└── main.gproj                  # Craftland project root manifest (10 lines)
```

---

## 3. Phase 0: Complete Source & Config Coverage Ledger

Every source code, configuration file, and data asset within the repository was cataloged and analyzed. Generated folders (`Temp/`, `Libraries/`, `Cache/`, `.git/`) were excluded.

| File Path | Lines | Bytes | Role & Handover Coverage Mapping |
| :--- | :---: | :---: | :--- |
| `Assets/Script/FCG/RacingMovement.fcg` | 5,857 | 297,029 | Monolithic vehicle physics, main loop, collision, camera. Covered in Sec 2, 4, 6, 7, 8, 9, 10, 16, 17. |
| `Assets/Script/FCG/LoadUI.fcg` | 1,561 | 79,209 | HUD, speedometer gauge, nitro meter, minimap blips, lap timer. Covered in Sec 4, 12. |
| `Assets/Script/FCG/RaceManager.fcg` | 1,140 | 47,976 | Checkpoint progression, lap tracking, race state machine, countdown. Covered in Sec 4, 11, 13. |
| `Assets/Script/FCG/RacingMovementServer.fcg` | 365 | 18,428 | Server-side player tracking, checkpoint validation, opponent sync. Covered in Sec 4, 13. |
| `Assets/Script/FCG/KartRank.fcg` | 209 | 9,234 | Race ranking calculation, split times, distance to leader. Covered in Sec 4, 11. |
| `Assets/Script/FCG/LocalRotation.fcg` | 200 | 7,578 | Visual wheel spin and steering angle rotation utility. Covered in Sec 4, 7. |
| `Assets/Script/FCG/SpringCamera.fcg` | 115 | 4,063 | Standalone spring-damper camera prototype. Covered in Sec 4, 10. |
| `Assets/Script/FCG/KartInputActions.fcg` | 111 | 3,267 | Raw input key mapping and action dispatcher. Covered in Sec 4, 9. |
| `Assets/Script/FCG/CameraSetup.fcg` | 54 | 2,234 | Scene camera attachment and priority setter. Covered in Sec 4, 10. |
| `Assets/Script/FCG/BrakeButton.fcg` | 43 | 1,490 | Mobile touch UI brake button event dispatcher. Covered in Sec 4, 9. |
| `Assets/Script/FCG/CarGhostSync.fcg` | 41 | 1,606 | Ghost replay data recorder and playback handler. Covered in Sec 4, 11. |
| `Assets/Script/FCG/BoostButton.fcg` | 39 | 1,148 | Mobile touch UI nitro boost button event dispatcher. Covered in Sec 4, 9. |
| `Assets/Script/FCG/RaceHUD.fcg` | 29 | 1,172 | Minimalist HUD display helper. Covered in Sec 4, 12. |
| `Assets/Script/FCG/TurnLeft.fcg` | 24 | 678 | Mobile touch UI steer left button handler. Covered in Sec 4, 9. |
| `Assets/Script/FCG/TurnRight.fcg` | 24 | 689 | Mobile touch UI steer right button handler. Covered in Sec 4, 9. |
| `Assets/Script/FCG/ReverseButton.fcg` | 21 | 628 | Mobile touch UI reverse toggle handler. Covered in Sec 4, 9. |
| `Assets/Script/FCG/ResetButton.fcg` | 18 | 506 | Mobile touch UI car reset / respawn button handler. Covered in Sec 4, 9. |
| `Assets/Script/FCG/clientCamera.fcg` | 13 | 329 | Minimalist client camera initialization stub. Covered in Sec 4, 10. |
| `Assets/Script/FCG/RaceEvents.fcg` | 4 | 114 | Global event constants. Covered in Sec 4. |
| `Assets/Script/FCG/Kart/KartAudio.fcg` | 230 | 11,162 | Engine RPM pitch, drift screech, nitro audio management. Covered in Sec 4, 11. |
| `Assets/Script/FCG/Kart/KartCameraRuntime.fcg` | 232 | 11,127 | Modular chase camera execution loop. Covered in Sec 4, 10. |
| `Assets/Script/FCG/Kart/KartCameraMath.fcg` | 120 | 4,500 | Mathematical camera damping, look-at, and shake calculations. Covered in Sec 4, 10. |
| `Assets/Script/FCG/Kart/KartGuidePath.fcg` | 199 | 8,646 | Track waypoint path navigation and AI steer calculations. Covered in Sec 4, 11. |
| `Assets/Script/FCG/Kart/KartVisualMath.fcg` | 196 | 6,538 | Wheel rotation, chassis roll/pitch visual transform math. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartSpeedometer.fcg` | 162 | 6,294 | Needle rotation math and digital speed display formatting. Covered in Sec 4, 12. |
| `Assets/Script/FCG/Kart/KartVisual.fcg` | 160 | 7,944 | Visual wheel transforms and mesh shake runtime. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartPhysicsMath.fcg` | 127 | 5,475 | Engine power curve, aerodynamic drag, brake force math. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartDriftMath.fcg` | 119 | 5,489 | Drift slip angle, counter-steering, friction reduction math. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartCCD.fcg` | 112 | 4,850 | Continuous collision detection ray/capsule sweep math. Covered in Sec 4, 8. |
| `Assets/Script/FCG/Kart/KartGroundProbe.fcg` | 109 | 4,549 | 4-point ground raycast suspension and normal probe. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartNitro.fcg` | 105 | 3,886 | 3-tier nitro tank charge, activation, and burn duration. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartPhysics.fcg` | 98 | 3,814 | Rigidbody velocity and force injection adapter. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartSkidmark.fcg` | 96 | 4,034 | Procedural skidmark ribbon generation on drift. Covered in Sec 4, 11. |
| `Assets/Script/FCG/Kart/KartMath.fcg` | 93 | 3,786 | Vector clamping, Lerp, Slerp, ExpDecay math helpers. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartSteeringMath.fcg` | 85 | 3,278 | Speed-sensitive steering falloff curve calculations. Covered in Sec 4, 7, 9. |
| `Assets/Script/FCG/Kart/KartSteeringRuntime.fcg` | 83 | 3,314 | Steering input smoothing and deadzone filtering. Covered in Sec 4, 9. |
| `Assets/Script/FCG/Kart/KartGroundingMath.fcg` | 76 | 2,983 | Surface normal alignment and slope pitch math. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartDriftState.fcg` | 62 | 2,921 | State machine for Drift None/Initiating/Active/Recovering. Covered in Sec 4, 7. |
| `Assets/Script/FCG/Kart/KartCheckpoint.fcg` | 46 | 1,623 | Modular checkpoint trigger detection. Covered in Sec 4, 11. |
| `Assets/Script/FCG/Kart/KartLifecycle.fcg` | 40 | 1,927 | Kart spawning, respawning, and destruction lifecycle. Covered in Sec 4. |
| `Assets/Script/FCG/Kart/KartCarFactory.fcg` | 17 | 694 | Vehicle prefab instantiator stub. Covered in Sec 4. |
| `Assets/Script/FCG/Kart/KartNetworkSync.fcg` | 13 | 529 | Unused network replication stub. Covered in Sec 4, 16. |
| `Assets/Script/Important.txt` | 67 | 7,103 | Critical project constraints, compiler quirks, network bugs. Covered in Sec 1, 16. |
| `Assets/Script/FCG_FILE_ROLES.txt` | 307 | 16,484 | Architectural map of script modularization. Covered in Sec 3, 4. |
| `ProjectSettings/Physics.asset` | 445 | 6,277 | Physics layers, gravity `Vector3(0, -9.81, 0)`. Covered in Sec 1, 8. |
| `ProjectSettings/PlayerSetting.asset` | 135 | 4,101 | Spawn coordinates, camera profiles. Covered in Sec 1. |
| `ProjectSettings/DataStorage.asset` | 58 | 1,207 | Server persistent storage table definitions. Covered in Sec 13. |
| `ProjectSettings/InputSystem.asset` | 54 | 1,198 | Virtual axis and key code configurations. Covered in Sec 9. |
| `ProjectSettings/Camera.asset` | 41 | 917 | Engine Camera FOV default (120°). Covered in Sec 10, 17. |
| `ProjectSettings/basicSetting.cproj` | 24 | 544 | Project metadata, build settings. Covered in Sec 1. |
| `fcconfig.json` | 10 | 374 | FCG compiler path and optimization flags. Covered in Sec 1, 15. |
| `main.gproj` | 10 | 238 | Craftland project package manifest. Covered in Sec 1. |
| `AGENTS.md` | 69 | 3,867 | Agent rules, MCP guidelines, Craftland rules. Covered in Sec 1. |

---

## 4. Binary Asset Inventory

The project contains **547 binary asset files** totaling approximately **118.4 MB**. Below is the grouped breakdown by asset type:

* **3D Models (`.fbx` - 63 files, ~46.2 MB):**
  * Supercar chassis meshes: `Supercar_Chassis_LOD0.fbx` (6.2 MB), `SportsCar_Body.fbx` (4.8 MB), `Kart_Model.fbx` (3.1 MB).
  * Wheel meshes: `Wheel_Front_Left.fbx`, `Wheel_Front_Right.fbx`, `Wheel_Rear_Left.fbx`, `Wheel_Rear_Right.fbx` (~450 KB each).
  * Environment & Stunts: `Loop_Track_Section.fbx` (7.4 MB), `Mega_Ramp_Modular.fbx` (5.1 MB), `Track_Barrier_Curved.fbx` (1.8 MB), `Bridge_Pillars.fbx` (2.4 MB).
* **Textures & Sprites (`.png`, `.jpg` - 244 files, ~51.8 MB):**
  * UI Elements: `Speedometer_Dial.png` (512x512, 420 KB), `Speedometer_Needle.png` (256x256, 85 KB), `Nitro_Gauge_Fill.png` (1024x256, 180 KB), `Minimap_Track_Overlay.png` (1024x1024, 1.2 MB).
  * Materials: PBR Albedo, Normal, Metallic-Gloss maps for vehicle body (`Car_Carbon_Normal.png`, `Car_Paint_Albedo.png`) and road surfaces (`Asphalt_Clean_Normal.jpg`, `Curb_YellowBlack_Albedo.jpg`).
* **Audio Clips (`.wav`, `.mp3` - 17 files, ~14.1 MB):**
  * `Engine_Idle_Loop.wav` (44.1kHz 16-bit, 1.2 MB)
  * `Engine_HighRPM_Loop.wav` (44.1kHz 16-bit, 2.4 MB)
  * `Drift_Tire_Screech_Loop.wav` (44.1kHz 16-bit, 890 KB)
  * `Nitro_Ignition_Burst.wav` (44.1kHz 16-bit, 410 KB)
  * `Nitro_Continuous_Whoosh.wav` (44.1kHz 16-bit, 1.8 MB)
  * `Wall_Crash_Heavy.wav` (44.1kHz 16-bit, 620 KB)
  * `Wall_Scrape_Sparks_Loop.wav` (44.1kHz 16-bit, 780 KB)
  * `Countdown_Beep.wav`, `Countdown_Go.wav` (~200 KB each)
  * `BGM_Cyber_Race_Theme.mp3` (128kbps stereo, 5.2 MB)
* **Visual Effects & Particles (`.particles` - 9 files, ~4.2 MB):**
  * `Nitro_Jet_Exhaust.particles`: Dual supersonic exhaust plume.
  * `Drift_Tire_Smoke.particles`: Bilateral tire burnout cloud.
  * `Wall_Scrape_Sparks.particles`: Contact-point friction shower.
  * `Air_Speed_Streaks.particles`: Screen-space radial velocity lines.
  * `Checkpoint_Pass_Rings.particles`: Emissive portal burst on pass.
* **Prefabs & Entities (`.prefab` - 35 files, ~2.1 MB):**
  * `Player_Kart_Default.prefab`: Root vehicle entity with Rigidbody, colliders, visual attachment bones.
  * `Checkpoint_Trigger_Standard.prefab`, `Speed_Pad_Booster.prefab`, `Obstacle_Dynamic_Box.prefab`.

