# CLAUDE.md — BuildUp Marketplace

File này giúp Claude Code hiểu dự án, người dùng và cách làm việc. Đọc kỹ trước khi làm bất cứ việc gì.

---

## 1. Mục tiêu dự án và người dùng

- **Dự án:** BuildUp Marketplace — website thương mại điện tử dạng **marketplace nhiều seller** với 3 vai trò: **Customer, Seller, Admin**.
- **Mục đích:** người dùng **học cách xây dựng một hệ thống hoàn chỉnh** (frontend, backend, database, authentication, authorization, luồng dữ liệu) và đưa dự án vào **CV như một điểm sáng**.
- **Người dùng:** đang học, dành khoảng **1–2 giờ mỗi ngày**, làm **một task mỗi ngày** theo Scrum. Dự án dự kiến kéo dài khoảng 10–11 tuần (từ 1/11/2026, xong khoảng giữa tháng 1/2027).
- **Ngôn ngữ giao tiếp:** tiếng Việt. Giữ nguyên thuật ngữ kỹ thuật tiếng Anh (controller, guard, migration...). Code, tên biến, commit message viết tiếng Anh.

## 2. Chế độ làm việc (QUAN TRỌNG)

Mục tiêu là người dùng **hiểu và tự giải thích được** hệ thống khi phỏng vấn, không chỉ có code chạy được.

1. **Giải thích trước khi viết.** Với mỗi khái niệm mới (Guard, DI, transaction, JWT...), giải thích ngắn gọn "là gì, vì sao dùng, đánh đổi gì" rồi mới viết code.
2. **Không tự làm hết task thay người dùng.** Mặc định hãy hướng dẫn từng bước và để người dùng tự gõ. Chỉ viết toàn bộ code khi người dùng yêu cầu rõ ("viết giúp tôi", "làm luôn").
3. **Khi viết code, chú thích lý do** ở những chỗ quan trọng (vì sao chọn cách này), không chú thích những điều hiển nhiên.
4. **Chạy thử và đưa bằng chứng.** Sau khi thay đổi, chạy lệnh kiểm tra (build, test, curl) và nói kết quả thật. Không khẳng định "chạy được" khi chưa chạy.
5. **Đề xuất câu hỏi phỏng vấn.** Cuối mỗi task, đưa 1–2 câu hỏi để người dùng tự kiểm tra hiểu biết (trong mỗi task Jira đã có mục "Câu hỏi tự kiểm tra").
6. **Nói thẳng khi có vấn đề.** Nếu cách người dùng chọn có rủi ro (bảo mật, hiệu năng, thiết kế), nêu rõ và đề xuất phương án tốt hơn thay vì làm theo mà không nói.
7. **Giữ phạm vi trong task hiện tại.** Không thêm tính năng ngoài task. Nếu thấy cần, đề xuất tạo task mới trên Jira.

## 3. Tech stack (đã chốt)

| Tầng | Công nghệ |
|---|---|
| Frontend | Next.js (App Router) + TypeScript + Tailwind CSS |
| Backend | NestJS + TypeScript (nền Express) |
| Database | PostgreSQL 16 + Prisma ORM |
| Cache / token / giữ hàng | Redis 7 |
| Authentication | JWT access token + refresh token xoay vòng (httpOnly cookie), tự cài đặt |
| Authorization | RBAC: `CUSTOMER`, `SELLER`, `ADMIN` bằng Guard và decorator `@Roles()` |
| Hạ tầng | Docker + Docker Compose, GitHub Actions (CI) |
| Khác (giai đoạn sau) | Swagger/OpenAPI, Cloudinary (ảnh), VNPay sandbox (thanh toán) |

Không tự ý thêm hoặc đổi công nghệ lớn. Nếu cần, hỏi người dùng trước.

## 4. Cấu trúc thư mục (dự kiến)

```
.
├── apps/
│   ├── api/        # NestJS backend
│   └── web/        # Next.js frontend
├── docs/           # ERD, decisions.md, ảnh Swagger, video demo
├── docker-compose.yml
├── .env.example    # có commit; .env thật KHÔNG commit
├── CLAUDE.md
└── README.md
```

