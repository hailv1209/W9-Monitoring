# AWS CLI Commands — CloudWatch Agent on EC2

## Môi trường

```bash
# Set biến môi trường (thay thế các giá trị phù hợp)
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export EC2_INSTANCE_ID="i-xxxxxxxxxxxxxxxxx"
export IAM_ROLE_NAME="CloudWatchAgentServerRole"
export IAM_PROFILE_NAME="CloudWatchAgentInstanceProfile"
```

---

## Bước 1: Cài đặt CloudWatch Agent

### Trên Amazon Linux / CentOS / RHEL

```bash
# Cài đặt agent package
sudo yum install amazon-cloudwatch-agent -y

# Xác nhận binary đã được cài
ls -la /opt/aws/amazon-cloudwatch-agent/bin/
```

### Trên Ubuntu / Debian

```bash
sudo apt-get update
sudo apt-get install amazon-cloudwatch-agent -y

# Xác nhận binary
ls -la /opt/aws/amazon-cloudwatch-agent/bin/
```

---

## Bước 2: Tạo IAM Role cho CloudWatch Agent

### 2.1 Tạo Trust Policy

```bash
cat > /tmp/cw-agent-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

### 2.2 Tạo IAM Role

```bash
aws iam create-role \
  --role-name $IAM_ROLE_NAME \
  --assume-role-policy-document file:///tmp/cw-agent-trust.json \
  --region $AWS_REGION
```

**Output mẫu:**
```json
{
  "Role": {
    "RoleName": "CloudWatchAgentServerRole",
    "Arn": "arn:aws:iam::123456789012:role/CloudWatchAgentServerRole",
    "RoleId": "AROAXXXXXXXXXXXXXX"
  }
}
```

### 2.3 Attach Managed Policy

```bash
aws iam attach-role-policy \
  --role-name $IAM_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --region $AWS_REGION
```

> ⚠️ **Lưu ý:** `CloudWatchAgentServerPolicy` là managed policy của AWS, cấp quyền:
> - `cloudwatch:PutMetricData` — gửi metrics lên CloudWatch
> - `logs:CreateLogGroup` / `logs:CreateLogStream` / `logs:PutLogEvents` — gửi logs

---

## Bước 3: Tạo Instance Profile và đính kèm vào EC2

### 3.1 Tạo Instance Profile

```bash
aws iam create-instance-profile \
  --instance-profile-name $IAM_PROFILE_NAME \
  --region $AWS_REGION
```

### 3.2 Thêm Role vào Instance Profile

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name $IAM_PROFILE_NAME \
  --role-name $IAM_ROLE_NAME \
  --region $AWS_REGION
```

### 3.3 Đính kèm Instance Profile vào EC2

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id $EC2_INSTANCE_ID \
  --iam-instance-profile Name=$IAM_PROFILE_NAME \
  --region $AWS_REGION
```

**Output mẫu:**
```json
{
  "IamInstanceProfileAssociation": {
    "InstanceId": "i-xxxxxxxxxxxxxxxxx",
    "IamInstanceProfile": {
      "Arn": "arn:aws:iam::123456789012:instance-profile/CloudWatchAgentInstanceProfile",
      "Id": "AIPAXXXXXXXXXXXXXXXXX"
    },
    "AssociationId": "iip-assoc-xxxxxxxxxxxxxxxxx",
    "State": "associated"
  }
}
```

### 3.4 Xác nhận IAM Role đã đính kèm

```bash
aws ec2 describe-iam-instance-profile-associations \
  --filters Name=instance-id,Values=$EC2_INSTANCE_ID \
  --region $AWS_REGION
```

---

## Bước 4: Chạy Configuration Wizard

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Wizard tạo file config tại: `/opt/aws/amazon-cloudwatch-agent/bin/config.json`

---

## Bước 5: Quản lý Service

### 5.1 Enable và Start

```bash
# Bật service (khởi động cùng hệ thống)
sudo systemctl enable amazon-cloudwatch-agent

