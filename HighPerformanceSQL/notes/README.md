# Ghi chú học High Performance MySQL — Tổng quan và hướng dẫn đọc

Bộ tài liệu học tiếng Việt cho cuốn *High Performance MySQL* (Fourth Edition). Mục tiêu là hiểu **vì sao** MySQL nhanh hay chậm và **cơ chế nào** đứng sau, từ kiến trúc, đo lường, phần cứng, cấu hình, schema, index, truy vấn cho tới replication, backup, scaling, cloud và compliance. Yêu cầu gốc nằm trong [`../init/DETAIL.md`](../init/DETAIL.md).

## 1. Ấn bản đã xác minh

Thông tin dưới đây được đọc trực tiếp từ trang nhan đề (trang in iii, PDF 5), trang bản quyền (trang in iv, PDF 6) và metadata của [PDF gốc](../init/high-performance-mysql-4th.pdf).

| Mục | Giá trị |
| --- | --- |
| Tên sách | *High Performance MySQL — Proven Strategies for Operating at Scale* |
| Tác giả | Silvia Botros và Jeremy Tinley |
| Lời tựa (Foreword) | Jeremy Cole |
| Nhà xuất bản | O'Reilly Media |
| Ấn bản | **Fourth Edition**, tháng 11/2021 |
| Lịch sử bản in | 2021-11-17: First Release |
| Bản quyền | © 2022 Silvia Botros và Jeremy Tinley |
| ISBN | 978-1-492-08051-0 |
| Tệp PDF | 389 trang, tạo ngày 2021-11-17 bằng Antenna House |

> - Đây **không phải** cuốn *High-Performance SQL* hay sách của Markus Winand. Tên đó xuất hiện trong prompt ban đầu của dự án nhưng không phải nguồn (xem [`../README.md`](../README.md)).
> - Mọi ghi chú dựa trên **ấn bản thứ tư (2021)**. Các ấn bản trước (2004, 2008, 2012) có nội dung và trọng tâm khác (Preface, tr. xvi).
> - Chương 3 (*Performance Schema*) do **Sveta Smirnova** viết (ghi ở trang đầu chương, tr. 41).

<a id="page-convention"></a>

### Quy ước số trang

- **"tr. N"** trong ghi chú là **số trang in**, tức số in ở chân trang.
- PDF 1–2 là bìa trước và bìa sau. Phần đầu sách (lời khen, nhan đề, bản quyền, mục lục, Foreword, Preface) đánh số La Mã i–xx, tương ứng PDF 3–22.
- Từ chương 1 trở đi: **trang PDF = trang in + 22**. Ví dụ chương 1 bắt đầu ở tr. 1, tức PDF 23. Độ lệch này đã được kiểm tra bằng số in ở chân trang tại tr. 2, 78, 191, 228, 278 và 340.
- Các tham chiếu chéo trong sách dùng số trang in. Ví dụ "JSON Data" on page 139 nằm ở PDF 161.

### Đặc điểm của nguồn

- Chữ trích xuất từ PDF rõ ràng. Không có trang scan hay trang thiếu chữ; các trang gần như trống chỉ là trang trắng cuối chương (tr. 40, 74, 98, 256, 312, 342, 352).
- Sách **không có** bản đồ minh họa đầu chương và **không có** danh sách tài liệu tham khảo cuối chương. Nguồn được dẫn bằng chú thích cuối trang và liên kết.
- Sách viết vào giai đoạn **MySQL 8.0**, có nhiều chỗ so sánh với 5.5, 5.6 và 5.7. Ghi chú sẽ nêu phiên bản áp dụng cho từng tính năng.

## 2. Cấu trúc cuốn sách

Ấn bản này **không chia thành Part**, và Preface **không có** mục "cách sách được tổ chức". Sách gồm 13 chương và 2 phụ lục.

