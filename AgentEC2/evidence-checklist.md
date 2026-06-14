# CloudWatch Agent on EC2 — Evidence Checklist

## Mục đích
File này ghi lại tiến độ hoàn thành bài Hands-On.

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