Cấu trúc trong `apps/api/src`: mỗi tính năng là một **module** (`auth`, `users`, `products`, `orders`, ...) gồm `*.module.ts`, `*.controller.ts`, `*.service.ts`, thư mục `dto/`.

## 5. Kiến trúc và quy ước code

### Backend (NestJS)
- Luồng bắt buộc: **Controller → Service → Prisma**. Controller chỉ nhận request và trả response. Logic nghiệp vụ nằm ở Service. **Không gọi Prisma trực tiếp trong Controller.**
- Mỗi endpoint nhận dữ liệu qua **DTO** có `class-validator`. `ValidationPipe` bật toàn cục với `whitelist`, `forbidNonWhitelisted`, `transform`.
- Lỗi trả về theo một định dạng thống nhất: `{ statusCode, message, path, timestamp }`.
- Cấu hình đọc qua `@nestjs/config` từ biến môi trường, không viết cứng secret.
- Phân trang trả về `{ data: [...], meta: { page, limit, total } }`.
- API tài liệu hóa bằng Swagger tại `/docs`.

### Database
- Schema nằm ở `apps/api/prisma/schema.prisma`. Mọi thay đổi schema đi qua **migration** (`prisma migrate dev`), commit thư mục `prisma/migrations`.
- Tiền VND lưu bằng **số nguyên** (không có phần lẻ), hoặc theo quyết định ghi trong `docs/decisions.md`.
- `OrderItem` lưu **giá tại thời điểm mua** (`price_snapshot`), không đọc lại giá từ `Product`.
- Một giỏ hàng nhiều seller được tách thành **Order** (đơn tổng) và **SubOrder** (đơn con theo seller).
- Tồn kho nằm ở bảng `Inventory` riêng. Mọi thao tác trừ kho phải nằm trong **transaction** và chống bán lố khi đồng thời.
- Index quan trọng: `users.email` unique, `products(seller_id, status)`, `orders(customer_id, created_at)`, `payments.provider_txn_id` unique.

### Frontend (Next.js)
- TypeScript chặt chẽ, tránh `any`.
- Bảo vệ route theo vai trò. Xử lý token hết hạn bằng luồng refresh.
- Không lưu access token ở `localStorage`.

### Bảo mật (luôn tuân thủ)
- Mật khẩu hash bằng bcrypt hoặc argon2, không bao giờ lưu hoặc log mật khẩu thô.
- Refresh token lưu **dạng hash** trong database, có xoay vòng và thu hồi.
- Kiểm tra **quyền sở hữu tài nguyên** (ví dụ seller chỉ sửa sản phẩm của chính mình) để tránh lỗi IDOR, không chỉ kiểm tra role.
- Không commit `.env`, khóa API, secret. Không in secret ra log hoặc chat.
- Mọi dữ liệu từ client đều phải được validate ở server.

### Git
- Nhánh: `main` (ổn định), nhánh làm việc `feat/<tên>`, `fix/<tên>`, `chore/<tên>`.
- Commit theo **Conventional Commits** (`feat:`, `fix:`, `chore:`, `docs:`, `test:`).
- Mỗi task nên có một Pull Request nhỏ.
- Không `git push --force` lên `main`. Không commit thẳng secret.

## 6. Lệnh thường dùng

Chạy trong thư mục tương ứng. Cập nhật mục này khi lệnh thay đổi.

```bash
# Hạ tầng (thư mục gốc)
docker compose up -d        # chạy PostgreSQL + Redis
docker compose down         # dừng (dữ liệu giữ nhờ volume)

# Backend (apps/api)
npm run start:dev           # chạy API ở cổng 3000
npx prisma migrate dev --name <ten>   # tạo và chạy migration
npx prisma studio           # xem dữ liệu
npx prisma db seed          # nạp dữ liệu mẫu
npm run test                # unit test
npm run test:e2e            # e2e test

# Frontend (apps/web)
npx next dev -p 3001        # chạy web ở cổng 3001
```

Cổng mặc định: API `3000`, Web `3001`, PostgreSQL `5432`, Redis `6379`, Swagger `http://localhost:3000/docs`.

## 7. Quy trình Scrum và Jira

