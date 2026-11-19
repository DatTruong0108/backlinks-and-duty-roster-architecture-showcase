# Tài liệu Thiết kế đã được Làm sạch — Nền tảng Theo dõi Backlink & Lịch Trực ban

Tài liệu về kiến trúc, mô hình dữ liệu và thuật toán cho một nền tảng nội bộ cấp production, được viết lại để có thể chia sẻ ra ngoài công ty. Tất cả sơ đồ đều dùng [Mermaid](https://mermaid.js.org/) và hiển thị chuẩn (native) trên GitHub, GitLab, VS Code (qua tiện ích Markdown Preview Mermaid Support) và hầu hết các trình xem Markdown khác.

## Hệ thống này là gì

Một monorepo TypeScript, một ứng dụng Next.js, một dự án Supabase (Postgres), với hai module nghiệp vụ:

| Module | Mục đích | Hướng đi của Dữ liệu |
|---|---|---|
| **Theo dõi Backlink** | Đọc các Google Sheets được thao tác bằng tay (as-is) định kỳ hàng ngày, xác thực, tính toán lại các số liệu dashboard và lưu trữ chúng ở dạng snapshot. | Sheets → DB → UI (chỉ đọc) |
| **Lịch Trực ban ("duty")** | Lịch luân phiên 24/7 thời gian thực cho một đội vận hành ở 3 khu vực: đổi ca, checklist trực tiếp, lịch sử sự cố, audit log. | Người dùng ghi dữ liệu qua các hàm DB được bảo vệ, với chế độ refetch thời gian thực |

## Tài liệu

| # | Tài liệu | Nội dung bao gồm |
|---|---|---|
| 1 | [Kiến trúc Hệ thống](01-system-architecture.md) | Monorepo 4 package, cấu trúc luồng chạy, ranh giới tin cậy, luồng write (ghi), các quyết định thiết kế chính yếu và giải pháp thay thế bị loại bỏ |
| 2 | [ERD — Mô hình dữ liệu](02-erd.md) | Toàn bộ 23 bảng Postgres dưới dạng 3 biểu đồ ER, từ điển bảng, các bất biến ở cấp độ schema |
| 3 | [Biểu đồ Tuần tự](03-sequence-diagrams.md) | Luồng rắc rối nhất (đổi ca: lũy đẳng + giờ server ngay trong transaction + khóa hàng), đồng bộ định kỳ (chống trùng lặp pg_cron), luồng làm mới qua thời gian thực, cơ chế lưu trữ log fail-closed |
| 4 | [Các đoạn mã & Mã giả](04-code-snippets.md) | Bộ phân giải bảng tính bằng header-anchor, RLS từ chối mặc định, hợp đồng hàm ghi lũy đẳng, quyết định khóa khung giờ, hàm tính giờ server, thuật toán luân phiên, mutex quá trình chạy, bộ lưu trữ fail-closed |

## Sự thật đã kiểm chứng (đếm từ kho mã nguồn)

| Hạng mục | Giá trị |
|---|---|
| Package trong Workspace | 4 (`core`, `sync`, `duty`, `web`) |
| Bảng cơ sở dữ liệu (Postgres) | 23 (12 backlink + 1 bảng truy cập + 10 trực ban) |
| SQL migrations (lịch sử migration) | 29 |
| Test files (file bài kiểm tra) | 282 (`*.test.ts` / `*.test.tsx`) |
| Lịch trình Pipelines | 2 workflow bằng GitHub Actions (đồng bộ, bảo trì) + 1 job dùng `pg_cron` |

## Chính sách làm sạch tài liệu (Sanitization)

Những gì đã bị xóa đi hoặc tổng quát hóa, để người đọc đánh giá xem đâu là thật và đâu là giả định:

| Loại thông tin cũ | Được thay bằng |
|---|---|
| Tên công ty, tên đội và tên sản phẩm, URL kho mã | "công ty", "đội vận hành", "tên module chung chung" |
| Tên thật của các site/domain, domain của đối thủ | `site-a.example`, `site-b.example`, `competitor.example` |
| Tên nhân viên, địa chỉ e-mail, ID tài khoản | `staff_1 … staff_5`, `user@company.example` |
| ID Google Sheet, project ref từ Supabase, tokens, keys | `<SHEET_ID>`, `<PROJECT_REF>`, `<GITHUB_TOKEN>` (tuyệt đối không để giá trị thực) |
| Cụm từ về hoạt động kinh doanh hoặc khu vực | "live-broadcast operations", các vùng `R1 / R2 / R3` |
| Con số nghiệp vụ (Mục tiêu hàng ngày, ngày tháng thật) | Giá trị minh họa, hoặc ví dụ có sẵn |
| Mọi mã định danh, nội dung tin nhắn không phải tiếng Anh | Dịch thẳng ra tiếng Anh/tiếng Việt |

Những phần không thay đổi: tên các bảng/cột mang tính chất chung (`site`, `output`, `duty_shift`, …), các thuật toán lõi, mô hình bảo mật, và cấu trúc kỹ thuật của schema. Khung giờ khóa (`15:30–19:00`) và múi giờ UTC+7 xuất hiện dưới dạng **giá trị cấu hình minh họa** — vì ngoài đời thật chúng cũng được lưu ở một cấu hình trên hàng của DB, không nằm ở trong code.

Không tài liệu nào chứa nguyên bản file code (verbatim): tất cả đoạn mã ở tài liệu 4 đều là hình minh họa độc lập cho từng kỹ thuật.