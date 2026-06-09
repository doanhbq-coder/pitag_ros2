# Hướng Dẫn Đánh Giá Sai Số PiTag Detector

## Mục Lục

1. [Tổng Quan Về Sai Số](#1-tổng-quan-về-sai-số)
2. [Các Chỉ Số Đánh Giá](#2-các-chỉ-số-đánh-giá)
3. [Cách Script Tính Toán](#3-cách-script-tính-toán)
4. [Chuẩn Bị Trước Khi Đánh Giá](#4-chuẩn-bị-trước-khi-đánh-giá)
5. [Thực Hiện Đánh Giá](#5-thực-hiện-đánh-giá)
6. [Đọc Và Phân Tích Kết Quả](#6-đọc-và-phân-tích-kết-quả)
7. [Đánh Giá Theo Nhiều Điều Kiện](#7-đánh-giá-theo-nhiều-điều-kiện)
8. [Mức Tham Chiếu Chất Lượng](#8-mức-tham-chiếu-chất-lượng)
9. [Nguyên Nhân Và Cách Cải Thiện Sai Số](#9-nguyên-nhân-và-cách-cải-thiện-sai-số)

---

## 1. Tổng Quan Về Sai Số

Khi detector nhận dạng một PiTag và tính ra pose (vị trí + hướng), kết quả có hai loại sai số:

```
Giá trị thực (ground truth)
         │
         │  ← Sai số hệ thống (bias): luôn lệch về 1 hướng cố định
         │                             Ví dụ: z luôn đo dài hơn 2mm
         ▼
   Giá trị trung bình detect được
         │
         │  ← Sai số ngẫu nhiên (noise): dao động quanh trung bình
         │                                Ví dụ: mỗi frame lệch ±0.5mm
         ▼
   Từng giá trị detect (mỗi frame)
```

| Loại sai số | Tên kỹ thuật | Đo bằng | Nguyên nhân chính |
|-------------|-------------|---------|-------------------|
| Sai số hệ thống | Bias / Accuracy | RMSE so với ground truth | Sai `LineWidthHeight` XML, camera chưa calibrate tốt |
| Sai số ngẫu nhiên | Noise / Precision | Std deviation | Nhiễu ảnh, rung động, ánh sáng không ổn định |
| Sai số nội tại | Reprojection error | Pixel | solvePnP hội tụ kém, chấm tròn nhỏ, mờ |

---

## 2. Các Chỉ Số Đánh Giá

### 2.1 Standard Deviation (Std Dev) — đo Noise

**Ý nghĩa:** Nếu đo lại cùng 1 vị trí nhiều lần, các kết quả phân tán bao nhiêu?

```
std = sqrt( Σ(xᵢ - x̄)² / N )
```

- `std = 0` → mọi frame cho đúng cùng 1 giá trị (lý tưởng)
- `std = 1mm` → mỗi frame sai lệch khoảng ±1mm so với trung bình
- **Không nói lên giá trị trung bình có đúng hay không**

### 2.2 RMSE (Root Mean Square Error) — đo Accuracy

**Ý nghĩa:** Trung bình khoảng cách từ mỗi kết quả đến giá trị thực.

```
RMSE = sqrt( Σ(xᵢ - x_thực)² / N )
```

- Cần biết `x_thực` (ground truth — đo bằng thước)
- RMSE = 0 → hoàn hảo
- RMSE bao gồm cả bias lẫn noise: `RMSE² = bias² + std²`

### 2.3 Bias — sai số hệ thống

```
bias = x̄_detected - x_thực
```

- `bias = +2mm` → detector luôn ước lượng xa hơn thực tế 2mm
- Bias không đổi theo thời gian → có thể bù trừ trong phần mềm nếu cần

### 2.4 Reprojection Error — chất lượng solvePnP

**Ý nghĩa:** Sau khi tính được R, t từ solvePnP, chiếu ngược 12 chấm 3D về ảnh 2D rồi đo khoảng cách (pixel) với vị trí chấm thực tế detect được.

```
12 điểm 3D trong model
    │  R, t từ solvePnP
    ▼
12 điểm 2D chiếu lại (projected)
    │
    ├── so sánh với ──→  12 điểm 2D detect thực tế
    │
    ▼
reproj_error = mean( sqrt(Δu² + Δv²) ) qua 12 điểm   [đơn vị: pixel]
```

- Đây là chỉ số **nội tại** — không cần thước đo ngoài
- Tự động publish vào field `score` của `/fiducials/detect_fiducials`
- Phản ánh chất lượng calibration và độ rõ nét của marker trong ảnh

---

## 3. Cách Script Tính Toán

### 3.1 Luồng hoạt động

```
Bật detector (fiducials node)
    │
    ▼
Bật evaluate_accuracy.py
    │
    ▼  Subscribe /fiducials/detect_fiducials
    │
    ├─ [Mỗi frame nhận được]
    │       Lấy: position (x,y,z), quaternion (qx,qy,qz,qw), score (reproj_error)
    │       Chuyển quaternion → Euler angles (roll, pitch, yaw)
    │       Lưu vào buffer: samples[tag_id].append(...)
    │
    ├─ [Khi đủ N mẫu]
    │       Tính: mean, std, RMSE, min, max cho mỗi trục
    │       In báo cáo ra terminal
    │       Lưu CSV
    │
    └─ Kết thúc
```

### 3.2 Chuyển đổi Quaternion → Euler

Script chuyển orientation từ quaternion `(qx, qy, qz, qw)` sang góc Euler `(roll, pitch, yaw)` theo convention ZYX:

```
roll  = atan2( 2(qw·qx + qy·qz),  1 - 2(qx² + qy²) )   [xoay quanh X]
pitch = asin(  2(qw·qy - qz·qx) )                         [xoay quanh Y]
yaw   = atan2( 2(qw·qz + qx·qy),  1 - 2(qy² + qz²) )   [xoay quanh Z]
```

### 3.3 Reprojection error được tính trong C++ (FiducialModelPi.cpp)

Sau khi `solvePnP` trả về:
```cpp
cv::projectPoints(obj_pts_3d, rvec, tvec, camera_matrix, dist_coeffs, projected_2d);

error = mean( sqrt( (projected[i].x - detected[i].x)² +
                    (projected[i].y - detected[i].y)² ) )  // qua 12 điểm
```

Giá trị này được lưu vào `t_pose.reproj_error` và publish qua `detection.score`.

---

## 4. Chuẩn Bị Trước Khi Đánh Giá

### 4.1 Checklist bắt buộc

- [ ] Camera đã **calibrate** (có file `.yaml` hợp lệ, không dùng camera_info mặc định)
- [ ] `LineWidthHeight` trong XML khớp **chính xác** kích thước tag in ra (đo bằng thước kẹp)
- [ ] Tag in rõ nét, không bị mờ, nhàu, cong vênh
- [ ] Môi trường đủ sáng, đồng đều (tránh bóng đổ trên tag)
- [ ] Camera và tag **cố định** trong suốt quá trình thu mẫu

### 4.2 Đo kích thước tag chính xác

Dùng thước kẹp (vernier caliper) đo từ **tâm chấm tròn góc trên-trái** đến **tâm chấm tròn góc trên-phải**:

```
  ●────────────────────●
 TL                   TR
  ←── LineWidthHeight ──→
```

Đây là giá trị `LineWidthHeight` trong XML, đơn vị **mét**.  
Ví dụ đo được `101.2mm` → set `value="0.1012"`.

### 4.3 Setup cho ground truth test

Dùng thước đo từ **mặt kính camera** đến **mặt phẳng tag** theo phương thẳng đứng với tag:

```
[Camera]
    │  ← khoảng cách Z thực (đo thẳng góc)
    │
[Tag phẳng, vuông góc với trục quang camera]
```

- Tag phải **vuông góc** với trục quang (không nghiêng)
- Đo đến **mặt phẳng tag** (không phải nền bàn)
- Dùng thước cứng, không dùng thước dây

---

## 5. Thực Hiện Đánh Giá

### 5.1 Khởi động hệ thống

```bash
# Terminal 1: camera
ros2 launch cob_fiducials usb_cam.launch.py

# Terminal 2: detector
source install/setup.bash
ros2 launch cob_fiducials fiducials.launch.py

# Xác nhận đang detect được
ros2 topic echo /fiducials/detect_fiducials --once
```

### 5.2 Repeatability test (đo noise)

**Mục tiêu:** Tag đứng yên → đo mức dao động của pose.

```bash
# Đặt tag lên bàn, không chạm vào trong lúc đo
source install/setup.bash

ros2 run cob_fiducials evaluate_accuracy \
    --mode repeatability \
    --n 200
```

- `--n 200`: thu 200 frame (khoảng 6–7 giây với camera 30fps)
- Trong lúc đo: **không chạm vào tag hoặc camera**
- Đặt tag xa nguồn rung (máy bơm, quạt, v.v.)

### 5.3 Ground truth test (đo sai số tuyệt đối)

**Mục tiêu:** So sánh Z detect được với khoảng cách thực đo bằng thước.

```bash
# Bước 1: đo khoảng cách thực bằng thước → ví dụ 0.502m
# Bước 2:
ros2 run cob_fiducials evaluate_accuracy \
    --mode ground_truth \
    --distance 0.502 \
    --n 100
```

**Lặp lại ở nhiều khoảng cách:**

```bash
for dist in 0.200 0.300 0.400 0.500 0.700 1.000; do
    echo "=== Đặt tag ở ${dist}m rồi nhấn Enter ==="
    read
    ros2 run cob_fiducials evaluate_accuracy \
        --mode ground_truth \
        --distance $dist \
        --n 100
done
```

### 5.4 Xem reprojection error realtime

```bash
# Theo dõi score (= reproj error [px]) liên tục
ros2 topic echo /fiducials/detect_fiducials \
    | grep -A1 "score:"
```

Hoặc trong Python:
```python
# xem score realtime
ros2 run cob_fiducials evaluate_accuracy --mode repeatability --n 50
```

---

## 6. Đọc Và Phân Tích Kết Quả

### 6.1 Ví dụ output và cách đọc

```
======================================================================
  KẾT QUẢ ĐÁNH GIÁ SAI SỐ  —  mode: GROUND_TRUTH
======================================================================

  Tag ID = 1  (100 mẫu)
  Trục       Trung bình      Std Dev  Đơn vị
  --------------------------------------------------
  X             -1.24 mm  ±0.38 mm       ← (A) trung bình lệch -1.24mm sang trái
  Y              0.82 mm  ±0.41 mm
  Z (depth)    503.15 mm  ±1.02 mm       ← (B) detect 503mm, thực 500mm

  Roll           -0.21°   ±0.15°
  Pitch           0.43°   ±0.18°
  Yaw             0.07°   ±0.22°         ← (C) noise góc ~0.2°

  Reproj err      0.91 px  ±0.14 px      ← (D) chất lượng solvePnP

  ── Ground Truth Z = 500.0 mm ──
  Sai số trung bình (bias) Z :  +3.15 mm ← (E) luôn đo xa hơn 3.15mm
  RMSE Z                     :   3.31 mm ← (F) sai số tổng hợp
  Sai số tương đối           :   0.63 %  ← (G) tỉ lệ so với khoảng cách

  ── Tổng hợp ──
  Noise vị trí 3D (|std|)    :  1.14 mm
  Noise góc 3D (|std|)       :  0.31°
  → Chất lượng: TỐT (< 1 px)
```

### 6.2 Giải thích từng dòng

| Ký hiệu | Ý nghĩa | Hành động nếu lớn |
|---------|---------|-------------------|
| **(A)** X/Y mean lệch | Tâm tag không chính giữa trục quang, hoặc `Offset` XML chưa đúng | Bình thường nếu nhỏ hơn 5mm |
| **(B)** Z mean | Khoảng cách detect trung bình | So sánh với thước để tính bias |
| **(C)** Std góc | Noise về hướng của tag | > 1° → cần cải thiện độ sáng/kích thước tag |
| **(D)** Reproj error | Chất lượng fit của solvePnP | > 2px → xem [Mục 9](#9-nguyên-nhân-và-cách-cải-thiện-sai-số) |
| **(E)** Bias Z | Sai số hệ thống theo chiều sâu | > 5mm → kiểm tra `LineWidthHeight` |
| **(F)** RMSE Z | Sai số tổng hợp (noise + bias) | Chỉ số chính để so sánh hệ thống |
| **(G)** Sai số % | Tỉ lệ lỗi so với khoảng cách thực | > 2% → calibrate lại camera |

### 6.3 File CSV đầu ra

Script lưu kết quả vào file `pitag_accuracy_<mode>_<timestamp>.csv`:

| Cột | Ý nghĩa |
|-----|---------|
| `tag_id` | ID của marker |
| `n_samples` | Số mẫu thu |
| `x_mean_mm`, `x_std_mm` | Vị trí X trung bình và noise [mm] |
| `y_mean_mm`, `y_std_mm` | Vị trí Y |
| `z_mean_mm`, `z_std_mm` | Vị trí Z (khoảng cách) |
| `roll/pitch/yaw_mean_deg` | Góc trung bình [độ] |
| `roll/pitch/yaw_std_deg` | Noise góc [độ] |
| `reproj_mean_px` | Reprojection error trung bình [pixel] |
| `gt_z_mm` | Ground truth Z nếu có |
| `z_bias_mm` | Bias theo Z = `z_mean - gt_z` |

**Vẽ đồ thị với Python:**

```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('pitag_accuracy_ground_truth_*.csv')
plt.bar(df['gt_z_mm'], df['z_bias_mm'])
plt.xlabel('Khoảng cách thực (mm)')
plt.ylabel('Bias Z (mm)')
plt.title('Sai số theo khoảng cách')
plt.grid(True)
plt.show()
```

---

## 7. Đánh Giá Theo Nhiều Điều Kiện

Để có đánh giá toàn diện, chạy theo bảng sau:

### 7.1 Theo khoảng cách

| Khoảng cách | Lệnh | Ghi chú |
|------------|------|---------|
| 20 cm | `--distance 0.200` | Cận — tag to trong ảnh |
| 30 cm | `--distance 0.300` | |
| 50 cm | `--distance 0.500` | Điển hình |
| 70 cm | `--distance 0.700` | |
| 100 cm | `--distance 1.000` | Xa — tag nhỏ trong ảnh |

### 7.2 Theo góc nghiêng (thủ công)

Script không tự xoay tag. Để đánh giá ảnh hưởng của góc nghiêng, nghiêng tag một góc cố định rồi chạy repeatability test:

```
Tag vuông góc (0°)  →  nghiêng 15°  →  nghiêng 30°  →  nghiêng 45°
```

```bash
# Sau mỗi lần điều chỉnh góc, chạy:
ros2 run cob_fiducials evaluate_accuracy --mode repeatability --n 100
```

So sánh `reproj_error` và `std` ở các góc khác nhau → biết được giới hạn góc detect ổn định.

### 7.3 Theo điều kiện ánh sáng

Chạy repeatability test trong 3 điều kiện:
- Đèn phòng bình thường
- Ánh sáng trực tiếp mạnh
- Ánh sáng yếu (buổi tối)

So sánh `reproj_error` và `std` → biết ngưỡng ánh sáng tối thiểu.

---

## 8. Mức Tham Chiếu Chất Lượng

### 8.1 Reprojection error

| Giá trị | Đánh giá | Hành động |
|---------|----------|-----------|
| < 1.0 px | **Tốt** — calibration chính xác | Không cần làm gì |
| 1.0 – 2.5 px | **Khá** — dùng được cho hầu hết ứng dụng | Có thể calibrate lại để cải thiện |
| 2.5 – 5.0 px | **Kém** — pose sẽ bị sai đáng kể | Cần calibrate lại camera |
| > 5.0 px | **Rất kém** — solvePnP thất bại | Tag quá nhỏ, mờ, hoặc camera_info sai |

### 8.2 Noise vị trí (std)

| Tag 10cm, khoảng cách 50cm | Đánh giá |
|---------------------------|----------|
| std_xyz < 0.5 mm | Tốt |
| std_xyz 0.5 – 2 mm | Chấp nhận được |
| std_xyz > 2 mm | Cần cải thiện (ánh sáng, kích thước tag) |

### 8.3 Noise góc (std)

| Std góc | Đánh giá |
|---------|----------|
| < 0.3° | Tốt |
| 0.3° – 1.0° | Chấp nhận |
| > 1.0° | Cần tag lớn hơn hoặc ánh sáng tốt hơn |

### 8.4 Bias Z (sai số khoảng cách)

| Bias / khoảng cách | Đánh giá | Nguyên nhân thường gặp |
|-------------------|----------|------------------------|
| < 0.5% | Tốt | |
| 0.5% – 2% | Khá | Sai nhẹ `LineWidthHeight` |
| > 2% | Kém | Sai `LineWidthHeight` hoặc camera chưa calibrate |

---

## 9. Nguyên Nhân Và Cách Cải Thiện Sai Số

### 9.1 Reprojection error cao (> 2 px)

```
Kiểm tra theo thứ tự:

1. Camera calibration
   → Chạy lại calibration với nhiều ảnh hơn (> 30 ảnh)
   → Dùng checkerboard chất lượng cao, in phẳng

2. LineWidthHeight sai
   → Đo lại bằng thước kẹp (vernier caliper)
   → Sai 1mm trên tag 10cm → sai ~1% khoảng cách

3. Tag quá nhỏ trong ảnh
   → Tối thiểu: mỗi chấm tròn cần ≥ 5px đường kính
   → Dùng tag lớn hơn (--fill-a4) hoặc đặt gần hơn

4. Ảnh mờ / thiếu sáng
   → Kiểm tra sharpness: ros2 topic echo /fiducials/detect_fiducials
   → Tăng exposure hoặc thêm đèn
```

### 9.2 Noise vị trí cao (std > 2 mm)

```
1. Rung động cơ học
   → Đặt camera và tag trên bề mặt ổn định
   → Tránh rung từ quạt, máy móc gần đó

2. Nhiễu ảnh (image noise)
   → Tăng exposure time (giảm ISO/gain)
   → Dùng camera có sensor tốt hơn

3. Ánh sáng thay đổi (đèn nháy, ánh sáng tự nhiên thay đổi)
   → Dùng đèn LED ổn định DC
   → Che chắn ánh sáng bên ngoài
```

### 9.3 Bias Z lớn (> 1%)

```
1. LineWidthHeight không khớp (nguyên nhân #1)
   → Đo lại kích thước in thực tế
   → Máy in có thể scale không đúng 100%
   → Cách kiểm tra nhanh: nếu bias dương (đo xa hơn thực) →
     LineWidthHeight trong XML đang nhỏ hơn thực tế

2. Camera matrix sai (fx, fy không chính xác)
   → Calibrate lại, kiểm tra residual error

3. Tag không vuông góc với trục quang
   → Khi nghiêng tag, Z đo được bằng khoảng cách đến tâm tag
     (không phải khoảng cách vuông góc từ camera đến mặt phẳng tag)
```

### 9.4 Sơ đồ chẩn đoán nhanh

```
reproj_error > 2px ?
    ├─ Có → Kiểm tra calibration và LineWidthHeight trước
    └─ Không ↓

std_xyz > 2mm ?
    ├─ Có → Kiểm tra ánh sáng, rung động, kích thước tag trong ảnh
    └─ Không ↓

bias_Z > 1% ?
    ├─ Có → Đo lại LineWidthHeight thật cẩn thận
    └─ Không ↓

→ Hệ thống hoạt động tốt ✓
```
