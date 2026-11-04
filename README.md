# X Nitro — High-Speed Arcade Racing & Vehicle Physics Engine

Trò chơi đua xe Arcade được phát triển trên nền tảng **Garena Craftland Studio**, sử dụng ngôn ngữ lập trình hướng đối tượng tĩnh **FCG** (Free Fire Craftland Scripting Language).

Ý tưởng dựa trên các game đua xe phổ biến làm nền tảng bao gồm : Asphalt8, ZingSpeed, Ace Racer, Mario Kart,...

Toàn bộ hệ thống vật lý xe (chuyển động, drift, độ bám mặt đường, chống xuyên vật thể), camera đuổi động (Chase Camera) và phân tách mạng Client - Server đều được tự lập trình hoàn toàn trong mã nguồn FCG.

---

## 🚀 Các Tính Năng Kỹ Thuật Nổi Bật

### 1. Mô Phỏng Động Học Xe Đua (Custom Vehicle Kinematics & Physics)
- **Động học bám đường (Ground Adherence & Suspension):** Hệ thống dò đường đa điểm (Multi-point Raycast Probing) tính toán pháp tuyến bề mặt (Surface Normal) để căn chỉnh hướng xe ôm sát mặt đường uốn lượn và dốc nghiêng.
- **Cơ chế Drift Kinematics:** Tự tính toán lực ma sát lốp xe, góc trượt (slip angle) và hệ số gia tốc tích lũy khi vào cua tốc độ cao (`KartDriftMath.fcg`).
- **Chống xuyên vật thể tốc độ cao (Continuous Collision Detection - CCD):** Thuật toán raycast dự đoán vị trí khung hình tiếp theo để khử hoàn toàn hiện tượng lọt dốc/xuyên tường (tunneling) khi xe đạt vận tốc cực hạn (`KartCCD.fcg`).

### 2. Nội Suy Tuyến Tính & Camera Bám Đuổi (Interpolation & Dynamic Chase Camera)
- **Làm mượt chuyển động (Lerp/Slerp):** Áp dụng nội suy tuyến tính (Linear Interpolation - `math.Lerp`) và nội suy hình cầu (Slerp cho Quaternions) để triệt tiêu hiện tượng giật cục của xe và camera khi thay đổi hướng đột ngột.
- **Chase Camera thông minh:** Camera tự động tính toán góc Pitch, Yaw, khoảng cách lùi theo vận tốc và góc nghiêng của thân xe, mang lại cảm giác tốc độ chân thực (`clientCamera.fcg`, `KartCameraMath.fcg`).

### 3. Kiến Trúc Mạng Phân Tách Client - Server
- **Client-Side Simulation (`[platform_client]`):** Toàn bộ tính toán vật lý, xử lý phím bấm (Steer, Brake, Boost, Reverse) và phản hồi camera được thực thi độc lập trên Client để đảm bảo độ trễ bằng 0 (Zero-latency feel).
- **Server Authoritative Race State (`[platform_server]`):** Server chịu trách nhiệm quản lý phòng đua, đếm ngược thời gian, xác thực thứ tự Checkpoint để chống gian lận (anti-cheat) và xếp hạng tay đua (`RacingMovementServer.fcg`, `RaceManager.fcg`).

---

## 📂 Cấu Trúc Mã Nguồn

```text
├── Assets/
│   └── Script/
│       └── FCG/
│           ├── Kart/                     # Module xe đua chuyên sâu
│           │   ├── KartPhysics.fcg       # Mô phỏng lực kéo, cản, gia tốc
│           │   ├── KartDriftMath.fcg     # Toán học xử lý trượt văng xe (drift)
│           │   ├── KartGroundProbe.fcg   # Bắn raycast dò bề mặt địa hình
│           │   ├── KartCCD.fcg           # Chống xuyên tường ở tốc độ cao
│           │   ├── KartCameraMath.fcg    # Tính toán góc xoay camera
│           │   ├── KartNitro.fcg         # Hệ thống tăng tốc tức thời (Nitro)
│           │   └── ...
│           ├── Main_Script/              # Bộ điều khiển kiến trúc trung tâm
│           │   ├── VehicleController.fcg # Bộ điều phối xe
│           │   ├── VehicleCamera.fcg     # Quản lý camera
│           │   └── VehicleServer.fcg     # Xử lý luồng server
│           ├── RacingMovement.fcg        # Hệ thống di chuyển xe chính
│           ├── RacingMovementServer.fcg  # Đồng bộ và xác thực server
│           ├── RaceManager.fcg           # Quản lý vòng đua và checkpoint
│           └── ... (48 FCG scripts)
├── docs/                                 # Bộ tài liệu kỹ thuật chi tiết
│   ├── 01_OVERVIEW_AND_INVENTORY.md      # Tổng quan dự án và stack công nghệ
│   ├── 02_MAIN_LOOP_AND_TIMING.md        # Vòng lặp chính và xử lý delta time
│   ├── 03_MODULE_REFERENCE.md            # Tham chiếu chi tiết từng module
│   ├── 04_FEATURE_CATALOGUE.md           # Danh mục toàn bộ tính năng
│   ├── 05_CORE_SOURCE_VERBATIM.md        # Mã nguồn cốt lõi
│   ├── 06_VEHICLE_AND_PHYSICS_MODEL.md   # Phân tích toán học mô hình vật lý
│   ├── 07_COLLISION_AND_TUNNELING.md     # Giải pháp xử lý va chạm & tunneling
│   ├── 08_CONTROLS_AND_CAMERA.md         # Hệ thống điều khiển & camera
│   └── 09_TRACK_UI_DATA_SYSTEMS.md       # Hệ thống UI và dữ liệu đường đua
├── fcconfig.json                         # Cấu hình biên dịch FCG
├── .gitignore                            # Cấu hình lọc mã nguồn sạch
└── README.md
```

---

## 📚 Tài Liệu Kỹ Thuật Chi Tiết
Bạn có thể tham khảo đầy đủ các phân tích chuyên sâu tại thư mục [`docs/`](docs/):
- [Mô hình Vật lý & Động lực học Xe](docs/06_VEHICLE_AND_PHYSICS_MODEL.md)
- [Thuật toán Xử lý Va chạm & Chống Xuyên Tường (CCD)](docs/07_COLLISION_AND_TUNNELING.md)
- [Hệ thống Điều khiển & Camera Đuổi](docs/08_CONTROLS_AND_CAMERA.md)
- [Kiến trúc Vòng lặp Chính & Thời gian](docs/02_MAIN_LOOP_AND_TIMING.md)

---

## 🛠️ Công Nghệ & Nền Tảng
- **Ngôn ngữ:** FCG (Free Fire Craftland Scripting Language - OOP, Static Typing).
- **Engine:** Garena Craftland Studio Engine.
- **Mô hình kiến trúc:** Client-Server Separation, Continuous Collision Detection (CCD), Linear Interpolation (Lerp).
