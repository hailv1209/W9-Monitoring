# AWS CLI Commands — Root Account Login Alarm

> **Best Practice:** Tài khoản root gần như không bao giờ được sử dụng. Bài này tạo alarm phát hiện **bất kỳ** lần đăng nhập/hoạt động nào của root và gửi email cảnh báo qua SNS.

## Môi trường

```bash
# Set biến môi trường (thay thế các giá trị phù hợp)
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export EMAIL="haileab542@gmail.com"

export TRAIL_NAME="RootAlarmTrail"
export TRAIL_BUCKET="root-alarm-trail-bucket-${AWS_ACCOUNT_ID}"
export LOG_GROUP_NAME="RootAlarmCloudTrailLogGroup"
export SNS_TOPIC_NAME="RootAlarmTopic"
export FILTER_NAME="RootAccountLoginFilter"
export ALARM_NAME="RootAlarm-RootLogin"
export METRIC_NAMESPACE="Security"
export METRIC_NAME="RootAccountLoginCount"

export SNS_TOPIC_ARN="arn:aws:sns:${AWS_REGION}:${AWS_ACCOUNT_ID}:${SNS_TOPIC_NAME}"
```

---

## Bước 1: Tạo SNS Topic & Email Subscription

```bash
# 1.1 Tạo SNS Topic
aws sns create-topic \
  --name $SNS_TOPIC_NAME \
  --region $AWS_REGION

# 1.2 Tạo Email Subscription
aws sns subscribe \
  --topic-arn $SNS_TOPIC_ARN \
  --protocol email \
  --notification-endpoint $EMAIL \
  --region $AWS_REGION
```

> ⚠️ **Lưu ý:** Vào hộp thư `haileab542@gmail.com` và nhấn **Confirm subscription** trong email từ `no-reply@sns.amazonaws.com`.

```bash
# 1.3 Kiểm tra subscription status (phải là "Confirmed" sau khi xác nhận)
aws sns list-subscriptions-by-topic \
  --topic-arn $SNS_TOPIC_ARN \
  --region $AWS_REGION
```

---

## Bước 2: Tạo S3 Bucket cho CloudTrail

> CloudTrail yêu cầu một S3 bucket để lưu trữ log files.

```bash
# 2.1 Tạo S3 bucket
aws s3api create-bucket \
  --bucket $TRAIL_BUCKET \
  --region $AWS_REGION \
  $([ "$AWS_REGION" != "us-east-1" ] && echo "--create-bucket-configuration LocationConstraint=$AWS_REGION")

# 2.2 Apply bucket policy cho CloudTrail
cat > /tmp/trail-bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${TRAIL_BUCKET}"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${TRAIL_BUCKET}/AWSLogs/${AWS_ACCOUNT_ID}/*",
      "Condition": { "StringEquals": { "s3:x-amz-acl": "bucket-owner-full-control" } }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket $TRAIL_BUCKET \
  --policy file:///tmp/trail-bucket-policy.json
```

---

## Bước 3: Tạo IAM Role cho CloudTrail ghi log sang CloudWatch

> CloudTrail cần một IAM Role với trust policy cho phép `cloudtrail.amazonaws.com` assume role, và permission policy cho phép ghi log vào CloudWatch Logs.

```bash
# 3.1 Tạo trust policy
cat > /tmp/cloudtrail-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# 3.2 Tạo IAM Role
aws iam create-role \
  --role-name CloudTrailRoleForCloudWatchLogs \
  --assume-role-policy-document file:///tmp/cloudtrail-trust-policy.json

# 3.3 Attach policy cho phép CloudTrail ghi vào CloudWatch Logs
cat > /tmp/cloudtrail-logs-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:${LOG_GROUP_NAME}:log-stream:*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name CloudTrailRoleForCloudWatchLogs \
  --policy-name CloudTrailLogsPolicy \
  --policy-document file:///tmp/cloudtrail-logs-policy.json
```

---

## Bước 4: Tạo CloudWatch Log Group

```bash
aws logs create-log-group \
  --log-group-name $LOG_GROUP_NAME \
  --region $AWS_REGION

# Set retention (optional — giữ log 90 ngày để tiết kiệm chi phí)
aws logs put-retention-policy \
  --log-group-name $LOG_GROUP_NAME \
  --retention-in-days 90 \
  --region $AWS_REGION
```

---

## Bước 5: Tạo CloudTrail Trail (gửi log sang CloudWatch)

