# Hands-On: Installing the CloudWatch Agent on EC2

## Mục tiêu
Cài đặt và cấu hình **Amazon CloudWatch Agent** trên EC2 instance để thu thập custom metrics (RAM, Disk, Process...) ngoài các metrics mặc định của EC2.

---

## Tổng quan kiến trúc

```
┌─────────────────┐     ┌──────────────────────────────┐     ┌─────────────────┐
│  EC2 Instance    │     │  CloudWatch Agent            │     │  CloudWatch     │
│  (Linux/         │────▶│  /opt/aws/amazon-cloudwatch- │────▶│  Custom         │
│   Ubuntu)        │     │  agent/bin/                  │     │  Metrics        │
└─────────────────┘     └──────────────────────────────┘     └─────────────────┘
                                 ▲
                                 │
                    config-wizard wizard
                    generates JSON config
```

---

## Bước 1: Cài đặt Agent Package

### Trên Amazon Linux / CentOS / RHEL / Fedora

```bash
sudo yum install amazon-cloudwatch-agent -y
```

### Trên Ubuntu / Debian

```bash
sudo apt-get update
sudo apt-get install amazon-cloudwatch-agent -y
```

> 📸 **[SCREENSHOT 1]: Kết quả cài đặt thành công — agent binary tồn tại tại `/opt/aws/amazon-cloudwatch-agent/bin/`**

---

## Bước 2: Chạy Configuration Wizard

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Wizard sẽ hỏi lần lượt các câu hỏi sau (chọn default nếu phù hợp):

```
===============================================
Welcome to the CloudWatch Agent Configuration Wizard
===============================================

On which OS are you planning to use the CloudWatch agent?
  1. linux
  2. windows
  3. macos
  Default choice: [1]

Which user should this agent run as?
  1. root
  2. cwagent
  3. others
  Default choice: [1]

Do you want to turn on StatsD daemon?
  1. yes
  2. no
  Default choice: [2]

Do you want to monitor metrics from CollectD?
  1. yes
  2. no
  Default choice: [2]

Which CloudWatch region do you want to use for storing metrics?
  Default choice: [us-east-1]
  Your choice: us-east-1

Which credential scheme should this agent use?
  1. AWS::Default
  2. AWS::NamedProfile
  Default choice: [1]

Do you want to enable custom metrics collection?
  1. Basic (CPU, Memory, Disk Space, Net, Processes)
  2. Advanced (include more metrics: diskio, ethstat, netstat, ss, top)
  Default choice: [2]

Do you want to monitor any of the logs?
  1. yes
  2. no
  Default choice: [2]

===============================================
Config file is saved as: /opt/aws/amazon-cloudwatch-agent/bin/config.json
===============================================
```

> 📸 **[SCREENSHOT 2a]: Wizard đang chạy — các câu hỏi và câu trả lời**
> 📸 **[SCREENSHOT 2b]: Wizard hoàn tất — file config.json đã được tạo**

---

## Bước 3: Đính kèm IAM Role cho EC2 (Prerequisite)

> ⚠️ **Quan trọng:** EC2 phải có IAM Role với policy `CloudWatchAgentServerPolicy` đính kèm thì agent mới gửi được metrics lên CloudWatch.

### 3.1 Tạo IAM Role

```bash
# Tạo trust policy file
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

# Tạo IAM Role
aws iam create-role \
  --role-name CloudWatchAgentServerRole \
  --assume-role-policy-document file:///tmp/cw-agent-trust.json
```

### 3.2 Attach Policy

```bash
# Attach managed policy
aws iam attach-role-policy \
  --role-name CloudWatchAgentServerRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

### 3.3 Tạo Instance Profile và đính kèm vào EC2

```bash
# Tạo instance profile
aws iam create-instance-profile \
  --instance-profile-name CloudWatchAgentInstanceProfile

# Thêm role vào instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name CloudWatchAgentInstanceProfile \
  --role-name CloudWatchAgentServerRole

# Đính kèm vào EC2 instance
aws ec2 associate-iam-instance-profile \
  --instance-id <YOUR_EC2_INSTANCE_ID> \
  --iam-instance-profile Name=CloudWatchAgentInstanceProfile
```

> 📸 **[SCREENSHOT 3a]: IAM Role đã tạo với policy CloudWatchAgentServerPolicy**
> 📸 **[SCREENSHOT 3b]: EC2 instance đã đính kèm IAM Role**

---

## Bước 4: Khởi động CloudWatch Agent

### 4.1 Enable và Start service

```bash
# Enable service (khởi động cùng hệ thống)
sudo systemctl enable amazon-cloudwatch-agent

# Start service ngay
sudo systemctl start amazon-cloudwatch-agent
```

### 4.2 Kiểm tra trạng thái service

```bash
# Xem trạng thái service
sudo systemctl status amazon-cloudwatch-agent

