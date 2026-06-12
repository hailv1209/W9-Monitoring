# CPU Alarm — Evidence Checklist

## Mục đích
File này ghi lại tiến độ hoàn thành bài Hands-On. Mỗi mục cần chụp screenshot khi hoàn thành.

---

## Phần 1: SNS Topic & Subscription

### [ ] 1.1 Tạo SNS Topic thành công

| Trường | Giá trị |
|--------|---------|
| Type | Standard |
| Topic name | `CPUAlarmTopic` |
| Display name | `CPU Alarm` |

**Screenshots cần chụp:**
- [SCREENSHOT 1a] Trang tạo SNS Topic — xác nhận các trường đã điền đúng
- [SCREENSHOT 1b] Trang SNS Topic sau khi tạo thành công (ARN hiển thị)

---

### [ ] 1.2 Tạo Email Subscription

| Trường | Giá trị |
|--------|---------|
| Protocol | Email |
| Endpoint | `<your-email@example.com>` |

**Screenshots cần chụp:**
- [SCREENSHOT 2a] Trang tạo Subscription với Email protocol
- [SCREENSHOT 2b] Hộp thoại xác nhận subscription đang chờ (Pending confirmation)

---

### [ ] 1.3 Xác nhận Subscription qua email

**Screenshots cần chụp:**
- [SCREENSHOT 3a] Email nhận được từ AWS Notifications
- [SCREENSHOT 3b] Trang Subscription sau khi xác nhận → Status: `Confirmed`

---

## Phần 2: CloudWatch Alarm

### [ ] 2.1 Chọn EC2 Metric (CPUUtilization)

| Trường | Giá trị |
|--------|---------|
| Service | EC2 |
| Metric | Per-Instance Metrics → CPUUtilization |
| Instance | `<YOUR_EC2_INSTANCE_ID>` |
| Statistic | Average |
| Period | 5 minutes (300s) |

**Screenshots cần chụp:**
- [SCREENSHOT 4a] Giao diện chọn metric EC2 → CPUUtilization
- [SCREENSHOT 4b] Trang metric details với các thông số đã cấu hình

---

### [ ] 2.2 Cấu hình Alarm Conditions

| Trường | Giá trị |
|--------|---------|
| Condition | Greater than |
| Threshold | 80 % |
| Datapoints to alarm | 1 out of 1 |

**Screenshots cần chụp:**
- [SCREENSHOT 5a] Cấu hình Threshold và Evaluation (Greater than 80%)
- [SCREENSHOT 5b] Review Alarm Configuration

---

### [ ] 2.3 Cấu hình SNS Notification Action

| Trường | Giá trị |
|--------|---------|
| Alarm state | ALARM |
| Action | Select SNS Topic → `CPUAlarmTopic` |
| Recovery notification | OK state → `CPUAlarmTopic` (tùy chọn) |

**Screenshots cần chụp:**
- [SCREENSHOT 6a] Phần Actions — Alarm state notification đã chọn SNS Topic
- [SCREENSHOT 6b] Phần Actions — OK state notification (recovery alert)

---

### [ ] 2.4 Alarm tạo thành công

**Screenshots cần chụp:**
- [SCREENSHOT 7] Trang CloudWatch Alarms — alarm `CPUAlarm-HighCPU` hiển thị với trạng thái `OK`

---

## Phần 3: Kiểm tra hoạt động (Trigger Test)

### [ ] 3.1 Tạo CPU Spike để kích hoạt Alarm

**Cách thực hiện:**
```bash
# Linux / Amazon EC2
sudo yum install -y stress   # Amazon Linux / CentOS / RHEL
sudo apt-get install -y stress   # Ubuntu / Debian

# Chạy stress test để tăng CPU
stress --cpu 4 --timeout 600s
```

**Screenshots cần chụp:**
- [SCREENSHOT 8a] SSH vào EC2, chạy lệnh `stress --cpu 4`
- [SCREENSHOT 8b] CloudWatch Dashboard — CPU Utilization metric đang tăng cao

---

### [ ] 3.2 Alarm chuyển sang ALARM state

**Screenshots cần chụp:**
- [SCREENSHOT 9a] CloudWatch Alarms — trạng thái chuyển từ `OK` → `ALARM`
- [SCREENSHOT 9b] Chi tiết Alarm — biểu đồ CPU với ngưỡng 80% và vùng ALARM

---

### [ ] 3.3 Nhận Email Alert

**Screenshots cần chụp:**
- [SCREENSHOT 10a] Email nhận được từ AWS Notifications (Subject: ALARM)
- [SCREENSHOT 10b] Nội dung email — thông tin chi tiết về alarm

---

## Phần 4: Cleanup (Xóa tài nguyên)

### [ ] 4.1 Xóa CloudWatch Alarm

```bash
aws cloudwatch delete-alarms \
  --alarm-names CPUAlarm-HighCPU \
  --region us-east-1
```

**Screenshot cần chụp:**
- [SCREENSHOT 11] Kết quả xóa alarm thành công (output trống / success)

---

### [ ] 4.2 Xóa SNS Topic

```bash
aws sns delete-topic \
  --topic-arn arn:aws:sns:us-east-1:ACCOUNT_ID:CPUAlarmTopic \
  --region us-east-1
```

**Screenshot cần chụp:**
- [SCREENSHOT 12] Kết quả xóa SNS Topic thành công

---

## Tổng kết số lượng Screenshots

| Phần | Số lượng Screenshots |
|------|---------------------|
| SNS Topic & Subscription | 4 |
| CloudWatch Alarm | 4 |
| Trigger Test | 4 |
| Cleanup | 2 |
| **Tổng cộng** | **14 screenshots** |

---

## Trạng thái hoàn thành

| # | Screenshot | Trạng thái | Ghi chú |
|---|-----------|-----------|---------|
| 1a | SNS Topic creation page | [ ] | |
| 1b | SNS Topic created (ARN visible) | [ ] | |
| 2a | Email Subscription creation | [ ] | |
| 2b | Subscription pending confirmation | [ ] | |
| 3a | Confirmation email received | [ ] | |
| 3b | Subscription status: Confirmed | [ ] | |
| 4a | EC2 CPUUtilization metric selection | [ ] | |
| 4b | Metric details configuration | [ ] | |
| 5a | Threshold configuration (>80%) | [ ] | |
| 5b | Alarm review page | [ ] | |
| 6a | Alarm state SNS action | [ ] | |
| 6b | OK state SNS action (recovery) | [ ] | |
| 7  | Alarm created — status OK | [ ] | |
| 8a | EC2 stress test running | [ ] | |
| 8b | CPU metric graph rising | [ ] | |
| 9a | Alarm state: ALARM | [ ] | |
| 9b | Alarm detail with graph | [ ] | |
| 10a | Email alert received (subject) | [ ] | |
| 10b | Email alert content | [ ] | |
| 11 | Delete alarm success | [ ] | |
| 12 | Delete SNS Topic success | [ ] | |

---

*Evidence Checklist — CPU Alarm → Email Alert via SNS — W9 Monitoring*
