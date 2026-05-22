# Phân chia công việc cho team phát triển hệ thống Risk Scoring

Tác giả: Võ Văn Đức  
Ngày: 22/05/2026

---

# Backend Team

## Task: Thiết kế cấu trúc lưu trữ events

### Mô tả / Hướng dẫn thực hiện

- Thiết kế bảng `events` đóng vai trò audit log bất biến.

### Ví dụ cấu trúc bảng `events`

| Cột         | Ý nghĩa                    |
| ----------- | -------------------------- |
| id          | ID của event               |
| user_id     | ID user                    |
| event_name  | Tên event                  |
| occurred_at | Thời điểm event xảy ra     |
| metadata    | Dữ liệu bổ sung dạng JSON  |
| created_at  | Thời gian lưu vào hệ thống |

- Tạo indexes cho:
  - user_id
  - occurred_at
  - event_name
- Tối ưu cho các truy vấn realtime gần đây theo user.
- Thiết kế retention strategy cho hot data.
- Chuẩn bị partitioning strategy nếu traffic tăng cao trong tương lai.
- Đảm bảo event history có thể replay và phục vụ audit/investigation.

---

## Task: Xây dựng API nhận events

### Mô tả / Hướng dẫn thực hiện

- Xây dựng API `takingEvent(event)` để hệ thống hiện tại có thể gửi user events vào risk engine.
- Kiểm tra tính hợp lệ của dữ liệu đầu vào trước khi xử lý.
- Lưu raw events vào PostgreSQL ngay khi nhận được request.
- Đẩy công việc xử lý risk vào hàng chờ BullMQ để worker xử lý nền.
- API phải luôn nhẹ, phản hồi nhanh và không được tính risk trực tiếp trong request lifecycle.
- Thiết kế theo hướng có thể xử lý lượng events lớn trong tương lai.

---

## Task: Xây dựng hệ thống rules tính điểm risk linh hoạt

### Mô tả / Hướng dẫn thực hiện

- Xây dựng hệ thống tính điểm risk theo hướng linh hoạt thay vì viết cố định trong source code.
- Hệ thống sẽ đọc các rules đang hoạt động trực tiếp từ database.

### Ví dụ cấu trúc bảng `risk_rules`

| Cột                 | Ý nghĩa                          |
| ------------------- | -------------------------------- |
| id                  | ID rule                          |
| event_name          | Event cần theo dõi               |
| threshold           | Số lần xảy ra                    |
| time_window_minutes | Khoảng thời gian kiểm tra        |
| score_impact        | Điểm risk cộng thêm              |
| is_active           | Rule có đang hoạt động hay không |

- Mỗi rule cần hỗ trợ:
  - số lần xảy ra hành vi
  - khoảng thời gian kiểm tra
  - số điểm risk được cộng thêm
  - bật/tắt rule

Ví dụ:

```text
user failed login 5 lần trong 10 phút
=> cộng thêm 30 điểm risk
```

- Các rules phải có thể thay đổi mà không cần deploy lại hệ thống.
- Thiết kế đủ linh hoạt để sau này có thể thêm nhiều kiểu rules và logic đánh giá khác nhau.

---

## Task: Setup BullMQ + Redis

### Mô tả / Hướng dẫn thực hiện

- Setup Redis phục vụ queue processing.
- Setup BullMQ queues và workers.
- Configure:
  - retries
  - concurrency
  - failed jobs handling
  - queue cleanup
- Thêm queue monitoring/logging.
- Đảm bảo workers có thể scale độc lập với API service.
- Đảm bảo jobs không bị mất khi worker restart/crash.

---

## Task: Xây dựng workers tính điểm risk

### Mô tả / Hướng dẫn thực hiện

- Nhận jobs từ BullMQ queue.
- Đọc recent user events từ PostgreSQL.
- Áp dụng các rules đang hoạt động để đánh giá hành vi người dùng.
- Tính toán risk scores mới.
- Đảm bảo cùng một event không bị cộng score nhiều lần nếu worker retry hoặc job chạy lại.

### Ví dụ cấu trúc bảng `user_risk_scores`

| Cột           | Ý nghĩa                     |
| ------------- | --------------------------- |
| user_id       | ID user                     |
| current_score | Điểm risk hiện tại          |
| risk_level    | Low / Medium / High         |
| updated_at    | Thời gian cập nhật gần nhất |

- Cập nhật bảng `user_risk_scores`.
- Ghi lịch sử vào `risk_score_history`.
- Tạo cảnh báo nếu score vượt threshold.
- Thiết kế workers theo hướng tránh xử lý trùng và an toàn khi retry.

---

## Task: Xây dựng hệ thống cảnh báo

### Mô tả / Hướng dẫn thực hiện

- Tạo alerts khi user vượt risk thresholds.
- Lưu alerts vào bảng `risk_alerts`.
- Chuẩn bị notification hooks cho:
  - Slack
  - Email
  - Internal dashboard notifications
- Hỗ trợ trạng thái cảnh báo:
  - open
  - investigating
  - resolved
- Đảm bảo alert flow có thể mở rộng thêm integrations trong tương lai.

---

## Task: Xây dựng pipeline đồng bộ dữ liệu lịch sử

### Mô tả / Hướng dẫn thực hiện

- Đồng bộ old events từ PostgreSQL sang BigQuery theo schedule.
- Cleanup hot data sau khi sync thành công.
- Đảm bảo không mất dữ liệu trong quá trình archival.
- Thiết kế pipeline theo hướng an toàn khi retry.
- Chuẩn bị support analytics/reporting workloads dài hạn.

---

# Frontend Team

## Task: Xây dựng dashboard theo dõi risk realtime

### Mô tả / Hướng dẫn thực hiện

- Xây dựng dashboard theo dõi risky users gần realtime.
- Hiển thị:
  - current risk scores
  - active alerts
  - recent risky activities
- Hỗ trợ cập nhật dữ liệu liên tục.
- Tối ưu trải nghiệm cho monitoring và vận hành hệ thống.

---

## Task: Xây dựng trang chi tiết risk của user

### Mô tả / Hướng dẫn thực hiện

- Hiển thị current risk score của user.
- Hiển thị recent events.
- Hiển thị score history timeline.
- Hiển thị các rules/reasons khiến score thay đổi.
- Tối ưu để phục vụ fraud investigation và audit usage.

---

## Task: Xây dựng giao diện quản lý rules

### Mô tả / Hướng dẫn thực hiện

- Xây dựng giao diện CRUD cho risk rules.
- Hỗ trợ:
  - bật/tắt rules
  - chỉnh thresholds
  - chỉnh score impacts
  - chỉnh time windows
- Giao diện cần dễ sử dụng cho ops/compliance team.
- Không yêu cầu technical knowledge để chỉnh rules.

---

## Task: Xây dựng giao diện quản lý alerts

### Mô tả / Hướng dẫn thực hiện

- Hiển thị danh sách alerts.
- Hỗ trợ:
  - resolve alerts
  - add investigation notes
  - filter/search alerts
  - xem lịch sử alerts
- Tối ưu flow cho operational investigation.

---

# Ghi chú

Assume toàn bộ developers là senior-level engineers.

Các tasks được thiết kế theo hướng:

- ownership rõ ràng
- dễ parallel development
- tránh coupling không cần thiết giữa các components
- dễ scale trong tương lai
- ưu tiên operational simplicity cho phiên bản đầu tiên
