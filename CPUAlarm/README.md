# Hands-On: CPU Alarm → Email Alert via SNS

## Mục tiêu
Gửi email cảnh báo khi EC2 CPU vượt quá **80%** trong **5 phút liên tiếp**.

---

## Tổng quan kiến trúc

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐     ┌─────────────┐
│  EC2        │     │  CloudWatch  │     │   SNS Topic   │     │  Email      │
│  Instance   │────▶│  Alarm       │────▶│               │────▶│  Alert      │
│  (CPU>80%)  │     │  (5min eval) │     │  cpu-alarm    │     │  (Subscriber)│
└─────────────┘     └──────────────┘     └───────────────┘     └─────────────┘
```

---

## Bước 1: Tạo SNS Topic & Subscription

### 1.1 Tạo SNS Topic

- Dịch vụ: **SNS** → **Topics** → **Create topic**
- **Type:** Standard
- **Name:** `CPUAlarmTopic`
- **Display name:** `CPU Alarm`
- **Tags (tùy chọn):** Environment = Production
- Nhấn **Create topic**

### 1.2 Thêm Email Subscription

- Vào topic vừa tạo → tab **Subscriptions** → **Create subscription**
- **Protocol:** Email
- **Endpoint:** `your-email@example.com`
- Nhấn **Create subscription**

### 1.3 Xác nhận Subscription qua email

- Mở email từ **AWS Notifications** (`no-reply@sns.amazonaws.com`)
- Nhấn vào link **Confirm subscription**
- Kiểm tra trạng thái: `Confirmed`

> 📸 **[SCREENSHOT 1]: SNS Topic đã tạo thành công**

---

## Bước 2: Tạo CloudWatch Alarm

### 2.1 Truy cập CloudWatch

- Dịch vụ: **CloudWatch** → **Alarms** → **Create alarm**

### 2.2 Chọn Metric

- Nhấn **Select metric** → **EC2** → **Per-Instance Metrics**
- Tìm metric: `CPUUtilization` cho instance cần giám sát
- Instance ID: `<YOUR_EC2_INSTANCE_ID>`
- Nhấn **Select metric**

### 2.3 Metric Details

| Tham số              | Giá trị                   |
|----------------------|---------------------------|
| Statistic            | Average                   |
| Period               | 5 minutes                 |
| Threshold type       | Static                    |
| CPU >                | 80                        |
| Datapoints to alarm  | 1 out of 1                |

### 2.4 Cấu hình Alarm

- **Alarm name:** `CPUAlarm-HighCPU`
- **Description:** `Alert when CPU > 80% for 5 minutes`
- **AWS managed key:** No

> 📸 **[SCREENSHOT 2]: CloudWatch Metric đã chọn**

---

## Bước 3: Cấu hình SNS Notification

### 3.1 Thêm Action cho Alarm State

- Phần **Notification** → nhấn **Add notification**
- **Alarm state:** ALARM
- **Select an SNS topic:** Chọn `CPUAlarmTopic`
- Nhấn **Next**

### 3.2 (Tùy chọn) Thêm Recovery Alert

- Thêm notification khác:
  - **Alarm state:** OK
  - **SNS topic:** `CPUAlarmTopic`
- Ý nghĩa: Gửi email thông báo khi CPU trở về bình thường

> 📸 **[SCREENSHOT 3]: Alarm Actions đã cấu hình**

---

## Bước 4: Tạo Alarm

- Kiểm tra lại các thông số → nhấn **Create alarm**

> 📸 **[SCREENSHOT 4]: CloudWatch Alarm đã tạo thành công**

---

## Bước 5: Kiểm tra Alarm (Tạo CPU spike để trigger)

### Cách 1: Stress test trên EC2 (Linux)

```bash
sudo yum install -y stress   # Amazon Linux / CentOS
# hoặc
sudo apt-get install -y stress   # Ubuntu/Debian

# Tạo CPU spike
stress --cpu 4 --timeout 600s
```

### Cách 2: Tạo CPU spike nhanh (busy loop)

```bash
# Chạy 1 vòng lặp vô tận để tăng CPU
while true; do echo "scale=5000; a(1)*100000" | bc -l > /dev/null; done
```

### Theo dõi trạng thái Alarm

- CloudWatch → **Alarms** → Xem trạng thái chuyển từ `OK` → `ALARM`

> 📸 **[SCREENSHOT 5]: Alarm chuyển sang ALARM state**

---

## Bước 6: Kiểm tra Email Alert

- Kiểm tra hộp thư email đã đăng ký
- Email sẽ có subject: `Amazon SNS Notification` hoặc `ALARM: "CPUAlarm-HighCPU"`

> 📸 **[SCREENSHOT 6]: Email alert nhận được**

---

## Cleanup (Xóa tài nguyên sau khi hoàn thành)

```bash
# 1. Xóa CloudWatch Alarm
aws cloudwatch delete-alarms --alarm-names CPUAlarm-HighCPU

# 2. Xóa SNS Topic (sẽ xóa toàn bộ subscriptions)
aws sns delete-topic --topic-arn arn:aws:sns:REGION:ACCOUNT_ID:CPUAlarmTopic
```

---

## Tóm tắt kiến thức

| Khái niệm       | Giải thích                                                    |
|-----------------|---------------------------------------------------------------|
| **SNS Topic**   | Kênh trung gân để gửi notification đến nhiều endpoint       |
| **SNS Subscription** | Đăng ký endpoint (email/SMS/Lambda...) để nhận thông báo |
| **CloudWatch Alarm** | Giám sát metric và kích hoạt action khi ngưỡng bị vượt |
| **Evaluation Period** | Số chu kỳ (period) metric phải vi phạm ngưỡng trước khi alarm |
| **Datapoints to Alarm** | 1 out of 1 = chỉ cần 1 lần vi phạm là kích hoạt alarm |

---

## Screenshot Checklist

| # | Mô tả Screenshot | Trạng thái |
|---|-----------------|------------|
| 1 | SNS Topic tạo thành công | [CHỖ CHỤP] |
| 2 | CloudWatch Metric đã chọn | [CHỤP ẢNH] |
| 3 | Alarm Actions cấu hình SNS | [CHỤP ẢNH] |
| 4 | CloudWatch Alarm tạo thành công | [CHỤP ẢNH] |
| 5 | Alarm chuyển sang ALARM state | [CHỤP ẢNH] |
| 6 | Email alert nhận được | [CHỤP ẢNH] |

---

*Hands-On: CPU Alarm → Email Alert via SNS — W9 Monitoring*
