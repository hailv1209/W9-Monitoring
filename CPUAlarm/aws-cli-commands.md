# AWS CLI Commands — CPU Alarm via SNS

## Môi trường

```bash
# Set biến môi trường (thay thế các giá trị phù hợp)
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export EC2_INSTANCE_ID="i-xxxxxxxxxxxxxxxxx"
export EMAIL="your-email@example.com"
export SNS_TOPIC_ARN="arn:aws:sns:${AWS_REGION}:${AWS_ACCOUNT_ID}:CPUAlarmTopic"
```

---

## Bước 1: Tạo SNS Topic

```bash
# Tạo SNS Topic
aws sns create-topic \
  --name CPUAlarmTopic \
  --region $AWS_REGION
```

**Output mẫu:**
```json
{
  "TopicArn": "arn:aws:sns:us-east-1:123456789012:CPUAlarmTopic"
}
```

---

## Bước 2: Thêm Email Subscription

```bash
# Tạo Email subscription
aws sns subscribe \
  --topic-arn arn:aws:sns:$AWS_REGION:$AWS_ACCOUNT_ID:CPUAlarmTopic \
  --protocol email \
  --notification-endpoint $EMAIL \
  --region $AWS_REGION
```

**Output mẫu:**
```json
{
  "SubscriptionArn": "arn:aws:sns:us-east-1:123456789012:CPUAlarmTopic:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "TopicArn": "arn:aws:sns:us-east-1:123456789012:CPUAlarmTopic"
}
```

> ⚠️ **Lưu ý:** Sau khi subscribe, AWS sẽ gửi email xác nhận. Vào hộp thư xác nhận bằng cách nhấn **Confirm subscription** trong email.

---

## Bước 3: Xác nhận Subscription (nếu dùng Lambda)

```bash
# Nếu cần xác nhận subscription qua CLI (thường dùng cho Lambda)
aws sns confirm-subscription \
  --topic-arn arn:aws:sns:$AWS_REGION:$AWS_ACCOUNT_ID:CPUAlarmTopic \
  --token <TOKEN_FROM_EMAIL> \
  --region $AWS_REGION
```

---

## Bước 4: Tạo CloudWatch Alarm

```bash
# Tạo CloudWatch Alarm giám sát CPU > 80%
aws cloudwatch put-metric-alarm \
  --alarm-name CPUAlarm-HighCPU \
  --alarm-description "Alert when EC2 CPU > 80% for 5 consecutive minutes" \
  --actions-enabled \
  --alarm-actions arn:aws:sns:$AWS_REGION:$AWS_ACCOUNT_ID:CPUAlarmTopic \
  --ok-actions arn:aws:sns:$AWS_REGION:$AWS_ACCOUNT_ID:CPUAlarmTopic \
  --insufficient-data-actions [] \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --datapoints-to-alarm 1 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --dimensions Name=InstanceId,Value=$EC2_INSTANCE_ID \
  --region $AWS_REGION
```

**Tham số quan trọng:**

| Tham số | Giá trị | Ý nghĩa |
|---------|---------|---------|
| `--period` | 300 | 5 phút (300 giây) |
| `--evaluation-periods` | 1 | Chỉ cần 1 lần vi phạm |
| `--threshold` | 80 | Ngưỡng CPU 80% |
| `--comparison-operator` | GreaterThanThreshold | Lớn hơn ngưỡng |
| `--treat-missing-data` | notBreaching | Missing data không trigger alarm |

---

## Bước 5: Kiểm tra trạng thái Alarm

```bash
# Xem danh sách tất cả alarms
aws cloudwatch describe-alarms \
  --alarm-names CPUAlarm-HighCPU \
  --region $AWS_REGION
```

**Output mẫu:**
```json
{
  "MetricAlarms": [
    {
      "AlarmName": "CPUAlarm-HighCPU",
      "AlarmArn": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:CPUAlarm-HighCPU",
      "StateValue": "OK",
      "MetricName": "CPUUtilization",
      "Namespace": "AWS/EC2",
      "Threshold": 80.0
    }
  ]
}
```

---

## Bước 6: Theo dõi Alarm state liên tục

```bash
# Theo dõi trạng thái alarm (lặp lại mỗi 10 giây)
watch -n 10 "aws cloudwatch describe-alarms --alarm-names CPUAlarm-HighCPU --query 'MetricAlarms[0].StateValue' --output text --region $AWS_REGION"
```

---

## Bước 7: Bật/Tắt Alarm Actions

