# Public Document

Kho tài liệu học bằng tiếng Việt từ sách và tài liệu kỹ thuật do người dùng cung cấp. Mỗi chủ đề hoặc cuốn sách là một dự án trong một thư mục riêng. Tài liệu học tập tập trung vào bản chất khái niệm, quá trình lập luận và các quyết định thiết kế hệ thống.

## Các dự án

| Thư mục | Mục đích | Trạng thái |
| --- | --- | --- |
| [DDIA](DDIA/README.md) | Học *Designing Data-Intensive Applications* của Martin Kleppmann và tư duy về hệ thống dữ liệu. | Đã tạo cấu trúc dự án; chưa có PDF sách để xác định ấn bản và viết ghi chú. |

Thêm dự án mới bằng một thư mục riêng cùng README mô tả nguồn tài liệu, mục tiêu học và tình trạng ghi chú.

## Skill dùng trong dự án

| Skill | Mục đích | Vị trí |
| --- | --- | --- |
| `read-doc-pdf` | Đọc PDF kỹ thuật đã cung cấp; xác minh phiên bản và mục lục; viết ghi chú tiếng Việt theo chương với ví dụ, đánh đổi và góc nhìn System Design. | [.agents/skills/read-doc-pdf/SKILL.md](.agents/skills/read-doc-pdf/SKILL.md) |

Trong Codex, gọi skill bằng `$read-doc-pdf` khi đã cung cấp PDF và chỉ rõ chương hoặc mục muốn học. Tên skill dùng dấu gạch nối theo quy ước của Codex; đây là skill cho nhu cầu `/read_doc_pdf` nêu trong yêu cầu dự án.
