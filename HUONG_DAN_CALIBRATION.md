# Hướng Dẫn Calibrate Camera cho PiTag Detection

## 1. Tổng Quan

Camera calibration xác định các thông số nội tại (intrinsics) của camera:
- **Camera matrix (K)**: tiêu cự fx, fy và điểm trung tâm cx, cy
- **Distortion coefficients (D)**: hệ số méo ống kính (barrel/pincushion)

Không có calibration → pose estimation trả về `{0,0,0}` không có nghĩa.

---

## 2. Tạo File In Checkerboard

```bash
cd ~/Documents/pitag_ros2/src/PiTag-generator

# Mặc định: 8x7 ô, mỗi ô 25mm (khuyến nghị)
python3 gen_checkerboard.py

# Hoặc tùy chỉnh kích thước ô (30mm nếu máy in A4 cảm thấy nhỏ)
python3 gen_checkerboard.py --square 30

# Mở file PDF để in
xdg-open checkerboard.pdf
```

### Khi In

| Yêu cầu | Chi tiết |
|---------|---------|
| Tỉ lệ in | **100% (Actual Size)** — KHÔNG chọn "Fit to Page" |
| Kiểm tra | Sau khi in, dùng thước đo 1 ô vuông → phải đúng **25mm** |
| Dán lên | Bìa cứng / mặt phẳng để tránh cong vênh |

---

## 3. Chạy Calibration

### Bước 1 — Khởi động camera

```bash
# Terminal 1
source /opt/ros/humble/setup.bash
source ~/Documents/pitag_ros2/install/setup.bash
ros2 launch cob_fiducials usb_cam.launch.py
```

### Bước 2 — Chạy công cụ calibration

```bash
# Terminal 2
source /opt/ros/humble/setup.bash
ros2 run camera_calibration cameracalibrator \
  --size 7x6 \
  --square 0.025 \
  --ros-args \
  -r image:=/camera/imx335/image_raw \
  -r camera:=/camera/imx335
```

> **Lưu ý:** `--size 7x6` = số **góc trong** (không phải số ô). Checkerboard 8x7 ô có 7x6 góc trong.
> Nếu dùng `--square 30mm` khi in thì đổi thành `--square 0.030`.

### Bước 3 — Di chuyển checkerboard

Giao diện hiện 4 thanh tiến trình:

| Thanh | Ý nghĩa | Cách lấp đầy |
|-------|---------|-------------|
| **X** | Vị trí ngang | Di chuyển sang trái / phải |
| **Y** | Vị trí dọc | Di chuyển lên / xuống |
| **Size** | Khoảng cách | Lại gần rồi ra xa camera |
| **Skew** | Góc nghiêng | Nghiêng checkerboard các hướng |

**Thứ tự di chuyển khuyến nghị:**

```
1. Giữa màn hình, thẳng góc với camera
2. Dịch sang trái → rồi sang phải
3. Dịch lên trên → rồi xuống dưới
4. Lại gần camera (~30cm) → ra xa (~120cm)
5. Nghiêng trái/phải khoảng 30-45°
6. Nghiêng lên/xuống khoảng 30-45°
7. Xoay tờ giấy theo chiều kim đồng hồ ~30°
8. Lặp lại ở nhiều góc kết hợp
```

Cần thu thập **ít nhất 40-50 mẫu** phân bố đều → 4 thanh chuyển xanh đậm.

### Bước 4 — Lưu kết quả

Khi cả 4 thanh đủ xanh:
1. Nhấn **CALIBRATE** → chờ 10-30 giây tính toán
2. Nhấn **SAVE** → lưu vào `/tmp/calibrationdata.tar.gz`
3. Nhấn **COMMIT** → ghi vào ROS (có thể bị crash, không sao)

---

## 4. Lưu File Calibration Thủ Công (nếu COMMIT bị crash)

```bash
# Giải nén file calibration
mkdir -p /tmp/calib_extract
tar -xf /tmp/calibrationdata.tar.gz -C /tmp/calib_extract ost.yaml

# Xem kết quả
cat /tmp/calib_extract/ost.yaml

# Tạo thư mục lưu trữ
mkdir -p ~/.ros/camera_info

# Copy và đổi tên theo tên camera
cp /tmp/calib_extract/ost.yaml ~/.ros/camera_info/imx335.yaml

# Sửa camera_name trong file
sed -i 's/camera_name: narrow_stereo/camera_name: imx335/' ~/.ros/camera_info/imx335.yaml
```

