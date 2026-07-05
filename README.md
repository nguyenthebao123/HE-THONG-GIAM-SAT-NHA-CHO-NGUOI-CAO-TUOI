# Hệ thống Nhà ở Thông minh (Smart Home) trên RTOS – ESP32

Đồ án môn học **RTOS** – Khoa Điện – Điện tử, Bộ môn Công nghệ Kỹ thuật Máy tính – Truyền thông (CCE), Trường Đại học Sư phạm Kỹ thuật TP.HCM (HCMUTE).

**Sinh viên thực hiện**
| MSSV | Họ và tên |
|---|---|
| | Huỳnh Võ Gia Bảo |
| | Nguyễn Thế Bảo |

**GVHD:** PGS.TS. Phan Văn Ca

---

## 1. Giới thiệu

Hệ thống nhà ở thông minh tích hợp 3 chức năng an toàn – tiện nghi trên nền một vi điều khiển **ESP32**, sử dụng **FreeRTOS** để chạy song song và điều phối ưu tiên giữa các nhiệm vụ:

- **Cảnh báo khí gas** – phát hiện rò rỉ khí gas bằng cảm biến MQ‑2, cảnh báo bằng buzzer/LED, tự động khoá toàn bộ hệ thống ở trạng thái an toàn khi có sự cố.
- **Cửa tự động** – mở/đóng cửa bằng động cơ servo dựa trên cảm biến siêu âm HC‑SR04 khi phát hiện người tiếp cận.
- **Nhận diện người bị ngất** – kết hợp cảm biến PIR (chuyển động) và RD‑03 (hiện diện) để phát hiện tình trạng bất động kéo dài, gửi cảnh báo qua Telegram Bot.
- **Điều khiển từ xa qua Telegram** – cho phép mở/khoá cửa bằng lệnh `/open`, `/close` từ điện thoại.

## 2. Kiến trúc hệ thống

Hệ thống tổ chức theo mô hình **Input – Processing – Output**:

```
[MQ2] [HC-SR04] [PIR] [RD-03]        (Input - cảm biến)
        │
        ▼
     [ ESP32 ]  ← xử lý, so sánh ngưỡng, điều phối task (FreeRTOS)
        │
        ▼
[Buzzer] [LED] [Servo] [Telegram Bot]   (Output - cảnh báo & chấp hành)
```

### Sơ đồ khối phần cứng

```
Buzzer ◄──┐        MQ2 ◄──┐      ┌──► RD03
Servo  ◄──┼── ESP32 ──────┴──────┤
LED    ◄──┘        │             └──► PIR
                    └──► SR04
```

## 3. Các task RTOS

| Task | Chức năng | Priority | Period |
|---|---|---|---|
| `gasTask` | Cảnh báo khí gas (ngưỡng: `gas > 5`) | **3 (cao nhất)** | 100 ms |
| `fallTask` | Cảnh báo ngất xỉu (đứng im > 10s / 15s) | 2 (trung bình) | 200 ms |
| `doorTask` | Cửa tự động (ngưỡng: `distance < 50`) | 1 (thấp nhất) | 100 ms |
| `telegramTask` | Nhận lệnh mở/đóng cửa qua Telegram Bot | 1 (thấp nhất) | 2000 ms |

**Kỹ thuật RTOS sử dụng:**
- **Mutex** (`xSemaphoreTake` / `xSemaphoreGive` với `portMAX_DELAY`) để bảo vệ tài nguyên dùng chung giữa các task, có priority inheritance để tránh priority inversion.
- **Suspend / Resume task** (`vTaskSuspend` / `vTaskResume`) để tạm dừng các task ưu tiên thấp khi phát hiện khí gas (chế độ khẩn cấp), và khôi phục khi môi trường an toàn.
- **Task ưu tiên phân tầng** đảm bảo tình huống nguy hiểm (khí gas) luôn được xử lý trước.

## 4. Phần cứng sử dụng

| STT | Linh kiện | Số lượng | Đơn giá | Thành tiền |
|---|---|---|---|---|
| 1 | ESP32 Doit DevKit V1 | 1 | 115.000đ | 115.000đ |
| 2 | Cảm biến khí gas MQ‑2 | 1 | 21.000đ | 21.000đ |
| 3 | Cảm biến siêu âm HC‑SR04 | 1 | 21.000đ | 21.000đ |
| 4 | Cảm biến hiện diện RD‑03 (Ai‑Thinker) | 1 | 70.000đ | 70.000đ |
| 5 | Cảm biến PIR | 1 | 18.000đ | 18.000đ |
| 6 | Servo (mở/đóng cửa) | 1 | – | – |
| 7 | Buzzer | 1 | – | – |
| 8 | LED cảnh báo | 1 | – | – |

*(Bảng đầy đủ xem trong tài liệu thiết kế – `docs/RTOS_Final.pdf`.)*

## 5. Cấu trúc thư mục (đề xuất)

```
.
├── README.md
├── docs/
│   └── RTOS_Final.pdf        # Technical Design Document đầy đủ
├── firmware/
│   └── smart_home.ino        # Source code ESP32 (FreeRTOS)
└── images/
    ├── wiring_diagram.png    # Sơ đồ nối dây (Fritzing)
    ├── flowchart.png         # Lưu đồ giải thuật
    └── final_product/        # Ảnh sản phẩm hoàn thiện
```

> Repo hiện tại đang được khởi tạo — code `.ino` đầy đủ và hình ảnh sẽ được bổ sung.

## 6. Hướng dẫn build & nạp code

1. Cài **Arduino IDE** (hoặc PlatformIO) và board package **ESP32**.
2. Cài các thư viện: `WiFi`, `HTTPClient`, `WiFiClientSecure`, `ESP32Servo` (hoặc `Servo`).
3. Cấu hình thông tin Wi‑Fi và `BOT_TOKEN` Telegram trong file cấu hình (`config.h` hoặc đầu file `.ino`).
4. Kết nối phần cứng theo sơ đồ trong `images/wiring_diagram.png`.
5. Build và nạp firmware vào ESP32 qua cổng USB.

## 7. Kết quả đạt được

- Hệ thống chạy ổn định, các task hoạt động song song không xung đột.
- Cảnh báo khí gas phản hồi nhanh (period 100 ms, priority cao nhất), tự động khoá các chức năng khác khi có sự cố để đảm bảo an toàn.
- Cửa tự động phản hồi chính xác theo khoảng cách đo được từ HC‑SR04.
- Phát hiện và cảnh báo tình trạng bất động (nghi ngất) sau 10s/15s không chuyển động, gửi thông báo khẩn qua Telegram.

## 8. Tài liệu tham khảo

1. A. A. Galadima, "Arduino as a learning tool," *ICECCO*, 2014.
2. V. Oza and P. Mehta, "Arduino Robotic Hand: Survey Paper," *ICSCET*, 2018.

---

*Đồ án phục vụ mục đích học tập – Bộ môn Công nghệ Kỹ thuật Máy tính – Truyền thông, HCMUTE.*
