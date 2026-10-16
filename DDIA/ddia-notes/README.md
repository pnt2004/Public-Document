# Ghi chú học DDIA — Tổng quan và hướng dẫn đọc

Bộ ghi chú tiếng Việt cho cuốn *Designing Data-Intensive Applications* (DDIA) của Martin Kleppmann. Ghi chú bám theo thứ tự lập luận của sách, tập trung vào bản chất khái niệm và các quyết định thiết kế hệ thống. Yêu cầu gốc nằm trong [`../init/DETAIL.md`](../init/DETAIL.md).

## 1. Ấn bản đã xác minh

Thông tin dưới đây được đọc trực tiếp từ trang bản quyền (trang in ii, trang PDF 4) và metadata của tệp [PDF gốc](../init/Designing%20Data%20Intensive%20Applications%20by%20Martin%20Kleppmann.pdf).

| Mục | Giá trị |
| --- | --- |
| Tên sách | *Designing Data-Intensive Applications — The Big Ideas Behind Reliable, Scalable, and Maintainable Systems* |
| Tác giả | Martin Kleppmann |
| Nhà xuất bản | O'Reilly Media |
| Ấn bản | **First Edition**, tháng 3/2017 |
| Lịch sử bản in | 2017-03-01: First Release |
| ISBN | 978-1-449-37332-0 |
| Tệp PDF | 613 trang, tạo ngày 2017-03-01 bằng Antenna House |

> Mọi ghi chú trong thư mục này dựa trên **ấn bản thứ nhất (2017)**. Nếu bạn đọc một ấn bản khác, số trang, thứ tự mục và nội dung có thể không khớp.

### Quy ước số trang

- **"tr. N"** trong ghi chú là **số trang in** trong sách. Đây là số bạn thấy ở chân trang.
- Phần đầu sách (bản quyền, mục lục, lời nói đầu) đánh số La Mã i–xx, tương ứng trang PDF 3–22.
- Từ phần nội dung chính trở đi: **trang PDF = trang in + 22**. Ví dụ: chương 1 bắt đầu ở tr. 3, tức trang PDF 25. Độ lệch này đã được kiểm tra ở tr. 1, 26, 78, 278, 553 và 591.
- Các tham chiếu chéo trong sách cũng dùng số trang in. Ví dụ: "Rebalancing Partitions" on page 209 nằm ở trang PDF 231.

## 2. Cấu trúc cuốn sách

Theo Preface (tr. xvi–xvii) và phần giới thiệu Part I (tr. 1), sách gồm 3 phần với 12 chương:

- **Part I — Foundations of Data Systems (chương 1–4):** những ý tưởng nền tảng đúng cho mọi hệ thống dữ liệu, dù chạy trên một máy hay trên cả cụm máy. Phần này bàn về mục tiêu cần đạt, mô hình dữ liệu, cách lưu trữ trên đĩa, và cách mã hóa dữ liệu khi schema thay đổi theo thời gian.
- **Part II — Distributed Data (chương 5–9):** chuyển từ dữ liệu trên một máy sang dữ liệu phân tán trên nhiều máy. Việc phân tán thường cần cho scalability, nhưng mang theo nhiều thách thức riêng: replication, partitioning, transactions, các vấn đề của hệ phân tán, consistency và consensus.
- **Part III — Derived Data (chương 10–12):** hệ thống tạo ra tập dữ liệu này từ tập dữ liệu khác. Trường hợp này phổ biến khi không có một database nào làm tốt mọi việc và ứng dụng phải ghép nhiều database, cache, index với nhau. Phần này đi từ batch processing đến stream processing, rồi tổng hợp lại ở chương 12.

### Mục lục chi tiết

Cột "Trang in" tính từ trang đầu chương tới trước trang đầu chương sau (theo bookmark PDF), kể cả danh sách tài liệu tham khảo. Vì bản đồ minh họa của mỗi chương được in ngay trước trang mở đầu chương, trang cuối của một khoảng thường là bản đồ (hoặc trang trắng) của chương kế tiếp. Phạm vi nội dung chính xác của từng chương được ghi ở đầu mỗi tệp ghi chú.