```bash
aws cloudtrail create-trail \
  --name $TRAIL_NAME \
  --s3-bucket-name $TRAIL_BUCKET \
  --cloud-watch-logs-log-group-arn "arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:${LOG_GROUP_NAME}" \
  --cloud-watch-logs-role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/CloudTrailRoleForCloudWatchLogs" \
  --include-global-service-events \
  --is-multi-region-trail \
  --region $AWS_REGION

# Bật logging cho trail
aws cloudtrail start-logging \
  --name $TRAIL_NAME \
  --region $AWS_REGION

# Kiểm tra trạng thái trail
aws cloudtrail describe-trails \
  --trail-name-list $TRAIL_NAME \
  --region $AWS_REGION

aws cloudtrail get-trail-status \
  --name $TRAIL_NAME \
  --region $AWS_REGION
```

**Output mẫu của `get-trail-status`:**
```json
{
  "IsLogging": true,
  "LatestDeliveryTime": "2026-06-13T15:30:00Z",
  "LatestNotificationTime": "2026-06-13T15:28:00Z",
  "StartLoggingTime": "2026-06-13T10:00:00Z"
}
```

---

## Bước 6: Tạo CloudWatch Metric Filter

> **Filter Pattern:** `{ $.userIdentity.type = "Root" && $.eventType != "AwsServiceEvent" }`
>
> **Metric Name:** `RootAccountLoginCount` | **Namespace:** `Security` | **Value:** `1`

```bash
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name $FILTER_NAME \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations \
    metricName=$METRIC_NAME,metricNamespace=$METRIC_NAMESPACE,metricValue=1,defaultValue=0 \
  --region $AWS_REGION

# Kiểm tra metric filter đã tạo
aws logs describe-metric-filters \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name-prefix $FILTER_NAME \
  --region $AWS_REGION
```

**Output mẫu:**
```json
{
  "metricFilters": [
    {
      "filterName": "RootAccountLoginFilter",
      "filterPattern": "{ $.userIdentity.type = \"Root\" && $.eventType != \"AwsServiceEvent\" }",
      "metricTransformations": [
        {
          "metricName": "RootAccountLoginCount",
          "metricNamespace": "Security",
          "metricValue": "1",
          "defaultValue": 0
        }
      ]
    }
  ]
}
```

---

## Bước 7: Tạo CloudWatch Alarm

> **Threshold:** `RootAccountLoginCount >= 1` trong 5-minute period — **BẤT KỲ** root login nào cũng trigger alarm.

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name $ALARM_NAME \
  --alarm-description "Alert when AWS root account is used (any single login triggers alarm)" \
  --actions-enabled \
  --alarm-actions $SNS_TOPIC_ARN \
  --ok-actions $SNS_TOPIC_ARN \
  --insufficient-data-actions [] \
  --metric-name $METRIC_NAME \
  --namespace $METRIC_NAMESPACE \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --datapoints-to-alarm 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --region $AWS_REGION

# Kiểm tra alarm đã tạo
aws cloudwatch describe-alarms \
  --alarm-names $ALARM_NAME \
  --query 'MetricAlarms[0].{Name:AlarmName,State:StateValue,Threshold:Threshold,Period:Period,Metric:MetricName,Namespace:Namespace}' \
  --output table \
  --region $AWS_REGION
```

**Tham số quan trọng:**

| Tham số | Giá trị | Ý nghĩa |
|---------|---------|---------|
| `--metric-name` | `RootAccountLoginCount` | Metric từ filter |
| `--namespace` | `Security` | Custom namespace |
| `--statistic` | `Sum` | Tổng số root login trong period |
| `--period` | `300` | 5 phút (300 giây) |
| `--evaluation-periods` | `1` | Chỉ cần 1 lần vi phạm |
| `--datapoints-to-alarm` | `1` | Cần 1/1 datapoints |
| `--threshold` | `1` | Bất kỳ root login nào cũng trigger |
| `--comparison-operator` | `GreaterThanOrEqualToThreshold` | ≥ threshold |
| `--treat-missing-data` | `notBreaching` | Missing data không trigger |

---

## Bước 8: Kiểm tra trạng thái Alarm

```bash
# Xem trạng thái hiện tại
aws cloudwatch describe-alarms \
  --alarm-names $ALARM_NAME \
  --query 'MetricAlarms[0].StateValue' \
  --output text \
  --region $AWS_REGION

# Xem lịch sử alarm (state changes)
aws cloudwatch describe-alarm-history \
  --alarm-name $ALARM_NAME \
  --history-item-type StateUpdate \
  --query 'AlarmHistoryItems[*].[Timestamp,HistorySummary]' \
  --output table \
  --region $AWS_REGION

