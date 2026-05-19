# Versioning — Core Business Service API

## Phiên bản hiện tại

| Phiên bản | Ngày | Trạng thái | Ghi chú |
|---|---|---|---|
| v1.0.0 | 2026-05-19 | Signed-off | Hợp đồng API đầu tiên giữa Access Gate (B3) và Core Business (B6) |

## Quy tắc versioning

Dùng **Semantic Versioning** (SemVer):

- **MAJOR** (x.0.0): Breaking change — thay đổi không tương thích ngược (VD: xóa field bắt buộc, đổi path)
- **MINOR** (1.x.0): Feature mới tương thích ngược (VD: thêm endpoint, thêm field tùy chọn)
- **PATCH** (1.0.x): Sửa lỗi, cập nhật description, không ảnh hưởng logic

## Lịch sử thay đổi

### v1.0.0 — 2026-05-19

- Khởi tạo hợp đồng API cho pair-10 (Access Gate → Core Business)
- Endpoints: `/health`, `/access/check`, `/policies/access/{policyId}`, `/decisions/{decisionId}`, `/decisions`
- Schema: AccessCheckRequest, AccessCheckResponse (oneOf: AllowDecision | DenyDecision), AccessPolicy, DecisionRecord, DecisionPage
- Error model: Problem Details (RFC 9457) cho tất cả 4xx/5xx
- Đàm phán 7 issue với Access Gate, đã sign-off
