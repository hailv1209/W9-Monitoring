# CPU Alarm — Evidence Checklist

## Mục đích
File này ghi lại tiến độ hoàn thành bài Hands-On. Mỗi mục cần chụp screenshot khi hoàn thành.

---

## Thông tin tài nguyên đã tạo

| Tài nguyên | Giá trị |
|-------------|---------|
| **EC2 Instance ID** | `i-06f3580ee2b478cf6` |
| **SNS Topic ARN** | `arn:aws:sns:us-east-1:593777010472:CPUAlarmTopic` |
| **CloudWatch Alarm** | `CPUAlarm-HighCPU` |
| **Email** | `haileab542@gmail.com` |

---

## Phần 1: SNS Topic & Subscription

### [ ] 1.1 Tạo SNS Topic thành công

**AWS Console Link:**
👉 https://console.aws.amazon.com/sns/v3/home#/topics/arn:aws:sns:us-east-1:593777010472:CPUAlarmTopic

**Screenshots cần chụp:**
- [SCREENSHOT 1a] Trang SNS Topic — ARN hiển thị đầy đủ

---

### [ ] 1.2 Tạo Email Subscription

**AWS Console Link:**
👉 https://console.aws.amazon.com/sns/v3/home#/subscriptions/arn:aws:sns:us-east-1:593777010472:CPUAlarmTopic

**Screenshots cần chụp:**
- [SCREENSHOT 2a] Danh sách Subscriptions — Email endpoint `haileab542@gmail.com` với Status: `Pending confirmation`

---

### [ ] 1.3 Xác nhận Subscription qua email

> ⚠️ **Kiểm tra email `haileab542@gmail.com`** — AWS đã gửi email xác nhận từ `no-reply@sns.amazonaws.com`. Nhấn **Confirm subscription** trong email.

**Screenshots cần chụp:**
- [SCREENSHOT 3a] Email nhận được từ AWS Notifications (Subject: "AWS Notification - Subscription Confirmation")
- [SCREENSHOT 3b] Trang Subscription sau khi xác nhận → Status: `Confirmed`

---

## Phần 2: CloudWatch Alarm

### [ ] 2.1 Alarm đã tạo thành công

**AWS Console Link:**
👉 https://console.aws.amazon.com/cloudwatch/home#alarmsV2:alarm/CPUAlarm-HighCPU

**Cấu hình đã tạo:**

| Trường | Giá trị |
|--------|---------|
| Alarm name | `CPUAlarm-HighCPU` |
| Metric namespace | `AWS/EC2` |
| Metric name | `CPUUtilization` |
| Instance ID | `i-06f3580ee2b478cf6` |
| Statistic | Average |
| Period | 300 seconds (5 minutes) |
| Threshold | 80 % |
| Condition | Greater than |
| Datapoints to alarm | 1 out of 1 |
| Alarm action | `CPUAlarmTopic` (SNS) |
| OK action | `CPUAlarmTopic` (SNS) |

**Screenshots cần chụp:**
- [SCREENSHOT 4a] CloudWatch Alarm detail — xem đầy đủ cấu hình alarm
- [SCREENSHOT 4b] Phần Actions — Alarm state và OK state đều trỏ đến `CPUAlarmTopic`

---

## Phần 3: Kiểm tra hoạt động (Trigger Test)

### [ ] 3.1 Tạo CPU Spike để kích hoạt Alarm

**Cách thực hiện:** SSH vào EC2 và chạy stress test:

```bash
# Kết nối EC2
ssh -i your-key.pem ec2-user@<EC2-PUBLIC-IP>

# Cài stress (Amazon Linux 2023)
sudo dnf install -y stress   # hoặc dùng yum

# Chạy stress test để tăng CPU
sudo stress --cpu 4 --timeout 600s
```

**Lấy EC2 Public IP:**
```bash
aws ec2 describe-instances --instance-ids i-06f3580ee2b478cf6 --query 'Reservations[0].Instances[0].PublicIpAddress' --output text
```

**Screenshots cần chụp:**
- [SCREENSHOT 5a] SSH terminal — lệnh `stress --cpu 4` đang chạy
- [SCREENSHOT 5b] CloudWatch Dashboard — CPU Utilization metric đang tăng cao vượt ngưỡng 80%

---

### [ ] 3.2 Alarm chuyển sang ALARM state

**AWS Console Link:**
👉 https://console.aws.amazon.com/cloudwatch/home#alarmsV2:alarm/CPUAlarm-HighCPU

> ⏱️ **Lưu ý:** Alarm cần 5 phút data vượt ngưỡng để chuyển sang ALARM. Sau khi chạy stress ~6-7 phút, alarm sẽ trigger.

**Screenshots cần chụp:**
- [SCREENSHOT 6a] CloudWatch Alarms list — trạng thái chuyển từ `OK` hoặc `INSUFFICIENT_DATA` → `ALARM`
- [SCREENSHOT 6b] Alarm detail — biểu đồ CPU với đường ngưỡng 80% và vùng ALARM highlighted

---

### [ ] 3.3 Nhận Email Alert

> ⚠️ **Kiểm tra email `haileab542@gmail.com`** — Email sẽ có Subject: "Amazon SNS Notification" hoặc "ALARM: CPUAlarm-HighCPU"

**Screenshots cần chụp:**
- [SCREENSHOT 7a] Email nhận được từ AWS Notifications (Subject: ALARM)
- [SCREENSHOT 7b] Nội dung email — thông tin chi tiết về alarm (Instance ID, CPU %, Threshold)

