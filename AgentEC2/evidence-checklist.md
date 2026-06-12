# CloudWatch Agent on EC2 — Evidence Checklist

## Mục đích
File này ghi lại tiến độ hoàn thành bài Hands-On. Mỗi mục cần chụp screenshot khi hoàn thành.

---

## Phần 1: Cài đặt Agent Package

### [ ] 1.1 Cài đặt Amazon CloudWatch Agent

**Câu lệnh:**
```bash
# Amazon Linux / CentOS / RHEL / Fedora
sudo yum install amazon-cloudwatch-agent -y

# Ubuntu / Debian
sudo apt-get update
sudo apt-get install amazon-cloudwatch-agent -y
```

**Screenshots cần chụp:**
- [SCREENSHOT 1a] Terminal — kết quả cài đặt thành công (output không có lỗi)
- [SCREENSHOT 1b] Xác nhận binary tồn tại: `ls -la /opt/aws/amazon-cloudwatch-agent/bin/`

---

## Phần 2: Chạy Configuration Wizard

### [ ] 2.1 Chạy Wizard

**Câu lệnh:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

**Screenshots cần chụp:**
- [SCREENSHOT 2a] Wizard đang chạy — trả lời các câu hỏi (chọn OS, User, region, metrics level...)
- [SCREENSHOT 2b] Wizard hoàn tất — xác nhận file `/opt/aws/amazon-cloudwatch-agent/bin/config.json` đã được tạo

---

## Phần 3: IAM Role cho EC2 (Prerequisite)

> ⚠️ **Bắt buộc:** EC2 phải có IAM Role đính kèm. Nếu chưa có, CloudWatch Agent **sẽ không gửi được metrics** lên CloudWatch.

### [ ] 3.1 Tạo IAM Role với CloudWatchAgentServerPolicy

**Screenshots cần chụp:**
- [SCREENSHOT 3a] Trang IAM → Roles — Role `CloudWatchAgentServerRole` đã được tạo
- [SCREENSHOT 3b] Trang Role details — tab **Permissions** → Policy `CloudWatchAgentServerPolicy` đã attach
- [SCREENSHOT 3c] IAM Role có **Trusted Entities** = `ec2.amazonaws.com`

---

### [ ] 3.2 Đính kèm IAM Role vào EC2 Instance

**Câu lệnh (thay YOUR_INSTANCE_ID):**
```bash
aws ec2 associate-iam-instance-profile \
  --instance-id <YOUR_EC2_INSTANCE_ID> \
  --iam-instance-profile Name=CloudWatchAgentInstanceProfile
```

**Screenshots cần chụp:**
- [SCREENSHOT 4a] AWS CLI — kết quả lệnh associate IAM instance profile thành công
- [SCREENSHOT 4b] EC2 Console → Instance Details → Tab **Security** → IAM Role đã hiển thị

---

## Phần 4: Khởi động CloudWatch Agent

### [ ] 4.1 Enable và Start Service

**Câu lệnh:**
```bash
sudo systemctl enable amazon-cloudwatch-agent
sudo systemctl start amazon-cloudwatch-agent
```

**Screenshots cần chụp:**
- [SCREENSHOT 5a] Terminal — output `Created symlink...` từ `systemctl enable`
- [SCREENSHOT 5b] Terminal — `systemctl start` không có lỗi (exit 0)

---

### [ ] 4.2 Kiểm tra Service Status

**Câu lệnh:**
```bash
sudo systemctl status amazon-cloudwatch-agent
```

**Screenshots cần chụp:**
- [SCREENSHOT 6a] Output `systemctl status` — hiển thị `active (running)`
- [SCREENSHOT 6b] Process đang chạy: `amazon-cloudwatch-agent`

---

## Phần 5: Verify Agent Status

### [ ] 5.1 Kiểm tra Agent bằng amazon-cloudwatch-agent-ctl

**Câu lệnh:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

**Screenshots cần chụp:**
- [SCREENSHOT 7a] Output JSON — `"status": "running"`
- [SCREENSHOT 7b] Output JSON — `"config": { "json": "/opt/aws/amazon-cloudwatch-agent/bin/config.json" }`

---

### [ ] 5.2 Xác nhận Custom Metrics trên CloudWatch Console