**Vì sao ấn bản này khác** (Preface, tr. xv–xvi). Các ấn bản trước tập trung vắt kiệt từng chút hiệu năng, qua nội bộ và các thiết lập tuning. Ấn bản thứ tư giữ mục tiêu đó nhưng đổi trọng tâm:

- giúp người đọc **ra quyết định có hiểu biết** về cách dùng MySQL;
- bắt đầu từ việc hiểu **MySQL được thiết kế ra sao**, để biết nó **giỏi và không giỏi** việc gì;
- các bản MySQL hiện đại có mặc định hợp lý, nên ít khi cần tuning trừ khi gặp vấn đề scaling rất cụ thể;
- các đội ngày nay phải lo **thay đổi schema, compliance và sharding**.

<a id="muc-luc-chi-tiet"></a>

### Mục lục chi tiết

Cột "Trang in" là phạm vi nội dung của chương, không tính trang trắng cuối chương. Cột "Ghi chú" liên kết tới ghi chú chương, đáp án câu tự kiểm tra và bài thực hành (nếu có).

| Chương | Tên | Trang in | Trang PDF | Các mục chính | Ghi chú |
| --- | --- | --- | --- | --- | --- |
| 1 | MySQL Architecture | 1–18 | 23–40 | MySQL’s Logical Architecture · Concurrency Control · Transactions · Multiversion Concurrency Control · Replication · Datafiles Structure · The InnoDB Engine | [ghi chú](chapter-01.md) · [đáp án](answers/chapter-01.md) · [thực hành](experiments/chapter-01.md) |
| 2 | Monitoring in a Reliability Engineering World | 19–39 | 41–61 | The Impact of Reliability Engineering on DBA Teams · Defining Service Level Goals · What to Measure · Measuring Long-Term Performance | [ghi chú](chapter-02.md) · [đáp án](answers/chapter-02.md) · [thực hành](experiments/chapter-02.md) |
| 3 | Performance Schema | 41–73 | 63–95 | Introduction to Performance Schema · Configuration · Using Performance Schema | [ghi chú](chapter-03.md) · [đáp án](answers/chapter-03.md) · [thực hành](experiments/chapter-03.md) |
| 4 | Operating System and Hardware Optimization | 75–97 | 97–119 | What Limits MySQL’s Performance? · How to Select CPUs for MySQL · Balancing Memory and Disk Resources · Solid-State Storage · RAID Performance Optimization · Network Configuration · Choosing a Filesystem | [ghi chú](chapter-04.md) · [đáp án](answers/chapter-04.md) · [thực hành](experiments/chapter-04.md) |
| 5 | Optimizing Server Settings | 99–124 | 121–146 | How MySQL’s Configuration Works · What Not to Do · Creating a MySQL Configuration File · Configuring Memory Usage · Configuring MySQL’s I/O Behavior · Configuring MySQL Concurrency · Safety Settings · Advanced InnoDB Settings | [ghi chú](chapter-05.md) · [đáp án](answers/chapter-05.md) · [thực hành](experiments/chapter-05.md) |
| 6 | Schema Design and Management | 125–154 | 147–176 | Choosing Optimal Data Types · Schema Design Gotchas in MySQL · Schema Management | [ghi chú](chapter-06.md) · [đáp án](answers/chapter-06.md) · [thực hành](experiments/chapter-06.md) |
| 7 | Indexing for High Performance | 155–190 | 177–212 | Indexing Basics · Indexing Strategies for High Performance · Index and Table Maintenance | [ghi chú](chapter-07.md) · [đáp án](answers/chapter-07.md) · [thực hành](experiments/chapter-07.md) |
| 8 | Query Performance Optimization | 191–226 | 213–248 | Why Are Queries Slow? · Slow Query Basics: Optimize Data Access · Ways to Restructure Queries · Query Execution Basics · Limitations of the MySQL Query Optimizer · Optimizing Specific Types of Queries | [ghi chú](chapter-08.md) · [đáp án](answers/chapter-08.md) · [thực hành](experiments/chapter-08.md) |
| 9 | Replication | 227–255 | 249–277 | Replication Overview · Replication Under the Hood · Replication Failover · Replication Topologies · Replication Administration and Maintenance · Replication Problems and Solutions | [ghi chú](chapter-09.md) · [đáp án](answers/chapter-09.md) · [thực hành](experiments/chapter-09.md) |
| 10 | Backup and Recovery | 257–286 | 279–308 | Why Backups? · Defining Recovery Requirements · Designing a MySQL Backup Solution · Managing and Backing Up Binary Logs · Backup and Recovery Tools · Backing Up Data · Recovering from a Backup | [ghi chú](chapter-10.md) · [đáp án](answers/chapter-10.md) · [thực hành](experiments/chapter-10.md) |
| 11 | Scaling MySQL | 287–311 | 309–333 | What Is Scaling? · Read- Versus Write-Bound Workloads · Functional Sharding · Scaling Reads with Read Pools · Queuing · Scaling Writes with Sharding | [ghi chú](chapter-11.md) · [đáp án](answers/chapter-11.md) · [thực hành](experiments/chapter-11.md) |
| 12 | MySQL in the Cloud | 313–324 | 335–346 | Managed MySQL · MySQL on Virtual Machines | [ghi chú](chapter-12.md) · [đáp án](answers/chapter-12.md) · [thực hành](experiments/chapter-12.md) |
| 13 | Compliance with MySQL | 325–341 | 347–363 | What Is Compliance? · Building for Compliance Controls | [ghi chú](chapter-13.md) · [đáp án](answers/chapter-13.md) · [thực hành](experiments/chapter-13.md) |
| A | Upgrading MySQL | 343–348 | 365–370 | Why Upgrade? · Upgrade Life Cycle · Testing Upgrades · Upgrading at Scale | [ghi chú](appendix-a.md) · [đáp án](answers/appendix-a.md) · [thực hành](experiments/appendix-a.md) |
| B | MySQL on Kubernetes | 349–351 | 371–373 | Provisioning Resources with Kubernetes | [ghi chú](appendix-b.md) · [đáp án](answers/appendix-b.md) |

