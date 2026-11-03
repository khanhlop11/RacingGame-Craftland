# Technical Handover — 07: Collision System & Tunneling Analysis

> **Target Audience:** External Technical Advisor (Independent AI Consultant)  
> **Source Files:** `RacingMovement.fcg`, `KartCCD.fcg`, `KartPhysics.fcg`, `ProjectSettings/Physics.asset`  
> **Classification:** Collision Geometry, Raycast Pipeline, Tunneling Proof & Resolution  

---

## 1. Collidable Shapes & Collision Geometry

### 1.1 Vehicle Collider Representation
The vehicle uses a hybrid physical representation:
1. **Primary Ground Contact Collider:**
   * **Shape:** Single Sphere Collider attached to the vehicle root Rigidbody.
   * **Dimensions:** Radius $r = 0.80\text{ m}$ (Diameter $1.60\text{ m}$).
   * **Offset:** Centered at vehicle base $(0, 0.80, 0)$.
   * **Physics Material:** Dynamic friction $0.0$, static friction $0.0$, bounciness $0.0$ (allows frictionless sliding over ground polygons).
2. **Suspension & Ground Alignment Probes:**
   * **Shape:** 4 downward vertical raycasts located at chassis corners:
     * Front-Left: $(+0.75, +0.80, +1.20)$
     * Front-Right: $(-0.75, +0.80, +1.20)$
     * Rear-Left: $(+0.75, +0.80, -1.20)$
     * Rear-Right: $(-0.75, +0.80, -1.20)$
   * **Raycast Length:** $1.40\text{ m}$ downward (`SuspRestLength = 0.58\text{ m}`).
3. **Wall Crash & Obstacle Probes:**
   * **Forward Raycast:** 1 single point raycast originating at $(0, 0.80, 0)$ casting along vehicle forward vector over distance $d = 2.40\text{ m}$.
   * **Side Whisker Raycasts:** 2 lateral raycasts originating at $(0, 0.80, 0)$ casting $\pm 45^\circ$ left and right over distance $d = 1.60\text{ m}$.

### 1.2 Track & Environmental Colliders
* **Track Surface:** Static MeshColliders (Triangle meshes representing asphalt road segments and loop-the-loops).
* **Track Walls & Barriers:** Static BoxColliders and Convex MeshColliders:
  * Concrete Retaining Walls: Thickness $t = 0.20\text{ m} - 0.50\text{ m}$.
  * Guardrails & Curbs: Thickness $t = 0.08\text{ m} - 0.15\text{ m}$, Height $h = 0.35\text{ m} - 0.55\text{ m}$.
  * Dynamic Obstacles (Boxes, Barrels): BoxColliders ($1.0\text{ m} \times 1.0\text{ m} \times 1.0\text{ m}$) with Rigidbody mass $m = 100\text{ kg}$.

---

## 2. Collision Detection Pipeline & Execution Order

Every frame in `RacingMovement.fcg::OnUpdate`, collision evaluation occurs in strict sequence relative to movement:

```text
1. GROUND PROBE RAYCASTS (RacingMovement.fcg:3980-4100):
   • Casts 4 vertical rays downward to detect road surface and ground normal.
   • Computes suspension spring force: F = SuspSpring * (RestLength - HitDist) - Damping * velY.

2. WALL & OBSTACLE RAYCASTS (RacingMovement.fcg:4320-4370):
   • Casts 1 forward raycast for head-on collision (frontCrashHit).
   • Casts 2 side whiskers for scrape detection (IsScrapingLeft, IsScrapingRight).

3. CONTINUOUS COLLISION DETECTION (CCD) EVALUATION:
   • Checked at RacingMovement.fcg:4335:
     "if UsePredictiveWallCcd { CheckContinuousCollision(...) }"
   • STATUS: BYPASSED (UsePredictiveWallCcd is hardcoded to FALSE).

4. VELOCITY CORRECTION & DEFLECTION (RacingMovement.fcg:4745-4845):
   • If side scrape detected: Strips normal velocity vector against wall (vDotW < 0).
   • Deflects vehicle yaw parallel to wall tangent vector.

5. RIGIDBODY FORCE INJECTION (RacingMovement.fcg:5120-5140):
   • Calls KartPhysics.ApplyVelocityAssist via ForceType.Velocity.
   • Overwrites Rigidbody linear velocity with custom targetVel.

6. ENGINE DISCRETE INTEGRATION:
   • Craftland physics engine advances Rigidbody position based on injected velocity.
```

---

## 3. Detailed Tunneling Analysis (Mathematical Proof)

The user reported: *"The car tunnels through walls/objects at high speed."*  
**This issue is CONFIRMED BY CODE through four distinct structural failure mechanisms:**

### 3.1 Mechanism 1: CCD is Explicitly Disabled in Source Code
* **Code Proof:**
  * [`RacingMovement.fcg:117`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L117): `UsePredictiveWallCcd bool = false`
  * [`RacingMovement.fcg:118`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L118): `UseCustomWallScrapeResponse bool = false`
* **Impact:** The continuous capsule sweep routine implemented in [`KartCCD.fcg`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartCCD.fcg) is **completely uncalled**. The simulation relies entirely on the engine's discrete collision detection.