| Phần | Chương | Tên chương | Trang in | Trang PDF | Các mục chính | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |
| I | 1 | Reliable, Scalable, and Maintainable Applications | 3–26 | 25–48 | Thinking About Data Systems · Reliability · Scalability · Maintainability | [chapter-01.md](chapter-01.md) |
| I | 2 | Data Models and Query Languages | 27–68 | 49–90 | Relational Model Versus Document Model · Query Languages for Data · Graph-Like Data Models | [chapter-02.md](chapter-02.md) |
| I | 3 | Storage and Retrieval | 69–110 | 91–132 | Data Structures That Power Your Database · Transaction Processing or Analytics? · Column-Oriented Storage | [chapter-03.md](chapter-03.md) |
| I | 4 | Encoding and Evolution | 111–144 | 133–166 | Formats for Encoding Data · Modes of Dataflow | [chapter-04.md](chapter-04.md) |
| II | 5 | Replication | 151–198 | 173–220 | Leaders and Followers · Problems with Replication Lag · Multi-Leader Replication · Leaderless Replication | [chapter-05.md](chapter-05.md) (kèm phần giới thiệu Part II) |
| II | 6 | Partitioning | 199–220 | 221–242 | Partitioning and Replication · Partitioning of Key-Value Data · Partitioning and Secondary Indexes · Rebalancing Partitions · Request Routing | [chapter-06.md](chapter-06.md) |
| II | 7 | Transactions | 221–272 | 243–294 | The Slippery Concept of a Transaction · Weak Isolation Levels · Serializability | [chapter-07.md](chapter-07.md) |
| II | 8 | The Trouble with Distributed Systems | 273–320 | 295–342 | Faults and Partial Failures · Unreliable Networks · Unreliable Clocks · Knowledge, Truth, and Lies | [chapter-08.md](chapter-08.md) |
| II | 9 | Consistency and Consensus | 321–384 | 343–406 | Consistency Guarantees · Linearizability · Ordering Guarantees · Distributed Transactions and Consensus | [chapter-09.md](chapter-09.md) |
| III | 10 | Batch Processing | 389–438 | 411–460 | Batch Processing with Unix Tools · MapReduce and Distributed Filesystems · Beyond MapReduce | [chapter-10.md](chapter-10.md) (kèm phần giới thiệu Part III) |
| III | 11 | Stream Processing | 439–488 | 461–510 | Transmitting Event Streams · Databases and Streams · Processing Streams | [chapter-11.md](chapter-11.md) |
| III | 12 | The Future of Data Systems | 489–552 | 511–574 | Data Integration · Unbundling Databases · Aiming for Correctness · Doing the Right Thing | [chapter-12.md](chapter-12.md) |