Phần cuối sách: Index (tr. 353–365), About the Authors và Colophon (tr. 366).

## 3. Các tệp trong thư mục

| Tệp | Nội dung | Trạng thái |
| --- | --- | --- |
| `README.md` | Ấn bản đã xác minh, mục lục, quy ước và hướng dẫn đọc (tệp này). | Hoàn thành |
| [`learning-roadmap.md`](learning-roadmap.md) | Mục tiêu của sách, kiến thức nền, nhóm chủ đề, lộ trình học theo thứ tự sách và cách học, thực hành, tự kiểm tra; danh sách các điểm sách tham chiếu sai, in sai hay đã lỗi thời so với MySQL 8.0.42. | Hoàn thành, đã cập nhật sau khi viết xong các chương |
| [`concept-map.md`](concept-map.md) | Quan hệ giữa các khái niệm và giữa các chương, dựng từ tham chiếu chéo trong sách và nội dung các chương; luồng chẩn đoán "MySQL chậm"; tra cứu nhanh khái niệm theo chương. | Hoàn thành, đã cập nhật sau khi viết xong các chương |
| [`glossary.md`](glossary.md) | Thuật ngữ Anh–Việt, định nghĩa ngắn, liên kết tới mục giải thích sâu trong từng chương. | Hoàn thành (13 chương, 2 phụ lục) |
| `chapter-01.md` … `chapter-13.md`, `appendix-a.md`, `appendix-b.md` | Ghi chú chuyên sâu từng chương và phụ lục, theo cấu trúc của `DETAIL.md`. Liên kết ở [mục lục](#muc-luc-chi-tiet). | Hoàn thành |
| `experiments/` | Bài thực hành chạy lại được. Phần lớn đã **chạy thật** trên MySQL 8.0.42 và có kết quả đo; phần cần công cụ hay hạ tầng không có (Linux, XtraBackup, cloud, Kubernetes…) ghi rõ **Chưa thực thi** kèm quy trình. Chương 12 chỉ có bài tập quyết định trên giấy; phụ lục B không có thực hành. | Hoàn thành |
| `answers/` | Giải thích đáp án cho mọi câu tự kiểm tra, tách khỏi ghi chú chương. | Hoàn thành |

Mọi chương và phụ lục đã được đọc đầy đủ trước khi viết. Các điểm sách tham chiếu sai, in sai hoặc đã lỗi thời so với MySQL 8.0.42 được tổng hợp ở [roadmap, mục 8](learning-roadmap.md#can-kiem-chung) và ghi chi tiết trong mục "Ghi chú kiểm tra nguồn" của từng chương.

## 4. Quy ước nhãn nội dung

| Nhãn | Ý nghĩa |
| --- | --- |
| *(không nhãn)* + "tr. N" | Diễn giải nội dung sách. Số trang giúp bạn đối chiếu bản gốc. |
| **Bổ sung ngoài sách** | Kiến thức thêm để giải thích rõ hơn, không có trong phần sách đang học (kèm nguồn nếu dùng tài liệu bên ngoài). Phép tính tự thực hiện cũng mang nhãn này. |
| **Ví dụ minh họa do AI xây dựng** | Ví dụ, dữ liệu, SQL hoặc tình huống giả định, không phải của tác giả. |
| **Dự đoán** | Điều người viết ghi chú kỳ vọng sẽ xảy ra (ví dụ plan mà optimizer có thể chọn), chưa được đo. |
| **Kết quả đã đo** | Chỉ dùng khi thí nghiệm thực sự đã chạy, kèm phiên bản MySQL và môi trường. Không có số liệu đo nào được dựng sẵn rồi trình bày như dữ liệu thật. Các thí nghiệm trong bộ ghi chú chạy trên **MySQL Community 8.0.42**, laptop Windows 11, mỗi chương một instance riêng (`--no-defaults`, datadir và cổng riêng, cấu hình mặc định trừ khi ghi khác), dữ liệu tự sinh bằng SQL. Thời gian đo trên laptop có nhiễu: hãy xem chúng như so sánh tương đối, không phải benchmark production. |

Ghi chú diễn giải bằng lời của người viết, không chép nguyên văn sách. Thuật ngữ tiếng Anh được giữ lại kèm giải thích tiếng Việt.

## 5. Hướng dẫn đọc

1. **Đọc [`learning-roadmap.md`](learning-roadmap.md) trước** để biết sách đi theo trình tự nào, cần kiến thức nền gì và nên học, thực hành ra sao.
2. **Đọc sách song song với ghi chú.** Mỗi chương ghi chú dẫn số trang in để bạn mở đúng đoạn trong PDF.
3. **Mỗi chương ghi chú có cấu trúc giống nhau** (theo `DETAIL.md`): bức tranh lớn → giải thích sâu theo từng mục sách → phân tích tình huống → đo lường và thực hành → liên kết và điều cần nhớ → câu hỏi tự kiểm tra.
4. **Tự trả lời câu hỏi trước**, rồi mới mở `answers/chapter-XX.md`.
5. **Làm thực hành trên môi trường thử nghiệm**, không chạy trên production. Bài thực hành ghi rõ phiên bản MySQL, giả thuyết, cách đo và cách khôi phục.
6. **Dùng [`concept-map.md`](concept-map.md)** để thấy các chương nối với nhau thế nào, và [`glossary.md`](glossary.md) khi gặp thuật ngữ lạ.

Sơ đồ được viết bằng Mermaid. GitHub và hầu hết trình xem Markdown hiện đại hiển thị được trực tiếp.
