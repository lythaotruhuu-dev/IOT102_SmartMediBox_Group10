[README.md](https://github.com/user-attachments/files/34878258/README.md)
# 💊 SmartMediBox – Tủ thuốc thông minh

Đồ án môn **IOT102** – Nhóm 10. Hệ thống tủ thuốc 3 ngăn (Sáng – Trưa – Tối) dùng **ESP32**, nhắc giờ uống thuốc theo thời gian thực, **tự mở đúng nắp ngăn thuốc**, theo dõi việc lấy thuốc và **cảnh báo lên điện thoại** qua Blynk khi người dùng quên thuốc.

> ⚠️ Đây là mô hình học tập, **không phải thiết bị y tế**. Hệ thống chỉ nhắc giờ và nhận biết ngăn thuốc đã được lấy ra hay chưa, không kiểm tra loại thuốc hay số viên.

## Mục lục

- [Tính năng](#tính-năng)
- [Phần cứng](#phần-cứng)
- [Sơ đồ đấu nối](#sơ-đồ-đấu-nối)
- [Nguyên lý hoạt động](#nguyên-lý-hoạt-động)
- [Cài đặt và nạp code](#cài-đặt-và-nạp-code)
- [Cấu hình Blynk](#cấu-hình-blynk)
- [Cấu hình thời gian và chế độ demo](#cấu-hình-thời-gian-và-chế-độ-demo)
- [Cấu trúc repo](#cấu-trúc-repo)
- [Kiểm thử và demo](#kiểm-thử-và-demo)
- [Hạn chế và hướng phát triển](#hạn-chế-và-hướng-phát-triển)
- [Thành viên](#thành-viên)

## Tính năng

**Chức năng lõi**
- Đặt giờ uống thuốc cho 3 buổi từ app Blynk, lưu vào bộ nhớ flash (không mất khi mất điện).
- Báo giờ theo thời gian thực bằng **RTC DS3231** và còi; vẫn đúng giờ khi mất WiFi.
- **Servo tự mở đúng nắp** ngăn thuốc khi đến giờ; mỗi ngăn có góc đóng/mở riêng.
- **LCD1602 I2C** hiển thị giờ, nhiệt độ, độ ẩm, buổi uống thuốc tiếp theo và trạng thái nhắc.
- Đo nhiệt độ, độ ẩm trong tủ bằng DHT11, hiển thị trên LCD và app.

**Chức năng nâng cao**
- Cảm biến hồng ngoại từng ngăn nhận biết thuốc **đã được lấy** (có lọc nhiễu 0,7 s).
- LED trạng thái từng ngăn, còi kêu theo mức độ (ngắt quãng, dồn dập).
- **Nhắc lại** sau 5 phút, **cảnh báo quên thuốc** sau 30 phút bằng thông báo đẩy và email (Blynk Event).
- Đồng bộ giờ RTC từ NTP (GMT+7) khi có Internet.
- 3 công tắc trên app để **mở/đóng nắp từ xa** (tự đóng lại sau 60 giây, không can thiệp khi đang nhắc uống thuốc).
- Kết nối WiFi và Blynk **không chặn**: mất mạng thì phần chạy tại chỗ vẫn hoạt động bình thường.

## Phần cứng

| STT | Linh kiện | SL | Vai trò |
|---|---|---|---|
| 1 | ESP32 DevKit | 1 | Xử lý trung tâm, WiFi |
| 2 | RTC DS3231 (pin CR2032) | 1 | Thời gian thực, giữ giờ khi mất điện |
| 3 | LCD1602 + module I2C | 1 | Hiển thị |
| 4 | DHT11 | 1 | Nhiệt độ, độ ẩm |
| 5 | Servo SG90 | 3 (+1 dự phòng) | Mở/đóng 3 nắp |
| 6 | Cảm biến hồng ngoại vật cản FC-51 | 3 (+1 dự phòng) | Nhận biết thuốc đã được lấy |
| 7 | LED 5 mm + điện trở 220 Ω | 3 | Trạng thái từng ngăn |
| 8 | Còi buzzer chủ động | 1 | Báo giờ, nhắc lại |
| 9 | Adapter 5 V 2 A + jack DC | 1 | Nguồn |
| 10 | Tụ hóa 470–1000 µF 16 V | 1–2 | Lọc nguồn servo |
| 11 | Breadboard, dây nối, vật liệu làm mô hình | — | — |

### Bảng chân GPIO

| Thiết bị | Chân ESP32 | Ghi chú |
|---|---|---|
| LCD1602 I2C (0x27) | SDA = GPIO21, SCL = GPIO22 | Chung bus I2C với RTC. Một số module có địa chỉ 0x3F |
| RTC DS3231 (0x68) | SDA = GPIO21, SCL = GPIO22 | |
| DHT11 | GPIO4 | |
| Servo Sáng / Trưa / Tối | GPIO13 / GPIO14 / GPIO27 | Cấp 5 V |
| IR Sáng / Trưa / Tối | GPIO34 / GPIO35 / GPIO32 | `LOW` = còn thuốc, `HIGH` = trống. Cấp 3V3 |
| LED Sáng / Trưa / Tối | GPIO25 / GPIO26 / GPIO33 | GPIO → R 220 Ω → LED → GND |
| Còi buzzer | GPIO23 | Còi chủ động. Nếu còi nhỏ hoặc chân nóng, thêm transistor S8050 + R 1 kΩ |

## Sơ đồ đấu nối

Sơ đồ khối, sơ đồ nguyên lý (ký hiệu IEC) và lưu đồ thuật toán nằm trong thư mục [`docs/`](docs/) (file draw.io, mở bằng [app.diagrams.net](https://app.diagrams.net)).

**Cấp nguồn**
- Adapter 5 V → thanh 5 V → chân **VIN** của ESP32, VCC của 3 servo và LCD.
- RTC, DHT11 và 3 IR lấy **3V3** từ ESP32 (chân GPIO của ESP32 không chịu được 5 V).
- **Nối chung GND** tất cả thiết bị.
- Gắn tụ 470–1000 µF gần chân nguồn servo. Nguồn servo đi dây ngắn, chắc, tránh breadboard rơ.
- Đừng cắm cáp USB vào ESP32 cùng lúc với adapter cấp VIN nếu chưa chắc board có diode chặn. Rút một trong hai khi nạp code.

## Nguyên lý hoạt động

1. Khởi động: khởi tạo LCD, RTC, DHT11, servo (đóng nắp), đọc giờ đã lưu, kết nối WiFi và Blynk.
2. Vòng lặp chính (không dùng `delay()` dài): đọc RTC, đọc DHT11, cập nhật LCD, xử lý từng ngăn bằng máy trạng thái.
3. Khi trùng giờ và ngăn còn thuốc: mở nắp, còi kêu, LED nháy.
4. Thuốc được lấy: tắt còi và LED, đợi 1,5 s rồi đóng nắp.
5. Chưa lấy: sau 5 phút nhắc lại, sau 30 phút đóng nắp và gửi cảnh báo quên thuốc lên điện thoại.

### Máy trạng thái từng ngăn

| Trạng thái | LED | Còi | Nắp | Chuyển sang |
|---|---|---|---|---|
| `IDLE` (còn thuốc) | Sáng | Tắt | Đóng | `ALERTING` khi đến giờ |
| `IDLE` (ngăn trống) | Tắt | Tắt | Đóng | Bỏ qua buổi khi đến giờ |
| `ALERTING` | Nháy chậm | Ngắt quãng | Mở | `TAKEN`, hoặc `REMINDED` sau 5 phút |
| `REMINDED` | Nháy nhanh | Kêu dồn dập | Mở | `TAKEN`, hoặc `MISSED` sau 30 phút |
| `TAKEN` | Tắt | Tắt | Đóng (sau 1,5 s) | `IDLE` sang ngày mới |
| `MISSED` | Nháy chậm | Tắt | Đóng | `TAKEN` nếu lấy muộn, `IDLE` sang ngày mới |

## Cài đặt và nạp code

**Yêu cầu:** Arduino IDE 2.x (hoặc 1.8.x), board **ESP32 by Espressif Systems**.

**Thư viện** (Library Manager):

| Thư viện | Ghi chú |
|---|---|
| Blynk | |
| ESP32Servo | |
| DHT sensor library (Adafruit) | Cài kèm *Adafruit Unified Sensor* |
| LiquidCrystal I2C | |
| RTClib (Adafruit) | |

**Các bước**
1. Clone repo, mở `SmartMediBox/SmartMediBox.ino` (file `.ino` phải nằm trong thư mục cùng tên).
2. Chọn board **ESP32 Dev Module**, chọn đúng cổng COM (cài driver CH340/CP2102 nếu cần).
3. Tạo file `secrets.h` (xem phần bên dưới), điền thông tin của bạn.
4. Nạp code, mở Serial Monitor ở **115200 baud**.

### Thông tin bí mật (Blynk token, WiFi)

**Không commit token và mật khẩu WiFi lên GitHub.** Tạo `SmartMediBox/secrets.h`:

```cpp
// secrets.h - KHONG commit file nay
#define BLYNK_TEMPLATE_ID   "YOUR_TEMPLATE_ID"
#define BLYNK_TEMPLATE_NAME "SmartMediBox"
#define BLYNK_AUTH_TOKEN    "YOUR_BLYNK_AUTH_TOKEN"
#define WIFI_SSID_VALUE     "YOUR_WIFI_NAME"      // WiFi 2.4 GHz
#define WIFI_PASS_VALUE     "YOUR_WIFI_PASSWORD"
```

Trong file `.ino`, thay các dòng `#define BLYNK_*` và hai biến WiFi bằng:

```cpp
#include "secrets.h"            // phải đứng TRƯỚC #include <BlynkSimpleEsp32.h>
char ssid[] = WIFI_SSID_VALUE;
char pass[] = WIFI_PASS_VALUE;
```

Thêm vào `.gitignore`:

```
SmartMediBox/secrets.h
```

Repo chỉ nên chứa `secrets.example.h` (bản mẫu, không có giá trị thật). Nếu lỡ commit token, hãy **tạo lại token** trong Blynk (Device Info → Regenerate) và đổi mật khẩu WiFi.

## Cấu hình Blynk

1. Tạo template **SmartMediBox** (ESP32, WiFi), tạo thiết bị từ template, lấy `TEMPLATE_ID` và `AUTH_TOKEN`.
2. Tạo các datastream (Virtual Pin):

| Pin | Tên | Kiểu | Ghi chú |
|---|---|---|---|
| V0 | Temperature | Double | °C |
| V1 | Humidity | Double | % |
| V2 | CurrentTime | String | Giờ thiết bị (HH:MM) |
| V3 / V4 / V5 | Morning / Noon / Evening Status | String | Trạng thái từng ngăn |
| V6 / V7 | Morning Hour / Minute | Integer | 0–23 / 0–59 |
| V8 / V9 | Noon Hour / Minute | Integer | |
| V10 / V11 | Evening Hour / Minute | Integer | |
| V12 / V13 / V14 | Morning / Noon / Evening Lid | Integer 0–1 | Công tắc mở nắp bằng tay |

3. Tạo 3 **Event** (bật *Enable notifications*, gửi push và email tới Device Owner). **Event Code phải khớp đúng với code:**

| Event Code | Khi nào gửi |
|---|---|
| `missed_morning` | Quên thuốc buổi Sáng |
| `missed_noon` | Quên thuốc buổi Trưa |
| `missed_evening` | Quên thuốc buổi Tối |

4. Dựng dashboard: Time/Number để đặt giờ, Label cho trạng thái, Gauge cho nhiệt độ và độ ẩm, Switch cho mở nắp. Bấm **Save and apply** ở template.
5. Trên điện thoại: đăng nhập app Blynk bằng đúng tài khoản chủ thiết bị, cho phép thông báo.

> Gói Blynk Free có giới hạn số datastream và số tin nhắn mỗi tháng (có thể thay đổi theo thời điểm). Code đã giảm số tin gửi (chỉ gửi nhiệt độ, độ ẩm khi thay đổi đáng kể hoặc tối đa mỗi 5 phút), nhưng bạn nên kiểm tra giới hạn thực tế trong Blynk Console.

Giao diện HTML tùy chỉnh (HTML Widget, thử nghiệm): [`blynk/SmartMediBox_dashboard.html`](blynk/SmartMediBox_dashboard.html).

## Cấu hình thời gian và chế độ demo

Tất cả mốc thời gian nằm trong khối **`[2] CAU HINH THOI GIAN`** ở đầu file `.ino`:

| Hằng số | Mặc định | Ý nghĩa |
|---|---|---|
| `DEFAULT_HOUR` / `DEFAULT_MINUTE` | 7:00, 12:00, 19:00 | Giờ uống thuốc mặc định |
| `DEMO_MODE` | `0` | `1`: nhắc lại sau 15 s, báo quên sau 30 s |
| `REMIND_MS` / `ALERT_MS` | 5 phút / 30 phút | Mốc nhắc lại và báo quên |
| `IR_DEBOUNCE_MS` | 700 ms | Lọc nhiễu IR |
| `CLOSE_DELAY_MS` | 1500 ms | Đợi sau khi lấy thuốc rồi mới đóng nắp |
| `MANUAL_OPEN_TIMEOUT_MS` | 60 s | Tự đóng nắp mở bằng tay |
| `SENSOR_SEND_MAX_MS` | 5 phút | Gửi nhiệt độ, độ ẩm tối đa mỗi 5 phút |
| `CLOCK_SEND_MS` | 15 phút | Gửi giờ thiết bị lên app |

Góc đóng/mở từng servo nằm ở mục `[3]` (mảng `CLOSE_ANGLE[]`, `OPEN_ANGLE[]`). Chỉnh cho khớp cơ khí từng nắp.

## Cấu trúc repo

```
SmartMediBox/
├── README.md
├── .gitignore
├── SmartMediBox/
│   ├── SmartMediBox.ino        # firmware
│   └── secrets.example.h       # bản mẫu (file secrets.h thật không commit)
├── blynk/
│   └── SmartMediBox_dashboard.html   # giao diện HTML Widget (tùy chọn)
└── docs/
    ├── SmartMediBox_So_do.drawio     # sơ đồ khối, sơ đồ nguyên lý, lưu đồ
    └── images/                       # ảnh mô hình, ảnh chụp app
```

## Kiểm thử và demo

- **Demo nhanh:** đặt `DEMO_MODE 1`, đặt giờ một ngăn là giờ hiện tại và để thuốc trong ngăn rồi không lấy: sau 15 s nhắc lại, sau 30 s báo quên và điện thoại nhận thông báo.
- **Kiểm tra servo riêng từng cái:** nạp sketch test chỉ điều khiển một servo, để loại trừ lỗi nguồn trước khi chạy chương trình chính.
- **Mất mạng:** các chức năng tại chỗ (báo giờ, mở nắp, nhắc lại, LCD) vẫn chạy; chỉ thông báo lên điện thoại là không gửi được.
- **Mạng yếu hoặc bị chặn (ví dụ ở trường):** dùng điểm phát WiFi từ điện thoại (băng tần 2.4 GHz, tên mạng không dấu).
- Các tình huống đã mô phỏng trên máy tính: lấy thuốc đúng giờ, ngăn trống, quên thuốc rồi lấy muộn, đổi giờ trong ngày, mất WiFi.

Link mô phỏng: https://wokwi.com/projects/478909463441983489
Link video demo: https://www.youtube.com/watch?v=POlntURg4F4
Slide báo cáo: https://www.canva.com/design/DAHaWXNFT4I/IflX71q-aVDcYSr0mMzHmg/edit

## Hạn chế và hướng phát triển

**Hạn chế**
- Mới ở mức mô hình, chưa kiểm nghiệm lâu dài với thuốc thật.
- Cảm biến hồng ngoại chỉ nhận biết có hay không có vật, không phân biệt loại thuốc hoặc số viên; chùm tia hẹp nên cần bố trí cơ khí phù hợp (ví dụ cốc đựng liều, nền đen mờ).
- DHT11 có sai số khoảng ±2 °C và ±5 % độ ẩm.
- Thông báo lên điện thoại cần Internet; nếu mất mạng đúng lúc quên thuốc, thông báo không được gửi bù.
- Trạng thái trong ngày (đã uống hay chưa) không được lưu khi mất điện.
- Cố định 3 ngăn, chưa mở rộng theo ngày trong tuần.

**Hướng phát triển**
- Pin dự phòng, lưu trạng thái trong ngày.
- Gửi bù thông báo khi có mạng lại, cảnh báo ngăn trống hoặc thuốc sắp hết.
- Loa nhắc bằng giọng nói (DFPlayer Mini).
- Lịch sử uống thuốc trên app.
- Cảm biến cân (load cell + HX711) để đếm số viên; nâng cấp DHT22 hoặc SHT31.
- Nhiều ngày, nhiều người dùng.

## Thành viên

| Họ và tên | MSSV | Phụ trách |
|---|---|---|
| Lý Thu Thảo | HE204379 | Blynk |
| Lê Bá Dũng | HE204561 | Wokwi |
| Nguyễn Tấn Dũng | HE204432 | Linh kiện |
| Đoàn Anh Tuấn | SE200905 | Lắp mạch |

## License

_[Chọn license nếu muốn công khai, ví dụ MIT, hoặc ghi "Dự án học tập"]_
