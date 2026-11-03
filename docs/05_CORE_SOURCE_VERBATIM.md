# Technical Handover — 05: Core Source Verbatim

> **Target Audience:** External Technical Advisor (Independent AI Consultant)  
> **Source Project:** Max Speed  
> **Hard Rule:** Verbatim source code without alterations, deletions, or formatting changes. Every block is preceded by its exact `path:lines` citation.  

---

## 1. Main Frame Loop & Timing Header

### `Assets/Script/FCG/RacingMovement.fcg:3516-3560`
```fcg
    event OnUpdate() {
        if !IsRunning  { return }
        // --- DeltaTime chuẩn xác theo thời gian thực (đo FPS tự động, khử 100% sai lệch tốc độ giữa PC và Mobile) ---
        FpsFrameCount += 1
        var secNow = time.GetLocalTimestamp()
        if secNow != FpsLastSec {
            if FpsFrameCount >= 20 && FpsFrameCount <= 240 {
                MeasuredFrameRate = FpsFrameCount as float
            }
            FpsFrameCount = 0
            FpsLastSec = secNow
        }

        var realDt = 1.0 / MeasuredFrameRate
        if realDt < 0.008 { realDt = 0.008 }
        if realDt > 0.050 { realDt = 0.050 }

        // Cập nhật mượt mà đếm ngược phòng chờ cho Flycam (dùng realDt theo thời gian thực)
        if IsLobbyActive {
            if LobbySmoothTimer > 0.0 {
                LobbySmoothTimer -= realDt
                if LobbySmoothTimer < 0.0 { LobbySmoothTimer = 0.0 }
            }
        }

        // --- Intro Flycam (Cinematic phòng chờ: dùng realDt theo thời gian thực) ---
        if IsIntroCamActive && !HasSwitchedStartCamera {
            UpdateIntroCamera(realDt)
        }

        // --- DeltaTime vật lý xe: Khôi phục chuẩn gốc 0.033s (30 FPS physics tick) ---
        // Đảm bảo 100% độ gắt khi bẻ lái, ôm cua và cảm giác rê bánh drift nguyên bản đã tune
        var dt = 0.033

        if PendingSpawnAtStart && !HasSpawnedAtStartSlot {
            SpawnRetryTimer -= dt
            if SpawnRetryTimer <= 0.0 {
                SpawnRetryTimer = 0.35
                if math.Abs(CpRespawnX) > 0.1 || math.Abs(CpRespawnZ) > 0.1 {
                    HasSpawnedAtStartSlot = true
                    PendingSpawnAtStart = false
                    SpawnKartAtRespawn()
                    SpawnDropTimer = 1.5
                    if !IsRaceStarted {
                        IsExternallyLocked = true
                    }
                }
            }
        }
```

---

## 2. Input Handling & Double Smoothing

### `Assets/Script/FCG/RacingMovement.fcg:4514-4528`
```fcg
            var rawTurnInput = KartMath.CalcTurnInput(IsTurningLeft, IsTurningRight)
            var turnSharp = 5.5
            if isDrift { turnSharp = 5.0 }
            if rawTurnInput * SmoothedTurnInput < 0.0 {
                turnSharp = 3.6
                if isDrift { turnSharp = 3.2 }
            }
            SmoothedTurnInput = SmoothedTurnInput + (rawTurnInput - SmoothedTurnInput) * (dt * turnSharp)
            if math.Abs(SmoothedTurnInput) < 0.03 && rawTurnInput == 0.0 { SmoothedTurnInput = 0.0 }
            turnInput = SmoothedTurnInput
            // Goal #3: explicit drift state machine owns drift flags.
            // FlatSpin remains a drift-like override mode.
            if IsFlatSpinning { IsBraking = true }

            UpdateDriftFlow(dt, IsBraking)
            UpdateDriftState(dt, isStrictlyGrounded, raycastHit, rawTurnInput)
```

---

## 3. Vehicle Motion, Speed & Steering Integration