Phần cuối sách: Glossary (tr. 553–558), Index (tr. 559–590), About the Author và Colophon (tr. 591). Phần giới thiệu Part II nằm ở tr. 145–148 (ghi chú ở [chapter-05.md, mục 0](chapter-05.md#part-ii)), phần giới thiệu Part III nằm ở tr. 385–387 (ghi chú ở [chapter-10.md, mục 0](chapter-10.md#part-iii)).

## 3. Các tệp trong thư mục

| Tệp | Nội dung | Trạng thái |
| --- | --- | --- |
| `README.md` | Ấn bản đã xác minh, mục lục, quy ước và hướng dẫn đọc (tệp này). | Hoàn thành |
| [`chapter-01.md`](chapter-01.md) | Chương 1: Reliability, Scalability, Maintainability. | Hoàn thành |
| [`chapter-02.md`](chapter-02.md) | Chương 2: mô hình relational, document, graph và các ngôn ngữ truy vấn. | Hoàn thành |
| [`chapter-03.md`](chapter-03.md) | Chương 3: hash index, LSM-tree, B-tree, OLTP và OLAP, lưu trữ theo cột. | Hoàn thành |
| [`chapter-04.md`](chapter-04.md) | Chương 4: định dạng mã hóa (JSON, Thrift, Protocol Buffers, Avro), schema evolution, luồng dữ liệu qua database, REST/RPC và message. | Hoàn thành |
| [`chapter-05.md`](chapter-05.md) | Phần giới thiệu Part II (shared-nothing, replication và partitioning) và Chương 5: single-leader, replication lag, multi-leader, leaderless, quorum, phát hiện ghi đồng thời. | Hoàn thành |
| [`chapter-06.md`](chapter-06.md) | Chương 6: phân vùng theo khoảng key và theo hash, skew và hot spot, secondary index cục bộ và toàn cục, rebalancing, định tuyến request. | Hoàn thành |
| [`chapter-07.md`](chapter-07.md) | Chương 7: ACID, các mức isolation yếu (read committed, snapshot isolation), lost update, write skew, phantom, serializability (thực thi tuần tự, 2PL, SSI). | Hoàn thành |
| [`chapter-08.md`](chapter-08.md) | Chương 8: partial failure, mạng không tin cậy, timeout, đồng hồ không tin cậy, process pause, đa số và fencing token, Byzantine fault, system model. | Hoàn thành |
| [`chapter-09.md`](chapter-09.md) | Chương 9: linearizability, CAP, causality và thứ tự, Lamport timestamp, total order broadcast, 2PC, XA, consensus, ZooKeeper. | Hoàn thành |
| [`chapter-10.md`](chapter-10.md) | Phần giới thiệu Part III (system of record, derived data) và Chương 10: công cụ Unix, MapReduce và HDFS, các thuật toán join, dataflow engine, xử lý đồ thị. | Hoàn thành |
| [`chapter-11.md`](chapter-11.md) | Chương 11: message broker và log, change data capture, event sourcing, thời gian và window, stream join, exactly-once. | Hoàn thành |
| [`chapter-12.md`](chapter-12.md) | Chương 12: tích hợp dữ liệu qua derived data, unbundling database, correctness end-to-end, timeliness và integrity, đạo đức và quyền riêng tư. | Hoàn thành |
| [`glossary.md`](glossary.md) | Thuật ngữ tiếng Anh kèm giải thích tiếng Việt, bổ sung dần theo từng chương. | Hoàn thành (chương 1–12) |
| [`system-design-concepts.md`](system-design-concepts.md) | Các nguyên lý System Design và liên kết đến chương liên quan. | Hoàn thành (P0–P130, chương 1–12) |

Mỗi chương chỉ được viết sau khi đã đọc phần sách tương ứng. Chương sau chỉ bắt đầu khi chương trước đã hoàn thành và được kiểm tra.

## 4. Quy ước nhãn nội dung

Theo yêu cầu trong `DETAIL.md`, ghi chú phân biệt rõ ba loại nội dung:

| Nhãn | Ý nghĩa |
| --- | --- |
| *(không nhãn)* + "tr. N" | Diễn giải nội dung sách. Số trang giúp bạn đối chiếu bản gốc. |
| **Bổ sung ngoài sách** | Kiến thức thêm để giải thích rõ hơn, không có trong phần sách đang học. Phép tính tự thực hiện từ số liệu trong sách cũng mang nhãn này. |
| **Ví dụ minh họa do AI xây dựng** | Ví dụ hoặc tình huống giả định, không phải của tác giả. |

Ghi chú diễn giải bằng lời của người viết, không chép nguyên văn sách. Thuật ngữ tiếng Anh được giữ lại kèm giải thích tiếng Việt.

## 5. Hướng dẫn đọc

1. **Đọc sách trước hoặc song song.** Mỗi chương ghi chú dẫn số trang để bạn mở đúng đoạn trong PDF.
2. **Đọc phần A của chương** để biết chương giải quyết vấn đề gì và nằm ở đâu trong cả cuốn sách.
3. **Đọc phần B theo thứ tự.** Phần này đi đúng trình tự mục của sách. Các khái niệm quan trọng được giải thích theo khung What → Why → How → Example → Trade-offs → System Design.
4. **Dùng phần C và D để ôn tập** qua sơ đồ và bảng so sánh.
5. **Làm phần E trước khi xem lời giải.** Hãy tự suy luận các tình huống thiết kế trước, rồi mới đọc lập luận gợi ý.
6. **Tự trả lời câu hỏi ở phần G**, sau đó mới xem phần đáp án.
7. **Tra [`glossary.md`](glossary.md)** khi gặp thuật ngữ lạ. Đọc [`system-design-concepts.md`](system-design-concepts.md) để thấy các nguyên lý kết nối với nhau qua nhiều chương.

Sơ đồ được viết bằng Mermaid. GitHub và hầu hết trình xem Markdown hiện đại hiển thị được trực tiếp.
