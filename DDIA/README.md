# DDIA — Designing Data-Intensive Applications

Dự án học cuốn *Designing Data-Intensive Applications* của Martin Kleppmann bằng tiếng Việt. Mục tiêu là hiểu các nguyên lý của hệ thống dữ liệu, cơ chế bên trong, nguyên nhân xuất hiện vấn đề và các đánh đổi khi thiết kế hệ thống; tài liệu không phải giáo trình lập trình theo một ngôn ngữ hoặc framework.

## Đầu vào và tình trạng

Thư mục [`init/`](init/) chứa [PDF sách gốc](init/Designing%20Data%20Intensive%20Applications%20by%20Martin%20Kleppmann.pdf) và [DETAIL.md](init/DETAIL.md), bản yêu cầu chi tiết về nội dung cần tạo. PDF đã có nhưng chưa được phân tích để xác nhận ấn bản, mục lục hoặc nội dung chương 1. Skill [`read-doc-pdf`](../.agents/skills/read-doc-pdf/SKILL.md) phải đọc README này, sau đó đọc `init/DETAIL.md` và PDF trước khi tạo ghi chú.

## Tài liệu dự kiến

Sau khi xác minh nguồn, ghi chú được tạo dần trong `DDIA/ddia-notes/`:

| Tệp | Nội dung |
| --- | --- |
| `README.md` | Ấn bản đã xác minh, mục lục và hướng dẫn đọc. |
| `glossary.md` | Thuật ngữ tiếng Anh và giải thích tiếng Việt. |
| `chapter-XX.md` | Giải thích theo từng chương, bám sát trình tự lập luận của sách. |
| `system-design-concepts.md` | Liên kết các nguyên lý System Design với chương liên quan. |

Phạm vi và cách viết cụ thể nằm trong `init/DETAIL.md`. Chỉ tạo ghi chú cho chương được yêu cầu sau khi đã đọc phần sách tương ứng; hoàn thành và kiểm tra từng chương trước khi chuyển sang chương tiếp theo.