```bash
# Tạm thời tắt alarm (không xóa)
aws cloudwatch disable-alarm-actions \
  --alarm-names CPUAlarm-HighCPU \
  --region $AWS_REGION

# Bật lại alarm
aws cloudwatch enable-alarm-actions \
  --alarm-names CPUAlarm-HighCPU \
  --region $AWS_REGION
```

---

## Bước 8: Test SNS Topic (Gửi notification thủ công)

```bash
# Gửi test notification đến SNS topic
aws sns publish \
  --topic-arn arn:aws:sns:$AWS_REGION:$AWS_ACCOUNT_ID:CPUAlarmTopic \
  --subject "TEST: CPU Alarm Notification" \
  --message "This is a test notification from AWS SNS. Your CPU Alarm is working correctly." \
  --region $AWS_REGION
```

---

## Bước 9: Xóa tài nguyên (Cleanup)

```bash
# Xóa CloudWatch Alarm
aws cloudwatch delete-alarms \
  --alarm-names CPUAlarm-HighCPU \
  --region $AWS_REGION

# Xóa SNS Topic (tự động xóa tất cả subscriptions)
aws sns delete-topic \
  --topic-arn arn:aws:sns:$AWS_REGION:$AWS_ACCOUNT_ID:CPUAlarmTopic \
  --region $AWS_REGION
```

---

## Bước 10: Deploy bằng CloudFormation

```bash
# Tạo Stack từ template
aws cloudformation create-stack \
  --stack-name CPUAlarmStack \
  --template-body file://cpu-alarm-template.yaml \
  --parameters \
    ParameterKey=EmailAddress,ParameterValue=$EMAIL \
    ParameterKey=InstanceId,ParameterValue=$EC2_INSTANCE_ID \
    ParameterKey=AlarmThreshold,ParameterValue=80 \
    ParameterKey=PeriodSeconds,ParameterValue=300 \
    ParameterKey=EvaluationPeriods,ParameterValue=1 \
  --capabilities CAPABILITY_IAM \
  --region $AWS_REGION

# Kiểm tra trạng thái Stack
aws cloudformation describe-stacks \
  --stack-name CPUAlarmStack \
  --query 'Stacks[0].StackStatus' \
  --output text \
  --region $AWS_REGION

# Cập nhật Stack (nếu có thay đổi)
aws cloudformation update-stack \
  --stack-name CPUAlarmStack \
  --template-body file://cpu-alarm-template.yaml \
  --parameters \
    ParameterKey=EmailAddress,ParameterValue=$EMAIL \
    ParameterKey=InstanceId,ParameterValue=$EC2_INSTANCE_ID \
    ParameterKey=AlarmThreshold,ParameterValue=80 \
    ParameterKey=PeriodSeconds,ParameterValue=300 \
    ParameterKey=EvaluationPeriods,ParameterValue=1 \
  --capabilities CAPABILITY_IAM \
  --region $AWS_REGION

# Xóa Stack (cleanup cuối cùng)
aws cloudformation delete-stack \
  --stack-name CPUAlarmStack \
  --region $AWS_REGION
```

---

## Lệnh hỗ trợ kiểm tra

```bash
# 1. Xem CPU metric hiện tại của EC2
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --dimensions Name=InstanceId,Value=$EC2_INSTANCE_ID \
  --region $AWS_REGION

# 2. Liệt kê EC2 instances (lấy Instance ID)
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=running \
  --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0],State.Name]' \
  --output table \
  --region $AWS_REGION

# 3. Kiểm tra SNS subscriptions
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:$AWS_REGION:$AWS_ACCOUNT_ID:CPUAlarmTopic \
  --region $AWS_REGION

# 4. Xem chi tiết alarm history
aws cloudwatch describe-alarm-history \
  --alarm-name CPUAlarm-HighCPU \
  --history-item-type StateUpdate \
  --query 'AlarmHistoryItems[*].[Timestamp,AlarmName,HistorySummary]' \
  --output table \
  --region $AWS_REGION
```

---

## Troubleshooting

| Vấn đề | Nguyên nhân | Giải pháp |
|--------|-------------|-----------|
| Không nhận được email | Subscription chưa Confirm | Kiểm tra email, nhấn Confirm subscription |
| Alarm không trigger | Threshold quá cao/thấp | Giảm ngưỡng hoặc kiểm tra metric |
| Alarm stuck ở INSUFFICIENT | Metric không có data | Kiểm tra EC2 đang chạy, metric có data |
| Notification không gửi | SNS Topic ARN sai | Kiểm tra ARN trong Alarm actions |
| Permission denied | IAM Role thiếu quyền | Thêm quyền sns:Publish vào SNS Topic Policy |

---

*AWS CLI Reference — CPU Alarm via SNS — W9 Monitoring*