### `Assets/Script/FCG/RacingMovement.fcg:4630-4712`
```fcg
            // Goal #4: continuous boost power (0..1) for scalable boost intensity.
            // Keep IsBoosting as the "boost request" flag (UI/network compatibility).
            BoostPowerTarget = KartNitro.CalcBoostPowerTargetFromTier(IsBoosting, NitroBoostTier)
            BoostPower = KartNitro.SmoothBoostPower(BoostPower, BoostPowerTarget, dt)

            if IsBoosting {
                DriftFlowTimer = 0.0
                if DriftState == DriftState_EXIT || DriftState == DriftState_HOLD {
                    DriftState = DriftState_NORMAL
                }
            }

            var shouldShowNitro = KartNitro.ShouldShowNitro(NitroCharge, PostDriftBoostTimer, isDrift)
            if shouldShowNitro && !HasNitro {
                HasNitro = true
                DispatchEventToEntity(ShowNitroButtonServer, thisEntity as entity<Player>, PlatformType.Client, nil)
            } else if !shouldShowNitro && HasNitro {
                HasNitro = false
                IsBoosting = false
                DispatchEventToEntity(HideNitroButtonServer, thisEntity as entity<Player>, PlatformType.Client, nil)
            }

            var accelInput = 0.0
            if IsRaceStarted && !IsReversing { accelInput = 1.0 }

            var physTopSpeed = KartMath.DisplaySpeedToPhysSpeed(TopSpeed, SpeedUnitScale)
            var physBoostTopSpeed = KartMath.DisplaySpeedToPhysSpeed(BoostTopSpeed, SpeedUnitScale)
            var physReverseSpeed = KartMath.DisplaySpeedToPhysSpeed(ReverseSpeed, SpeedUnitScale)
            var driftSustainSpeed = physTopSpeed * DriftSustainRatio

            if IsBoosting { IsNitroRecovering = false }
            var boostTarget = BoostPower
            BoostBlend = KartPhysicsMath.CalcBoostBlendNext(BoostBlend, boostTarget, BoostRampIn, BoostRampOut, dt)

            var tierTopMult = KartNitro.CalcBoostTopMult(NitroBoostTier)
            var maxSpeed = KartPhysicsMath.CalcMaxSpeed(physTopSpeed, physBoostTopSpeed, BoostPower, BoostBlend, PostDriftBoostTimer, PostDriftNitroTopMult * tierTopMult, isDriftActive, DriftSustainRatio, DriftBlend, IsTurningLeft, IsTurningRight, TurnSpeedRatio)
            if IsBoosting {
                maxSpeed = physBoostTopSpeed * tierTopMult
                if PostDriftBoostTimer > 0.0 { maxSpeed = maxSpeed * PostDriftNitroTopMult }
            }
            if IsAiAutopilot {
                maxSpeed = KartMath.DisplaySpeedToPhysSpeed(AiDesiredSpeedKmh, SpeedUnitScale)
            }

            var curLatSpdEarly = vel.X * rgtVecFlat.X + vel.Y * rgtVecFlat.Y + vel.Z * rgtVecFlat.Z
            var turnRate = CalcSteerRate(turnInput, isDriftActive, CurrentSpeedKmh, localFwdSpeed, curLatSpdEarly, dt)
            if IsAiAutopilot {
                turnRate = AiTurnRate
            }

            PhysicsQuat = KartSteeringRuntime.ApplySteerRotation(PhysicsQuat, nx, ny, nz, turnRate, dt)
```

---

## 4. Collision Detection, Scrape & Wall Deflection