# Xem metric datapoints hiện tại
aws cloudwatch get-metric-statistics \
  --namespace $METRIC_NAMESPACE \
  --metric-name $METRIC_NAME \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --region $AWS_REGION
```

---

## Bước 9: Test — Kích hoạt Root Login

> ⚠️ **Cảnh báo bảo mật:** Chỉ thực hiện khi bạn đã sẵn sàng test. Đăng nhập root và **đăng xuất ngay lập tức** sau khi xong.

```bash
# Bước 1: Mở browser ở chế độ incognito
# Bước 2: Truy cập https://console.aws.amazon.com/
# Bước 3: Chọn "Sign in as root user"
# Bước 4: Nhập email + password của root
# Bước 5: Sau khi vào console thành công → Sign out

# Sau khi đăng nhập bằng root, đợi ~5-10 phút cho CloudTrail đẩy log

# Kiểm tra CloudTrail log stream mới nhất
aws logs describe-log-streams \
  --log-group-name $LOG_GROUP_NAME \
  --order-by LastEventTime \
  --descending \
  --max-items 3 \
  --region $AWS_REGION

# Kiểm tra metric có datapoint chưa
aws cloudwatch get-metric-statistics \
  --namespace $METRIC_NAMESPACE \
  --metric-name $METRIC_NAME \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --region $AWS_REGION

# Kiểm tra alarm state
aws cloudwatch describe-alarms \
  --alarm-names $ALARM_NAME \
  --query 'MetricAlarms[0].StateValue' \
  --output text \
  --region $AWS_REGION
```

---

## Bước 10: Test SNS Topic (gửi notification thủ công)

```bash
# Gửi test notification
aws sns publish \
  --topic-arn $SNS_TOPIC_ARN \
  --subject "TEST: Root Alarm Notification" \
  --message "This is a test. Your Root Account Login Alarm is configured correctly. Any root login will trigger this alarm." \
  --region $AWS_REGION
```

---

## Bước 11: Xóa tài nguyên (Cleanup)

> Sau khi demo xong, xóa tài nguyên theo thứ tự để tránh lỗi dependency.

```bash
# 11.1 Xóa CloudWatch Alarm
aws cloudwatch delete-alarms \
  --alarm-names $ALARM_NAME \
  --region $AWS_REGION

# 11.2 Xóa Metric Filter
aws logs delete-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name $FILTER_NAME \
  --region $AWS_REGION

# 11.3 Tắt logging và xóa CloudTrail
aws cloudtrail stop-logging --name $TRAIL_NAME --region $AWS_REGION
aws cloudtrail delete-trail --name $TRAIL_NAME --region $AWS_REGION

# 11.4 Xóa IAM Role và policies
aws iam delete-role-policy \
  --role-name CloudTrailRoleForCloudWatchLogs \
  --policy-name CloudTrailLogsPolicy
aws iam delete-role \
  --role-name CloudTrailRoleForCloudWatchLogs

# 11.5 Xóa CloudWatch Log Group
aws logs delete-log-group \
  --log-group-name $LOG_GROUP_NAME \
  --region $AWS_REGION

# 11.6 Xóa SNS Topic (sẽ xóa luôn subscriptions)
aws sns delete-topic \
  --topic-arn $SNS_TOPIC_ARN \
  --region $AWS_REGION

# 11.7 Xóa S3 bucket (chỉ khi bucket này do bạn tạo)
aws s3 rm s3://$TRAIL_BUCKET --recursive
aws s3api delete-bucket --bucket $TRAIL_BUCKET --region $AWS_REGION
```

---

## Bước 12: Deploy bằng CloudFormation

```bash
# Tạo Stack
aws cloudformation create-stack \
  --stack-name RootAlarmStack \
  --template-body file://root-alarm-template.yaml \
  --parameters \
    ParameterKey=EmailAddress,ParameterValue=$EMAIL \
    ParameterKey=TrailName,ParameterValue=$TRAIL_NAME \
    ParameterKey=LogGroupName,ParameterValue=$LOG_GROUP_NAME \
    ParameterKey=MetricFilterName,ParameterValue=$FILTER_NAME \
    ParameterKey=AlarmName,ParameterValue=$ALARM_NAME \
    ParameterKey=SNSTopicName,ParameterValue=$SNS_TOPIC_NAME \
  --capabilities CAPABILITY_IAM \
  --region $AWS_REGION

