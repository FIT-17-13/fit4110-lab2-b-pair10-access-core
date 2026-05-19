# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: Access Gate ↔ Core Business (Pair 10)
- Product: B
- Provider: Core Business (Nhóm 12 — B6)
- Consumer: Access Gate (B3)
- Phiên: v1.0
- Ngày: 2026-05-19

---

## Issue #1 — Response time SLA cho POST /access/check

- Raised by: Consumer (Access Gate)
- Endpoint: `POST /access/check`
- Concern: Access Gate cần response rất nhanh để tránh kẹt cổng khi có nhiều người xếp hàng quẹt thẻ. Nếu Core Business trả chậm, cổng sẽ bị ùn tắc.
- Proposal: Consumer yêu cầu SLA response time dưới 200ms cho endpoint này.
- Resolution: Accepted
- Rationale: 200ms là ngưỡng hợp lý cho trải nghiệm realtime. Core Business sẽ tối ưu query policy bằng cache in-memory và chỉ trả về field tối thiểu cần thiết.
- Impact: Core Business cần thiết kế index database và cache layer cho bảng policy. Không trả về field thừa trong response để giảm payload size.

---

## Issue #2 — Fail-open hay fail-closed khi Core Business down

- Raised by: Consumer (Access Gate)
- Endpoint: `POST /access/check`
- Concern: Khi Core Business không phản hồi (timeout, 5xx), Access Gate nên mở cổng (fail-open) hay đóng cổng (fail-closed)? Đây là vấn đề an ninh quan trọng.
- Proposal: Consumer đề xuất fail-open cho nhân viên (EMPLOYEE) trong giờ hành chính, fail-closed cho VISITOR và ngoài giờ.
- Resolution: Modified
- Rationale: Thống nhất: Access Gate giữ bản cache policy gần nhất (TTL 5 phút). Khi Core timeout, Gate dùng cached policy để quyết định. Nếu không có cache, fail-closed cho tất cả. Core Business sẽ set `expiresAt` trong AllowDecision để Gate biết khi nào cache hết hạn.
- Impact: AccessCheckResponse cần trường `expiresAt` (nullable) để Access Gate cache. Core Business cần đảm bảo high availability (>99.5%).

---

## Issue #3 — Chuẩn hóa định dạng cardId và gateId

- Raised by: Provider (Core Business)
- Endpoint: `POST /access/check`, `GET /decisions`
- Concern: Hai bên có thể dùng format cardId khác nhau (VD: Core dùng UUID, Gate dùng mã RFID). Nếu không thống nhất sẽ gây lỗi parse và mapping sai.
- Proposal: Provider đề xuất dùng pattern cố định: cardId = `RFID-YYYY-NNN`, gateId = `GATE-NN`.
- Resolution: Accepted
- Rationale: Pattern cố định giúp validate tại API layer bằng JSON Schema regex, phát hiện lỗi sớm trước khi xử lý nghiệp vụ. Đủ ngắn để hiển thị trên dashboard.
- Impact: Cả hai bên cần tuân thủ pattern trong openapi.yaml. Schema dùng `pattern: '^RFID-[0-9]{4}-[0-9]{3}$'` và `'^GATE-[0-9]{2}$'`.

---

## Issue #4 — Discriminator cho AccessCheckResponse (ALLOW/DENY)

- Raised by: Provider (Core Business)
- Endpoint: `POST /access/check`
- Concern: Response có thể là ALLOW hoặc DENY với cấu trúc khác nhau (ALLOW có policyId + expiresAt, DENY có reasonDetail). Consumer cần biết cách parse chính xác.
- Proposal: Provider đề xuất dùng `oneOf` + `discriminator` trên field `decision` với mapping: `ALLOW → AllowDecision`, `DENY → DenyDecision`.
- Resolution: Accepted
- Rationale: Discriminator giúp Consumer parse response chính xác bằng code-gen, tránh phải đoán kiểu dữ liệu. Đây cũng là yêu cầu kỹ thuật của OpenAPI 3.1 trong Lab 02.
- Impact: Consumer cần handle cả 2 nhánh AllowDecision và DenyDecision. AllowDecision có `policyId` (bắt buộc), DenyDecision có `reasonDetail` (nullable).

---

## Issue #5 — Nullable fields và union type với null

- Raised by: Provider (Core Business)
- Endpoint: `POST /access/check`, `GET /policies/access/{policyId}`
- Concern: Một số field có thể không có giá trị (VD: `expiresAt` khi policy vĩnh viễn, `reasonDetail` khi lý do đã rõ từ `reasonCode`, `policyId` khi DENY vì không tìm thấy policy). Cần quy ước cách biểu diễn "không có giá trị".
- Proposal: Provider đề xuất dùng union type `type: [string, "null"]` thay vì `nullable: true` (deprecated trong OpenAPI 3.1).
- Resolution: Accepted
- Rationale: OpenAPI 3.1 dùng JSON Schema 2020-12, `nullable` đã deprecated. Union type `[string, "null"]` là cách chuẩn. Consumer cần check null trước khi parse các field này.
- Impact: Consumer cần xử lý null cho: `expiresAt`, `reasonDetail`, `policyId` (trong DenyDecision), `description` (trong AccessPolicy), `updatedAt`, `detail` và `instance` (trong Problem).

---

## Issue #6 — Error response chuẩn Problem Details (RFC 9457)

- Raised by: Provider (Core Business)
- Endpoint: Tất cả endpoints
- Concern: Consumer cần hiểu cấu trúc error response nhất quán để xử lý lỗi tự động (retry, hiển thị message, log). Nếu mỗi endpoint trả error khác nhau, Consumer rất khó maintain.
- Proposal: Provider đề xuất dùng Problem Details (RFC 9457) cho tất cả response 4xx/5xx, content-type `application/problem+json`, với schema `Problem` gồm: type, title, status, detail (nullable), instance (nullable), errors (array).
- Resolution: Accepted
- Rationale: Problem Details là chuẩn HTTP API error, giúp Consumer parse lỗi bằng một schema duy nhất. Trường `errors` array cho phép trả nhiều lỗi validation cùng lúc (VD: cả cardId và gateId đều sai format).
- Impact: Consumer implement một error handler chung cho tất cả API call tới Core Business. Provider commit dùng content-type `application/problem+json` cho mọi error response.

---

## Issue #7 — Anti-passback constraint

- Raised by: Provider (Core Business)
- Endpoint: `POST /access/check`
- Concern: Thẻ không được quẹt vào 2 lần liên tiếp mà chưa có quẹt ra. Core Business cần Consumer gửi đúng hướng quẹt thẻ.
- Proposal: Consumer phải cung cấp trường `direction` với Enum là `IN` hoặc `OUT` trong request.
- Resolution: Accepted
- Rationale: Cần field `direction` để duy trì state trong Core Business. Nếu phát hiện vi phạm, Core sẽ DENY với reasonCode `PASSBACK_VIOLATION`.
- Impact: Thêm trường `direction: IN/OUT` vào AccessCheckRequest. Thêm reasonCode `PASSBACK_VIOLATION` vào DenyDecision.

---

# Chốt hợp đồng v1.0

Provider sign-off: Nhóm 12 — Core Business (B6)
Consumer sign-off: Access Gate (B3)
Witness (GV/TA):   _________________
Date:              2026-05-19

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| oas3-unused-component | Có thể một số response chưa được dùng hết | Sẽ bổ sung endpoint trong phiên bản sau |
