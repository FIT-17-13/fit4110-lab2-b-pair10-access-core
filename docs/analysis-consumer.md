# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán: Pair 10 — Access Gate → Core Business
- Product: B
- Consumer service: Access Gate (B3)
- Provider service: Core Business (Nhóm 12 — B6)
- Người viết: Nhóm 10 (Access Gate) & Nhóm 12 (Core Business)
- Ngày: 2026-05-19

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `AccessCheckRequest` | Payload gửi cho Core Business khi có người quẹt thẻ tại cổng | cardId, gateId, direction, timestamp | - |
| `AllowDecision` / `DenyDecision` | Payload nhận về để biết nên mở hay đóng cổng | decision, decisionId, cardId, gateId, reasonCode | policyId (với ALLOW), expiresAt, reasonDetail |
| `AccessPolicy` | Policy chi tiết để cache local, dùng làm fallback khi mất mạng | policyId, name, scope, allowedDays, status | description, expiresAt |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| POST | `/access/check` | Bắt buộc gọi realtime mỗi khi thẻ được quẹt tại Access Gate | Quyết định ALLOW/DENY kèm reason và metadata (dưới 200ms) |
| GET | `/policies/access/{policyId}` | Khi Access Gate nhận AccessCheckResponse có policyId chưa cache, gọi để lấy chi tiết | Chi tiết policy để lưu in-memory cache |
| GET | `/health` | Chạy nền 10 giây/lần để theo dõi trạng thái Core Business | HTTP 200 OK |

---

## 3. Error case Consumer cần xử lý

Tối thiểu 5 case.

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Request sai schema (VD: format cardId sai do máy đọc lỗi) | Reject yêu cầu ngay, chớp đèn đỏ tại cổng |
| 401 | Thiếu Bearer token hoặc token hết hạn | Thử refresh token 1 lần, nếu vẫn lỗi báo admin |
| 404 | Endpoint không tồn tại | Log cảnh báo phiên bản API bất đồng bộ |
| 422 | Vi phạm rule nghiệp vụ (GateID chưa khai báo trên Core) | Đóng cổng (fail-closed), chớp đèn cảnh báo |
| 500 | Core Business sập hoặc quá tải | Fallback dùng local policy cache, nếu không có thì fail-closed |

---

## 4. Giả định bổ sung

- Giả định 1: Access Gate không cần gửi toàn bộ dữ liệu thẻ, chỉ gửi `cardId`. Core Business tự query thông tự query thông tin thẻ từ CSDL của nó.
- Giả định 2: Hướng quẹt thẻ (IN/OUT) là quan trọng để Core Business check anti-passback.
- Giả định 3: Nếu Core Business phản hồi chậm > 500ms, Consumer sẽ abort request và tự dùng fallback để tránh kẹt người.

---

## 5. Câu hỏi cho Provider

1. Discriminator cho `AccessCheckResponse` là trường nào? (VD: dùng `decision: "ALLOW" | "DENY"`)
2. Rate limit cho `/access/check` là bao nhiêu? Cổng vào giờ cao điểm có thể đẩy 100 req/s.
3. Có cần retry request nếu bị lỗi mạng (ví dụ HTTP 502/504) không?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider đổi kiểu format thời gian | Parse lỗi | Dùng chuẩn ISO 8601 (RFC 3339) `date-time` |
| Bất đồng bộ UUID (VD Gate tự gen một ID, Core trả ID khác) | Khó trace log | Core Business sinh `decisionId` và trả về |
| Kẹt mạng giờ cao điểm | Timeout | Giữ SLA < 200ms, Consumer set timeout HTTP client là 500ms |