### `Assets/Script/FCG/RacingMovement.fcg:4745-4845`
```fcg
            var isWallScrapingNow = hasInstantScrape && (ActiveWallNormX != 0.0 || ActiveWallNormZ != 0.0)
            if isWallScrapingNow {
                var wallScrapeNx = ActiveWallNormX
                var wallScrapeNz = ActiveWallNormZ

                var curFwd = math.RotateVector(Vector3{0.0, 0.0, 1.0}, PhysicsQuat)

                // BẺ HƯỚNG TỪ TỪ KHI CÓ HƯỚNG HOẶC ĐÂM CHÉO VÀO TƯỜNG (intoWall > 0.005)
                var intoWall = -(curFwd.X * wallScrapeNx + curFwd.Z * wallScrapeNz)
                var isQualifySideScrape = CurrentSpeedKmh > 15.0 && !IsReversing && intoWall > 0.005
                if isQualifySideScrape {
                    // Vector tiếp tuyến tường cùng chiều tiến của xe
                    var tanX = -wallScrapeNz
                    var tanZ = wallScrapeNx
                    if curFwd.X * tanX + curFwd.Z * tanZ < 0.0 {
                        tanX = wallScrapeNz
                        tanZ = -wallScrapeNx
                    }

                    // Góc Yaw tiếp tuyến của bờ tường (Degrees)
                    var wallYaw = math.ATan2(tanX, tanZ)
                    var curYaw = physEuler.Y
                    var yawDiff = wallYaw - curYaw
                    while yawDiff > 180.0 { yawDiff -= 360.0 }
                    while yawDiff < -180.0 { yawDiff += 360.0 }

                    if turnRate * yawDiff < 0.0 {
                        curYaw -= turnRate * dt
                        yawDiff = wallYaw - curYaw
                        while yawDiff > 180.0 { yawDiff -= 360.0 }
                        while yawDiff < -180.0 { yawDiff += 360.0 }
                    }

                    var intoFactor = intoWall * 2.2
                    if intoFactor > 1.0 { intoFactor = 1.0 }
                    if intoFactor < 0.35 { intoFactor = 0.35 }

                    var speedScale = CurrentSpeedKmh / 90.0
                    if speedScale < 0.9 { speedScale = 0.9 }
                    if speedScale > 2.2 { speedScale = 2.2 }

                    var alignStep = dt * 9.0 * intoFactor * speedScale
                    if alignStep > 0.26 { alignStep = 0.26 }

                    var deltaYaw = yawDiff * alignStep
                    var minStepDeg = dt * 85.0 * intoFactor * speedScale
                    if math.Abs(yawDiff) > minStepDeg && math.Abs(deltaYaw) < minStepDeg {
                        if yawDiff > 0.0 { deltaYaw = minStepDeg } else { deltaYaw = -minStepDeg }
                    }

                    var newYaw = curYaw + deltaYaw
                    physEuler.Y = newYaw
                    PhysicsQuat = math.EulerAngleToQuaternion(Vector3{physEuler.X, newYaw, physEuler.Z})
                    FacingAngle = KartSteeringRuntime.NormalizeFacingAngle(newYaw)
                    VelocityQuat = math.SlerpQuaternion(VelocityQuat, PhysicsQuat, dt * 18.0)
                }

                // CHỈ LOẠI BỎ VẬN TỐC ĐÂM VÀO TƯỜNG, TUYỆT ĐỐI KHÔNG ÉP DÍNH VÀO TƯỜNG
                var vDotW = vel.X * wallScrapeNx + vel.Z * wallScrapeNz
                if vDotW < 0.0 {
                    vel.X -= vDotW * wallScrapeNx
                    vel.Z -= vDotW * wallScrapeNz
                }

                // Triệt tiêu 100% phản lực nảy lên không trung khi cạ/đâm tường ở tốc độ cao
                if vel.Y > 0.0 { vel.Y = 0.0 }
                if KartModel != nil {
                    KartModel<Rigidbody>.Velocity = vel
                    KartModel<Rigidbody>.AngularVelocity = Vector3{0.0, 0.0, 0.0}
                }

                if DriveForwardSpeed < physTopSpeed * 0.55 {
                    DriveForwardSpeed = physTopSpeed * 0.55
                }

                isCrashing = false
                Frame_IsCrashing = false
            }
```

---

## 5. Rigidbody Physics Injection

### `Assets/Script/FCG/Kart/KartPhysics.fcg:10-31`
```fcg
    func ApplyVelocityAssist(kartModel entity<Rigidbody>, vel Vector3, targetVelX float, targetVelY float, targetVelZ float, assist float) {
        if kartModel == nil { return }
        physics.AddForce(kartModel as PhysicalObject,
            Vector3{(targetVelX - vel.X) * assist, (targetVelY - vel.Y) * assist, (targetVelZ - vel.Z) * assist},
            ForceType.Velocity)
    }

    func ApplyAngularDamping(kartModel entity<Rigidbody>, isStableGrounded bool) {
        if kartModel == nil { return }
        var angDampFactor = 0.0
        if isStableGrounded {
            angDampFactor = 0.04
        } else {
            angDampFactor = 0.30
        }
        var av = kartModel<Rigidbody>.AngularVelocity
        kartModel<Rigidbody>.AngularVelocity = Vector3{av.X * angDampFactor, av.Y * angDampFactor, av.Z * angDampFactor}
    }
```

