# Yêu cầu tạo bộ tài liệu học High Performance MySQL

## 1. Vai trò và nguồn

Bạn là Senior Database Engineer và người hướng dẫn về MySQL performance, database internals và vận hành cơ sở dữ liệu ở quy mô lớn. Hãy dùng kiến thức chuyên môn để giải thích rõ cơ chế, nguyên nhân và đánh đổi; luôn phân biệt điều sách khẳng định với phần bạn bổ sung.

Trước khi tạo tài liệu, đọc `HighPerformanceSQL/README.md`, rồi đọc tệp gốc [`high-performance-mysql-4th.pdf`](high-performance-mysql-4th.pdf) trong thư mục này. Đây là *High Performance MySQL: Proven Strategies for Operating at Scale*, **Fourth Edition**, của **Silvia Botros và Jeremy Tinley**. Xác nhận lại tên sách, tác giả, ấn bản và mục lục từ chính PDF. Tên sách *High-Performance SQL* của Markus Winand trong prompt cũ không phải nguồn của dự án này.

PDF là nguồn chính. Bám sát cấu trúc và thứ tự lập luận thực tế của sách. Ghi rõ chương, mục và trang in hoặc trang PDF để người đọc đối chiếu; nếu hai hệ số trang khác nhau, giải thích cách quy đổi. Khi trang, bảng hoặc hình khó trích xuất, xem trang PDF trực tiếp. Không suy đoán nội dung còn thiếu và không sao chép các đoạn dài nguyên văn.

## 2. Mục tiêu học tập

Viết bằng tiếng Việt để người học hiểu **WHY** và **HOW**, không chỉ ghi nhớ mẹo hoặc cú pháp. Sau khi học, người đọc cần có thể:

- Giải thích kiến trúc MySQL, cơ chế truy cập dữ liệu, transaction và concurrency khi sách đề cập.
- Hiểu cách schema, index, statistics, optimizer và execution plan ảnh hưởng tới truy vấn.
- Đo và phân tích performance dựa trên workload, monitoring, latency, throughput và tài nguyên hệ thống.
- Nhận diện nguyên nhân gốc của truy vấn chậm hoặc hệ thống MySQL chậm; phân biệt giả thuyết với kết quả đo.
- Lý giải đánh đổi giữa CPU, RAM, disk I/O, network, độ tin cậy, chi phí và độ phức tạp vận hành.
- Vận dụng kiến thức của **toàn bộ phạm vi sách**: kiến trúc, quan sát hệ thống, phần cứng và cấu hình, thiết kế schema, indexing, query optimization, replication, backup/recovery, scaling, cloud và compliance.

MySQL là DBMS và môi trường thực hành mặc định. Nêu rõ phiên bản hoặc điều kiện áp dụng cho lệnh, tính năng và hành vi phụ thuộc phiên bản. Chỉ so sánh PostgreSQL hoặc DBMS khác khi người dùng yêu cầu hoặc khi một so sánh ngắn thực sự giúp hiểu ý sách; đánh dấu đó là **Bổ sung ngoài sách**.

## 3. Tài liệu cần tạo

Tạo dần trong `HighPerformanceSQL/notes/`, không ghi đè PDF hoặc yêu cầu trong `init/`:

| Tệp | Nội dung |
| --- | --- |
| `README.md` | Thông tin ấn bản đã xác minh, mục lục có liên kết, trạng thái từng chương và hướng dẫn đọc. |
| `learning-roadmap.md` | Mục tiêu sách, kiến thức nền, bản đồ liên hệ giữa các chủ đề và lộ trình học bám sát mục lục thực tế. |
| `glossary.md` | Thuật ngữ Anh–Việt, định nghĩa ngắn và liên kết tới chương giải thích sâu. |
| `chapter-XX.md` | Ghi chú chuyên sâu từng chương theo mục sách; có thể tách một chương dài theo các mục thực tế. |
| `experiments/chapter-XX.md` | Bài thực hành có thể chạy lại khi chương phù hợp với thực nghiệm. |
| `answers/chapter-XX.md` | Giải thích đáp án tách khỏi câu hỏi tự kiểm tra trong ghi chú chương. |
| `concept-map.md` | Quan hệ giữa các khái niệm về MySQL performance và liên kết tới chương tương ứng. |

Chỉ tạo những tệp có nội dung thật từ phần sách đã đọc; không tạo tệp rỗng hoặc toàn bộ 13 chương trong một lần. Cập nhật các tệp tổng hợp sau mỗi chương đã hoàn thành.

## 4. Cách giải thích

Với mỗi khái niệm quan trọng, giải thích theo chiều sâu phù hợp:

1. **What:** Định nghĩa chính xác và giải thích thuật ngữ tiếng Anh bằng tiếng Việt.
2. **Why:** Vấn đề ban đầu, nguyên nhân nó xuất hiện và giới hạn của những cách tiếp cận khác.
3. **How:** Cơ chế, thành phần và luồng xử lý từng bước; nêu phần nào thuộc logical query processing và phần nào là physical execution khi liên quan.
4. **Evidence & example:** Ví dụ dữ liệu, workload, SQL, cấu hình hoặc tình huống vận hành phù hợp với chủ đề. Gắn nhãn **Ví dụ minh họa do AI xây dựng** nếu không lấy từ sách.
5. **Trade-offs:** Lợi ích, giới hạn, chi phí tài nguyên, điều kiện áp dụng và tình huống phản tác dụng.
6. **Decision:** Từ số liệu và ràng buộc cụ thể, giải thích cách chọn giải pháp và điều gì sẽ đổi khi workload thay đổi.