**Câu lệnh để verify (hoặc dùng Console):**
```bash
aws cloudwatch list-metrics \
  --namespace CWAgent \
  --region us-east-1
```

**Screenshots cần chụp:**
- [SCREENSHOT 8a] CloudWatch Console → **Metrics** → **All metrics** → Namespace `CWAgent` đã xuất hiện
- [SCREENSHOT 8b] Danh sách metrics: `mem_used_percent`, `disk_used_percent`, `cpu_usage_idle`... (các custom metrics)

---

### [ ] 5.3 Xem Dashboard với Custom Metrics

**Screenshots cần chụp:**
- [SCREENSHOT 9] CloudWatch Dashboard — biểu đồ custom metric (ví dụ: Memory Used % theo thời gian)

---

## Phần 6: Kiểm tra Logs (Tùy chọn — Troubleshooting)

### [ ] 6.1 Xem Agent Logs

**Câu lệnh:**
```bash
sudo journalctl -u amazon-cloudwatch-agent -f
# Hoặc xem log file trực tiếp:
sudo cat /var/log/amazon/cloudwatch-agent.log
```

**Screenshots cần chụp:**
- [SCREENSHOT 10] Log file — không có lỗi ERROR, agent khởi động thành công

---

## Phần 7: Cleanup (Xóa tài nguyên sau khi hoàn thành)

### [ ] 7.1 Stop và Disable Service

```bash
sudo systemctl stop amazon-cloudwatch-agent
sudo systemctl disable amazon-cloudwatch-agent
```

### [ ] 7.2 Gỡ IAM Role khỏi EC2

```bash
aws ec2 disassociate-iam-instance-profile \
  --instance-id <YOUR_EC2_INSTANCE_ID> \
  --iam-instance-profile Name=CloudWatchAgentInstanceProfile
```

### [ ] 7.3 Xóa IAM Role và Policy

```bash
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

**Screenshots cần chụp:**
- [SCREENSHOT 11a] `systemctl stop/disable` không lỗi
- [SCREENSHOT 11b] IAM Role đã bị xóa khỏi AWS

---

## Tổng kết số lượng Screenshots

| Phần | Số lượng Screenshots |
|------|---------------------|
| Cài đặt Agent Package | 2 |
| Configuration Wizard | 2 |
| IAM Role tạo thành công | 3 |
| IAM Role đính kèm EC2 | 2 |
| Enable & Start Service | 2 |
| Verify Agent Status | 3 |
| CloudWatch Metrics Dashboard | 1 |
| Agent Logs | 1 |
| Cleanup | 2 |
| **Tổng cộng** | **18 screenshots** |

---

## Trạng thái hoàn thành

| # | Screenshot | Trạng thái | Ghi chú |
|---|-----------|-----------|---------|
| 1a | Agent package install success | [ ] | |
| 1b | Binary exists at /opt/aws/... | [ ] | |
| 2a | Wizard running | [ ] | |
| 2b | Wizard complete, config.json created | [ ] | |
| 3a | IAM Role created | [ ] | |
| 3b | CloudWatchAgentServerPolicy attached | [ ] | |
| 3c | Trusted Entity: ec2.amazonaws.com | [ ] | |
| 4a | associate-iam-instance-profile CLI success | [ ] | |
| 4b | EC2 Console — IAM Role attached | [ ] | |
| 5a | systemctl enable output | [ ] | |
| 5b | systemctl start success | [ ] | |
| 6a | systemctl status = active (running) | [ ] | |
| 6b | Process amazon-cloudwatch-agent running | [ ] | |
| 7a | agent-ctl status: running | [ ] | |
| 7b | agent-ctl config.json path | [ ] | |
| 8a | CloudWatch CWAgent namespace visible | [ ] | |
| 8b | Custom metrics list (mem_used, disk_used...) | [ ] | |
| 9 | Custom metrics graph on Dashboard | [ ] | |
| 10 | Agent log — no ERROR | [ ] | |
| 11a | systemctl stop/disable success | [ ] | |
| 11b | IAM Role deleted from AWS | [ ] | |

---

*Evidence Checklist — CloudWatch Agent on EC2 — W9 Monitoring*