---

## 6. Camera Position, Orientation & FOV Application

### `Assets/Script/FCG/RacingMovement.fcg:5655-5735`
```fcg
            var totalBobX = (speedBobX + nitroBobX) as float
            var totalBobY = (speedBobY + nitroBobY) as float
            var totalBobZ = (speedBobZ + nitroBobZ) as float
            // Giữ xe nằm ngay chính giữa màn hình (triệt tiêu độ lệch trôi ngang dư thừa, chỉ giữ dao động điện ảnh nhỏ khi drift)
            var camCenterSide = (CamDriftSide * 0.30 + totalBobX) as float
            var localOffset = Vector3{camCenterSide, BaseCamHeight + CurrentDropOffset + CamDriftHeightDrop + CamNitroHeightDrop + totalBobY, camBackZ + totalBobZ}
            
            var rotatedOffset = math.RotateVector(localOffset, CamQuat)
            var lookAheadVec = math.RotateVector(Vector3{0.0, 0.0, CamLookAheadOffset}, CamQuat)
            
            var desiredCamPos = Vector3{ 
                rawCarPos.X + rotatedOffset.X + lookAheadVec.X, 
                rawCarPos.Y + rotatedOffset.Y + lookAheadVec.Y, 
                rawCarPos.Z + rotatedOffset.Z + lookAheadVec.Z}

            var camUp = math.RotateVector(Vector3{0.0, 1.0, 0.0}, CamQuat)
            CurrentCamPos = desiredCamPos

            var camEntity = MyClientCam as entity<Entity>

            // Dual-Pivot Target Node: Ngắm vào trọng tâm thân xe/nắp capo để nâng xe lên giữa màn hình
            var camFwd = math.RotateVector(Vector3{0.0, 0.0, 1.0}, CamQuat)
            var tX = rawCarPos.X + camFwd.X * (4.5 + CamLookAheadOffset) + camUp.X * 0.65
            var tY = rawCarPos.Y + camFwd.Y * (4.5 + CamLookAheadOffset) + camUp.Y * 0.65
            var tZ = rawCarPos.Z + camFwd.Z * (4.5 + CamLookAheadOffset) + camUp.Z * 0.65
            var lookAtTarget = Vector3{tX, tY, tZ}

            var aimX = lookAtTarget.X - CurrentCamPos.X
            var aimY = lookAtTarget.Y - CurrentCamPos.Y
            var aimZ = lookAtTarget.Z - CurrentCamPos.Z
            var aimLen = math.Sqrt(aimX * aimX + aimY * aimY + aimZ * aimZ) as float
            if aimLen < 0.10 { aimLen = 0.10 }
            var aimDir = Vector3{aimX / aimLen, aimY / aimLen, aimZ / aimLen}

            var lookQ = math.LookRotation(aimDir, camUp)
            var lookEuler = math.QuaternionToEulerAngle(lookQ)

            var totalRollDeg = CamDriftRoll + StuntCamRoll + CamAirRoll
            // Cộng thêm BaseCamPitch vào góc chúc để nâng xe rời khỏi mép đáy, định vị hoàn hảo ở trung tâm màn hình
            var eX = lookEuler.X + BaseCamPitch + finalShakeX
            var eY = lookEuler.Y + finalShakeY + CamDriftYawInward
            var eZ = lookEuler.Z + totalRollDeg + finalShakeZ
            var targetCamEuler = Vector3{eX, eY, eZ}

            if IsIntroBlendingToCar {
                IntroBlendTimer += dt
                if IntroBlendTimer >= IntroBlendDuration {
                    IntroBlendTimer = IntroBlendDuration
                    IsIntroBlendingToCar = false
                }
                var blendT = IntroBlendTimer / IntroBlendDuration
                if blendT < 0.0 { blendT = 0.0 }
                if blendT > 1.0 { blendT = 1.0 }

                var smoothT = blendT * blendT * (3.0 - 2.0 * blendT)

                var blendedPos = math.LerpVector3(IntroBlendStartPos, CurrentCamPos, smoothT)
                camEntity<Transform>.Position = blendedPos

                var qIntro = math.EulerAngleToQuaternion(IntroBlendStartRot)
                var qTarget = math.EulerAngleToQuaternion(targetCamEuler)
                var blendedQ = math.SlerpQuaternion(qIntro, qTarget, smoothT)
                camEntity<Transform>.Rotation = math.QuaternionToEulerAngle(blendedQ)

                var blendedFOV = math.Lerp(IntroBlendStartFOV, CurrentFOV, smoothT)
                MyClientCam<camera.CustomCamera>.FOV = blendedFOV
            } else {
                camEntity<Transform>.Position = CurrentCamPos
                camEntity<Transform>.Rotation = targetCamEuler
            }
```