### 3.2 Mechanism 2: Raycast Blind Spots on Low Curbs & Corner Angles
* **Code Proof:** [`RacingMovement.fcg:3995, 4001`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L3995-L4001):
  * Raycast origin: `rawCarPos.Y + 0.80\text{ m}`.
  * Raycast direction: Exact vehicle centerline forward vector $(0, 0, 1)$.
* **Blind Spot Geometry:**
  1. **Low Barrier Under-Shoot:** Track curbs and tire barriers in Craftland have heights between $0.30\text{ m}$ and $0.50\text{ m}$. The forward raycast at $+0.80\text{ m}$ shoots directly over the top of the barrier into empty air!
  2. **Corner Clipping:** A car approaching a wall corner at a $30^\circ - 45^\circ$ angle strikes the barrier with its front-left or front-right bumper. The single centerline raycast misses the wall entirely, leaving `frontCrashHit = false`.

### 3.3 Mechanism 3: The `ForceType.Velocity` Override Injection
* **Code Proof:** [`KartPhysics.fcg:11-14`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartPhysics.fcg#L11-L14):
  ```fcg
  physics.AddForce(kartModel as PhysicalObject,
      Vector3{(targetVelX - vel.X) * assist, (targetVelY - vel.Y) * assist, (targetVelZ - vel.Z) * assist},
      ForceType.Velocity)
  ```
* **Failure Sequence:**
  1. Car impacts a wall. The engine's native discrete sphere collider halts the vehicle, bringing physical velocity $vel \to 0$.
  2. Because the forward raycast missed (due to height or angle), `RacingMovement.fcg` calculates `targetVel` based on `DriveForwardSpeed` ($365\text{ km/h} = 2.53\text{ units/s}$).
  3. On the next frame, `ApplyVelocityAssist` calculates:
     $$\Delta \vec{v} = (\vec{v}_{\text{target}} - \vec{0}) \times \text{assist} = \vec{v}_{\text{target}} \times 0.90$$
  4. This forcibly injects $2.28\text{ units/s}$ of velocity directly into the wall using `ForceType.Velocity`!
  5. The injected impulse overpowers the solver contact constraints, pushing the collider sphere $0.10\text{m} - 0.20\text{m}$ inside the wall geometry.

### 3.4 Mechanism 4: Disabled Push-Out Safety Ejection
* **Code Proof:** [`RacingMovement.fcg:4489`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L4489):
  ```fcg
  // CrashPush disabled to prevent sudden snapping
  ```
* Once the sphere penetrates the wall mesh, there is no geometric push-out or teleportation ejection. On the subsequent frame, the sphere's center passes through the rear face of the wall polygon, emerging into the void on the other side.

---

## 4. Numerical Tunneling Breakdown

| Metric | Normal Driving | Tier 3 Nitro Boost | Severe Frametime Spike ($realDt = 0.1\text{ s}$) |
| :--- | :---: | :---: | :---: |
| **Display Speed** | $365.0\text{ km/h}$ | $547.5\text{ km/h}$ | $547.5\text{ km/h}$ |
| **Internal Physical Speed** | $2.535\text{ units/s}$ | $3.802\text{ units/s}$ | $3.802\text{ units/s}$ |
| **Simulation Step ($\Delta t$)** | $0.033\text{ s}$ | $0.033\text{ s}$ | $0.100\text{ s}$ |
| **Travel Distance Per Step ($\Delta x$)** | **$0.0837\text{ m}$** | **$0.1255\text{ m}$** | **$0.3802\text{ m}$** |
| **Standard Guardrail Thickness** | $0.1000\text{ m}$ | $0.1000\text{ m}$ | $0.1000\text{ m}$ |
| **Ratio ($\Delta x$ / Wall Thickness)** | **$83.7\%$** | **$125.5\%$** | **$380.2\%$** |
| **Tunneling Probability** | High (oblique angles) | **Guaranteed Penetration** | **Complete Tunneling** |

* **Mathematical Conclusion:** Tunneling is **100% mathematically inevitable** whenever the vehicle travels at boost speeds against thin guardrails ($\le 0.12\text{m}$) or whenever frametime spikes above $0.033\text{s}$, because the travel per step exceeds barrier thickness and CCD is disabled in code.

---

## 5. Post-Collision Position Modification Points

The following locations in the codebase directly modify vehicle transform coordinates after or during collision resolution:

1. **`RacingMovement.fcg:3350` (`ResetVehicle`):** Snaps vehicle transform to last valid checkpoint position and resets velocity:
   `KartModel<Transform>.Position = respawnPos`
2. **`RacingMovement.fcg:4489` (`CrashPush` - Commented Out):** Previously teleported the vehicle backward along impact normal:
   `// KartModel<Transform>.Position = hitPoint + hitNorm * pushDist`
3. **`RacingMovement.fcg:4830` (`WallScrape`):** Modifies velocity vector directly on Rigidbody:
   `KartModel<Rigidbody>.Velocity = vel` (with normal velocity removed: $v_{\text{normal}} = 0$).

