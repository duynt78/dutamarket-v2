# DUTAMARKET V2

Cloudflare Workers + D1 + OpenAI API + GitHub deployment. Giao diện web responsive chạy Safari. Bản V2 hỗ trợ tạo bài marketing thật, lưu D1, duyệt, lên lịch và xuất bản **Facebook Page** nếu đã cấu hình kết nối hợp lệ. Instagram/TikTok chưa có adapter xuất bản, chỉ lưu nội dung và lịch; các tác vụ không hỗ trợ sẽ báo lỗi rõ ràng.

## Triển khai bằng GitHub và Cloudflare Dashboard (không cần máy tính)

1. Tạo repository **private** `dutamarket-v2` trên GitHub. Tải toàn bộ các file trong gói lên repository, giữ nguyên cấu trúc thư mục.
2. Cloudflare → **Workers & Pages** → **D1** → Create database `dutamarket-db`. Sao chép **Database ID** và thay `REPLACE_WITH_D1_DATABASE_ID` trong `wrangler.jsonc`.
3. Mở D1 database → Console → dán toàn bộ `schema.sql` → Execute để tạo bảng. Chỉ chạy schema này trên database DUTAMARKET mới, không dùng database dự án khác.
4. Workers & Pages → Create Worker → Import repository GitHub `dutamarket-v2` (nếu tính năng Git integration khả dụng trên tài khoản). Đặt lệnh deploy `npx wrangler deploy`; cài đặt `npm install` nếu được yêu cầu. Cloudflare Workers Builds có thể tự triển khai khi push vào GitHub.
5. Worker → Settings → Variables and Secrets → thêm **Secrets**: `ADMIN_TOKEN` (chuỗi bí mật dài ngẫu nhiên), `OPENAI_API_KEY` (khóa API riêng). Tùy chọn `OPENAI_MODEL` (mặc định gpt-4o-mini). Không lưu secrets trong GitHub hoặc HTML.
6. Kiểm tra Worker có binding `DB` trỏ tới D1 `dutamarket-db`, và Cron `*/5 * * * *`. Mở `https://<worker>.workers.dev/api/health` phải trả `ok:true` và `database:true`.
7. Mở trang chủ Worker, nhập ADMIN_TOKEN, tạo bài bằng AI, lưu, duyệt, lên lịch. **Lưu ý:** API key AI phải có quota/billing; không cam kết miễn phí.

## Facebook Page (tùy chọn, chỉ sau khi được Meta cấp quyền)

Cần Meta App và Page access token có quyền đăng nội dung Page (ví dụ `pages_manage_posts` và các quyền liên quan theo phiên bản API). Không dùng user token tùy tiện. V2 dùng Graph API `POST /{page-id}/feed` cho bài dạng văn bản; không hỗ trợ media. Trước khi sử dụng thực tế, bổ sung OAuth callback và lưu token được mã hóa: bản V2 **chưa cung cấp giao diện OAuth** và không có endpoint để tạo connection. Bảng `connections` là cấu trúc dành cho giai đoạn tích hợp tiếp theo, **không lưu token thô trong D1 ở production**. Không tự nhập token vào bảng này khi chưa bổ sung mã hóa và vòng đời token.

## An toàn và giới hạn

- Đây là MVP quản trị một người dùng, xác thực bằng `ADMIN_TOKEN`, chưa phải hệ thống đăng nhập nhiều tài khoản. Token nhập trong web được lưu ở sessionStorage; chỉ dùng trên thiết bị tin cậy, không chia sẻ.
- Chưa có mã hóa token mạng xã hội, OAuth, rate limiting, audit bảo mật, retry idempotent, upload media hoặc analytics thực tế. **Chưa nên bật đăng bài tự động production** cho đến khi bổ sung các phần này.
- Cron chạy mỗi 5 phút, chỉ xử lý tối đa 10 bài/lượt; trạng thái `publishing` có thể cần khôi phục thủ công nếu Worker bị dừng giữa chừng. Facebook có thể đã đăng bài trước khi Worker ghi kết quả, do đó không tự động retry khi không rõ kết quả.
- Cấu hình `wrangler.jsonc` dùng Wrangler 4, Cloudflare Git Build. Không trộn với dự án dùng `cf` CLI.

## API

`GET /api/health` public. Các API còn lại yêu cầu `Authorization: Bearer ADMIN_TOKEN`.

`POST /api/ai/generate`, `GET/POST /api/posts`, `POST /api/posts/:id/approve`, `POST /api/posts/:id/schedule`, `GET/POST /api/brands`, `GET /api/logs`, `POST /api/publish/run`.
