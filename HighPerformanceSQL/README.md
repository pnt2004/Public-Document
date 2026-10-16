# High Performance MySQL

Dự án tạo tài liệu học tiếng Việt từ *High Performance MySQL: Proven Strategies for Operating at Scale* của Silvia Botros và Jeremy Tinley. Trọng tâm là hiểu cơ chế và đánh đổi về hiệu năng MySQL trên toàn bộ hệ thống, từ truy vấn, schema và index đến monitoring, vận hành, replication và scaling.

## Đọc trước khi tạo tài liệu

1. Đọc README này để xác định đúng dự án và nguồn.
2. Đọc [init/DETAIL.md](init/DETAIL.md) để biết mục tiêu, định dạng đầu ra và quy trình học cụ thể.
3. Đọc [PDF gốc](init/high-performance-mysql-4th.pdf) để xác minh và dẫn nguồn cho từng nội dung.

PDF hiện có trong `init/` là **Fourth Edition** của *High Performance MySQL*, do **Silvia Botros và Jeremy Tinley** viết. Trang nhan đề và trang bản quyền của PDF xác nhận ấn bản này; mục lục có 13 chương. Tên *High-Performance SQL* của Markus Winand trong bản prompt ban đầu là một cuốn khác và không phải nguồn của dự án này. MySQL là hệ quản trị mặc định cho ví dụ và thực hành của dự án.

## Kết quả dự kiến

Tài liệu được tạo theo từng giai đoạn trong `notes/`: thông tin nguồn và mục lục, learning roadmap, glossary, concept map, ghi chú theo chương, bài thực hành phù hợp và phần giải thích đáp án riêng. Cấu trúc và tiêu chuẩn nội dung nằm trong [init/DETAIL.md](init/DETAIL.md).

**Trạng thái:** Đã có PDF và prompt được chuẩn hóa; chưa tạo tài liệu học. Lần đầu dùng `$read-doc-pdf` cho dự án này sẽ xác minh mục lục và tạo roadmap trước khi viết chương 1.