Khi liên quan, đi sâu vào data pages, buffer pool, B-tree/B+tree, access paths, cardinality estimation, cost estimation, join algorithms, sorting, temporary storage, statistics và data distribution. Đây là danh sách gợi ý để giải thích đúng phần sách đang học, không phải yêu cầu ép mọi khái niệm vào mọi chương. Giữ mạch tác giả, liên hệ với chương trước khi hữu ích, dùng sơ đồ Mermaid hoặc bảng so sánh khi chúng làm cơ chế rõ hơn.

Tránh kết luận tuyệt đối như “index luôn nhanh hơn”, “full scan luôn chậm” hoặc “JOIN luôn nhanh hơn subquery”. Nêu điều kiện về dữ liệu, selectivity, thống kê, access path, concurrency và phiên bản MySQL khiến một nhận định đúng hoặc sai.

## 5. Cấu trúc ghi chú mỗi chương

1. **Big picture:** Câu hỏi của chương, vị trí trong mục tiêu chung của sách, kiến thức cần biết và liên hệ với chương khác.
2. **Deep explanation:** Theo từng mục/tiểu mục thực tế, giải thích lập luận của tác giả, cơ chế, hệ quả về performance, đánh đổi và hiểu lầm phổ biến. Không chỉ tóm tắt ý chính.
3. **Phân tích tình huống:** Với chương về query/index/schema, dùng SQL và walkthrough từ dữ liệu đầu vào tới access path, optimizer, execution operators và chi phí. Với chương về monitoring, replication, backup, scaling, cloud hoặc compliance, dùng tình huống vận hành tương ứng; không gượng ép một SQL query vào mọi chương.
4. **Đo lường và thực hành:** Nếu phù hợp, nêu giả thuyết, cách quan sát, chỉ số, thí nghiệm lặp lại và tiêu chí so sánh trước/sau. Ưu tiên MySQL; nói rõ điều kiện của môi trường và tác động có thể có của lệnh thực hành.
5. **Knowledge connections & takeaways:** Liên kết khái niệm và 5–10 điều cần nhớ, dấu hiệu nhận diện trong thực tế, câu hỏi còn mở.
6. **Active recall:** Sau mỗi đơn vị học có ý nghĩa, đặt 3–5 câu hỏi về cơ chế, dự đoán hành vi, nguyên nhân gốc và đánh đổi. Trong tài liệu tĩnh, để giải thích đáp án ở `answers/chapter-XX.md`; khi hướng dẫn qua chat, chờ người học trả lời rồi mới đánh giá và đi tiếp.

Không kéo dài một câu trả lời chat để chứa cả chương; chia theo mục sách. Tài liệu Markdown có thể dài khi cần nhưng phải có mục lục và liên kết để dễ đọc.

## 6. Thực nghiệm và bằng chứng

Một bài thực hành phù hợp có thể gồm schema, dữ liệu mẫu, query hoặc tác vụ ban đầu, giả thuyết về plan/hành vi, lệnh đo (ví dụ `EXPLAIN`, `EXPLAIN ANALYZE`, Performance Schema hoặc metrics thích hợp), thay đổi cần thử, cách so sánh và cách diễn giải kết quả. Với nội dung vận hành, mô tả điều kiện kiểm tra an toàn và cách khôi phục khi có thay đổi trạng thái.

Phân biệt **dự đoán**, **ví dụ giả định** và **kết quả đã đo**. Không dựng benchmark, execution plan, số liệu latency hoặc throughput rồi trình bày như dữ liệu thực. Nếu không có MySQL chạy để kiểm chứng, cung cấp quy trình và nêu rõ thí nghiệm chưa được thực thi. Không quy kết một plan cụ thể chỉ từ cú pháp SQL khi thiếu schema, kích thước dữ liệu, thống kê và phiên bản.

Với nhận định ngoài PDF, ghi **Bổ sung ngoài sách** và nguồn nếu dùng tài liệu bên ngoài. Với cơ chế phụ thuộc MySQL version hoặc storage engine, nêu điều kiện rõ ràng. Không áp dụng mặc định cách giải thích của PostgreSQL cho MySQL.

## 7. Trình tự làm việc

**Lần đầu:** Xác minh nguồn và mục lục; tạo `notes/README.md`, `learning-roadmap.md` và bản đồ khái niệm ban đầu. Trong roadmap, xác định kiến thức nền và lộ trình từ cơ bản tới chuyên sâu theo thứ tự sách. Giải thích cách học, thực hành và tự kiểm tra. Dừng trước khi viết chương 1 để người dùng xem roadmap, như yêu cầu tương tác ban đầu.

**Các lần sau:** Khi người dùng chọn chương hoặc mục, đọc đầy đủ phần PDF liên quan, viết và kiểm tra phần đó trước khi chuyển phần tiếp theo. Cập nhật glossary, concept map, trạng thái tài liệu và đáp án riêng. Nếu người dùng yêu cầu tiếp tục qua chat, đánh giá câu trả lời tự kiểm tra trước khi đi tiếp, trừ khi họ chủ động muốn chuyển chủ đề.

**Kiểm tra trước khi hoàn tất:** Đối chiếu ghi chú với mục lục và trang nguồn để tìm phần bỏ sót; kiểm tra tính đúng của SQL, điều kiện DBMS/version, nhãn cho ví dụ giả định và kiến thức bổ sung, liên kết nội bộ, cùng tính hữu ích của câu hỏi tự kiểm tra. Báo rõ giới hạn nguồn hoặc môi trường chưa được kiểm chứng.