---

## 5. Kết Quả Calibration (Ví dụ Camera TUOPUONE IMX335)

File lưu tại: `~/.ros/camera_info/imx335.yaml`

```yaml
image_width: 640
image_height: 480
camera_name: imx335
camera_matrix:
  rows: 3
  cols: 3
  data: [489.50507, 0.0, 303.41207,
         0.0, 490.74064, 240.73248,
         0.0, 0.0, 1.0]
distortion_model: plumb_bob
distortion_coefficients:
  rows: 1
  cols: 5
  data: [-0.416793, 0.152571, -0.000992, 0.007617, 0.000000]
rectification_matrix:
  rows: 3
  cols: 3
  data: [1.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 1.0]
projection_matrix:
  rows: 3
  cols: 4
  data: [385.63974, 0.0, 306.13146, 0.0,
         0.0, 435.91244, 240.51519, 0.0,
         0.0, 0.0, 1.0, 0.0]
```

**Giải thích các thông số:**

| Ký hiệu | Giá trị | Ý nghĩa |
|---------|---------|---------|
| `fx` | 489.5 | Tiêu cự theo chiều ngang (pixel) |
| `fy` | 490.7 | Tiêu cự theo chiều dọc (pixel) |
| `cx` | 303.4 | Điểm trung tâm ảnh X (pixel) |
| `cy` | 240.7 | Điểm trung tâm ảnh Y (pixel) |
| `D[0]` | -0.417 | Méo hướng tâm bậc 2 (barrel distortion) |
| `D[1]` | 0.153 | Méo hướng tâm bậc 4 |

---

## 6. Chạy Hệ Thống Với Calibration

```bash
# Terminal 1 — camera với calibration
ros2 launch cob_fiducials usb_cam.launch.py \
  camera_info_url:=file:///home/neo/.ros/camera_info/imx335.yaml

# Terminal 2 — PiTag detector
ros2 launch cob_fiducials fiducials.launch.py

# Terminal 3 — xem kết quả pose 3D
source ~/Documents/pitag_ros2/install/setup.bash
ros2 topic echo /fiducials/detect_fiducials
```

**Kết quả detection với calibration đúng:**

```yaml
detections:
- label: tag_0
  id: 0
  pose:
    pose:
      position:
        x: -0.052    # lệch trái 5.2cm
        y:  0.018    # lệch lên 1.8cm
        z:  0.634    # cách camera 63.4cm  ← quan trọng nhất
      orientation:
        w: 0.997
        x: 0.021
        y: 0.070
        z: 0.003
```

---

## 7. Đánh Giá Chất Lượng Calibration

Sau khi nhấn CALIBRATE, terminal in ra **reprojection error**:

```
mono pinhole calibration...
D = [...]
K = [...]
```

Nếu không thấy số lỗi rõ ràng, kiểm tra qua cách khác:

| Chỉ tiêu | Tốt | Chấp nhận được | Cần calibrate lại |
|---------|-----|----------------|------------------|
| Số mẫu | ≥ 60 | 40-60 | < 40 |
| Phân bố X, Y | Đều khắp | Hơi lệch | Tập trung 1 góc |
| Skew | Có mẫu nghiêng | Ít nghiêng | Không có |

**Khi nào nên calibrate lại:**
- Thay đổi độ phân giải camera (640x480 ↔ 1280x720)
- Thay ống kính hoặc tháo lắp camera
- Kết quả pose 3D trông không hợp lý

---

## 8. Calibrate Với Độ Phân Giải Khác

Nếu dùng resolution khác (ví dụ 1280x720):

```bash
# Chạy camera ở resolution mới
ros2 launch cob_fiducials usb_cam.launch.py \
  image_width:=1280 image_height:=720

# Calibrate lại
ros2 run camera_calibration cameracalibrator \
  --size 7x6 --square 0.025 \
  --ros-args \
  -r image:=/camera/imx335/image_raw \
  -r camera:=/camera/imx335

# Lưu với tên khác
cp /tmp/calib_extract/ost.yaml ~/.ros/camera_info/imx335_1280x720.yaml
```

Mỗi resolution cần file calibration riêng.
