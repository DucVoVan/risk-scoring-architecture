# Đề xuất kiến trúc hệ thống Risk Scoring gần realtime

Tác giả: Võ Văn Đức  
Ngày: 22/05/2026

# 1. Mục tiêu hệ thống

Xây dựng một hệ thống chấm điểm rủi ro người dùng theo thời gian gần realtime, có khả năng:

- Nhận và lưu toàn bộ user events
- Tính toán risk score liên tục theo hành vi người dùng
- Phát hiện user nguy hiểm trong thời gian ngắn
- Cảnh báo admin gần realtime
- Cho phép thay đổi rule mà không cần deploy code
- Có lịch sử dữ liệu để audit và phân tích sau này

---

# 2. System Architecture Proposal

```text
Existing Platform
      |
      | takingEvent(event)
      v
Risk Ingestion API
      |
      +--> Lưu raw event vào PostgreSQL (events/audit log)
      |
      +--> Đẩy job vào BullMQ + Redis
                        |
                        v
                Risk Evaluation Workers
                        |
                        +--> Load recent events
                        +--> Load active rules
                        +--> Tính risk score
                        +--> Update user_risk_scores
                        +--> Insert risk_score_history
                        +--> Tạo risk_alerts nếu cần
      |
      +--> Đồng bộ historical events sang BigQuery
```

---

# 3. Các bảng dữ liệu chính

## events

Lưu raw user events như một audit log bất biến.

## risk_rules

Lưu rules tính điểm risk.

## user_risk_scores

Lưu điểm risk hiện tại của user.

## risk_alerts

Lưu các cảnh báo đã tạo.

## risk_score_history

Lưu lịch sử thay đổi score theo thời gian.

---

# 4. Ý tưởng chính

## 4.1 Event-driven processing

Mỗi hành động của user trong hệ thống sẽ được xem là một event.

Ví dụ:

- failed_card_attempt
- login_from_new_device
- otp_failed
- withdraw_requested
- transfer_created
- password_changed

Khi user thực hiện bất kỳ thao tác nào (A, B, C, D...), hệ thống hiện tại sẽ gọi:

```text
takingEvent(event)
```

Ngay khi event được nhận:

- event sẽ được validate
- lưu ngay vào PostgreSQL
- sau đó đẩy vào queue để worker xử lý risk scoring nền

Việc lưu event ngay khi xảy ra giúp hệ thống luôn có đầy đủ lịch sử hành vi người dùng để:

- tính risk score realtime
- kiểm tra các pattern bất thường
- audit và investigation
- replay scoring khi rules thay đổi
- phân tích dữ liệu dài hạn

Thay vì xử lý theo batch mỗi giờ như hiện tại, hệ thống sẽ xử lý gần như ngay sau khi event xảy ra.

Điều này giúp phát hiện sớm các hành vi bất thường và giảm đáng kể độ trễ trong việc detect risky users.

---

## 4.2 takingEvent(event) phải nhẹ

Function `takingEvent(event)` không nên tính risk trực tiếp.

Nó chỉ nên:

- validate event
- lưu raw event vào bảng events
- đẩy job vào queue
- trả response ngay

Việc tính risk sẽ được worker xử lý nền.

Cách này giúp:

- API phản hồi nhanh
- không block request user
- giảm tải cho main application
- dễ scale worker độc lập

---

## 4.3 Worker xử lý risk nền

BullMQ + Redis được dùng làm tầng queue xử lý nền.

Redis lưu trạng thái jobs, còn BullMQ giúp quản lý queue, retry khi lỗi và phân phối jobs cho workers.

Cách này giúp hệ thống không cần tính risk trực tiếp trong API nhưng vẫn có thể xử lý gần realtime.

Worker sẽ liên tục lấy jobs từ queue để:

- đọc lịch sử event gần đây của user
- áp dụng các rules đang active
- tính risk score mới
- Để tránh phải tính lại toàn bộ lịch sử events mỗi lần user có hành vi mới, worker nên ưu tiên cơ chế cập nhật điểm theo hướng incremental scoring.

  Ví dụ:

  user có thêm một failed_card_attempt
  => chỉ cộng thêm phần score tương ứng thay vì tính lại toàn bộ history từ đầu.