---

## 7. Track Waypoint & Checkpoint Positioning

### `Assets/Script/FCG/RacingMovement.fcg:3600-3635`
```fcg
            if WaypointNodes != nil && list.Length(WaypointNodes as List<object>) > 0 {
                var wpCount = list.Length(WaypointNodes as List<object>)
                var targetWp = WaypointNodes[CurrentWaypointIndex] as Vector3
                var dX = targetWp.X - rawCarPos.X
                var dY = targetWp.Y - rawCarPos.Y
                var dZ = targetWp.Z - rawCarPos.Z
                var distSq = dX * dX + dY * dY + dZ * dZ
                if distSq < 144.0 { // 12.0m threshold
                    CurrentWaypointIndex = (CurrentWaypointIndex + 1) % wpCount
                }
                // Spline Projection Progress
                var nextWpIdx = (CurrentWaypointIndex + 1) % wpCount
                var nextWp = WaypointNodes[nextWpIdx] as Vector3
                var segVec = Vector3{nextWp.X - targetWp.X, nextWp.Y - targetWp.Y, nextWp.Z - targetWp.Z}
                var segLen = math.Sqrt(segVec.X * segVec.X + segVec.Y * segVec.Y + segVec.Z * segVec.Z) as float
                if segLen > 0.01 {
                    var toCar = Vector3{rawCarPos.X - targetWp.X, rawCarPos.Y - targetWp.Y, rawCarPos.Z - targetWp.Z}
                    TrackSplineT = (toCar.X * segVec.X + toCar.Y * segVec.Y + toCar.Z * segVec.Z) / (segLen * segLen)
                    if TrackSplineT < 0.0 { TrackSplineT = 0.0 }
                    if TrackSplineT > 1.0 { TrackSplineT = 1.0 }
                }
            }
```

---

## 8. Index of Other Subsystem Functions (Signature + Description)

* `func CalcSteerRate(turnInput float, isDrift bool, speedKmh float, fwdSpeed float, latSpeed float, dt float) float` ([`RacingMovement.fcg:2580`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L2580)): Computes speed-dependent steering rate with non-linear falloff and counter-steer sensitivity multipliers.
* `func UpdateDriftState(dt float, grounded bool, hit bool, rawTurn float)` ([`RacingMovement.fcg:2750`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L2750)): Manages drift entry, active slide yaw clamping, and exit traction recovery timers.
* `func UpdateVisualTransforms(dt float)` ([`RacingMovement.fcg:5750`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/RacingMovement.fcg#L5750)): Rotates 4 wheel meshes around pitch axis based on linear velocity and steers front wheel hubs based on `SmoothedTurnInput`.
* `func UpdateEngineAudio(speedKmh float, isDrifting bool, isNitro bool)` ([`KartAudio.fcg:45`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartAudio.fcg#L45)): Scales engine audio source pitch between $0.8\times$ and $2.2\times$ and modulates tire screech loop gain.
* `func CheckContinuousCollision(pos Vector3, vel Vector3, dt float) bool` ([`KartCCD.fcg:30`](file:///c:/Max%20Speed23-07/maxspeed_published/Max%20Speed/Assets/Script/FCG/Kart/KartCCD.fcg#L30)): Ray-casts forward capsule along velocity vector over $vel \times dt$ distance; returns true if sweep penetrates obstacle.

