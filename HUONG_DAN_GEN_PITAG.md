# Hướng Dẫn Chi Tiết: PiTag Generator

## 1. Cài Đặt (Ubuntu 20.04 / 22.04 trở lên)

```bash
# Bước 1: Cài thư viện hệ thống cần thiết cho cairosvg
sudo apt-get install libcairo2-dev

# Bước 2: Cài các thư viện Python
pip3 install svgwrite cairosvg

# Nếu gặp lỗi "externally managed environment":
pip3 install svgwrite cairosvg --break-system-packages
```

---

## 2. Sinh Marker Theo ID (Cách Nhanh — Khuyến Nghị)

File `pitag_by_id.py` tra cứu cross-ratio từ database của `cob_fiducials` và tự động gọi `pi-tag_gen.py`.

### Chế độ tương tác (nhập ID qua terminal)

```bash
python3 pitag_by_id.py
```

Ví dụ phiên làm việc:
```
Nhập ID (hoặc 'list'/'exit'): 1
  Kích thước marker cm [mặc định 10.0]: 
  Căn giữa trang A4? [Y/n]: 
  Xuất thêm PDF? [Y/n]: 
  In thông tin kỹ thuật lên marker? [y/N]: 
→ Tạo ra: pitag_ID1.svg và pitag_ID1.pdf
```

### Chế độ dòng lệnh (tích hợp vào script)

```bash
# Sinh marker ID=1, kích thước 10cm, xuất A4 + PDF
python3 pitag_by_id.py 1

# Sinh marker ID=3, kích thước 15cm, có thông tin kỹ thuật
python3 pitag_by_id.py 3 --size 15 --info

# Chỉ SVG, không PDF
python3 pitag_by_id.py 0 --no-pdf

# Tên file tùy chọn
python3 pitag_by_id.py 25 --output my_marker.svg

# Liệt kê tất cả ID có sẵn
python3 pitag_by_id.py --list
```

### Phóng to lấp đầy tờ A4 (`--fill-a4`)

```bash
# Tạo pitag_ID1.pdf với tag phóng to vừa khít A4 (margin 5mm mỗi cạnh)
python3 pitag_by_id.py 1 --fill-a4
```

Flag `--fill-a4` tự động scale **tất cả tham số tỷ lệ thuận** (marker_size, circle_radius, circle_clearance)
để outer square lấp đầy chiều ngắn A4 (21cm − 2×5mm = 20cm). Kết quả in ra:

```
[FILL-A4] marker_size=16.393 cm | circle_radius=1.475 cm | circle_clearance=0.328 cm
  -> Dùng LineWidthHeight value="0.1639" trong XML model
```

