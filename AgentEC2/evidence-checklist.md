# CloudWatch Agent on EC2 — Evidence Checklist

## Mục đích
File này ghi lại tiến độ hoàn thành bài Hands-On. Mỗi mục cần chụp screenshot khi hoàn thành.

---

## Phần 1: Cài đặt Agent Package

### [x] 1.1 Cài đặt Amazon CloudWatch Agent

**Câu lệnh:**
```bash
# Amazon Linux / CentOS / RHEL / Fedora
sudo yum install amazon-cloudwatch-agent -y

# Ubuntu / Debian
sudo apt-get update
sudo apt-get install amazon-cloudwatch-agent -y
```

**Screenshots:**

![SCREENSHOT 1a](asset/1a.png)
*Terminal — kết quả cài đặt thành công*

![SCREENSHOT 1b](asset/1b.png)
*Binary tồn tại tại `/opt/aws/amazon-cloudwatch-agent/bin/`*

---

## Phần 2: Chạy Configuration Wizard

### [x] 2.1 Chạy Wizard

**Câu lệnh:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

**Screenshots:**

![SCREENSHOT 2a](asset/2a.png)
*Wizard đang chạy — trả lời các câu hỏi*

![SCREENSHOT 2b](asset/2b.png)
*Wizard hoàn tất — file config.json đã được tạo*

---

## Phần 3: IAM Role cho EC2 (Prerequisite)

> ⚠️ **Bắt buộc:** EC2 phải có IAM Role đính kèm. Nếu chưa có, CloudWatch Agent **sẽ không gửi được metrics** lên CloudWatch.

### [x] 3.1 Tạo IAM Role với CloudWatchAgentServerPolicy

**Screenshots:**

![SCREENSHOT 3a](asset/3a.png)
*IAM Console → Roles → Role `CloudWatchAgentServerRole` đã được tạo*

![SCREENSHOT 3b](asset/3b.png)
*Tab **Permissions** → Policy `CloudWatchAgentServerPolicy` đã attach*

![SCREENSHOT 3c](asset/3c.png)
*IAM Role có **Trusted Entities** = `ec2.amazonaws.com`*

---

### [x] 3.2 Đính kèm IAM Role vào EC2 Instance

**Screenshots:**

![SCREENSHOT 4a](asset/4a.png)
*AWS CLI — kết quả lệnh associate IAM instance profile thành công*

![SCREENSHOT 4b](asset/4b.png)
*EC2 Console → Instance Details → IAM Role đã hiển thị*

---

## Phần 4: Khởi động CloudWatch Agent

### [x] 4.1 Enable và Start Service

**Câu lệnh:**
```bash
sudo systemctl enable amazon-cloudwatch-agent
sudo systemctl start amazon-cloudwatch-agent
```

**Screenshots:**

![SCREENSHOT 5a](asset/5a.png)
*Output `systemctl enable` — Created symlink*

![SCREENSHOT 5b](asset/5b.png)
*`systemctl start` không có lỗi*

---

### [x] 4.2 Kiểm tra Service Status

**Câu lệnh:**
```bash
sudo systemctl status amazon-cloudwatch-agent
```

**Screenshots:**

![SCREENSHOT 6a+b](asset/6a+b.png)
*Output `systemctl status` — hiển thị `active (running)` và Process `amazon-cloudwatch-agent`*

---

## Phần 5: Verify Agent Status

### [x] 5.1 Kiểm tra Agent bằng amazon-cloudwatch-agent-ctl

**Câu lệnh:**
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

**Screenshots:**

![SCREENSHOT 7a+b](asset/7a+b.png)
*Output JSON — `"status": "running"` và `"config"` path*

---

### [x] 5.2 Xác nhận Custom Metrics trên CloudWatch Console

**Screenshots:**

![SCREENSHOT 8a](asset/8a.png)
*CloudWatch Console → **Metrics** → Namespace `CWAgent` đã xuất hiện*

![SCREENSHOT 8b](asset/8b.png)
*Danh sách metrics: `mem_used_percent`, `disk_used_percent`, `cpu_usage_idle`...*

---

### [x] 5.3 Xem Dashboard với Custom Metrics

**Screenshots:**

![SCREENSHOT 9](asset/9.png)
*CloudWatch Dashboard — biểu đồ custom metric (Memory Used %)*

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

> ⚠️ **Quan trọng:** Sau khi hoàn thành bài lab, hãy xóa các tài nguyên để tránh phát sinh phí.

### Xóa EC2 Instance

**AWS Console:**
1. Truy cập EC2 Console → Instances
2. Chọn instance `i-02ce20bce9e63ccd7`
3. Actions → Instance State → **Terminate**

**CLI:**
```powershell
aws ec2 terminate-instances --instance-ids i-02ce20bce9e63ccd7
```

### Xóa IAM Role và Instance Profile