Cách này giúp giảm đáng kể chi phí xử lý khi event volume tăng cao.

- cập nhật bảng user_risk_scores
- ghi lịch sử vào risk_score_history
- tạo alert nếu vượt ngưỡng

Ví dụ:

```text
6 lần failed_card_attempt trong 10 phút
=> +40 điểm risk
```

---

## 4.4 Risk score dạng liên tục

Thay vì chỉ dùng `low/high`, hệ thống sẽ dùng score dạng số:

| Score  | Level  |
| ------ | ------ |
| 0-30   | Low    |
| 31-70  | Medium |
| 71-100 | High   |

Điều này giúp đánh giá hành vi mềm dẻo và chính xác hơn.

---

## 4.5 Rule có thể thay đổi mà không cần sửa code

Rules sẽ được lưu trong database thay vì hard-code.

Ví dụ:

```text
Event: failed_card_attempt
Window: 10 phút
Threshold: >= 6 lần
Risk impact: +40
```

Ops/compliance team có thể:

- thêm rule
- chỉnh threshold
- chỉnh score impact
- bật/tắt rule

mà không cần developer deploy lại hệ thống.

Điều này giúp hệ thống dễ dàng thích nghi với các thay đổi về business rules và các kiểu gian lận mới phát sinh theo thời gian.

---

## 4.6 Lưu toàn bộ event history

Toàn bộ events sẽ được lưu lại để phục vụ:

- audit
- fraud investigation
- trend analysis
- replay risk scoring
- future ML training

Tuy nhiên, hệ thống sẽ không lưu event trực tiếp vào BigQuery để phục vụ realtime scoring.

Lý do là vì BigQuery phù hợp cho phân tích dữ liệu lớn theo dạng batch analytics, nhưng không tối ưu cho các tác vụ realtime operational processing.

Do đó, event sẽ được lưu trước vào PostgreSQL để phục vụ:

- realtime scoring
- worker processing
- recent event queries
- alert generation

PostgreSQL chỉ nên giữ dữ liệu “hot” gần đây phục vụ realtime processing, ví dụ:

- 7 ngày
- 14 ngày
- hoặc 30 ngày gần nhất

Trước khi xoá các events cũ khỏi PostgreSQL, hệ thống sẽ đồng bộ dữ liệu sang BigQuery để phục vụ analytics và audit dài hạn.

---

# 5. Reasoning / Trade-offs chính

## PostgreSQL thay vì BigQuery cho realtime scoring

### Reasoning

Worker cần query rất nhanh theo user và khoảng thời gian ngắn gần đây.

### Trade-off

PostgreSQL không phù hợp lưu historical data cực lớn lâu dài.

Giải pháp:

- chỉ giữ hot data ngắn hạn trong PostgreSQL
- đồng bộ historical data sang BigQuery

---

## BullMQ + Redis thay vì tính risk trực tiếp trong API

### Reasoning

Không muốn request user phải chờ risk scoring chạy xong.

API chỉ nên:

- nhận event
- lưu event
- enqueue job
- trả response ngay

### Trade-off

Tôi chấp nhận thêm Redis và queue workers để đổi lại việc API phản hồi nhanh hơn và không phải xử lý risk scoring trực tiếp trong request của user.

Đổi lại:

- API phản hồi nhanh hơn
- retry được khi worker lỗi
- scale worker độc lập
- xử lý near realtime ổn định hơn

---

# 6. Kết luận

Thiết kế này ưu tiên giải quyết đúng vấn đề hiện tại của hệ thống:

- detect risky behaviour đủ nhanh
- không làm chậm main application
- dễ thay đổi business rules
- vẫn giữ được khả năng audit và phân tích dữ liệu lâu dài

Đồng thời tránh over-engineering cho phiên bản đầu tiên nhưng vẫn đủ khả năng mở rộng khi event volume tăng trong tương lai.