# Khởi động service
sudo systemctl start amazon-cloudwatch-agent
```

### 5.2 Kiểm tra Status

```bash
sudo systemctl status amazon-cloudwatch-agent
```

**Output mẫu:**
```
● amazon-cloudwatch-agent.service - Amazon CloudWatch Agent
   Loaded: loaded (/etc/systemd/system/amazon-cloudwatch-agent.service; enabled; vendor preset: disable>
   Active: active (running) since ...
```

### 5.3 Stop / Restart / Disable

```bash
# Stop
sudo systemctl stop amazon-cloudwatch-agent

# Restart
sudo systemctl restart amazon-cloudwatch-agent

# Disable (ngăn khởi động cùng hệ thống)
sudo systemctl disable amazon-cloudwatch-agent
```

---

## Bước 6: Verify Agent Status

```bash
# Kiểm tra trạng thái agent (chế độ EC2)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

**Output mẫu:**
```json
{
  "status": "running",
  "metrics_collection_interval": 60,
  "start_time": "2026-06-12T08:00:00+00:00",
  "config": {
    "ecs": false,
    "ssm": false,
    "xml": false,
    "json": "/opt/aws/amazon-cloudwatch-agent/bin/config.json"
  }
}
```

### Các chế độ khác của `-m` flag

| Flag | Mô tả |
|------|--------|
| `-m ec2` | Chạy trên EC2 (dùng IAM Role) |
| `-m onPrem` | Chạy on-premises (dùng credentials file) |
| `-m ecs` | Chạy trong ECS container |

---

## Bước 7: Verify Custom Metrics trên CloudWatch

### 7.1 List CWAgent Metrics

```bash
aws cloudwatch list-metrics \
  --namespace CWAgent \
  --region $AWS_REGION
```

### 7.2 Get Memory Used Percent

```bash
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_used_percent \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum \
  --dimensions Name=InstanceId,Value=$EC2_INSTANCE_ID \
  --region $AWS_REGION
```

### 7.3 Get Disk Used Percent

```bash
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name disk_used_percent \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --dimensions Name=InstanceId,Value=$EC2_INSTANCE_ID Name=device,Value=/ Name= fstype,Value=xfs Name= path,Value=/ \
  --region $AWS_REGION
```

### 7.4 Get CPU Idle (từ custom agent)

```bash
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name cpu_usage_idle \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --dimensions Name=InstanceId,Value=$EC2_INSTANCE_ID Name=cpu,Value=cpu0 \
  --region $AWS_REGION
```

---

## Bước 8: Lưu Config vào SSM Parameter Store (Tùy chọn)

Thay vì lưu config file trên EC2, bạn có thể lưu vào SSM Parameter Store để quản lý tập trung.

### 8.1 Lưu config lên SSM

```bash
aws ssm put-parameter \
  --name AmazonCloudWatch-linux \
  --type String \
  --value file:///opt/aws/amazon-cloudwatch-agent/bin/config.json \
  --region $AWS_REGION
```

### 8.2 Start agent với SSM config

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -m ssm \
  -a start
```

---

## Bước 9: Cấu hình CloudWatch Dashboard với Custom Metrics

```bash
# Tạo Dashboard
aws cloudwatch put-dashboard \
  --dashboard-name EC2-System-Monitoring \
  --dashboard-body file://dashboard-body.json \
  --region $AWS_REGION
```

**dashboard-body.json mẫu:**
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["CWAgent", "mem_used_percent", "InstanceId", "i-xxxxxxxxxxxxxxxxx"],
          [".", "disk_used_percent", ".", ".", "/dev/xvda1"]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Memory & Disk Usage",
        "width": 12,
        "height": 6
      }
    }
  ]
}
```

---

## Bước 10: Cleanup (Xóa tài nguyên)

### 10.1 Gỡ IAM Role khỏi EC2

```bash
# Lấy Association ID trước
ASSOC_ID=$(aws ec2 describe-iam-instance-profile-associations \
  --filters Name=instance-id,Values=$EC2_INSTANCE_ID \
  --query 'IamInstanceProfileAssociations[0].AssociationId' \
  --output text \
  --region $AWS_REGION)

aws ec2 disassociate-iam-instance-profile \
  --association-id $ASSOC_ID \
  --region $AWS_REGION
```

### 10.2 Xóa IAM Role và Instance Profile

```bash
# Detach policy
aws iam detach-role-policy \
  --role-name $IAM_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
  --region $AWS_REGION

# Remove role từ instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name $IAM_PROFILE_NAME \
  --role-name $IAM_ROLE_NAME \
  --region $AWS_REGION

# Delete instance profile
aws iam delete-instance-profile \
  --instance-profile-name $IAM_PROFILE_NAME \
  --region $AWS_REGION

# Delete role
aws iam delete-role \
  --role-name $IAM_ROLE_NAME \
  --region $AWS_REGION
```

### 10.3 Xóa SSM Parameter (nếu đã lưu config ở SSM)

```bash
aws ssm delete-parameter \
  --name AmazonCloudWatch-linux \
  --region $AWS_REGION
```

### 10.4 Xóa CloudWatch Dashboard

```bash
aws cloudwatch delete-dashboards \
  --dashboard-names EC2-System-Monitoring \
  --region $AWS_REGION
```

---

## Troubleshooting

| Vấn đề | Nguyên nhân | Giải pháp |
|--------|-------------|-----------|
| Agent không gửi được metrics | EC2 thiếu IAM Role | Đính kèm CloudWatchAgentServerRole |
| `status: stopped` | Service chưa start | `sudo systemctl start amazon-cloudwatch-agent` |
| Metrics không xuất hiện trên CW | Config file sai | Kiểm tra `/opt/aws/amazon-cloudwatch-agent/bin/config.json` |
| `access denied` khi gửi metrics | IAM Role thiếu quyền | Verify CloudWatchAgentServerPolicy đã attach |
| Wizard bị treo | Non-interactive shell | Chạy với `sudo -i` hoặc `TERM=xterm` |
| Metrics có giá trị NaN | Agent chưa thu thập đủ data | Chờ 5-10 phút rồi query lại |

---

## Lệnh kiểm tra nhanh

```bash
# 1. Kiểm tra agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status

# 2. Kiểm tra service
sudo systemctl status amazon-cloudwatch-agent

# 3. Xem logs
sudo journalctl -u amazon-cloudwatch-agent -n 50

# 4. Kiểm tra config file
sudo cat /opt/aws/amazon-cloudwatch-agent/bin/config.json | jq .

# 5. List custom metrics
aws cloudwatch list-metrics --namespace CWAgent --region $AWS_REGION

# 6. Kiểm tra IAM Role trên EC2
aws ec2 describe-iam-instance-profile-associations \
  --filters Name=instance-id,Values=$EC2_INSTANCE_ID
```

---

*AWS CLI Reference — CloudWatch Agent on EC2 — W9 Monitoring*
