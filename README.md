# Public Document

Kho tài liệu học bằng tiếng Việt từ sách và tài liệu kỹ thuật do người dùng cung cấp. Mỗi chủ đề hoặc cuốn sách là một dự án trong một thư mục riêng. Tài liệu học tập tập trung vào bản chất khái niệm, quá trình lập luận và các quyết định thiết kế hệ thống.

## Các dự án

| Thư mục | Mục đích | Trạng thái |
| --- | --- | --- |
| [DDIA](DDIA/README.md) | Học *Designing Data-Intensive Applications* của Martin Kleppmann và tư duy về hệ thống dữ liệu. | Đã xác minh ấn bản (First Edition, 2017) và mục lục; đã có [ghi chú chương 1](DDIA/ddia-notes/chapter-01.md). |

Mỗi dự án mới cần một thư mục riêng gồm `README.md` mô tả dự án và `init/` chứa **tệp nguồn gốc** cùng `DETAIL.md` ghi rõ cần tạo những gì, phạm vi, mức độ chi tiết và định dạng đầu ra. Skill là quy trình dùng chung; `DETAIL.md` là yêu cầu cụ thể của từng dự án.

## Skill dùng trong dự án

| Skill | Mục đích | Vị trí |
| --- | --- | --- |
| `read-doc-pdf` | Đọc `README.md` và `init/DETAIL.md` của dự án trước, rồi kiểm tra PDF nguồn và tạo tài liệu tiếng Việt theo yêu cầu riêng của dự án. | [.agents/skills/read-doc-pdf/SKILL.md](.agents/skills/read-doc-pdf/SKILL.md) |

Trong Codex, gọi skill bằng `$read-doc-pdf` và nêu dự án, chương hoặc mục cần học. Skill sẽ đọc README của dự án, yêu cầu trong `init/DETAIL.md` và tệp gốc trước khi viết. Tên skill dùng dấu gạch nối theo quy ước của Codex; đây là skill cho nhu cầu `/read_doc_pdf` nêu trong yêu cầu dự án.