---

## Phần 4: Cleanup (Xóa tài nguyên)

### [ ] 4.1 Xóa CloudWatch Alarm

**AWS Console Link:**
👉 https://console.aws.amazon.com/cloudwatch/home#alarmsV2:

```bash
aws cloudwatch delete-alarms --alarm-names CPUAlarm-HighCPU
```

**Screenshot cần chụp:**
- [SCREENSHOT 8] Alarm đã bị xóa (không còn trong danh sách Alarms)

---

### [ ] 4.2 Xóa EC2 Instance

**AWS Console Link:**
👉 https://console.aws.amazon.com/ec2/home#Instances:v=3;search=i-06f3580ee2b478cf6

```bash
aws ec2 terminate-instances --instance-ids i-06f3580ee2b478cf6
```

**Screenshot cần chụp:**
- [SCREENSHOT 9] EC2 instance đã terminated (trạng thái: terminated)

---

### [ ] 4.3 Xóa SNS Topic

**AWS Console Link:**
👉 https://console.aws.amazon.com/sns/v3/home#/topics

```bash
aws sns delete-topic --topic-arn arn:aws:sns:us-east-1:593777010472:CPUAlarmTopic
```

**Screenshot cần chụp:**
- [SCREENSHOT 10] SNS Topic đã bị xóa (không còn trong danh sách Topics)

---

## AWS Console Quick Links

| Dịch vụ | Link |
|---------|------|
| **SNS Topics** | https://console.aws.amazon.com/sns/v3/home#/topics |
| **SNS Subscriptions** | https://console.aws.amazon.com/sns/v3/home#/subscriptions |
| **CloudWatch Alarms** | https://console.aws.amazon.com/cloudwatch/home#alarmsV2: |
| **CloudWatch Metrics** | https://console.aws.amazon.com/cloudwatch/home#metricsV2: |
| **EC2 Instances** | https://console.aws.amazon.com/ec2/home#Instances:v=3; |
| **EC2 Instance Details** | https://console.aws.amazon.com/ec2/home#Instances:instanceId=i-06f3580ee2b478cf6 |

---

## Trạng thái hoàn thành

| # | Screenshot | AWS Console Link | Trạng thái |
|---|-----------|-----------------|-----------|
| 1a | SNS Topic — ARN visible | [SNS Topic](https://console.aws.amazon.com/sns/v3/home#/topics/arn:aws:sns:us-east-1:593777010472:CPUAlarmTopic) | [ ] |
| 2a | Email Subscription — Pending | [SNS Subscriptions](https://console.aws.amazon.com/sns/v3/home#/subscriptions) | [ ] |
| 3a | Confirmation email received | Email box | [ ] |
| 3b | Subscription status: Confirmed | [SNS Subscriptions](https://console.aws.amazon.com/sns/v3/home#/subscriptions) | [ ] |
| 4a | Alarm detail page | [CloudWatch Alarm](https://console.aws.amazon.com/cloudwatch/home#alarmsV2:alarm/CPUAlarm-HighCPU) | [ ] |
| 4b | Alarm Actions configured | [CloudWatch Alarm](https://console.aws.amazon.com/cloudwatch/home#alarmsV2:alarm/CPUAlarm-HighCPU) | [ ] |
| 5a | EC2 stress test running | SSH terminal | [ ] |
| 5b | CPU metric rising | [CloudWatch Metrics](https://console.aws.amazon.com/cloudwatch/home#metricsV2:namespace=AWS/EC2) | [ ] |
| 6a | Alarm state: ALARM | [CloudWatch Alarms](https://console.aws.amazon.com/cloudwatch/home#alarmsV2:alarm/CPUAlarm-HighCPU) | [ ] |
| 6b | Alarm graph with ALARM zone | [CloudWatch Alarm](https://console.aws.amazon.com/cloudwatch/home#alarmsV2:alarm/CPUAlarm-HighCPU) | [ ] |
| 7a | Email alert received | Email box | [ ] |
| 7b | Email alert content | Email box | [ ] |
| 8  | Alarm deleted | [CloudWatch Alarms](https://console.aws.amazon.com/cloudwatch/home#alarmsV2:) | [ ] |
| 9  | EC2 terminated | [EC2 Instances](https://console.aws.amazon.com/ec2/home#Instances:v=3;) | [ ] |
| 10 | SNS Topic deleted | [SNS Topics](https://console.aws.amazon.com/sns/v3/home#/topics) | [ ] |

---

## AWS CLI Commands đã chạy

```bash
# 1. Tạo EC2
aws ec2 run-instances --image-id ami-0152204c1a187337c --instance-type t2.micro

# 2. Tạo SNS Topic
aws sns create-topic --name CPUAlarmTopic

# 3. Thêm Email Subscription
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:593777010472:CPUAlarmTopic \
  --protocol email --notification-endpoint haileab542@gmail.com

# 4. Tạo CloudWatch Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name CPUAlarm-HighCPU \
  --alarm-actions arn:aws:sns:us-east-1:593777010472:CPUAlarmTopic \
  --ok-actions arn:aws:sns:us-east-1:593777010472:CPUAlarmTopic \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 300 --evaluation-periods 1 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-06f3580ee2b478cf6
```

---

*Evidence Checklist — CPU Alarm → Email Alert via SNS — W9 Monitoring*