**CLI:**
```powershell
# 1. Detach policy
aws iam detach-role-policy --role-name CloudWatchAgentServerRole --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# 2. Remove role from instance profile
aws iam remove-role-from-instance-profile --instance-profile-name CloudWatchAgentInstanceProfile --role-name CloudWatchAgentServerRole

# 3. Delete instance profile
aws iam delete-instance-profile --instance-profile-name CloudWatchAgentInstanceProfile

# 4. Delete role
aws iam delete-role --role-name CloudWatchAgentServerRole
```

---

## Tổng kết số lượng Screenshots

| Phần | Screenshots |
|------|-------------|
| Cài đặt Agent Package | 2 ✅ |
| Configuration Wizard | 2 ✅ |
| IAM Role tạo thành công | 3 ✅ |
| IAM Role đính kèm EC2 | 2 ✅ |
| Enable & Start Service | 2 ✅ |
| Verify Agent Status | 2 ✅ |
| CloudWatch Metrics Dashboard | 1 ✅ |
| Agent Logs | 0 ❌ (tùy chọn) |
| **Tổng cộng** | **16 screenshots** ✅ |

---

## Trạng thái hoàn thành

| # | Screenshot | Trạng thái | Ghi chú |
|---|-----------|-----------|---------|
| 1a | Agent package install success | ✅ Hoàn thành | |
| 1b | Binary exists at /opt/aws/... | ✅ Hoàn thành | |
| 2a | Wizard running | ✅ Hoàn thành | |
| 2b | Wizard complete, config.json created | ✅ Hoàn thành | |
| 3a | IAM Role created | ✅ Hoàn thành | |
| 3b | CloudWatchAgentServerPolicy attached | ✅ Hoàn thành | |
| 3c | Trusted Entity: ec2.amazonaws.com | ✅ Hoàn thành | |
| 4a | associate-iam-instance-profile CLI success | ✅ Hoàn thành | |
| 4b | EC2 Console — IAM Role attached | ✅ Hoàn thành | |
| 5a | systemctl enable output | ✅ Hoàn thành | |
| 5b | systemctl start success | ✅ Hoàn thành | |
| 6a | systemctl status = active (running) | ✅ Hoàn thành | |
| 6b | Process amazon-cloudwatch-agent running | ✅ Hoàn thành | |
| 7a | agent-ctl status: running | ✅ Hoàn thành | |
| 7b | agent-ctl config.json path | ✅ Hoàn thành | |
| 8a | CloudWatch CWAgent namespace visible | ✅ Hoàn thành | |
| 8b | Custom metrics list (mem_used, disk_used...) | ✅ Hoàn thành | |
| 9 | Custom metrics graph on Dashboard | ✅ Hoàn thành | |
| 10 | Agent log — no ERROR | ❌ Tùy chọn | Không bắt buộc |
| 11a | EC2 terminated | ❌ Chưa thực hiện | Cần cleanup |
| 11b | IAM Role deleted from AWS | ❌ Chưa thực hiện | Cần cleanup |

---

## Hướng dẫn Cleanup Chi Tiết (Tránh phí oan)

### Bước 1: Xóa EC2 Instance

**Cách 1 — AWS Console:**
1. Mở: [https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:)
2. Tick chọn instance `i-02ce20bce9e63ccd7`
3. Actions → Instance State → **Terminate instance**
4. Confirm

**Cách 2 — AWS CLI:**
```powershell
aws ec2 terminate-instances --instance-ids i-02ce20bce9e63ccd7
```

### Bước 2: Xóa IAM Role và Instance Profile

**AWS Console:**
1. Mở: [https://console.aws.amazon.com/iamv2/home#/roles](https://console.aws.amazon.com/iamv2/home#/roles)
2. Tìm `CloudWatchAgentServerRole` → Delete
3. Mở: [https://console.aws.amazon.com/iamv2/home#/instance_profiles](https://console.aws.amazon.com/iamv2/home#/instance_profiles)
4. Tìm `CloudWatchAgentInstanceProfile` → Delete

**AWS CLI:**
```powershell
# 1. Detach policy
aws iam detach-role-policy --role-name CloudWatchAgentServerRole --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# 2. Remove role from instance profile
aws iam remove-role-from-instance-profile --instance-profile-name CloudWatchAgentInstanceProfile --role-name CloudWatchAgentServerRole

# 3. Delete instance profile
aws iam delete-instance-profile --instance-profile-name CloudWatchAgentInstanceProfile

# 4. Delete role
aws iam delete-role --role-name CloudWatchAgentServerRole
```

### Bước 3: Xác nhận đã xóa hết

```powershell
# Kiểm tra EC2
aws ec2 describe-instances --instance-ids i-02ce20bce9e63ccd7
# Sẽ báo lỗi "InvalidInstanceID.NotFound"

# Kiểm tra IAM Role
aws iam get-role --role-name CloudWatchAgentServerRole
# Sẽ báo lỗi "NoSuchEntity"

# Kiểm tra Instance Profile
aws iam get-instance-profile --instance-profile-name CloudWatchAgentInstanceProfile
# Sẽ báo lỗi "NoSuchEntity"
```

---

*Evidence Checklist — CloudWatch Agent on EC2 — W9 Monitoring*
