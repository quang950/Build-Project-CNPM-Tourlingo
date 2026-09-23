# Lộ trình xây dựng đồ án: Thuyết minh tự động đa ngôn ngữ (MultiGuide)

**Vai trò:** BE Dev — tạo các service độc lập giao tiếp với nhau, kèm CI/CD **Timeline:** Tuần 7 kiểm tra tiến độ giữa kỳ, tuần 11-12 nộp báo cáo cuối kỳ

---

## Giai đoạn 0 — Chuẩn bị nền tảng (Tuần 1-2)

1. **Chốt use case cụ thể** với nhóm: app MultiGuide cho 1 khu du lịch mẫu, khách quét QR → chọn ngôn ngữ → thanh toán → GPS → nghe audio.
2. **Chốt danh sách service** cần xây (đừng làm quá nhiều service, sinh viên nên giữ ở mức tối thiểu để kịp deadline):
   - `auth-service` — quản lý session sau khi quét QR
   - `payment-service` — xử lý thanh toán (có thể mock/sandbox, không cần cổng thanh toán thật)
   - `location-service` — nhận GPS, xác định điểm gần nhất
   - `content-service` — trả về thông tin + link audio theo ngôn ngữ
3. **Chọn công nghệ**: NestJS (đang học) cho từng service, PostgreSQL cho database, Docker để đóng gói mỗi service, GitHub để version control.
4. **Tạo GitHub repo** — quyết định: 1 repo chứa 4 folder (monorepo) hay 4 repo riêng (polyrepo). Với nhóm 2-3 người, **nên chọn monorepo** để dễ quản lý, tránh rối khi mới học.
5. **Chia việc trong nhóm**: ai phụ trách service nào, ai lo phần FE (nếu có), thống nhất convention đặt tên API, format response JSON chung.

## Giai đoạn 1 — Thiết kế hệ thống (Tuần 2-3)

1. Vẽ lại 3 sơ đồ flow đã làm (thanh toán, GPS, nghe thuyết minh) thành **use case diagram** chính thức cho báo cáo.
2. Thiết kế **ERD** cho từng service cần lưu dữ liệu:
   - `location-service`: bảng địa điểm (tên, lat/long, bán kính nhận diện)
   - `content-service`: bảng nội dung theo địa điểm + ngôn ngữ (địa điểm, ngôn ngữ, text, link audio)
   - `payment-service`: bảng giao dịch (mã giao dịch, trạng thái, thời gian)
3. Thiết kế **API contract** giữa các service trước khi code — mỗi service cần định nghĩa rõ endpoint, input/output. VD:
   - `POST /location/check` → input: lat, long → output: địa điểm gần nhất (nếu có)
   - `GET /content/:locationId?lang=ja` → output: text + audio URL
4. Quyết định cách các service gọi nhau: **REST API đơn giản** (dùng `fetch`/`axios` giữa các service) là đủ cho đồ án — không cần message queue (Kafka/RabbitMQ) vì sẽ quá phức tạp so với thời gian có.

## Giai đoạn 2 — Xây dựng từng service (Tuần 4-6)

Thứ tự làm nên đi từ service **đơn giản, không phụ thuộc service khác trước**, để có cái chạy được sớm:

1. **`location-service` trước** — chỉ cần nhận lat/long, so sánh với bảng địa điểm trong DB, trả về kết quả. Không phụ thuộc service nào khác.
2. **`content-service`** — nhận `locationId` + `lang`, trả về nội dung. Cũng độc lập.
3. **`auth-service`** — sinh session token đơn giản sau khi "quét QR" (có thể chỉ là 1 UUID lưu tạm, chưa cần OAuth phức tạp).
4. **`payment-service`** — mock giao dịch (trả về `success: true/false` giả lập, chưa cần tích hợp cổng thanh toán thật trừ khi giảng viên yêu cầu).

Với mỗi service, áp dụng đúng pattern **Controller → Service → Repository** đã học:

- Controller nhận request, validate input cơ bản (dùng DTO + class-validator của NestJS).
- Service xử lý logic (VD: tính khoảng cách GPS bằng công thức Haversine).
- Repository dùng TypeORM/Prisma để query database.

## Giai đoạn 3 — Kết nối các service + CI/CD (Tuần 6-7) → chuẩn bị kiểm tra giữa kỳ

1. Viết đoạn code trong `location-service` gọi sang `content-service` khi xác định được vị trí (HTTP call giữa 2 service).
2. Test luồng end-to-end thủ công bằng Postman: giả lập gửi GPS → nhận về nội dung + audio.
3. Setup **Docker Compose** để chạy cả 4 service + database cùng lúc bằng 1 lệnh (`docker-compose up`) — giúp demo dễ hơn nhiều.
4. Setup **CI/CD cơ bản** bằng GitHub Actions: mỗi lần push code, tự động chạy lint + test (chưa cần deploy lên server thật, chỉ cần chứng minh có pipeline).
5. **Chuẩn bị báo cáo tiến độ giữa kỳ**: demo được ít nhất 1-2 service chạy độc lập + gọi nhau thành công.

## Giai đoạn 4 — Hoàn thiện tính năng + Testing (Tuần 8-10)

1. Hoàn thiện `auth-service` và `payment-service`, nối toàn bộ 4 service thành 1 luồng hoàn chỉnh đúng theo flow đã vẽ.
2. Viết **unit test** cơ bản cho phần logic quan trọng nhất (VD: hàm tính khoảng cách GPS, hàm kiểm tra thanh toán).
3. Xử lý các trường hợp lỗi (error handling) đã thiết kế trong flowchart: GPS không có quyền, thanh toán thất bại, không có audio cho ngôn ngữ đã chọn.
4. Nếu có FE (React Native/React.js), phối hợp để FE gọi được đúng API đã thiết kế.
5. Viết tài liệu API (Swagger — NestJS hỗ trợ sẵn `@nestjs/swagger`) để dễ báo cáo và để FE dùng.

## Giai đoạn 5 — Tích hợp, demo, viết báo cáo (Tuần 11-12) → nộp cuối kỳ

1. Test lại toàn bộ luồng nhiều lần, đảm bảo demo mượt (chuẩn bị sẵn data mẫu: vài địa điểm, vài ngôn ngữ).
2. Chụp lại kiến trúc hệ thống (sơ đồ service, ERD, flowchart) đưa vào báo cáo.
3. Viết báo cáo theo cấu trúc chuẩn: Lý do chọn đề tài → Mục tiêu → Đối tượng & phạm vi (đã làm) → Cơ sở lý thuyết (microservices, Controller-Service-Repository) → Thiết kế hệ thống → Triển khai → Kết quả → Kết luận.
4. Quay video demo hoặc chuẩn bị demo trực tiếp (phòng khi máy chiếu/mạng lỗi lúc báo cáo).
5. Chuẩn bị trả lời câu hỏi của giảng viên về lý do chọn kiến trúc microservices thay vì 3 lớp thông thường (đề tài số 2).

---

### Lưu ý quan trọng

- Đừng cố làm quá nhiều service hay tính năng phức tạp (message queue, service discovery, k8s...) — với thời gian 1 học kỳ và đang mới học NestJS, **chạy đúng, chạy ổn định 4 service đơn giản** đã đủ điểm và đủ để chứng minh hiểu kiến trúc.
- Việc học NestJS/Node.js hiện tại nên ưu tiên đủ nhanh để bắt kịp Giai đoạn 2 (Tuần 4-6) — nếu học chậm hơn dự kiến, có thể lùi bớt tính năng phụ (VD: bỏ payment-service thật, chỉ mock).