> **Quan trọng:** Sau khi in, phải tạo XML model với `LineWidthHeight` khớp kích thước thực.
> Xem hướng dẫn tạo XML model ở [Mục 9](#9-tạo-xml-model-cho-kích-thước-mới).

`--fill-a4` bỏ qua `--size` và tự bật `--A4`. Vẫn có thể kết hợp `--no-pdf` hoặc `--info`.

### Tất cả tùy chọn

| Tham số | Mặc định | Ý nghĩa |
|---------|----------|---------|
| `id` | bắt buộc | ID marker cần tạo |
| `--size` | `10.0` | Kích thước marker (cm) |
| `--radius` | `0.9` | Bán kính vòng tròn (cm) |
| `--clearance` | `0.2` | Khoảng trắng quanh vòng tròn (cm) |
| `--output` | `pitag_ID<n>.svg` | Tên file đầu ra |
| `--no-a4` | — | Không căn giữa A4 |
| `--no-pdf` | — | Không xuất PDF |
| `--info` | — | In thông tin kỹ thuật lên marker |
| `--fill-a4` | — | Phóng to tag lấp đầy A4, bỏ qua `--size` |
| `--list` | — | Liệt kê toàn bộ ID trong database |

---

## 3. Database ID Mặc Định

**Tổng cộng 41 ID** được định nghĩa sẵn trong `cob_fiducials` (file `piTagIni_*.xml`).  
ID **không liên tục** — một số số bị bỏ qua để đảm bảo cross-ratio đủ khác biệt.

### Nhóm cơ bản (piTagIni_0.xml, piTagIni_1.xml) — dùng phổ biến nhất

| ID | AB0  | AC0  | AB1  | AC1  |
|----|------|------|------|------|
| 0  | 0.40 | 0.60 | 0.20 | 0.80 |
| 1  | 0.30 | 0.55 | 0.25 | 0.70 |
| 2  | 0.25 | 0.52 | 0.25 | 0.62 |
| 3  | 0.45 | 0.75 | 0.40 | 0.74 |

### Nhóm mở rộng (piTagIni_2.xml) — ID 0 đến 33 (bỏ 14)

| ID | AB0  | AC0  | AB1  | AC1  | ID | AB0  | AC0  | AB1  | AC1  |
|----|------|------|------|------|----|------|------|------|------|
| 4  | 0.30 | 0.55 | 0.25 | 0.50 | 5  | 0.25 | 0.50 | 0.30 | 0.65 |
| 6  | 0.25 | 0.50 | 0.30 | 0.75 | 7  | 0.40 | 0.65 | 0.25 | 0.50 |
| 8  | 0.25 | 0.50 | 0.40 | 0.75 | 9  | 0.30 | 0.55 | 0.25 | 0.65 |
| 10 | 0.30 | 0.65 | 0.25 | 0.65 | 11 | 0.25 | 0.60 | 0.30 | 0.75 |
| 12 | 0.40 | 0.65 | 0.25 | 0.65 | 13 | 0.40 | 0.75 | 0.25 | 0.65 |
| 15 | 0.30 | 0.65 | 0.25 | 0.70 | 16 | 0.30 | 0.75 | 0.25 | 0.75 |
| 17 | 0.40 | 0.65 | 0.25 | 0.70 | 18 | 0.40 | 0.75 | 0.25 | 0.70 |
| 19 | 0.30 | 0.55 | 0.35 | 0.65 | 20 | 0.35 | 0.60 | 0.30 | 0.65 |
| 21 | 0.35 | 0.60 | 0.30 | 0.75 | 22 | 0.40 | 0.65 | 0.35 | 0.65 |
| 23 | 0.35 | 0.60 | 0.40 | 0.75 | 24 | 0.30 | 0.55 | 0.35 | 0.75 |
| 25 | 0.30 | 0.65 | 0.35 | 0.75 | 26 | 0.35 | 0.70 | 0.30 | 0.75 |
| 27 | 0.40 | 0.65 | 0.35 | 0.75 | 28 | 0.40 | 0.75 | 0.35 | 0.75 |
| 29 | 0.30 | 0.55 | 0.45 | 0.75 | 30 | 0.45 | 0.70 | 0.30 | 0.65 |
| 31 | 0.45 | 0.70 | 0.30 | 0.75 | 32 | 0.40 | 0.65 | 0.45 | 0.75 |
| 33 | 0.45 | 0.70 | 0.40 | 0.75 |    |      |      |      |      |

### Nhóm đặc biệt (piTagIni_3.xml)

| ID | AB0  | AC0  | AB1  | AC1  |
|----|------|------|------|------|
| 36 | 0.40 | 0.60 | 0.35 | 0.60 |
| 38 | 0.40 | 0.60 | 0.20 | 0.50 |
| 48 | 0.25 | 0.45 | 0.20 | 0.55 |
| 55 | 0.30 | 0.55 | 0.25 | 0.55 |
| 64 | 0.20 | 0.45 | 0.20 | 0.80 |
| 69 | 0.20 | 0.55 | 0.25 | 0.70 |
| 73 | 0.25 | 0.70 | 0.20 | 0.80 |
| 79 | 0.35 | 0.65 | 0.25 | 0.75 |

> **ID tối đa là 79.** Số lượng ID không phải vô hạn — mỗi ID phải có cross-ratio đủ khác biệt để detector không nhầm. ID 14, 34, 35, 37, 39–47, 49–54, 56–63, 65–68, 70–72, 74–78 không được dùng.

---

## 4. Sinh Hàng Loạt Marker

```bash
# Tạo tất cả marker ID 0 đến 3 cùng lúc
for id in 0 1 2 3; do
    python3 pitag_by_id.py $id --size 10
done
```

Hoặc tạo file `generate_all.sh`:

```bash
#!/bin/bash
cd "$(dirname "$0")"
for id in 0 1 2 3 25 36; do
    python3 pitag_by_id.py "$id" --size 10
    echo "Done ID=$id"
done
echo "Hoàn thành!"
```

```bash
chmod +x generate_all.sh
./generate_all.sh
```

---

## 5. Cú Pháp Thủ Công (pi-tag_gen.py trực tiếp)

```
python3 pi-tag_gen.py AB0 AC0 AB1 AC1 [tùy chọn]
```

### Tham số bắt buộc

PiTag mã hóa ID bằng **Cross-ratio** (tỉ lệ chéo). Mỗi marker có 2 dòng điểm:

```
Dòng 0 (cạnh Trên và Trái):   A ---[B]---[C]--- D
Dòng 1 (cạnh Dưới và Phải):   A ---[B]---[C]--- D
```

| Tham số | Ý nghĩa | Khoảng hợp lệ |
|---------|---------|----------------|
| `AB0`   | Vị trí điểm B trên Dòng 0 | 0.05 – 0.45 |
| `AC0`   | Vị trí điểm C trên Dòng 0 | 0.55 – 0.95 |
| `AB1`   | Vị trí điểm B trên Dòng 1 | 0.05 – 0.45 |
| `AC1`   | Vị trí điểm C trên Dòng 1 | 0.55 – 0.95 |

**Ràng buộc:** `AB < AC` trên mỗi dòng; Cross-ratio 0 > Cross-ratio 1.

### Tham số tùy chọn

| Tham số | Mặc định | Ý nghĩa |
|---------|----------|---------|
| `--marker_size` | `10.0` | Kích thước marker (cm) |
| `--circle_radius` | `0.9` | Bán kính vòng tròn (cm) |
| `--circle_clearance` | `0.2` | Khoảng trắng quanh vòng tròn (cm) |
| `--output_file` | `pitag_marker.svg` | Tên file đầu ra |
| `--A4` | tắt | Căn giữa trang A4 |
| `--pdf` | tắt | Xuất thêm PDF |
| `--show_info` | tắt | In thông tin kỹ thuật lên marker |

### Ví dụ

```bash
# Lệnh đầy đủ tất cả option (từ README gốc)
python3 pi-tag_gen.py 0.40 0.60 0.30 0.70 \
    --output_file pitag_marker.svg \
    --A4 \
    --pdf \
    --show_info \
    --marker_size 10.0 \
    --circle_radius 0.9 \
    --circle_clearance 0.2

# ID=0, in chuẩn A4
python3 pi-tag_gen.py 0.40 0.60 0.20 0.80 --A4 --pdf --show_info

# ID=1, kích thước 5cm
python3 pi-tag_gen.py 0.30 0.55 0.25 0.70 --marker_size 5.0 --circle_radius 0.45 --output_file pitag_ID1_5cm.svg
```

---

## 6. Lưu Ý Khi Chọn Tham Số

| Vấn đề | Giải pháp |
|--------|-----------|
| "Crossratio 0 must be greater" | Cross-ratio 0 < Cross-ratio 1. Dùng `pitag_by_id.py` để tránh lỗi này |
| Vòng tròn bị chồng lên nhau | Tăng `--clearance` hoặc giảm `--radius` |
| Marker quá lớn so với A4 | Dùng `--fill-a4` để tự động tính kích thước phù hợp |
| Muốn tag to nhất có thể để detect từ xa | Dùng `--fill-a4` — tất cả tham số scale tỷ lệ thuận, chấm tròn cũng to theo |
| File PDF không tạo được | `sudo apt-get install libcairo2-dev && pip3 install cairosvg --upgrade` |

---

## 7. Kích Thước Tag và XML Model Tương Ứng

Mỗi kích thước tag cần một XML model riêng với `LineWidthHeight` khớp chính xác.  
**Nếu sai kích thước → pose tính ra sẽ sai (lệch tỉ lệ).**

### Các model có sẵn

| File XML | Kích thước tag | Ghi chú |
|----------|---------------|---------|
| `piTagIni_0.xml` | 10 cm | IDs 0–3, dùng tag đơn lẻ |
| `piTagIni_1.xml` | 5 cm | IDs 0–3 |
| `piTagIni_2.xml` | 10 cm | IDs 0–33 |
| `piTagIni_0_15cm.xml` | 15 cm | IDs 0–3 |
| `piTagIni_0_20cm.xml` | 20 cm | IDs 0–3 |

### Tạo XML model cho kích thước mới (ví dụ fill-A4 = 16.4 cm)

```bash
cd /path/to/pitag_ros2  # thư mục gốc workspace

# 1. Copy từ file gốc
cp src/cob_fiducials/common/files/models/piTagIni_0.xml \
   src/cob_fiducials/common/files/models/piTagIni_0_164mm.xml

# 2. Sửa LineWidthHeight (thay 0.100 bằng giá trị in ra bởi --fill-a4, ví dụ 0.1639)
sed -i 's/value="0.100"/value="0.1639"/g' \
   src/cob_fiducials/common/files/models/piTagIni_0_164mm.xml

# 3. Cài vào install/
colcon build --packages-select cob_fiducials
```

### Chạy detector với model mới

```bash
ros2 launch cob_fiducials fiducials.launch.py model_filename:=piTagIni_0_164mm.xml
```

---

## 8. Mở và Chỉnh Sửa File SVG

```bash
inkscape pitag_ID1.svg          # Mở bằng Inkscape
sudo apt-get install inkscape   # Cài nếu chưa có
```

---

## 9. Cấu Trúc Marker

```
+--[TL]----[B0]----[C0]----[TR]--+
|                                 |
[B0]    +-----------+          [B1]
|       |           |             |
[C0]    |     +     |          [C1]
|       |   (tâm)   |             |
[B1]    +-----------+          [C1]
|                                 |
+--[BL]----[B1]----[C1]----[BR]--+
```

- **TL, TR, BL, BR**: 4 vòng tròn góc — dùng để định vị marker
- **B0, C0**: 2 vòng tròn mã hóa trên cạnh Trên/Trái (Dòng 0)
- **B1, C1**: 2 vòng tròn mã hóa trên cạnh Dưới/Phải (Dòng 1)
- **Tâm đỏ**: điểm gốc tọa độ của marker
