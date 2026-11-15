# Scale_ana – Khối chuẩn hóa tín hiệu analog

## 1. Chức năng

`Scale_ana` là Function Block dùng để kiểm tra và chuẩn hóa giá trị tín hiệu analog đầu vào từ miền giá trị thô (Raw) sang miền giá trị kỹ thuật (Engineering).

FB thực hiện:
- Kiểm tra cấu hình miền Raw và miền kỹ thuật.
- Phát hiện tín hiệu Raw thấp hơn hoặc cao hơn giới hạn cấu hình.
- Xác nhận lỗi bằng bộ định thời 1 giây.
- Chốt lỗi thấp/cao và cho phép Reset khi tín hiệu đã trở về vùng hợp lệ.
- Tính toán giá trị kỹ thuật `Current_A`.
- Tổng hợp trạng thái lỗi `Fault`.

## 2. Giao diện FB

### Input

| Tên | Kiểu | Giá trị mặc định | Mô tả |
|---|---|---:|---|
| `Enable` | Bool | — | Cho phép FB kiểm tra và chuẩn hóa tín hiệu |
| `Raw_I` | Int | — | Giá trị Raw đầu vào |
| `RawMax_I` | Int | `27648` | Giới hạn Raw lớn nhất |
| `RawMin_I` | Int | `0` | Giới hạn Raw nhỏ nhất |
| `PerMax_I` | Real | `100.0` | Giá trị kỹ thuật lớn nhất |
| `PerMin_I` | Real | `0.0` | Giá trị kỹ thuật nhỏ nhất |
| `Reset` | Bool | — | Xóa lỗi thấp/cao khi Raw đã trở về vùng hợp lệ |

### Output

| Tên | Kiểu | Mô tả |
|---|---|---|
| `Current_A` | Real | Giá trị analog sau chuẩn hóa |
| `OverCurrent` | Bool | Lỗi Raw vượt giới hạn trên |
| `UnderCurrent` | Bool | Lỗi Raw thấp hơn giới hạn dưới |
| `ConfigFault` | Bool | Lỗi cấu hình miền đo |
| `Fault` | Bool | Lỗi tổng hợp |

## 3. Nguyên lý hoạt động

### 3.1. Kiểm tra cấu hình

FB tính:
- `RawSpan_I = RawMax_I - RawMin_I`
- `PerSpan_I = PerMax_I - PerMin_I`

`ConfigFault` được kích hoạt khi `RawSpan_I <= 0` hoặc `PerSpan_I <= 0`.

### 3.2. Kiểm tra giới hạn Raw

- `RawUnderRange = TRUE` khi `Raw_I < RawMin_I`.
- `RawOverRange = TRUE` khi `Raw_I > RawMax_I`.

Hai bộ định thời độc lập xác nhận tình trạng vượt giới hạn:
- Raw thấp liên tục 1 giây → đặt `UnderCurrent`.
- Raw cao liên tục 1 giây → đặt `OverCurrent`.

Điều kiện chạy timer: `Enable = TRUE`, không có `ConfigFault` và điều kiện vượt giới hạn tương ứng đang tồn tại.

### 3.3. Reset lỗi

Khi `Reset = TRUE`:
- Xóa `UnderCurrent` nếu `RawUnderRange = FALSE`.
- Xóa `OverCurrent` nếu `RawOverRange = FALSE`.

Vì vậy, Reset không xóa lỗi nếu tín hiệu Raw vẫn nằm ngoài giới hạn tương ứng.

### 3.4. Chuẩn hóa tín hiệu

Chỉ tính toán khi:
- `Enable = TRUE`.
- Không có `ConfigFault`.
- Không có `UnderCurrent` hoặc `OverCurrent`.
- Raw hiện tại không thấp hơn `RawMin_I` và không cao hơn `RawMax_I`.

Công thức:

```text
Current_A = ((Raw_I - RawMin_I) / (RawMax_I - RawMin_I))
            * (PerMax_I - PerMin_I) + PerMin_I
```

Giá trị sau tính toán được giới hạn trong khoảng `PerMin_I` đến `PerMax_I`.

### 3.5. Tổng hợp lỗi

```text
Fault = ConfigFault OR UnderCurrent OR OverCurrent
```

## 4. Dữ liệu nội bộ

| Tên | Kiểu | Chức năng |
|---|---|---|
| `Time_check_1` | TON_TIME | Xác nhận lỗi Raw thấp trong 1 giây |
| `Time_check_2` | TON_TIME | Xác nhận lỗi Raw cao trong 1 giây |
| `RawSpan_I` | DInt | Độ rộng miền Raw |
| `PerSpan_I` | Real | Độ rộng miền kỹ thuật |
| `I` | Real | Giá trị trung gian sau chuẩn hóa |
| `RawOverRange` | Bool | Cờ Raw vượt giới hạn trên |
| `RawUnderRange` | Bool | Cờ Raw thấp hơn giới hạn dưới |

## 5. Lưu ý khi sử dụng

- Cần cấu hình `RawMin_I`, `RawMax_I`, `PerMin_I` và `PerMax_I` đúng với module analog và cảm biến thực tế.
- Giá trị mặc định `RawMax_I = 27648`, `PerMin_I = 0.0`, `PerMax_I = 100.0` chỉ là giá trị mặc định trong FB, không đảm bảo phù hợp với mọi loại tín hiệu.
- `Current_A` chỉ được cập nhật khi các điều kiện chuẩn hóa hợp lệ. Khi `Enable` OFF hoặc có lỗi, code hiện tại không gán giá trị thay thế cho `Current_A`, do đó giá trị có thể giữ giá trị của chu kỳ trước.
- `UnderCurrent` và `OverCurrent` là các cờ chốt; cần tín hiệu `Reset` và Raw trở về vùng hợp lệ để xóa.
- `ConfigFault` được tính trực tiếp từ tham số cấu hình, không phải cờ chốt.
- FB hiện không kiểm tra riêng tín hiệu đứt dây theo chuẩn 4–20 mA; việc phát hiện phụ thuộc vào giới hạn Raw được cấu hình.

## 6. Tệp

- `Scale_ana.scl`: mã nguồn Function Block `Scale_ana`.
