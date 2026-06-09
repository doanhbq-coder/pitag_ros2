# Hướng Dẫn Sử Dụng: cob_fiducials (ROS 2 Humble)

## Mục Lục

1. [Tổng Quan](#1-tổng-quan)
2. [Cấu Trúc Package](#2-cấu-trúc-package)
3. [Luồng Dữ Liệu](#3-luồng-dữ-liệu)
4. [Topics, Services, TF](#4-topics-services-tf)
5. [Tham Số Cấu Hình](#5-tham-số-cấu-hình)
6. [XML Model File](#6-xml-model-file)
7. [Khởi Động Node](#7-khởi-động-node)
8. [Tích Hợp Với Ứng Dụng Khác](#8-tích-hợp-với-ứng-dụng-khác)
9. [Hiệu Chỉnh Camera (Calibration)](#9-hiệu-chỉnh-camera-calibration)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Tổng Quan

`cob_fiducials` là package ROS 2 nhận diện và tính pose (vị trí + hướng 3D) của các fiducial marker từ ảnh camera.  
Hỗ trợ hai loại marker:

| Loại | Mô tả |
|------|-------|
| **PiTag** (`TYPE_PI`) | Marker hình vuông với 12 chấm tròn, mã hóa ID bằng cross-ratio. Phù hợp cho camera đơn sắc, nhận diện được khi xoay/nghiêng |
| **ArUco** (`TYPE_ARUCO`) | Marker mã nhị phân phổ biến (không được đề cập trong hướng dẫn này) |

**Node chính:** `fiducials` (thực thi: `cob_fiducials/fiducials`)

---

## 2. Cấu Trúc Package

```
cob_fiducials/
├── common/
│   ├── include/cob_fiducials/
│   │   ├── AbstractFiducialModel.h     # Interface chung
│   │   ├── FiducialDefines.h           # Struct t_pose, enum t_FiducialType
│   │   └── pi/
│   │       ├── FiducialModelPi.h
│   │       └── FiducialPiParameters.h
│   ├── src/
│   │   ├── AbstractFiducialModel.cpp   # ApplyExtrinsics
│   │   └── pi/
│   │       └── FiducialModelPi.cpp     # Thuật toán detect + solvePnP
│   └── files/models/                   # XML model files
│       ├── piTagIni_0.xml              # IDs 0–3, 10cm, tag đơn
│       ├── piTagIni_1.xml              # IDs 0–3, 5cm
│       ├── piTagIni_2.xml              # IDs 0–33, 10cm
│       ├── piTagIni_3.xml              # Multi-tag board + extras
│       ├── piTagIni_0_15cm.xml         # IDs 0–3, 15cm
│       └── piTagIni_0_20cm.xml         # IDs 0–3, 20cm
└── ros/
    ├── src/
    │   └── fiducials.cpp               # ROS 2 node
    └── launch/
        ├── fiducials.launch.py         # Launch file chính
        ├── fiducials_0.yaml            # Config: piTagIni_0.xml, 10cm
        ├── fiducials_1.yaml            # Config: piTagIni_1.xml, 5cm
        └── usb_cam.launch.py           # Launch camera USB
```

---

## 3. Luồng Dữ Liệu

```
Camera
  │
  ├─[sensor_msgs/Image]──────────────────────────────────────────┐
  └─[sensor_msgs/CameraInfo]──────────────────────────────────┐  │
                                                               │  │
                                                    ┌──────────▼──▼──────────┐
                                                    │    CobFiducialsNode     │
                                                    │                         │
                                                    │  1. Khởi tạo camera     │
                                                    │     matrix (1 lần)      │
                                                    │                         │
                                                    │  2. Chuyển ảnh → gray   │
                                                    │     (+ invert nếu IR)   │
                                                    │                         │
                                                    │  3. FiducialModelPi     │
                                                    │     ::GetPose()         │
                                                    │   - Detect ellipses     │
                                                    │   - Match cross-ratio   │
                                                    │   - solvePnP → R, t     │
                                                    │                         │
                                                    │  4. Publish kết quả     │
                                                    └──────────┬──────────────┘
                                                               │
               ┌───────────────────────────────────────────────┤
               │               │               │               │
    ┌──────────▼──┐  ┌─────────▼──┐  ┌────────▼────┐  ┌──────▼──────────┐
    │ /fiducials/ │  │ /fiducials/ │  │ /fiducials/ │  │   TF2 tree      │
    │   image     │  │detect_fiduc-│  │ fiducial_   │  │ camera_frame    │
    │ (debug img) │  │   ials      │  │marker_array │  │   → marker_N    │
    └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────┘
    sensor_msgs/     DetectionArray   MarkerArray        geometry_msgs/
    Image            (pose + ID)      (RViz arrows)      TransformStamped
```

### Chi tiết bước 3 — Thuật toán detect PiTag

```
Ảnh grayscale
    │
    ├─ Phát hiện ellipses (contour fitting)
    │
    ├─ Nhóm thành cụm 12 chấm (4 góc + 8 chấm encoding)
    │
    ├─ Tính cross-ratio từ 2 dòng điểm
    │      cross_ratio = (AB/BD) / (AC/CD)
    │
    ├─ So khớp với database XML → tìm ID
    │
    └─ solvePnP(model_3d_points, image_2d_points, camera_matrix)
           → rotation R (3×3), translation t (3×1) [đơn vị: mét]
```

---

## 4. Topics, Services, TF

Node chạy trong namespace `/fiducials/`. Tất cả topic bên dưới đều có prefix này.

### Subscribe (đầu vào)

| Topic | Kiểu | Mô tả |
|-------|------|-------|
| `image_color` | `sensor_msgs/Image` | Ảnh từ camera (remapped từ topic camera thực) |
| `camera_info` | `sensor_msgs/CameraInfo` | Thông số nội tại camera (fx, fy, cx, cy) — chỉ đọc 1 lần đầu |

> **QoS:** `SensorDataQoS` (BEST_EFFORT) để tương thích với hầu hết camera driver.

### Publish (đầu ra)

| Topic | Kiểu | Điều kiện | Mô tả |
|-------|------|-----------|-------|
| `/fiducials/image` | `sensor_msgs/Image` | `publish_2d_image: true` | Ảnh debug với trục tọa độ vẽ lên marker. Xem bằng `rqt_image_view` |
| `/fiducials/detect_fiducials` | `DetectionArray` | `MODE_TOPIC` hoặc `MODE_TOPIC_AND_SERVICE` | Mảng tất cả marker detect được, mỗi phần tử gồm pose + ID + label |
| `/fiducials/fiducial_detection_array` | `DetectionArray` | `publish_tf: true` | Giống `detect_fiducials`, dùng nội bộ cho TF |
| `/fiducials/fiducial_marker_array` | `MarkerArray` | `publish_marker_array: true` | Arrows RGB cho RViz (X=đỏ, Y=xanh lá, Z=xanh dương) |

### Services

| Service | Kiểu | Mô tả |
|---------|------|-------|
| `/fiducials/get_fiducials` | `DetectObjects` | Trả về danh sách marker theo yêu cầu. Request: `object_name` (ID muốn lọc, hoặc `""` / `"ALL"` để lấy tất cả). Timeout: 5 giây |
| `/fiducials/stop_tf` | `std_srvs/Empty` | Dừng publish TF (giữ nguyên frame cuối cùng trên cây TF) |

### TF Frames

| Frame | Mô tả |
|-------|-------|
| `<camera_frame_id>` | Frame camera — lấy từ `header.frame_id` của ảnh đến |
| `marker_<ID>` | Frame marker — publish với tần suất 10 Hz khi `publish_tf: true` |

Ví dụ với camera frame `camera_ir_optical_frame` và detect tag ID 1:
```
camera_ir_optical_frame
    └── marker_1   (transform: vị trí + hướng của marker so với camera)
```

### Cấu trúc `DetectionArray`

```
DetectionArray
├── header          (stamp, frame_id = camera frame)
└── detections[]
    ├── label       "tag_1", "tag_2", ...
    ├── id          int (ID của marker)
    ├── detector    string
    ├── score       float (sharpness nếu bật, 0 nếu không)
    └── pose
        └── pose
            ├── position    {x, y, z}  [mét, tâm marker trong camera frame]
            └── orientation {x, y, z, w}  [quaternion]
```

---

## 5. Tham Số Cấu Hình

Khai báo trong YAML config hoặc trực tiếp trong launch file.

| Tham số | Kiểu | Bắt buộc | Mặc định | Mô tả |
|---------|------|----------|----------|-------|
| `fiducial_type` | string | ✓ | — | `TYPE_PI` hoặc `TYPE_ARUCO` |
| `ros_node_mode` | string | ✓ | — | `MODE_TOPIC`, `MODE_SERVICE`, hoặc `MODE_TOPIC_AND_SERVICE` |
| `model_directory` | string | ✓ | — | Đường dẫn thư mục chứa XML model (thêm `/` cuối) |
| `model_filename` | string | ✓ | — | Tên file XML model (ví dụ `piTagIni_0.xml`) |
| `publish_2d_image` | bool | | `false` | Publish ảnh debug với trục tọa độ |
| `publish_tf` | bool | | `false` | Publish TF transform cho mỗi marker |
| `publish_marker_array` | bool | | `false` | Publish RViz MarkerArray |
| `debug_verbosity` | int | | `1` | Mức độ log: 1=tất cả, 2=warnings+errors |
| `use_ir_reflective_markers` | bool | | `false` | Đảo ngược ảnh trước khi detect (cho camera IR với chấm phản quang sáng trên nền tối) |
| `compute_sharpness_measure` | bool | | `false` | Tính độ sắc nét marker, lưu vào `score` |
| `sharpness_calibration_parameter_m` | double | | `0.0` | Hệ số m của hàm sharpness = m × pixel_count + n |
| `sharpness_calibration_parameter_n` | double | | `0.0` | Hệ số n |
| `log_or_calibrate_sharpness_measurements` | bool | | `false` | Ghi log để hiệu chỉnh đường cong sharpness |

### Chọn `ros_node_mode`

| Mode | Khi nào dùng |
|------|-------------|
| `MODE_TOPIC` | Ứng dụng subscribe topic liên tục (realtime) |
| `MODE_SERVICE` | Ứng dụng chỉ cần pose khi yêu cầu (tiết kiệm CPU) |
| `MODE_TOPIC_AND_SERVICE` | Cần cả hai (mặc định khuyến nghị) |

---

## 6. XML Model File

File XML định nghĩa tham số vật lý của từng marker ID.

### Cấu trúc

```xml
<FiducialDetector>
    <PI>
        <ID value="1" />

        <!-- Kích thước tag đo bằng thước (mét): khoảng cách tâm-tâm 2 chấm góc -->
        <LineWidthHeight value="0.100" />

        <!-- Cross-ratio: vị trí chấm B, C trên mỗi dòng, tính theo % của LineWidthHeight -->
        <CrossRatioLine0 AB="0.30" AC="0.55" />
        <CrossRatioLine1 AB="0.25" AC="0.70" />

        <!-- Offset: vị trí tâm tag trong object frame (0,0 = trục tọa độ nằm đúng tâm tag) -->
        <!-- Dùng giá trị khác 0 cho multi-tag board (tâm board là gốc tọa độ) -->
        <Offset x="0.0" y="0.0" />

        <!-- Vùng dùng để tính sharpness (không ảnh hưởng detect) -->
        <SharpnessArea x="-0.01" y="-0.01" width="0.12" height="0.12"/>
    </PI>
    <!-- Thêm <PI>...</PI> cho các ID khác -->
</FiducialDetector>
```

### Quy tắc quan trọng

- **`LineWidthHeight` phải khớp chính xác kích thước tag in thực tế** (đo bằng thước từ tâm chấm góc trên-trái đến tâm chấm góc trên-phải). Sai số → pose sai.
- **`Offset x="0.0" y="0.0"`** → trục tọa độ nằm đúng tâm tag (khuyến nghị cho tag đơn lẻ).
- **Cross-ratio phải thỏa:** `CrossRatio_0 > CrossRatio_1`. Node sẽ bỏ qua tag vi phạm điều kiện này khi load.

### Các file model có sẵn

| File | Kích thước | IDs | Ghi chú |
|------|-----------|-----|---------|
| `piTagIni_0.xml` | 10 cm | 0–3 | Tag đơn, offset (0,0) |
| `piTagIni_1.xml` | 5 cm | 0–3 | Tag đơn, offset (0,0) |
| `piTagIni_2.xml` | 10 cm | 0–33 | IDs 4–33 offset (0,0); IDs 0–3 cấu hình multi-board |
| `piTagIni_3.xml` | 10 cm | 0–3 + extras | Multi-tag board cho IDs 0–3 |
| `piTagIni_0_15cm.xml` | 15 cm | 0–3 | Tag đơn, offset (0,0) |
| `piTagIni_0_20cm.xml` | 20 cm | 0–3 | Tag đơn, offset (0,0) |
| `fpiTagIni_0.xml` | 10 cm | 1–6 | Fast PiTag (thuật toán nhanh hơn) |

### Tạo model cho kích thước mới

```bash
# Từ thư mục gốc workspace
SIZE=0.1639   # kích thước tính bằng mét (ví dụ: fill-A4 ≈ 16.39cm)

cp src/cob_fiducials/common/files/models/piTagIni_0.xml \
   src/cob_fiducials/common/files/models/piTagIni_0_custom.xml

sed -i "s/value=\"0.100\"/value=\"${SIZE}\"/g" \
   src/cob_fiducials/common/files/models/piTagIni_0_custom.xml

colcon build --packages-select cob_fiducials
```

---

## 7. Khởi Động Node

### Cách 1: Launch file (khuyến nghị)

```bash
# Mặc định: piTagIni_0.xml, 10cm, topic mode
ros2 launch cob_fiducials fiducials.launch.py

# Chỉ định model khác
ros2 launch cob_fiducials fiducials.launch.py \
    model_filename:=piTagIni_2.xml

# Camera IR với marker phản quang
ros2 launch cob_fiducials fiducials.launch.py \
    use_ir_reflective_markers:=true

# Tùy chỉnh topic camera
ros2 launch cob_fiducials fiducials.launch.py \
    image_topic:=/camera/color/image_raw \
    camera_info_topic:=/camera/color/camera_info
```

### Tham số launch

| Tham số | Mặc định | Mô tả |
|---------|----------|-------|
| `image_topic` | `/camera/imx335/image_mono8` | Topic ảnh đầu vào |
| `camera_info_topic` | `/camera/imx335/camera_info` | Topic camera info |
| `model_filename` | `piTagIni_0.xml` | File XML model |
| `fiducial_type` | `TYPE_PI` | Loại marker |
| `ros_node_mode` | `MODE_TOPIC_AND_SERVICE` | Chế độ node |
| `publish_tf` | `true` | Publish TF |
| `publish_2d_image` | `true` | Publish ảnh debug |
| `publish_marker_array` | `true` | Publish RViz markers |
| `use_ir_reflective_markers` | `false` | Đảo ảnh cho marker IR |

### Cách 2: Run trực tiếp với YAML

```bash
ros2 run cob_fiducials fiducials \
    --ros-args \
    --params-file src/cob_fiducials/ros/launch/fiducials_0.yaml \
    -p model_directory:=$(ros2 pkg prefix cob_fiducials)/share/cob_fiducials/common/files/models/
```

### Xem ảnh debug

```bash
ros2 run rqt_image_view rqt_image_view /fiducials/image
```

### Xem pose trong terminal

```bash
ros2 topic echo /fiducials/detect_fiducials
```

### Xem TF

```bash
ros2 run tf2_tools view_frames   # Xuất PDF cây TF
ros2 run tf2_ros tf2_echo camera_ir_optical_frame marker_1
```

### Visualize trong RViz2

```bash
ros2 run rviz2 rviz2
```
Thêm display:
- **Image** → topic `/fiducials/image`
- **MarkerArray** → topic `/fiducials/fiducial_marker_array`
- **TF** → bật frame `marker_1`, `marker_2`, ...

---

## 8. Tích Hợp Với Ứng Dụng Khác

### Subscribe topic (Python)

```python
import rclpy
from rclpy.node import Node
from cob_object_detection_msgs.msg import DetectionArray

class TagSubscriber(Node):
    def __init__(self):
        super().__init__('tag_subscriber')
        self.sub = self.create_subscription(
            DetectionArray,
            '/fiducials/detect_fiducials',
            self.callback,
            10
        )

    def callback(self, msg):
        for det in msg.detections:
            p = det.pose.pose.position
            q = det.pose.pose.orientation
            self.get_logger().info(
                f"Tag {det.id}: pos=({p.x:.3f}, {p.y:.3f}, {p.z:.3f}) m"
            )
```

### Gọi service (Python)

```python
from cob_object_detection_msgs.srv import DetectObjects
from std_msgs.msg import String

client = node.create_client(DetectObjects, '/fiducials/get_fiducials')
req = DetectObjects.Request()
req.object_name = String(data='ALL')  # hoặc 'tag_1' để lọc ID cụ thể
future = client.call_async(req)
```

### Đọc TF (Python)

```python
import tf2_ros
from geometry_msgs.msg import TransformStamped

tf_buffer = tf2_ros.Buffer()
tf_listener = tf2_ros.TransformListener(tf_buffer, node)

try:
    t = tf_buffer.lookup_transform(
        'camera_ir_optical_frame',  # target frame
        'marker_1',                 # source frame
        rclpy.time.Time()
    )
    print(f"marker_1 tại: {t.transform.translation}")
except Exception as e:
    print(f"Không tìm thấy transform: {e}")
```

---

## 9. Hiệu Chỉnh Camera (Calibration)

Node cần `camera_info` hợp lệ để tính pose chính xác. Nếu chưa calibrate:

```bash
# Cài ROS 2 camera calibration
sudo apt install ros-humble-camera-calibration

# Chạy calibration với checkerboard 8x6
ros2 run camera_calibration cameracalibrator \
    --size 8x6 \
    --square 0.025 \
    image:=/camera/imx335/image_raw \
    camera:=/camera/imx335
```

Lưu file `.yaml` và truyền vào usb_cam:
```bash
ros2 launch cob_fiducials usb_cam.launch.py \
    camera_info_url:=file:///home/user/camera_calib.yaml
```

---

## 10. Troubleshooting

| Triệu chứng | Nguyên nhân | Giải pháp |
|------------|-------------|-----------|
| Node không detect dù tag trước mặt | `camera_info` chưa nhận | Kiểm tra `ros2 topic echo /camera/.../camera_info` |
| Trục tọa độ lệch xa so với tâm tag | Sai `Offset` trong XML hoặc sai `LineWidthHeight` | Dùng `Offset x="0.0" y="0.0"` và đo lại kích thước tag |
| Detect không ổn định (nhấp nháy) | Ánh sáng không đủ hoặc tag quá nhỏ trong ảnh | Tăng kích thước tag; dùng `--fill-a4` khi in; bổ sung đèn |
| `solvePnP` sai pose (tag nghiêng) | Ảnh bị distortion cao, chưa undistort | Calibrate camera chính xác |
| Topic ảnh không nhận được | QoS mismatch | Camera driver phải publish với BEST_EFFORT |
| `[ERROR] Initializing fiducial detector FAILED` | Sai `model_directory` hoặc file XML không tồn tại | Kiểm tra đường dẫn; chạy `colcon build` lại |
| Pose có z âm hoặc ngược | Camera coordinate convention | Ảnh normal: z ra trước mặt camera là dương |
| Detect nhầm tag (ID sai) | Cross-ratio 2 tag quá gần nhau | Chọn ID có delta cross-ratio > 1.5 từ database |
