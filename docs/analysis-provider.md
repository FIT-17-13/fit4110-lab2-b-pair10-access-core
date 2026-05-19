# Phân tích yêu cầu — vai Provider

- Cặp đàm phán: Pair 10 — Access Gate → Core Business
- Product: B
- Provider service: Core Business (Nhóm 12 — B6)
- Consumer service: Access Gate (B3)
- Người viết: Nhóm 12
- Ngày: 2026-05-19

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `AccessCheckResponse` | Kết quả kiểm tra quyền ra/vào, dùng `oneOf` (AllowDecision / DenyDecision) | decision, decisionId, cardId, gateId, reasonCode, checkedAt | policyId (null nếu DENY), expiresAt, reasonDetail |
| `AccessPolicy` | Chi tiết policy truy cập campus | policyId, name, scope, allowedTimeStart, allowedTimeEnd, allowedDays, status, priority, createdAt | description, updatedAt, expiresAt |
| `DecisionRecord` | Bản ghi lịch sử quyết định truy cập | decisionId, decision, cardId, gateId, direction, reasonCode, checkedAt | policyId, reasonDetail |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| GET | `/health` | Kiểm tra service hoạt động | Trước khi bắt đầu xử lý, hoặc health check định kỳ |
| POST | `/access/check` | Kiểm tra policy ra/vào realtime | Mỗi khi có lượt quẹt thẻ tại cổng, cần quyết định ALLOW/DENY |
| GET | `/policies/access/{policyId}` | Tra cứu chi tiết policy | Khi Access Gate cần cache hoặc hiển thị thông tin policy |
| GET | `/decisions/{decisionId}` | Tra cứu lịch sử quyết định | Khi cần audit hoặc tra cứu quyết định cũ |
| GET | `/decisions` | Lấy danh sách quyết định (pagination) | Khi cần xem lịch sử nhiều quyết định |

---

## 3. Error case

Tối thiểu 5 case.

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload sai định dạng (cardId không đúng pattern RFID-YYYY-NNN) | `Problem` với errors chứa field vi phạm |
| 401 | Thiếu Bearer token hoặc token hết hạn | `Problem` title "Chưa xác thực" |
| 403 | Token hợp lệ nhưng service không có quyền gọi endpoint | `Problem` title "Không có quyền" |
| 404 | Policy hoặc Decision không tồn tại với ID đã cho | `Problem` title "Không tìm thấy" |
| 422 | Timestamp quẹt thẻ nằm trong tương lai, hoặc gateId không tồn tại | `Problem` title "Vi phạm quy tắc nghiệp vụ" |
| 500 | Lỗi hệ thống không mong muốn (database timeout, v.v.) | `Problem` title "Lỗi hệ thống" |

---

## 4. Giả định bổ sung

Ghi rõ những điểm user story chưa nói nhưng Provider cần giả định.

- Giả định 1: Mỗi lượt quẹt thẻ tạo một DecisionRecord mới với UUID riêng, Core Business lưu lại để audit.
- Giả định 2: Policy được đánh theo priority — khi nhiều policy match, policy có priority nhỏ nhất (ưu tiên cao nhất) được áp dụng.
- Giả định 3: Response time cho POST /access/check phải dưới 200ms để không gây kẹt cổng — nếu Core Business timeout thì Access Gate tự quyết định theo default policy.

---

## 5. Câu hỏi cho Consumer

1. Access Gate có cần Core Business trả về danh sách tất cả policy áp dụng cho card, hay chỉ cần policy match đầu tiên?
2. Khi Core Business lỗi hoặc timeout, Access Gate sẽ fail-open (mở cổng) hay fail-closed (đóng cổng)?
3. Access Gate có cần idempotencyKey cho mỗi lượt quẹt để tránh tạo decision trùng khi retry?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên field không thống nhất giữa Core và Gate | Consumer parse lỗi response | Chốt naming convention camelCase trong `openapi.yaml`, dùng pattern RFID-YYYY-NNN và GATE-NN |
| Payload lớn hoặc response chậm | Gate bị kẹt, timeout | SLA response < 200ms, chỉ trả field tối thiểu trong AccessCheckResponse |
| Policy thay đổi khi Gate đang cache | Gate dùng policy cũ, quyết định sai | Dùng expiresAt để cache có thời hạn, gate refresh khi hết hạn |
| Core Business down | Gate không thể kiểm tra policy | Thống nhất fail-open/fail-closed, Access Gate có default policy fallback |