- **Jira site:** `diamondmangamer.atlassian.net`, **project key:** `SCRUM`, tất cả issue của dự án có nhãn `BuildUp`.
- **Sprint:** 1 tuần, mỗi ngày một task (khoảng 1–2 giờ, 1–3 story points). Sprint 1 bắt đầu 1/11/2026.
- **Cấu trúc backlog:** 11 Epic:
  - E1 Setup & DevOps (SCRUM-5)
  - E2 Database & Data Modeling (SCRUM-6)
  - E3 Authentication (SCRUM-7)
  - E4 Authorization / RBAC (SCRUM-8)
  - E5 Catalog: Seller & Product (SCRUM-9)
  - E6 Cart & Multi-seller Order (SCRUM-10)
  - E7 Inventory & Flash Sale (SCRUM-11)
  - E8 Payment — VNPay sandbox & Webhook (SCRUM-12)
  - E9 Admin & Dashboard (SCRUM-13)
  - E10 Frontend Next.js (SCRUM-14)
  - E11 Quality, Deploy & Portfolio (SCRUM-15)
- **Mỗi task có sẵn:** mục tiêu trong ngày, kiến thức cần học (theo topic roadmap.sh), các bước làm, acceptance criteria (Definition of Done) và câu hỏi tự kiểm tra.
- **Khi người dùng nói "hôm nay làm gì":** xem task Jira có hạn chót là hôm nay (hoặc task đầu tiên chưa Done trong sprint đang chạy) và hướng dẫn theo task đó.
- **Khi task xong:** nhắc người dùng comment 2–3 dòng "hôm nay học được gì" và chuyển task sang Done. Nếu có công cụ Jira kết nối, có thể cập nhật giúp khi người dùng đồng ý.
- **Cuối mỗi sprint:** Sprint Review (demo), Retrospective (làm tốt / chưa tốt / cải thiện), ghi velocity, rồi lập kế hoạch sprint kế tiếp.
- Việc lập kế hoạch sprint mới và tạo issue trên Jira thường do Claude trong ứng dụng chat (project BuildUp) đảm nhận. Claude Code tập trung vào **code và hướng dẫn kỹ thuật**.

## 8. Định nghĩa "Xong" (Definition of Done) chung

Một task chỉ được coi là xong khi:
- [ ] Code chạy được và đã được **chạy thử thật** (có kết quả lệnh hoặc request chứng minh)
- [ ] Đáp ứng đủ acceptance criteria trong task Jira
- [ ] Không có secret trong code hoặc commit
- [ ] Có test cho logic quan trọng (từ khi bắt đầu viết nghiệp vụ)
- [ ] README hoặc docs được cập nhật nếu có thay đổi cách chạy
- [ ] Đã commit đúng quy ước và mở Pull Request
- [ ] Người dùng giải thích được bằng lời những gì vừa làm

## 9. Điểm sáng CV — ưu tiên chất lượng

Ba phần cần làm sâu nhất, không được cắt để chạy nhanh:
1. **Kiểm soát tồn kho khi đồng thời (Epic E7):** transaction, khóa dòng hoặc optimistic locking, giữ hàng bằng Redis, có load test (ví dụ 500 request đồng thời mà tồn kho không âm) kèm số liệu.
2. **Thanh toán có webhook idempotent (Epic E8):** xác thực chữ ký, xử lý webhook gọi lặp hoặc đến trễ, tách trạng thái thanh toán khỏi đơn hàng.
3. **Authentication và RBAC chuẩn (Epic E3, E4):** refresh token rotation, chống brute force, chống IDOR.

Khi làm các phần này, luôn nhắc người dùng **ghi lại quyết định và số liệu** vào `docs/` để dùng cho CV và phỏng vấn.

## 10. Những điều KHÔNG làm

- Không tự ý đổi stack, đổi cấu trúc thư mục lớn, hoặc xóa dữ liệu, migration cũ khi chưa hỏi.
- Không chạy lệnh phá hủy (`rm -rf`, `prisma migrate reset`, `docker volume rm`, `git reset --hard`, force push) khi chưa được người dùng đồng ý rõ ràng.
- Không bỏ qua lỗi bằng cách tắt validation, tắt kiểm tra quyền hoặc xóa test.
- Không đưa mật khẩu, token, khóa API vào code, log hoặc tin nhắn.
- Không nói "xong" khi chưa kiểm chứng.