# Kiểm tra trạng thái Stack
aws cloudformation describe-stacks \
  --stack-name RootAlarmStack \
  --query 'Stacks[0].StackStatus' \
  --output text \
  --region $AWS_REGION

# Cập nhật Stack (nếu cần)
aws cloudformation update-stack \
  --stack-name RootAlarmStack \
  --template-body file://root-alarm-template.yaml \
  --parameters \
    ParameterKey=EmailAddress,ParameterValue=$EMAIL \
    ParameterKey=TrailName,ParameterValue=$TRAIL_NAME \
    ParameterKey=LogGroupName,ParameterValue=$LOG_GROUP_NAME \
    ParameterKey=MetricFilterName,ParameterValue=$FILTER_NAME \
    ParameterKey=AlarmName,ParameterValue=$ALARM_NAME \
    ParameterKey=SNSTopicName,ParameterValue=$SNS_TOPIC_NAME \
  --capabilities CAPABILITY_IAM \
  --region $AWS_REGION

# Xóa Stack (cleanup cuối cùng — xóa S3 bucket trước nếu Retain)
aws s3 rm s3://root-alarm-trail-bucket-${AWS_ACCOUNT_ID} --recursive
aws cloudformation delete-stack \
  --stack-name RootAlarmStack \
  --region $AWS_REGION
```

---

## Lệnh hỗ trợ kiểm tra

```bash
# 1. Liệt kê CloudTrail trails
aws cloudtrail describe-trails --region $AWS_REGION

# 2. Trạng thái trail (IsLogging phải = true)
aws cloudtrail get-trail-status --name $TRAIL_NAME --region $AWS_REGION

# 3. Tìm kiếm sự kiện root login trong CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=root \
  --max-results 10 \
  --region $AWS_REGION

# 4. Liệt kê metric filters
aws logs describe-metric-filters --region $AWS_REGION

# 5. Liệt kê alarms
aws cloudwatch describe-alarms --region $AWS_REGION

# 6. Liệt kê SNS subscriptions
aws sns list-subscriptions --region $AWS_REGION
```

---

## Troubleshooting

| Vấn đề | Nguyên nhân | Giải pháp |
|--------|-------------|-----------|
| Không nhận được email | Subscription chưa Confirm | Kiểm tra email, nhấn Confirm subscription |
| Metric filter không match | Filter pattern sai | Đảm bảo pattern đúng: `{ $.userIdentity.type = "Root" && $.eventType != "AwsServiceEvent" }` |
| Alarm ở INSUFFICIENT_DATA | CloudTrail chưa đẩy log | Đợi ~5-10 phút sau khi tạo trail |
| Không thấy log trong CloudWatch | CloudWatch Logs chưa được bật trong Trail | Bật lại "Enabled CloudWatch Logs" trong trail config |
| Permission denied khi tạo trail | IAM Role thiếu quyền | Kiểm tra trust policy cho `cloudtrail.amazonaws.com` và permission policy cho `logs:PutLogEvents` |
| Alarm không trigger | Metric value = 0 | Kiểm tra root có thực sự đăng nhập chưa; kiểm tra metric filter với `aws logs test-metric-filter` |
| S3 bucket policy bị reject | Bucket policy không đúng region | Đảm bảo bucket policy chỉ định đúng region và account ID |
| CloudTrail không tạo được | S3 bucket policy thiếu | Đảm bảo bucket đã áp dụng policy cho phép CloudTrail `s3:PutObject` |

---

## Giải thích Filter Pattern chi tiết

```
{ $.userIdentity.type = "Root" && $.eventType != "AwsServiceEvent" }
```

| Thành phần | Ý nghĩa |
|------------|---------|
| `$.userIdentity.type` | Trường `userIdentity.type` trong CloudTrail event JSON |
| `= "Root"` | Giá trị chính xác là `Root` (đăng nhập/hoạt động bằng root user) |
| `&&` | AND logic |
| `$.eventType` | Trường `eventType` trong CloudTrail event |
| `!= "AwsServiceEvent"` | Loại trừ các sự kiện do AWS service tự sinh (ví dụ: Health API checks) |

> **Kết quả:** Bất kỳ API call nào do root user thực hiện (Console login, S3, EC2, IAM...) đều trigger metric `1`. Loại trừ `AwsServiceEvent` giúp giảm false positive.

---

*AWS CLI Reference — Root Account Login Alarm via CloudTrail + CloudWatch — W9 Monitoring*
