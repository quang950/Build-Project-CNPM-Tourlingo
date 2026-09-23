//🧠 Luật cho AI
# AGENT.md — MultiGuide (Thuyết minh tự động đa ngôn ngữ)

Đây là file context cho AI coding agent (Claude Code, Cursor, v.v.) khi làm việc trên repo này. Đọc file này trước khi thực hiện bất kỳ task nào để hiểu đúng kiến trúc, convention và mục tiêu dự án.

## 1. Tổng quan dự án

- **Tên đề tài**: Thuyết minh tự động đa ngôn ngữ (đề tài môn Công nghệ phần mềm, lựa chọn số 3: BE Dev tạo các services giao tiếp với nhau).
- **Use case**: Khách du lịch quét mã QR tại 1 khu du lịch → mở app → chọn ngôn ngữ → thanh toán 2 USD → bật GPS → khi đến gần 1 địa điểm, app tự nhận diện và phát audio thuyết minh theo ngôn ngữ đã chọn.
- **Phạm vi**: Demo trên 1 khu du lịch mẫu, hỗ trợ vài ngôn ngữ chính (Việt/Anh/Nhật). Chỉ tập trung phần BE — FE là React Native + React.js do thành viên khác phụ trách (nếu có).
- **Team**: Nhóm 2-3 sinh viên, giao tiếp qua GitHub. Người viết context này (Quang) phụ trách BE Dev.
- **Deadline**: Tuần 7 — kiểm tra tiến độ giữa kỳ. Tuần 11-12 — nộp báo cáo cuối kỳ.

## 2. Kiến trúc hệ thống

Hệ thống chia thành **4 service độc lập**, mỗi service tự chạy riêng và giao tiếp với nhau qua REST API (không dùng message queue — quá phức tạp so với thời gian đồ án):

| Service | Nhiệm vụ | Phụ thuộc |
|---|---|---|
| `auth-service` | Sinh session token sau khi quét QR | Không |
| `payment-service` | Xử lý (mock) thanh toán 2 USD | Không |
| `location-service` | Nhận GPS, xác định địa điểm gần nhất | Gọi sang `content-service` |
| `content-service` | Trả nội dung + link audio theo địa điểm & ngôn ngữ | Không |

**Nguyên tắc quan trọng khi code**: mỗi service là 1 đơn vị độc lập, không được import trực tiếp code giữa các service — mọi giao tiếp phải qua HTTP call (`axios`/`fetch`), không share database giữa các service.

## 3. Kiến trúc bên trong mỗi service (layer)

Mỗi service dùng NestJS và tổ chức theo pattern **Controller → Service → Repository**:

- **Controller**: chỉ nhận request, validate input bằng DTO (`class-validator`), gọi xuống Service, trả response. Không chứa business logic.
- **Service**: chứa toàn bộ logic nghiệp vụ (VD: tính khoảng cách GPS bằng công thức Haversine, kiểm tra điều kiện thanh toán).
- **Repository**: chỉ giao tiếp database qua TypeORM/Prisma, không chứa logic nghiệp vụ.

Không được để Controller gọi thẳng Repository, và không được để Repository chứa logic tính toán.

## 4. Tech stack

- **Framework**: NestJS (TypeScript)
- **Database**: PostgreSQL (mỗi service có database/schema riêng, không share)
- **ORM**: TypeORM hoặc Prisma (chọn 1, thống nhất dùng chung cho cả 4 service)
- **Containerize**: Docker + Docker Compose (chạy toàn bộ 4 service + DB bằng 1 lệnh)
- **CI/CD**: GitHub Actions (lint + test tự động khi push)
- **API docs**: Swagger (`@nestjs/swagger`)
- **Repo structure**: monorepo — 1 repo GitHub chứa 4 folder service riêng biệt

```
/multiguide
  /auth-service
  /payment-service
  /location-service
  /content-service
  docker-compose.yml
  README.md
```

## 5. API contract (draft — cập nhật khi code thật)

- `POST /auth/session` → input: mã QR → output: session token
- `POST /payment/checkout` → input: session token → output: `{ success: boolean, transactionId }`
- `POST /location/check` → input: `{ lat, long }` → output: địa điểm gần nhất (nếu có) hoặc null
- `GET /content/:locationId?lang=ja` → output: `{ title, text, audioUrl }`

## 6. Database schema (draft)

- `location-service`: bảng `locations` (id, name, lat, long, radius)
- `content-service`: bảng `contents` (id, locationId, lang, title, text, audioUrl)
- `payment-service`: bảng `transactions` (id, status, createdAt)
- `auth-service`: bảng `sessions` (id, token, createdAt) — có thể dùng in-memory/Redis thay vì DB nếu đơn giản hơn

## 7. Xử lý lỗi cần có (theo flowchart đã thiết kế)

- GPS không được cấp quyền → trả lỗi rõ ràng để FE hiện thông báo yêu cầu bật GPS
- Thanh toán thất bại → cho phép retry, không cấp session hợp lệ
- Không có audio cho ngôn ngữ đã chọn → fallback về tiếng Anh hoặc trả lỗi có nghĩa (không crash)

## 8. Coding convention

- Đặt tên biến/hàm bằng tiếng Anh, comment có thể tiếng Việt nếu cần giải thích logic nghiệp vụ đặc thù.
- Mỗi service commit riêng, message rõ ràng theo format `[service-name] mô tả ngắn`.
- Không hard-code giá trị nhạy cảm (URL service khác, secret) — dùng biến môi trường (`.env`).

## 9. Trạng thái hiện tại / việc cần làm tiếp

Xem chi tiết lộ trình đầy đủ tại file lộ trình đồ án (6 giai đoạn, từ chuẩn bị đến nộp báo cáo). Thứ tự build service: `location-service` → `content-service` → `auth-service` → `payment-service`.