# Xem logs gần đây
sudo journalctl -u amazon-cloudwatch-agent -f
```

> 📸 **[SCREENSHOT 4a]: Lệnh `systemctl enable` thành công**
> 📸 **[SCREENSHOT 4b]: Service `amazon-cloudwatch-agent` đang chạy (active/running)**

---

## Bước 5: Verify — Kiểm tra Agent Status

```bash
# Kiểm tra trạng thái agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

**Output mong đợi:**

```json
{
  "status": "running",
  "metrics_collection_interval": 60,
  "start_time": "2026-06-12T00:00:00+00:00",
  "config": {
    "ecs": false,
    "xml": false,
    "json": "/opt/aws/amazon-cloudwatch-agent/bin/config.json"
  }
}
```

### Kiểm tra metrics đã được gửi lên CloudWatch

```bash
# Xem metrics Memory % Used từ EC2
aws cloudwatch get-metric-statistics \
  --namespace CWAgent \
  --metric-name mem_used_percent \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --dimensions Name=InstanceId,Value=<YOUR_EC2_INSTANCE_ID> \
  --region us-east-1
```

> 📸 **[SCREENSHOT 5a]: Lệnh `amazon-cloudwatch-agent-ctl -m ec2 -a status` trả về `status: running`**
> 📸 **[SCREENSHOT 5b]: CloudWatch Console — metrics custom đã xuất hiện (mem_used_percent, disk_used_percent...)**

---

## Bước 6: Tạo CloudWatch Alarm trên Custom Metrics (Tùy chọn)

Giờ bạn có thể tạo alarm dựa trên custom metrics. Ví dụ: RAM > 80%:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name EC2-HighMemory \
  --alarm-description "Alert when EC2 Memory > 80%" \
  --actions-enabled \
  --alarm-actions arn:aws:sns:us-east-1:ACCOUNT_ID:CPUAlarmTopic \
  --metric-name mem_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=<YOUR_EC2_INSTANCE_ID>
```

> 📸 **[SCREENSHOT 6]: CloudWatch Alarm dựa trên custom metric (Memory > 80%)**

---

## Cleanup (Xóa tài nguyên sau khi hoàn thành)

```bash
# Stop và disable service
sudo systemctl stop amazon-cloudwatch-agent
sudo systemctl disable amazon-cloudwatch-agent

# Xóa config file
sudo rm /opt/aws/amazon-cloudwatch-agent/bin/config.json

# Xóa IAM Role khỏi EC2
aws ec2 disassociate-iam-instance-profile \
  --instance-id <YOUR_EC2_INSTANCE_ID> \
  --iam-instance-profile Name=CloudWatchAgentInstanceProfile

# Detach policy và xóa IAM Role
aws iam detach-role-policy \
  --role-name CloudWatchAgentServerRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam remove-role-from-instance-profile \
  --instance-profile-name CloudWatchAgentInstanceProfile \
  --role-name CloudWatchAgentServerRole

aws iam delete-instance-profile \
  --instance-profile-name CloudWatchAgentInstanceProfile

aws iam delete-role \
  --role-name CloudWatchAgentServerRole
```

---

## Tóm tắt kiến thức

| Khái niệm | Giải thích |
|-----------|-----------|
| **CloudWatch Agent** | Agent chạy trên EC2, thu thập system-level metrics (RAM, Disk, Process...) mà AWS không thu thập mặc định |
| **CWAgent namespace** | Custom metrics từ agent được ghi vào namespace `CWAgent` (khác với `AWS/EC2`) |
| **SSM Parameter Store** | Config file có thể lưu ở SSM Parameter Store để quản lý tập trung |
| **CloudWatchAgentServerPolicy** | IAM managed policy cấp quyền `cloudwatch:PutMetricData` để agent gửi metrics |
| **config.json** | File cấu hình do wizard tạo, nằm ở `/opt/aws/amazon-cloudwatch-agent/bin/config.json` |

---

## Screenshot Checklist

| # | Mô tả Screenshot | Trạng thái |
|---|-----------------|------------|
| 1 | Agent package cài đặt thành công | [CHỖ CHỤP] |
| 2a | Wizard đang chạy | [CHỖ CHỤP] |
| 2b | Wizard hoàn tất, config.json tạo xong | [CHỖ CHỤP] |
| 3a | IAM Role với CloudWatchAgentServerPolicy | [CHỖ CHỤP] |
| 3b | EC2 đã đính kèm IAM Role | [CHỖ CHỤP] |
| 4a | systemctl enable thành công | [CHỖ CHỤP] |
| 4b | Service amazon-cloudwatch-agent active/running | [CHỖ CHỤP] |
| 5a | amazon-cloudwatch-agent-ctl status = running | [CHỖ CHỤP] |
| 5b | CloudWatch Console — custom metrics xuất hiện | [CHỖ CHỤP] |
| 6 | Alarm trên custom metric (tùy chọn) | [CHỖ CHỤP] |

---

*Hands-On: Installing CloudWatch Agent on EC2 — W9 Monitoring*
