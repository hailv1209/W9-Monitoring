# Asset folder — Screenshots

Folder này chứa toàn bộ screenshot cho bài **"Alert on AWS Root Account Login"**. Đặt file đúng tên đã quy ước bên dưới để `evidence-checklist.md` hiển thị ảnh.

> Khi chưa có ảnh, các link ảnh trong `evidence-checklist.md` sẽ bị broken — điều này là bình thường. Cứ chụp và lưu đúng tên file theo danh sách bên dưới.

## Danh sách screenshot cần chụp

| File | Phần | Mô tả |
|------|------|--------|
| `1a.png` | 1.1 | Trang Create trail — điền tên `RootAlarmTrail`, tick "Apply trail to all regions" |
| `1b.png` | 1.2 | Mục CloudWatch Logs đã bật, log group `RootAlarmCloudTrailLogGroup` được chọn, IAM Role được tạo |
| `1c.png` | 1.2 | Trail `RootAlarmTrail` hiển thị ở trang Trails list với status **Logging: ON** |
| `2a.png` | 2.1 | Trang Define pattern — Filter pattern đã điền đúng |
| `2b.png` | 2.1 | Trang Assign metric — namespace `Security`, metric name `RootAccountLoginCount`, value `1` |
| `2c.png` | 2.1 | Trang Review and create — filter `RootAccountLoginFilter` đã được tạo |
| `3a.png` | 3.1 | Trang SNS Topic — ARN `arn:aws:sns:us-east-1:593777010472:RootAlarmTopic` |
| `3b.png` | 3.1 | Subscription list — Email `haileab542@gmail.com`, Status: `Pending confirmation` |
| `3c.png` | 3.1 | Subscription list — Email `haileab542@gmail.com`, Status: `Confirmed` |
| `4a.png` | 3.2 | Trang Specify metric — namespace `Security`, metric `RootAccountLoginCount`, statistic `Sum` |
| `4b.png` | 3.2 | Trang Define alarm conditions — Threshold ≥ 1, Period 5 minutes |
| `4c.png` | 3.2 | Trang Configure actions — Alarm state trigger `In alarm`, SNS topic `RootAlarmTopic` |
| `4d.png` | 3.2 | Trang Review and create — Alarm `RootAlarm-RootLogin` đã được tạo |
| `4e.png` | 3.2 | CloudWatch Alarms list — alarm `RootAlarm-RootLogin` hiển thị với đầy đủ cấu hình |
| `5a.png` | 4.1 | AWS Console sign-in page — chọn "Sign in as root user" |
| `5b.png` | 4.1 | Form nhập email + password của root account |
| `6a.png` | 4.2 | Log stream mới nhất trong `RootAlarmCloudTrailLogGroup` chứa event root login |
| `6b.png` | 4.2 | Nội dung JSON của event: `"userIdentity": { "type": "Root", ... }` |
| `7a.png` | 4.3 | CloudWatch Metrics — namespace `Security`, metric `RootAccountLoginCount` có datapoint ≥ 1 |
| `7b.png` | 4.3 | Chi tiết metric — Sum = 1 trong period vừa qua |
| `8a.png` | 4.4 | Alarms list — `RootAlarm-RootLogin` ở trạng thái **ALARM** (đỏ) |
| `8b.png` | 4.4 | Alarm history tab — entry "Alarm updated from OK to ALARM" |
| `8c.png` | 4.4 | Alarm detail — đồ thị metric `RootAccountLoginCount` vượt threshold 1 |
| `9a.png` | 4.5 | Inbox email từ AWS Notifications — Subject: "ALARM: RootAlarm-RootLogin" |
| `9b.png` | 4.5 | Nội dung email — Alarm reason, threshold, metric, thời gian trigger |
| `10.png` | 5.1 | Console xác nhận tất cả tài nguyên đã được xóa |

## Tổng cộng: 25 screenshots
