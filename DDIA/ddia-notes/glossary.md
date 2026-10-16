# Thuật ngữ DDIA (Glossary)

Thuật ngữ tiếng Anh kèm giải thích tiếng Việt, bổ sung dần theo từng chương đã học. Số trang là trang in của ấn bản thứ nhất (2017). Thuật ngữ đánh dấu **(G)** cũng có trong Glossary cuối sách (tr. 553–558). Phần giải thích ở đây là diễn giải, đôi khi kèm ngữ cảnh từ chương đang học.

Đã cập nhật đến: **Chương 12** (toàn bộ sách, kèm phần giới thiệu Part II và Part III).

- [Chương 1 — Reliable, Scalable, and Maintainable Applications](#chuong-1)
- [Chương 2 — Data Models and Query Languages](#chuong-2)
- [Chương 3 — Storage and Retrieval](#chuong-3)
- [Chương 4 — Encoding and Evolution](#chuong-4)
- [Chương 5 — Replication](#chuong-5)
- [Chương 6 — Partitioning](#chuong-6)
- [Chương 7 — Transactions](#chuong-7)
- [Chương 8 — The Trouble with Distributed Systems](#chuong-8)
- [Chương 9 — Consistency and Consensus](#chuong-9)
- [Chương 10 — Batch Processing](#chuong-10)
- [Chương 11 — Stream Processing](#chuong-11)
- [Chương 12 — The Future of Data Systems](#chuong-12)

<a id="chuong-1"></a>

## Chương 1 — Reliable, Scalable, and Maintainable Applications

### Khái niệm chung

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Data-intensive application** | Ứng dụng thiên về dữ liệu | Ứng dụng mà thách thức chính là khối lượng, độ phức tạp hoặc tốc độ thay đổi của dữ liệu, không phải sức mạnh CPU. | tr. 3 |
| **Compute-intensive application** | Ứng dụng thiên về tính toán | Ứng dụng mà nút thắt chính là chu kỳ CPU. | tr. 3 |
| **Data system** | Hệ thống dữ liệu | Tên chung cho database, cache, search index, message queue, stream và batch processing. Cũng chỉ hệ thống ghép từ các thành phần này sau một API. | tr. 4–5 |
| **Functional requirements** | Yêu cầu chức năng | Hệ thống phải làm gì: lưu, truy xuất, tìm kiếm, xử lý dữ liệu… | tr. 22 |
| **Nonfunctional requirements** | Yêu cầu phi chức năng | Thuộc tính chung như bảo mật, reliability, compliance, scalability, compatibility, maintainability. | tr. 22 |

### Reliability

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Reliability** | Độ tin cậy | Tiếp tục hoạt động đúng (đúng chức năng, đủ hiệu năng, an toàn) ngay cả khi có trục trặc. | tr. 6 |
| **Fault** | Lỗi thành phần | Một thành phần của hệ thống đi lệch khỏi đặc tả của nó. | tr. 7 |
| **Failure** | Sự cố hệ thống | Toàn bộ hệ thống ngừng cung cấp dịch vụ cần thiết cho người dùng. | tr. 7 |
| **Fault-tolerant / resilient** **(G)** | Chịu lỗi | Lường trước và đối phó được với một số loại fault, ngăn chúng thành failure. Glossary của sách nhấn mạnh khả năng tự phục hồi. | tr. 6, 555 |
| **Chaos Monkey** | — | Công cụ của Netflix, cố tình gây fault (kill tiến trình ngẫu nhiên) để kiểm tra liên tục cơ chế chịu lỗi. | tr. 7 |
| **MTTF** (*mean time to failure*) | Thời gian trung bình đến khi hỏng | Với ổ cứng, khoảng 10–50 năm. Nghĩa là cụm 10.000 ổ hỏng khoảng một ổ mỗi ngày. | tr. 7 |
| **Redundancy** | Dư thừa, dự phòng | Thêm thành phần dự phòng (RAID, nguồn kép, CPU thay nóng, máy phát điện) để thay thế khi một thành phần hỏng. | tr. 7 |
| **RAID** | — | Cấu hình nhiều ổ đĩa để dữ liệu sống sót khi một ổ hỏng. Sách nêu như một ví dụ về dư thừa phần cứng. | tr. 7 |
| **Hot-swappable** | Thay nóng | Linh kiện có thể thay khi máy vẫn đang chạy. | tr. 7 |
| **High availability** | Tính sẵn sàng cao | Giảm tối đa thời gian dịch vụ ngừng hoạt động. | tr. 8 |
| **Elasticity** | Tính co giãn | Khả năng thêm hoặc bớt tài nguyên linh hoạt. Nền tảng cloud ưu tiên điều này hơn độ tin cậy của từng máy. | tr. 8, 17 |
| **Rolling upgrade** | Nâng cấp cuốn chiếu | Vá hoặc nâng cấp từng node một mà cả hệ thống không phải dừng. Chi tiết ở Chương 4. | tr. 8 |
| **Systematic error** | Lỗi có tính hệ thống | Lỗi phần mềm tương quan giữa các node (cùng code, cùng bug), khó lường trước. | tr. 8 |
| **Runaway process** | Tiến trình mất kiểm soát | Tiến trình ngốn hết tài nguyên dùng chung: CPU, bộ nhớ, ổ đĩa, mạng. | tr. 8 |
| **Cascading failure** | Lỗi dây chuyền | Fault nhỏ ở một thành phần kích hoạt fault ở thành phần khác, rồi lan tiếp. | tr. 9 |
| **Process isolation** | Cô lập tiến trình | Tách tiến trình để lỗi của một tiến trình không kéo theo tiến trình khác. | tr. 9 |
| **Sandbox environment** | Môi trường thử nghiệm | Môi trường ngoài production, đầy đủ tính năng, để thử với dữ liệu thật mà không ảnh hưởng người dùng thật. | tr. 9 |
| **Telemetry** | Đo xa | Thuật ngữ từ ngành khác (tên lửa) cho việc giám sát chi tiết: metric hiệu năng, tỷ lệ lỗi. | tr. 10 |

### Scalability

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Scalability** | Khả năng mở rộng | Khả năng hệ thống đối phó với tải tăng. Không phải nhãn một chiều: luôn phải hỏi "tăng theo chiều nào?". | tr. 10–11 |
| **Load parameters** | Tham số tải | Vài con số mô tả tải hiện tại: request/s, tỷ lệ đọc/ghi, người dùng đồng thời, tỷ lệ cache hit… | tr. 11 |
| **Fan-out** | Phân tán đầu ra | Gốc từ điện tử. Trong hệ thống giao dịch, là số request tới dịch vụ khác cần để phục vụ một request đến. Với Twitter, là số timeline phải cập nhật cho mỗi tweet. | tr. 11 |
| **Home timeline** | Dòng thời gian chính | Danh sách tweet của những người mà một người dùng theo dõi. | tr. 11 |
| **Throughput** | Thông lượng | Số bản ghi xử lý được mỗi giây, hoặc tổng thời gian chạy job trên tập dữ liệu cho trước. Thước đo chính của batch processing. | tr. 13 |
| **Skew** **(G)** | Độ lệch | Ở chương 1 (chú thích iii): dữ liệu không chia đều cho các worker, làm batch job chạy lâu hơn lý tưởng. Glossary của sách nêu hai nghĩa: (1) tải phân bố lệch giữa các partition, còn gọi là *hot spot*; (2) bất thường về thời gian khiến sự kiện xuất hiện sai thứ tự (*read skew*, *write skew*, *clock skew*). Các nghĩa này sẽ được làm rõ ở chương 6–10. | tr. 13, 557 |
| **Response time** | Thời gian phản hồi | Thời gian client chờ từ lúc gửi đến lúc nhận response = service time + độ trễ mạng + queueing delay. | tr. 13–14 |
| **Service time** | Thời gian phục vụ | Thời gian thực sự xử lý request. | tr. 14 |
| **Latency** | Độ trễ | Theo định nghĩa chặt của sách: thời gian request nằm chờ được xử lý. Thực tế thường dùng lẫn với response time. | tr. 14 |
| **Queueing delay** | Độ trễ xếp hàng | Thời gian request chờ trong hàng đợi. Thường chiếm phần lớn response time ở percentile cao. | tr. 14–15 |
| **Outlier** | Giá trị ngoại lai | Request chậm bất thường so với đa số. | tr. 14 |
| **Mean (arithmetic mean)** | Trung bình cộng | Tổng n giá trị chia cho n. Không cho biết bao nhiêu người dùng thực sự gặp độ trễ đó. | tr. 14 |
| **Percentile** **(G)** | Phân vị | Ngưỡng mà một tỷ lệ phần trăm giá trị nằm dưới. Ví dụ: p95 = t nghĩa là 95% request nhanh hơn t. | tr. 14–15, 556 |
| **Median / p50** | Trung vị | Điểm giữa của danh sách đã sắp xếp: một nửa request nhanh hơn, một nửa chậm hơn. | tr. 14–15 |
| **p95, p99, p999** | Phân vị 95, 99, 99,9 | Các percentile cao, dùng để thấy độ tệ của outlier. | tr. 15 |
| **Tail latency** | Độ trễ đuôi | Các percentile cao của response time. Ảnh hưởng trực tiếp tới trải nghiệm người dùng. | tr. 15 |
| **SLO** (*service level objective*) | Mục tiêu mức dịch vụ | Mục tiêu về hiệu năng và tính sẵn sàng, thường diễn đạt bằng percentile. | tr. 15 |
| **SLA** (*service level agreement*) | Thỏa thuận mức dịch vụ | Hợp đồng về hiệu năng và tính sẵn sàng. Khách hàng có thể đòi hoàn tiền nếu không đạt. | tr. 15 |
| **Head-of-line blocking** | Chặn đầu hàng | Vài request chậm ở đầu hàng đợi làm các request phía sau phải chờ, dù chúng xử lý nhanh. | tr. 16 |
| **Tail latency amplification** | Khuếch đại độ trễ đuôi | Request gọi càng nhiều backend thì càng dễ dính một lời gọi chậm, nên tỷ lệ request người dùng chậm tăng lên. | tr. 16 |
| **Rolling window** | Cửa sổ trượt | Khoảng thời gian gần nhất (vd: 10 phút) dùng để tính percentile liên tục. | tr. 16 |
| **Forward decay, t-digest, HdrHistogram** | — | Các thuật toán hoặc thư viện xấp xỉ percentile với chi phí CPU và bộ nhớ nhỏ. | tr. 16 |
| **Histogram** | Biểu đồ tần suất | Đếm số giá trị trong từng khoảng. Cách đúng để gộp response time từ nhiều máy là cộng các histogram. | tr. 16 |
| **Scaling up / vertical scaling** | Mở rộng theo chiều dọc | Chuyển sang một máy mạnh hơn. | tr. 17 |
| **Scaling out / horizontal scaling** | Mở rộng theo chiều ngang | Phân phối tải ra nhiều máy nhỏ hơn. | tr. 17 |
| **Shared-nothing architecture** **(G)** | Kiến trúc không chia sẻ | Các node độc lập, mỗi node có CPU, bộ nhớ, ổ đĩa riêng, nối với nhau qua mạng thông thường. Part II tập trung vào kiến trúc này (xem [chương 5](#chuong-5)). | tr. 17, 146–147, 557 |
| **Elastic system** | Hệ thống co giãn | Tự động thêm tài nguyên khi phát hiện tải tăng. Đối lập với scale thủ công. | tr. 17 |
| **Stateless / stateful** | Không trạng thái / có trạng thái | Dịch vụ stateless dễ phân tán. Data system stateful khi phân tán thì thêm nhiều độ phức tạp. | tr. 18 |
| **Magic scaling sauce** | "Nước sốt scale thần kỳ" | Cách gọi đùa cho một kiến trúc scale chung cho mọi trường hợp, thứ không tồn tại. | tr. 18 |

### Maintainability

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Maintainability** | Khả năng bảo trì | Nhiều người khác nhau làm việc hiệu quả với hệ thống theo thời gian. Gồm operability, simplicity, evolvability. | tr. 6, 18–19 |
| **Legacy system** | Hệ thống kế thừa | Hệ thống cũ, khó chịu khi bảo trì theo những cách riêng. | tr. 18 |
| **Technical debt** | Nợ kỹ thuật | Chi phí phải "trả" sau này vì những lựa chọn tạm bợ trước đó. Sách liệt kê việc trả nợ kỹ thuật là một phần của bảo trì. | tr. 18 |
| **Operability** | Khả năng vận hành | Giúp đội vận hành dễ giữ hệ thống chạy trơn tru. | tr. 19 |
| **Capacity planning** | Lập kế hoạch năng lực | Lường trước nhu cầu tài nguyên trước khi thiếu. | tr. 20 |
| **Simplicity** | Sự đơn giản | Giúp kỹ sư mới dễ hiểu hệ thống bằng cách loại bỏ độ phức tạp. Không phải sự đơn giản của giao diện người dùng. | tr. 19–21 |
| **Big ball of mud** | Cục bùn to | Phần mềm sa lầy trong độ phức tạp. | tr. 20 |
| **Accidental complexity** | Độ phức tạp phát sinh | Độ phức tạp không vốn có trong bài toán (theo góc nhìn người dùng), chỉ phát sinh từ cách cài đặt (Moseley và Marks). | tr. 21 |
| **Abstraction** | Trừu tượng hóa | Che giấu chi tiết cài đặt sau một giao diện gọn gàng, tái sử dụng được. Ví dụ: ngôn ngữ bậc cao, SQL. | tr. 21 |
| **Evolvability** | Khả năng tiến hóa | Giúp dễ thay đổi hệ thống khi yêu cầu thay đổi, tức sự linh hoạt ở cấp data system. Còn gọi là extensibility, modifiability, plasticity. | tr. 19, 22 |
| **TDD** (*test-driven development*) | Phát triển hướng kiểm thử | Kỹ thuật Agile, sách nêu như ví dụ công cụ thích ứng với thay đổi ở quy mô nhỏ. | tr. 22 |
| **Refactoring** | Tái cấu trúc | Cải thiện cấu trúc mà không đổi hành vi. Sách mở rộng câu hỏi lên cấp data system: "refactor" Twitter từ cách 1 sang cách 2. | tr. 22 |

### Bổ sung ngoài sách (thuật ngữ ngành liên quan, không có trong chương 1)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **Essential complexity** | Độ phức tạp vốn có của bài toán, không loại bỏ được, chỉ quản lý được. Đối lập với accidental complexity, theo Fred Brooks (*No Silver Bullet*, tài liệu [31] của chương 1). | Simplicity |
| **Fan-out on write / fan-out on read** | Tên thường dùng trong ngành cho cách 2 (tính sẵn khi ghi) và cách 1 (tính khi đọc) của Twitter. | Describing Load |
| **Coordinated omission** | Lỗi đo khi client sinh tải chờ response trước rồi mới gửi tiếp, làm số đo lạc quan hơn thực tế. Thuật ngữ do Gil Tene phổ biến. | Describing Performance |

<a id="chuong-2"></a>

## Chương 2 — Data Models and Query Languages

### Mô hình dữ liệu và lịch sử

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Data model** | Mô hình dữ liệu | Cách biểu diễn dữ liệu ở một lớp. Ảnh hưởng tới cả cách viết phần mềm lẫn cách nghĩ về bài toán. Ứng dụng là nhiều lớp mô hình chồng lên nhau. | tr. 27–28 |
| **Relational model** | Mô hình quan hệ | Do Edgar Codd đề xuất năm 1970: dữ liệu là các relation, mỗi relation là tập không thứ tự các tuple. | tr. 28 |
| **Relation / tuple** | Quan hệ / bộ | Trong SQL gọi là table (bảng) và row (hàng). | tr. 28 |
| **RDBMS** | Hệ quản trị CSDL quan hệ | *Relational database management system*. | tr. 28 |
| **Hierarchical model** | Mô hình phân cấp | Dữ liệu là cây các bản ghi lồng nhau, mỗi bản ghi đúng một cha. Dùng trong IMS. Giống mô hình JSON của document database. | tr. 36 |
| **IMS** | — | *Information Management System* của IBM (1968), làm cho chương trình Apollo. Database doanh nghiệp phổ biến nhất thập niên 1970. | tr. 36 |
| **Network model / CODASYL model** | Mô hình mạng | Tổng quát hóa hierarchical model: bản ghi có thể có nhiều cha. Liên kết giống con trỏ. Chuẩn hóa bởi ủy ban CODASYL. | tr. 36–37 |
| **Access path** | Đường truy cập | Trong CODASYL: đường đi từ bản ghi gốc dọc theo chuỗi liên kết. Là cách duy nhất để tới một bản ghi, do lập trình viên tự chọn. | tr. 37 |
| **Query optimizer** | Bộ tối ưu truy vấn | Thành phần tự quyết định thứ tự thực thi, index và phương pháp join. Thực chất là tự chọn access path. | tr. 37–38, 43 |
| **Object databases / XML databases** | CSDL đối tượng / XML | Các đối thủ của relational model (cuối 1980s đến đầu 1990s, và đầu 2000s), không kéo dài. | tr. 29 |
| **NoSQL** | — | Ban đầu là hashtag cho một meetup năm 2009 về database phân tán, mã nguồn mở, phi relational. Sau được diễn giải là *Not Only SQL*. Không chỉ một công nghệ cụ thể. | tr. 29 |
| **Polyglot persistence** | Lưu trữ đa ngôn ngữ | Dùng relational database song song với nhiều loại datastore phi relational, mỗi loại cho use case phù hợp. | tr. 29 |
| **Transaction processing / batch processing** | Xử lý giao dịch / xử lý theo lô | Hai use case gốc của relational database trong xử lý dữ liệu doanh nghiệp: giao dịch bán hàng, ngân hàng, đặt vé; và hóa đơn, lương, báo cáo. | tr. 28 |

### Relational và document

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Impedance mismatch** | Lệch trở kháng | Sự lệch giữa đối tượng trong code và bảng, hàng, cột trong database. Thuật ngữ mượn từ điện tử. | tr. 29–30 |
| **ORM** (*object-relational mapping*) | Ánh xạ đối tượng–quan hệ | Framework như ActiveRecord, Hibernate. Giảm code chuyển đổi nhưng không che giấu hoàn toàn khác biệt. | tr. 30 |
| **Document model / document database** | Mô hình / CSDL tài liệu | Lưu dữ liệu thành document tự chứa (JSON, XML, BSON). Ví dụ: MongoDB, RethinkDB, CouchDB, Espresso. | tr. 31 |
| **One-to-many** | Một-nhiều | Một người có nhiều công việc. Tạo thành cấu trúc cây, khớp tự nhiên với document. | tr. 30, 32 |
| **Many-to-one** | Nhiều-một | Nhiều người cùng một vùng. Cần tham chiếu và join. | tr. 34 |
| **Many-to-many** | Nhiều-nhiều | Người – tổ chức – trường học; người giới thiệu người. Điểm phân biệt quan trọng giữa các mô hình. | tr. 35, 49 |
| **Normalized / normalization** **(G)** | Chuẩn hóa | Cấu trúc không có dư thừa hay trùng lặp: dữ liệu đổi thì chỉ sửa một chỗ. Quy tắc kinh nghiệm: nhân bản giá trị lẽ ra lưu một chỗ là chưa chuẩn hóa. | tr. 33, 556 |
| **Denormalize** **(G)** | Phi chuẩn hóa | Cố ý đưa vào một ít dư thừa (thường dạng cache hay index) để đọc nhanh hơn. Giống một kết quả truy vấn tính sẵn. | tr. 34, 554 |
| **Join** **(G)** | Kết nối | Ghép các bản ghi có điểm chung, thường khi một bản ghi tham chiếu bản ghi khác (foreign key, document reference, edge) và truy vấn cần lấy bản ghi được trỏ tới. | tr. 34, 555 |
| **Foreign key / document reference** | Khóa ngoại / tham chiếu document | Định danh duy nhất dùng để tham chiếu mục liên quan, trong relational và document. Được giải lúc đọc. | tr. 38 |
| **Shredding** | Băm nhỏ | Kỹ thuật relational tách một cấu trúc dạng document ra nhiều bảng. Có thể làm schema cồng kềnh. | tr. 39 |
| **Schema** **(G)** | Lược đồ | Mô tả cấu trúc dữ liệu: các trường và kiểu dữ liệu. Có thể được kiểm tra ở nhiều thời điểm và thay đổi theo thời gian. | tr. 39, 556 |
| **Schemaless** | Không schema | Cách gọi gây hiểu lầm cho document database: thực tế luôn có schema ngầm. | tr. 39 |
| **Schema-on-read** | Schema khi đọc | Cấu trúc ngầm, chỉ diễn giải khi đọc. Giống kiểm tra kiểu động. | tr. 39–40 |
| **Schema-on-write** | Schema khi ghi | Schema tường minh, database đảm bảo dữ liệu ghi vào tuân theo. Giống kiểm tra kiểu tĩnh. | tr. 39–40 |
| **Migration** | Di chuyển schema | Đổi schema bằng `ALTER TABLE` và `UPDATE`. `ALTER TABLE` thường nhanh, trừ MySQL (sao chép bảng). `UPDATE` bảng lớn luôn chậm. | tr. 40 |
| **Heterogeneous data** | Dữ liệu không đồng nhất | Các mục trong collection không cùng cấu trúc. Trường hợp schema-on-read có lợi. | tr. 40 |
| **Locality** **(G)** | Tính cục bộ | Đặt dữ liệu thường cần cùng lúc vào cùng một chỗ. Lợi thế của document khi cần cả document. | tr. 41, 555 |
| **BSON** | — | Biến thể nhị phân của JSON mà MongoDB dùng để lưu document. | tr. 41 |
| **Interleaved tables / column-family** | Bảng xen kẽ / họ cột | Cách đạt locality ngoài document model: Spanner cho hàng bảng con nằm trong bảng cha; Bigtable (Cassandra, HBase) dùng column-family. | tr. 41 |
| **Nonsimple domains** | Miền không đơn | Trong mô tả gốc của Codd: giá trị của một hàng có thể là một relation lồng bên trong, giống JSON trong SQL ngày nay. | tr. 42 |

### Ngôn ngữ truy vấn

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Declarative** **(G)** | Khai báo | Mô tả thuộc tính kết quả cần có, không phải các bước để đạt được. Query optimizer quyết định cách thực thi. | tr. 43, 554 |
| **Imperative** | Mệnh lệnh | Chỉ định các thao tác theo thứ tự nhất định. IMS và CODASYL dùng API truy vấn imperative. | tr. 42–43 |
| **Relational algebra** | Đại số quan hệ | Nền toán học của relational model. SQL bám khá sát cấu trúc của nó. Ví dụ: toán tử chọn σ (sigma). | tr. 42–43 |
| **CSS selector / XPath / XSL** | — | Ngôn ngữ declarative để chọn phần tử và tạo kiểu cho document trong trình duyệt. Sách dùng làm phép so sánh với SQL. | tr. 44–45 |
| **DOM API** | — | API imperative của trình duyệt để thao tác document. Ví dụ đối lập với CSS. | tr. 45 |
| **MapReduce** | — | Mô hình lập trình xử lý lượng dữ liệu lớn theo lô trên nhiều máy (Google). Nằm giữa declarative và imperative. Chi tiết ở Chương 10. | tr. 46 |
| **Map (collect) / reduce (fold, inject)** | — | Hai hàm của MapReduce, có gốc từ lập trình hàm. Map phát ra cặp key-value; reduce gộp các value cùng key. | tr. 46–47 |
| **Pure function** | Hàm thuần | Chỉ dùng dữ liệu đầu vào, không truy vấn thêm, không side effect. Cho phép chạy ở bất kỳ đâu, thứ tự bất kỳ, chạy lại khi lỗi. | tr. 48 |
| **Aggregation pipeline** | — | Ngôn ngữ truy vấn declarative của MongoDB (từ 2.2), cú pháp JSON, sức biểu đạt tương đương một tập con SQL. | tr. 48 |

### Graph

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Graph** **(G)** | Đồ thị | Cấu trúc gồm vertex và edge. | tr. 49, 555 |
| **Vertex (node, entity)** | Đỉnh | Thứ có thể tham chiếu tới: người, địa điểm, trang web… | tr. 49 |
| **Edge (relationship, arc)** | Cạnh | Kết nối từ vertex này tới vertex khác. | tr. 49 |
| **Property graph** | Đồ thị thuộc tính | Vertex và edge đều có định danh và properties (key-value); edge có label, tail vertex, head vertex. Neo4j, Titan, InfiniteGraph. | tr. 50–51 |
| **Tail vertex / head vertex** | Đỉnh đầu / đỉnh cuối | Vertex nơi edge bắt đầu và nơi edge kết thúc. | tr. 50–51 |
| **Label** | Nhãn | Mô tả loại quan hệ của edge, cho phép nhiều loại thông tin trong một graph. | tr. 51 |
| **Traverse** | Duyệt | Đi theo một đường qua chuỗi vertex, xuôi hoặc ngược. | tr. 51 |
| **PageRank** | — | Thuật toán trên web graph để xác định độ phổ biến của trang web. | tr. 49 |
| **Cypher** | — | Ngôn ngữ truy vấn declarative cho property graph (Neo4j). Ký hiệu mũi tên; `*0..` nghĩa là "không hoặc nhiều lần". | tr. 52–54 |
| **Recursive common table expression** | Biểu thức bảng chung đệ quy | Cú pháp `WITH RECURSIVE` (từ SQL:1999) để biểu diễn duyệt độ dài thay đổi trong SQL. Dùng được nhưng vụng. | tr. 54 |
| **Triple-store** | Kho bộ ba | Lưu thông tin dạng (subject, predicate, object). Gần tương đương property graph. Datomic, AllegroGraph. | tr. 55 |
| **Subject / predicate / object** | Chủ ngữ / vị ngữ / tân ngữ | Ba thành phần của triple. Object là giá trị thì là property; object là vertex thì predicate là edge. | tr. 55–56 |
| **Turtle / Notation3 (N3)** | — | Định dạng người đọc được cho dữ liệu RDF. Turtle là tập con của N3. | tr. 56–57 |
| **Semantic web** | Web ngữ nghĩa | Ý tưởng website xuất bản dữ liệu máy đọc được để kết hợp thành "database của mọi thứ". Bị thổi phồng, chưa thành hiện thực. | tr. 57 |
| **RDF** | — | *Resource Description Framework*: định dạng dữ liệu nhất quán cho semantic web. Subject, predicate, object thường là URI. | tr. 57–58 |
| **URI namespace** | Không gian tên URI | Trong RDF, URI dùng làm namespace để dữ liệu nhiều bên kết hợp không xung đột. Không nhất thiết truy cập được. | tr. 58 |
| **SPARQL** | — | Ngôn ngữ truy vấn cho triple-store RDF, đọc là "sparkle". Ra đời trước Cypher; Cypher mượn pattern matching từ nó. | tr. 59 |
| **Gremlin** | — | Ngôn ngữ truy vấn graph imperative. | tr. 50 |
| **Pregel** | — | Framework xử lý graph (Chương 10). | tr. 50 |
| **Datalog** | — | Ngôn ngữ truy vấn cũ (học thuật 1980s), nền tảng cho các ngôn ngữ sau. Dữ liệu dạng predicate(subject, object); truy vấn xây từ rule. Dùng trong Datomic, Cascalog. | tr. 60–61 |
| **Rule (Datalog)** | Luật | Định nghĩa predicate mới suy ra từ dữ liệu hoặc rule khác, có thể đệ quy. Áp dụng khi mọi predicate vế phải của `:-` khớp. | tr. 61–62 |

### Bổ sung ngoài sách (liên quan chương 2)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **`$lookup` (MongoDB)** | Toán tử kiểu join trong aggregation pipeline của MongoDB, xuất hiện sau thời điểm sách viết. Hãy kiểm tra tài liệu phiên bản đang dùng. | Many-to-one, convergence |
| **Backfill** | Chạy nền theo từng lô nhỏ để đưa dữ liệu cũ về định dạng mới sau khi đổi schema, thay vì một `UPDATE` khổng lồ. | Schema flexibility |

<a id="chuong-3"></a>

## Chương 3 — Storage and Retrieval

### Nền tảng và log-structured storage

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Storage engine** | Bộ máy lưu trữ | Phần của database lo việc lưu dữ liệu và tìm lại. Có loại tối ưu cho OLTP, có loại cho analytics. | tr. 69 |
| **Log** **(G)** | Nhật ký | Trong sách: chuỗi bản ghi **chỉ ghi nối thêm**, không nhất thiết người đọc được. Khác với log ứng dụng. | tr. 71, 556 |
| **Append-only** | Chỉ ghi nối thêm | Chỉ thêm vào cuối file, không sửa dữ liệu đã ghi. | tr. 71, 75 |
| **Index** **(G)** | Chỉ mục | Cấu trúc phụ dẫn xuất từ dữ liệu chính, giúp tìm nhanh bản ghi theo giá trị của một trường. Đọc nhanh hơn, ghi chậm hơn. | tr. 71, 555 |
| **Hash map / hash table** | Bảng băm | Cấu trúc ánh xạ key tới value trong bộ nhớ. Dùng làm hash index trỏ tới byte offset trong file. | tr. 72 |
| **Hash** **(G)** | Hàm băm | Hàm biến input thành một số trông ngẫu nhiên; cùng input luôn cho cùng output; hai input khác nhau có thể trùng output (va chạm). | tr. 75, 555 |
| **Byte offset** | Vị trí byte | Vị trí trong file nơi value bắt đầu. | tr. 72 |
| **Bitcask** | — | Storage engine mặc định của Riak: hash index trong bộ nhớ + log trên đĩa. Mọi key phải vừa RAM. | tr. 72–73 |
| **Segment** | Đoạn | Một file log có kích thước giới hạn. Đầy thì đóng, ghi sang segment mới. | tr. 73 |
| **Compaction** | Nén gọn | Bỏ key trùng trong log, chỉ giữ cập nhật mới nhất của mỗi key. | tr. 73 |
| **Merge (segment)** | Gộp | Gộp nhiều segment thành một segment mới, thường cùng lúc với compaction, chạy ở nền. | tr. 73–74 |
| **Tombstone** | Bia mộ | Bản ghi xóa đặc biệt nối vào log. Báo quá trình merge bỏ các giá trị cũ của key đã xóa. | tr. 74 |
| **Checksum** | Mã kiểm tra | Dùng để phát hiện và bỏ qua bản ghi ghi dở khi crash. | tr. 75 |
| **Sequential vs random write** | Ghi tuần tự / ngẫu nhiên | Ghi tuần tự nhanh hơn nhiều, nhất là trên ổ đĩa từ. | tr. 75 |
| **Range query** | Truy vấn khoảng | Quét mọi key giữa một giá trị nhỏ nhất và một giá trị lớn nhất. Hash index không làm hiệu quả được. | tr. 75, 79 |
| **SSTable** (*Sorted String Table*) | Bảng chuỗi đã sắp xếp | Segment trong đó cặp key-value sắp theo key, mỗi key một lần. Merge kiểu mergesort, index thưa, nén theo khối. | tr. 76–77 |
| **Sparse index** | Index thưa | Index trong bộ nhớ chỉ chứa offset của một số key (vd mỗi vài KB một key). | tr. 77 |
| **Memtable** | Bảng trong bộ nhớ | Cây cân bằng trong bộ nhớ (red-black, AVL) nhận lượt ghi, được flush thành SSTable khi vượt vài MB. | tr. 78 |
| **LSM-tree** (*Log-Structured Merge-Tree*) | Cây gộp cấu trúc log | Cấu trúc index gồm memtable và chuỗi SSTable được merge nền (O'Neil và cộng sự). LevelDB, RocksDB, Cassandra, HBase. | tr. 78–79 |
| **Term dictionary / postings list** | Từ điển thuật ngữ / danh sách xuất hiện | Trong full-text index: key là từ, value là danh sách ID document chứa từ đó. Lucene lưu trong file kiểu SSTable. | tr. 79 |
| **Bloom filter** | Bộ lọc Bloom | Cấu trúc tiết kiệm bộ nhớ xấp xỉ một tập hợp. Cho biết key **không có**, tránh đọc đĩa vô ích. | tr. 79 |
| **Size-tiered / leveled compaction** | Compaction theo kích thước / theo tầng | Hai chiến lược compaction. Size-tiered: SSTable nhỏ, mới merge vào cái lớn, cũ. Leveled: chia khoảng key thành SSTable nhỏ theo level, từng bước hơn, ít tốn đĩa. | tr. 79 |

### B-tree và so sánh

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **B-tree** | Cây B | Cấu trúc index phổ biến nhất (1970). Cây các page cố định, giữ key có thứ tự, ghi đè page tại chỗ. | tr. 79–81 |
| **Page / block** | Trang / khối | Đơn vị kích thước cố định (thường 4 KB) mà B-tree đọc ghi mỗi lần. | tr. 80 |
| **Root / leaf page** | Trang gốc / trang lá | Root là nơi bắt đầu mọi lượt tra. Leaf chứa từng key với value (hoặc tham chiếu tới value). | tr. 80–81 |
| **Branching factor** | Hệ số phân nhánh | Số tham chiếu tới page con trong một page; thường vài trăm. | tr. 81 |
| **Page split** | Tách trang | Page đầy khi chèn thì tách thành hai page đầy một nửa, page cha được cập nhật. Giữ cây cân bằng, độ sâu O(log n). | tr. 81 |
| **WAL** (*write-ahead log*, *redo log*) | Nhật ký ghi trước | File chỉ ghi nối thêm; mọi thay đổi B-tree ghi vào đây trước khi áp dụng, để phục hồi sau crash. | tr. 82 |
| **Orphan page** | Trang mồ côi | Page không thuộc page cha nào, xuất hiện khi crash giữa lúc tách page. | tr. 82 |
| **Latch** | Chốt | Khóa nhẹ bảo vệ cấu trúc dữ liệu của cây khi nhiều thread truy cập. | tr. 82 |
| **Copy-on-write** | Sao chép khi ghi | Ghi page đã sửa ra chỗ mới và tạo phiên bản page cha mới, thay cho ghi đè + WAL (vd LMDB). | tr. 82–83 |
| **B+ tree** | — | Biến thể viết tắt key ở page bên trong để tăng branching factor; thường không được phân biệt với B-tree. | tr. 83 |
| **Fractal tree** | — | Biến thể B-tree mượn ý tưởng log-structured để giảm seek đĩa. | tr. 83 |
| **Write amplification** | Khuếch đại ghi | Một lần ghi vào database dẫn tới nhiều lần ghi xuống đĩa trong vòng đời dữ liệu. Quan trọng với workload ghi nhiều và SSD. | tr. 84 |
| **Fragmentation** | Phân mảnh | Dung lượng bị bỏ trống trong page (sau khi tách page, hoặc hàng không vừa). B-tree bị nhiều hơn LSM-tree. | tr. 84 |

### Các loại index khác và in-memory

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Primary key** **(G)** | Khóa chính | Giá trị định danh duy nhất một bản ghi; thường do hệ thống sinh ra. | tr. 85, 556 |
| **Secondary index** **(G)** | Chỉ mục phụ | Cấu trúc phụ duy trì cạnh dữ liệu chính để tìm bản ghi theo điều kiện khác. Key không duy nhất: value là danh sách ID hàng, hoặc nối ID hàng vào key. | tr. 85–86, 557 |
| **Heap file** | Tệp heap | Nơi lưu hàng không theo thứ tự; index chỉ chứa tham chiếu tới vị trí trong heap. | tr. 86 |
| **Forwarding pointer** | Con trỏ chuyển tiếp | Để lại ở vị trí cũ trong heap khi hàng phải dời sang chỗ mới vì to ra. | tr. 86 |
| **Clustered index** | Chỉ mục cụm | Lưu hàng ngay trong index. Primary key của InnoDB luôn là clustered index. | tr. 86 |
| **Covering index / index with included columns** | Chỉ mục phủ | Lưu thêm một số cột trong index để vài truy vấn trả lời chỉ bằng index. | tr. 86 |
| **Concatenated index** | Chỉ mục ghép | Index nhiều cột bằng cách nối cột này sau cột kia, như danh bạ (họ, tên). Chỉ giúp khi lọc theo cột đầu. | tr. 87 |
| **Multi-dimensional index** | Chỉ mục nhiều chiều | Truy vấn nhiều cột cùng lúc, nhất là dữ liệu không gian. | tr. 87 |
| **Space-filling curve** | Đường cong lấp không gian | Biến tọa độ nhiều chiều thành một số để dùng B-tree thường. | tr. 87 |
| **R-tree** | Cây R | Spatial index chuyên dụng (vd PostGIS trên GiST của PostgreSQL). | tr. 87 |
| **Fuzzy query** | Truy vấn mờ | Tìm key tương tự (sai chính tả, đồng nghĩa…), cần kỹ thuật khác index chính xác. | tr. 88 |
| **Edit distance** | Khoảng cách sửa | Số chữ cái được thêm, bớt hoặc thay để biến từ này thành từ kia. | tr. 88 |
| **Finite state automaton / trie / Levenshtein automaton** | Ôtômat hữu hạn / cây tiền tố | Cấu trúc Lucene dùng làm index trong bộ nhớ cho term dictionary, hỗ trợ tìm từ trong một edit distance. | tr. 88 |
| **In-memory database** | CSDL trong bộ nhớ | Phục vụ đọc hoàn toàn từ bộ nhớ; đĩa (nếu có) chỉ là log để bền. Nhanh vì tránh chi phí mã hóa cho đĩa. | tr. 88–89 |
| **Durable** **(G)** | Bền vững | Lưu dữ liệu sao cho tin rằng không mất dù có nhiều loại sự cố. | tr. 88–89, 554 |
| **Cache** **(G)** | Bộ đệm | Thành phần nhớ dữ liệu dùng gần đây để đọc lại nhanh hơn; không đầy đủ, thiếu thì phải lấy từ nơi lưu chậm hơn. | tr. 89, 553 |
| **Anti-caching** | Chống cache | Đẩy dữ liệu ít dùng nhất từ bộ nhớ xuống đĩa và tải lại khi cần, ở mức từng bản ghi. Index vẫn phải vừa RAM. | tr. 89–90 |
| **NVM** (*non-volatile memory*) | Bộ nhớ không mất dữ liệu khi mất điện | Công nghệ có thể làm thay đổi thiết kế storage engine trong tương lai. | tr. 90 |

### Analytics và lưu trữ theo cột

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Transaction** | Giao dịch | Một nhóm thao tác đọc và ghi tạo thành một đơn vị logic. Không nhất thiết có ACID. | tr. 90 |
| **OLTP** **(G)** | Xử lý giao dịch trực tuyến | Mẫu truy cập: truy vấn nhanh đọc hoặc ghi số ít bản ghi, thường theo key. | tr. 90, 556 |
| **OLAP** **(G)** | Xử lý phân tích trực tuyến | Mẫu truy cập: tổng hợp (count, sum, average) trên số lượng lớn bản ghi. | tr. 91, 556 |
| **Business intelligence** | Trí tuệ kinh doanh | Báo cáo từ truy vấn phân tích giúp ban lãnh đạo ra quyết định. | tr. 91 |
| **Data warehouse** **(G)** | Kho dữ liệu | Database riêng kết hợp dữ liệu từ nhiều hệ thống OLTP, chuẩn bị cho phân tích. | tr. 91–92, 554 |
| **ETL** **(G)** (*Extract–Transform–Load*) | Trích xuất–Biến đổi–Nạp | Lấy dữ liệu từ database nguồn, biến đổi sang dạng hợp với phân tích, nạp vào data warehouse. | tr. 92, 554 |
| **Drill-down / slicing and dicing** | Đi sâu / cắt lát | Thao tác khám phá dữ liệu trong công cụ phân tích. | tr. 93 |
| **SQL-on-Hadoop** | — | Dự án mã nguồn mở chạy SQL phân tích: Hive, Spark SQL, Impala, Presto, Tajo, Drill. | tr. 93 |
| **Star schema / dimensional modeling** | Lược đồ hình sao | Fact table ở giữa, dimension table bao quanh. | tr. 93–95 |
| **Fact table** | Bảng sự kiện | Mỗi hàng là một sự kiện; gồm thuộc tính và foreign key tới dimension. Có thể cực lớn. | tr. 93–94 |
| **Dimension table** | Bảng chiều | Mô tả ai, cái gì, ở đâu, khi nào, như thế nào, tại sao của sự kiện. Ngày giờ cũng thường là dimension. | tr. 94–95 |
| **Snowflake schema** | Lược đồ bông tuyết | Biến thể star schema: dimension chia tiếp thành subdimension. Chuẩn hóa hơn, nhưng star đơn giản hơn cho analyst. | tr. 95 |
| **Row-oriented storage** | Lưu theo hàng | Mọi giá trị của một hàng nằm cạnh nhau. Cách của hầu hết database OLTP. | tr. 96 |
| **Column-oriented storage** | Lưu theo cột | Mọi giá trị của một cột nằm cạnh nhau, mỗi cột một file; truy vấn chỉ đọc cột cần. | tr. 96–97 |
| **Parquet** | — | Định dạng lưu theo cột hỗ trợ mô hình document, dựa trên Dremel của Google. | tr. 96 |
| **Bitmap encoding** | Mã hóa bitmap | Mỗi giá trị khác nhau của cột thành một bitmap, mỗi hàng một bit. Lọc bằng AND, OR từng bit. | tr. 97–99 |
| **Run-length encoding (RLE)** | Mã hóa độ dài chuỗi | Lưu số lượng phần tử lặp liên tiếp thay vì từng phần tử. Nén bitmap thưa và cột đã sắp xếp. | tr. 98, 100 |
| **Column family** | Họ cột | Khái niệm của Bigtable, Cassandra, HBase. Gây hiểu lầm: thực chất vẫn lưu theo hàng. | tr. 99 |
| **Vectorized processing** | Xử lý vector hóa | Xử lý khúc dữ liệu cột nén vừa L1 cache trong vòng lặp chặt, toán tử chạy trực tiếp trên dữ liệu nén. | tr. 99 |
| **SIMD** | — | *Single-instruction-multi-data*: lệnh CPU xử lý nhiều dữ liệu cùng lúc. | tr. 99 |
| **Sort key** | Khóa sắp xếp | Cột dùng để sắp xếp hàng trong column store; vừa làm index vừa giúp nén. | tr. 100 |
| **Materialize** **(G)** | Vật chất hóa | Tính ngay và ghi ra kết quả, thay vì tính theo yêu cầu. | tr. 101, 556 |
| **Materialized view / virtual view** | View vật chất hóa / view ảo | Materialized: bản sao kết quả truy vấn trên đĩa, phải cập nhật khi dữ liệu gốc đổi. Virtual: lối tắt, mở rộng thành truy vấn gốc lúc chạy. | tr. 101–102 |
| **Data cube / OLAP cube** | Khối dữ liệu | Lưới tổng hợp theo các dimension; một loại materialized view. Đọc rất nhanh nhưng kém linh hoạt. | tr. 102–103 |

### Bổ sung ngoài sách (liên quan chương 3)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **False positive (Bloom filter)** | Bloom filter có thể báo "có thể có" cho key thực ra không có, nhưng không bao giờ báo sai rằng key không có. | Performance optimizations |
| **Z-order curve / Hilbert curve** | Hai space-filling curve hay dùng: điểm gần nhau trong không gian thường có số gần nhau. | Multi-dimensional indexes |

<a id="chuong-4"></a>

## Chương 4 — Encoding and Evolution

### Tương thích và mã hóa

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Rolling upgrade / staged rollout** | Nâng cấp cuốn chiếu | Triển khai bản mới lên vài node một lúc, kiểm tra rồi đi tiếp. Không downtime, nhưng code cũ và mới cùng chạy. | tr. 112, 139 |
| **Backward compatibility** | Tương thích ngược | Code **mới** đọc được dữ liệu do code **cũ** ghi. Thường dễ. | tr. 112 |
| **Forward compatibility** | Tương thích xuôi | Code **cũ** đọc được dữ liệu do code **mới** ghi. Khó hơn: phải bỏ qua phần thêm vào. | tr. 112 |
| **Encoding / serialization / marshalling** | Mã hóa | Biến cấu trúc dữ liệu trong bộ nhớ thành chuỗi byte tự chứa. Không phải encryption. Sách dùng chữ encoding để tránh nhầm với nghĩa khác của *serialization* trong transaction (Chương 7). | tr. 113 |
| **Decoding / parsing / deserialization / unmarshalling** | Giải mã | Biến chuỗi byte trở lại cấu trúc trong bộ nhớ. | tr. 113 |
| **Language-specific format** | Định dạng riêng của ngôn ngữ | `java.io.Serializable`, Ruby `Marshal`, Python `pickle`, Kryo. Tiện nhưng khóa ngôn ngữ, gây lỗ hổng bảo mật, bỏ qua versioning. | tr. 113–114 |
| **IEEE 754 double / 2⁵³** | Số thực độ chính xác kép | Số nguyên lớn hơn 2⁵³ không biểu diễn chính xác được; vấn đề khi JavaScript parse số lớn trong JSON. | tr. 114 |
| **Base64** | — | Mã hóa dữ liệu nhị phân thành văn bản; làm dữ liệu to thêm 33%. | tr. 114 |
| **XML Schema / JSON Schema** | — | Ngôn ngữ schema tùy chọn, mạnh nhưng phức tạp. | tr. 115, 127 |
| **Data interchange format** | Định dạng trao đổi dữ liệu | Định dạng gửi dữ liệu giữa các tổ chức; ở đây việc thống nhất quan trọng hơn hiệu quả. | tr. 115 |
| **Binary JSON variants** | Biến thể nhị phân của JSON | MessagePack, BSON, BJSON, UBJSON, BISON, Smile. Vẫn chứa tên trường. | tr. 115–116 |
| **MessagePack** | — | Mã hóa nhị phân cho JSON; bản ghi mẫu 66 byte so với 81 byte JSON. | tr. 116 |

### Định dạng nhị phân có schema

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Thrift / Protocol Buffers (protobuf)** | — | Thư viện mã hóa nhị phân cần schema (Facebook, Google; mở mã 2007–08). Dùng field tag. | tr. 117 |
| **IDL** (*interface definition language*) | Ngôn ngữ định nghĩa giao diện | Ngôn ngữ viết schema của Thrift, Protocol Buffers, Avro. | tr. 117 |
| **Code generation** | Sinh code | Từ schema sinh lớp trong nhiều ngôn ngữ. Hữu ích cho ngôn ngữ kiểu tĩnh, ít ý nghĩa cho kiểu động. | tr. 118, 127 |
| **BinaryProtocol / CompactProtocol** | — | Hai định dạng nhị phân của Thrift: 59 byte và 34 byte cho bản ghi mẫu. | tr. 118–119 |
| **Field tag** | Thẻ trường | Số định danh trường trong schema, thay cho tên trường trong dữ liệu. Đổi tên trường được, **không đổi tag được**, không dùng lại tag. | tr. 119–121 |
| **Variable-length integer (varint)** | Số nguyên độ dài thay đổi | Bit cao nhất mỗi byte báo còn byte nữa; số nhỏ tốn ít byte (1337 chỉ 2 byte). | tr. 119 |
| **required / optional** | Bắt buộc / tùy chọn | Không ảnh hưởng mã hóa; `required` chỉ bật kiểm tra lúc chạy. Trường thêm sau phải optional hoặc có mặc định. | tr. 120–121 |
| **repeated** | Lặp | Đánh dấu của Protocol Buffers thay cho kiểu list: cùng tag xuất hiện nhiều lần. Cho phép đổi optional thành repeated. | tr. 121 |
| **Schema evolution** | Tiến hóa schema | Thay đổi schema theo thời gian mà vẫn giữ backward và forward compatibility. | tr. 120 |
| **Avro** | — | Định dạng nhị phân (Hadoop, 2009), schema không có tag, dữ liệu chỉ là giá trị nối nhau; bản ghi mẫu 32 byte. | tr. 122–123 |
| **Avro IDL** | — | Ngôn ngữ schema của Avro cho người viết; ngoài ra có dạng JSON cho máy đọc. | tr. 122 |
| **Writer's schema** | Schema của người ghi | Phiên bản schema ứng dụng dùng khi mã hóa dữ liệu. | tr. 123 |
| **Reader's schema** | Schema của người đọc | Schema ứng dụng kỳ vọng khi giải mã. Chỉ cần tương thích với writer's schema; Avro phân giải theo tên trường. | tr. 123–124 |
| **Union type** | Kiểu hợp | Kiểu của Avro cho phép một trong nhiều kiểu, ví dụ {null, long}. Cách duy nhất để trường được là null. | tr. 125 |
| **Default value** | Giá trị mặc định | Điền khi reader cần trường mà dữ liệu không có. Avro chỉ cho thêm hoặc xóa trường có mặc định. | tr. 124 |
| **Alias** | Bí danh | Tên cũ của trường khai báo trong reader's schema; đổi tên trường nhờ alias là backward nhưng không forward compatible. | tr. 125 |
| **Object container file** | Tệp chứa đối tượng | Định dạng file Avro: writer's schema ở đầu file, rồi nhiều bản ghi. Tự mô tả. | tr. 125, 127 |
| **Schema version database** | CSDL phiên bản schema | Lưu các phiên bản schema; là tài liệu và cho phép kiểm tra tương thích. Phiên bản là số tăng dần hoặc hash. | tr. 125–126 |
| **Dynamically generated schema** | Schema sinh động | Schema sinh tự động từ nguồn khác (vd từ bảng database). Avro hợp hơn Thrift, protobuf. | tr. 126 |
| **Self-describing** | Tự mô tả | File chứa đủ metadata (schema) để đọc mà không cần thông tin ngoài. | tr. 127 |
| **ASN.1 / DER** | — | Ngôn ngữ schema chuẩn hóa từ 1984, có tag; DER dùng cho chứng chỉ X.509. Phức tạp, tài liệu kém. | tr. 127–128 |
| **ODBC / JDBC** | — | API mà driver database dùng để decode giao thức mạng riêng của database. | tr. 128 |

### Luồng dữ liệu

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Dataflow** | Luồng dữ liệu | Cách dữ liệu đi giữa các tiến trình: qua database, qua service, qua message. | tr. 128–129 |
| **Unknown fields** | Trường lạ | Trường do code mới thêm mà code cũ không biết; phải được giữ nguyên khi code cũ đọc, sửa, ghi lại. | tr. 129–130 |
| **Data outlives code** | Dữ liệu sống lâu hơn code | Code thay trong vài phút, nhưng dữ liệu cũ vẫn ở định dạng gốc trong database nhiều năm. | tr. 130 |
| **Migration** | Di chuyển dữ liệu | Ghi lại dữ liệu theo schema mới; tốn kém với dataset lớn. | tr. 130 |
| **Archival storage** | Lưu trữ lâu dài | Snapshot hoặc dump database, nên mã hóa bằng schema mới nhất; hợp với Avro hoặc Parquet. | tr. 131 |
| **Service** | Dịch vụ | API mà server phơi ra qua mạng; chỉ cho input, output định sẵn (đóng gói). | tr. 131–132 |
| **Ajax** | — | JavaScript trong trình duyệt dùng `XMLHttpRequest` làm HTTP client. | tr. 131 |
| **SOA / microservices** | Kiến trúc hướng dịch vụ / vi dịch vụ | Chia ứng dụng thành service nhỏ theo chức năng, triển khai và tiến hóa độc lập. | tr. 132 |
| **Web service** | Dịch vụ web | Service dùng HTTP làm giao thức nền. | tr. 132 |
| **Middleware** | Phần mềm trung gian | Phần mềm hỗ trợ service gọi service trong cùng tổ chức. | tr. 132 |
| **REST / RESTful** | — | Triết lý thiết kế trên nguyên tắc HTTP: URL định danh tài nguyên, dùng tính năng HTTP. | tr. 133 |
| **HATEOAS** | — | *Hypermedia as the engine of application state*; chủ đề tranh cãi trong phe REST. | tr. 133 |
| **SOAP / WS-\* / WSDL** | — | Giao thức XML cho request API; rừng tiêu chuẩn WS-\*; WSDL mô tả API và cho phép sinh code. | tr. 133 |
| **OpenAPI / Swagger** | — | Định dạng mô tả RESTful API và sinh tài liệu. | tr. 133 |
| **RPC** (*remote procedure call*) | Gọi thủ tục từ xa | Làm request tới service từ xa trông giống gọi hàm cục bộ. Có lỗi căn bản. | tr. 134 |
| **EJB / RMI / DCOM / CORBA** | — | Các công nghệ RPC cũ: giới hạn Java, giới hạn Microsoft, hoặc quá phức tạp và không tương thích phiên bản. | tr. 134 |
| **Location transparency** | Trong suốt về vị trí | Abstraction che giấu việc đối tượng ở cục bộ hay ở xa. Hợp với actor model hơn RPC. | tr. 134, 138 |
| **Timeout** | Hết thời gian chờ | Request trả về không có kết quả; không biết request đã tới nơi chưa. | tr. 134 |
| **Idempotent / idempotence** **(G)** | Lũy đẳng | Thao tác thử lại an toàn: thực hiện nhiều lần có cùng hiệu quả như một lần. Cần để thử lại request mạng. | tr. 134, 555 |
| **gRPC / Finagle / Rest.li** | — | Framework RPC thế hệ mới: gRPC dùng Protocol Buffers, Finagle dùng Thrift, Rest.li dùng JSON qua HTTP. | tr. 135 |
| **Future / promise** | — | Đóng gói hành động bất đồng bộ có thể thất bại; dễ gửi nhiều request song song. | tr. 135 |
| **Stream (gRPC)** | Luồng | Một lời gọi gồm chuỗi request và response theo thời gian. | tr. 135 |
| **Service discovery** | Khám phá dịch vụ | Cho client biết địa chỉ IP và cổng của service. | tr. 135 |
| **API versioning** | Đánh phiên bản API | Chưa có thống nhất: số phiên bản trong URL, trong header Accept, hoặc lưu theo API key phía server. | tr. 136 |
| **Asynchronous** **(G)** | Bất đồng bộ | Không chờ việc hoàn thành và không giả định nó mất bao lâu. | tr. 137, 553 |
| **Message broker / message queue / message-oriented middleware** | Trung gian tin nhắn | Trung gian lưu tạm message: đệm, gửi lại, tách rời người gửi và nhận. RabbitMQ, ActiveMQ, HornetQ, NATS, Kafka. | tr. 137 |
| **Queue / topic** | Hàng đợi / chủ đề | Đích có tên mà producer gửi message tới và consumer (subscriber) nhận từ đó. Topic chỉ một chiều. | tr. 137–138 |
| **Producer / consumer / subscriber** | Bên sản xuất / bên tiêu thụ / người đăng ký | Bên gửi và bên nhận message qua broker. | tr. 137 |
| **Reply queue** | Hàng đợi trả lời | Queue riêng để consumer gửi trả lời cho người gửi ban đầu, tạo luồng giống RPC. | tr. 138 |
| **Actor model** | Mô hình actor | Logic đóng gói trong actor có trạng thái riêng, giao tiếp bằng message bất đồng bộ, xử lý một message một lúc. | tr. 138 |
| **Distributed actor framework** | Framework actor phân tán | Dùng actor model qua nhiều node: Akka, Orleans, Erlang OTP. Vẫn cần tương thích để rolling upgrade. | tr. 138–139 |

### Bổ sung ngoài sách (liên quan chương 4)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **Zigzag encoding** | Đặt bit dấu ở cuối (mã hóa n thành 2n với số không âm) để số âm nhỏ cũng tốn ít byte. Thrift CompactProtocol và Avro dùng cách này: 1337 thành `f2 14`. | Varint |
| **Schema registry** | Dịch vụ lưu phiên bản schema; producer gắn ID schema vào message, consumer dùng ID lấy writer's schema; có thể từ chối schema phá tương thích. | Avro, message broker |
| **Idempotency key** | Khóa do client gửi kèm request để server nhận ra request trùng khi client thử lại. | RPC, idempotence |

<a id="chuong-5"></a>

## Chương 5 — Replication

Gồm cả thuật ngữ trong phần giới thiệu Part II (tr. 145–148).

### Phân tán dữ liệu (phần giới thiệu Part II)

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Shared-memory architecture** | Kiến trúc chia sẻ bộ nhớ | Nhiều CPU, RAM, đĩa ghép dưới một hệ điều hành, coi như một máy (scale up). Chi phí tăng nhanh hơn tuyến tính, chỉ ở một vị trí địa lý. | tr. 146 |
| **NUMA** (*nonuniform memory access*) | Truy cập bộ nhớ không đồng nhất | Trong máy lớn, có bank bộ nhớ gần CPU này hơn CPU khác; muốn hiệu quả vẫn phải chia việc theo vùng bộ nhớ, tức vẫn cần partitioning. | tr. 146 |
| **Shared-disk architecture** | Kiến trúc chia sẻ đĩa | Nhiều máy có CPU, RAM riêng nhưng dùng chung dãy đĩa (NAS, SAN). Tranh chấp và locking giới hạn khả năng mở rộng. | tr. 146 |
| **Hot-swappable** | Thay nóng | Linh kiện thay được (đĩa, RAM, CPU) mà không tắt máy. | tr. 146 |
| **Node** **(G)** | Nút | Một máy hay máy ảo chạy phần mềm database, giao tiếp với node khác qua mạng. | tr. 146, 556 |
| **Redundancy** | Dư thừa | Có nhiều bản sao để khi một máy hỏng thì máy khác thay. | tr. 145, 147 |
| **Replication** **(G)** | Nhân bản | Giữ bản sao của cùng dữ liệu trên nhiều node, để dữ liệu vẫn truy cập được khi có node không với tới. | tr. 147, 151, 556–557 |
| **Partitioning / sharding** **(G)** | Phân vùng | Chia dataset lớn thành các partition nhỏ hơn và đặt lên các node khác nhau (Chương 6). | tr. 147, 556 |

### Leader, follower và replication log

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Replica** | Bản sao | Node lưu một bản sao database. | tr. 152 |
| **Leader-based replication** (*active/passive*, *master–slave*) | Nhân bản dựa trên leader | Một replica nhận ghi, các replica khác áp dụng thay đổi theo đúng thứ tự từ replication log. | tr. 152 |
| **Leader / master / primary** **(G)** | Nút chủ | Replica được chỉ định nhận ghi. Có thể được bầu qua một giao thức hoặc do quản trị viên chọn. | tr. 152, 555 |
| **Follower / read replica / slave / secondary / hot standby** **(G)** | Nút theo | Replica không nhận ghi trực tiếp từ client, chỉ áp dụng thay đổi nhận từ leader. | tr. 152, 555 |
| **Replication log / change stream** | Log nhân bản | Chuỗi thay đổi leader gửi cho follower. Glossary gọi đây là một vai trò của log. | tr. 152, 556 |
| **Synchronous / asynchronous replication** **(G)** | Nhân bản đồng bộ / bất đồng bộ | Đồng bộ: leader chờ follower xác nhận rồi mới báo thành công. Bất đồng bộ: leader không chờ. | tr. 153–155, 553, 557 |
| **Semi-synchronous** | Bán đồng bộ | Một follower đồng bộ, các follower khác bất đồng bộ; dữ liệu luôn có trên ít nhất hai node. | tr. 154–155 |
| **Durable** **(G)** | Bền vững | Lưu dữ liệu sao cho tin rằng nó không mất dù có fault. Replication bất đồng bộ làm yếu durability. | tr. 155, 554 |
| **Chain replication** | Nhân bản chuỗi | Biến thể của replication đồng bộ, dùng trong Microsoft Azure Storage. | tr. 155 |
| **Consensus** **(G)** | Đồng thuận | Bài toán khiến nhiều node đồng ý về một giá trị, ví dụ node nào làm leader (Chương 9). | tr. 155, 157, 554 |
| **Consistent snapshot** | Ảnh chụp nhất quán | Trạng thái database tại một thời điểm, chụp mà không cần khóa toàn bộ; dùng để dựng follower mới và backup. | tr. 156 |
| **Log sequence number / binlog coordinates** | Vị trí trong log | Tên gọi vị trí chính xác trong replication log của PostgreSQL và MySQL. | tr. 156 |
| **Caught up / catch-up recovery** | Bắt kịp / phục hồi bắt kịp | Follower đã xử lý hết thay đổi tồn đọng; follower khởi động lại thì xin các thay đổi từ vị trí cuối đã xử lý. | tr. 156 |
| **Failover** **(G)** | Chuyển đổi dự phòng | Chuyển vai trò leader sang node khác khi leader hỏng: phát hiện, chọn leader mới, cấu hình lại. | tr. 157, 554 |
| **Timeout** **(G)** | Hết thời gian chờ | Cách thường dùng để đoán node đã chết; quá ngắn thì failover không cần thiết. | tr. 157–158, 557 |
| **Controller node** | Nút điều khiển | Node đã được bầu từ trước, có quyền chỉ định leader mới. | tr. 157 |
| **Split brain** **(G)** | Hai leader cùng lúc | Hai node cùng tin mình là leader và cùng nhận ghi; dễ mất hoặc hỏng dữ liệu. | tr. 158, 557 |
| **Fencing / STONITH** | Rào chắn / "bắn node kia" | Cơ chế tắt một node khi phát hiện hai leader (*Shoot The Other Node In The Head*). Bàn kỹ ở Chương 8. | tr. 158 |
| **Statement-based replication** | Nhân bản theo câu lệnh | Gửi câu lệnh SQL cho follower thực thi lại. Hỏng với hàm không tất định, thứ tự, side effect. | tr. 158–159 |
| **Nondeterministic function** | Hàm không tất định | Hàm cho kết quả khác nhau mỗi lần chạy, như `NOW()`, `RAND()`. | tr. 159 |
| **WAL shipping** | Gửi write-ahead log | Gửi chính WAL của storage engine cho follower. Gắn chặt với định dạng lưu trữ. | tr. 159–160 |
| **Zero-downtime upgrade** | Nâng cấp không dừng hệ thống | Nâng cấp follower trước rồi failover; chỉ làm được nếu giao thức replication cho phép lệch phiên bản. | tr. 160 |
| **Logical log / row-based replication** | Log logic / nhân bản theo hàng | Log thay đổi ở mức hàng, tách khỏi storage engine; cho phép khác phiên bản và dễ parse. | tr. 160 |
| **Change data capture** | Thu thập thay đổi dữ liệu | Dùng logical log để đưa thay đổi của database sang hệ thống khác (Chương 11). | tr. 161 |
| **Trigger-based replication** | Nhân bản bằng trigger | Trigger ghi thay đổi vào bảng riêng, tiến trình ngoài đọc và nhân bản. Linh hoạt nhưng tốn kém, dễ lỗi. | tr. 161 |

### Replication lag và các đảm bảo khi đọc

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Read-scaling architecture** | Kiến trúc mở rộng đọc | Thêm follower để tăng năng lực đọc; chỉ thực tế với replication bất đồng bộ. | tr. 161 |
| **Eventual consistency** | Nhất quán cuối cùng | Ngừng ghi và chờ thì các replica sẽ giống nhau; "cuối cùng" không có giới hạn thời gian. | tr. 162 |
| **Replication lag** | Độ trễ nhân bản | Khoảng trễ từ khi leader ghi tới khi follower phản ánh; thường dưới một giây, có thể lên vài phút. | tr. 162 |
| **Stale** | Lỗi thời | Giá trị cũ, chưa phản ánh lần ghi mới nhất. | tr. 162, 178 |
| **Read-after-write / read-your-writes consistency** | Đọc thấy ghi của chính mình | Người dùng tải lại trang luôn thấy cập nhật chính họ đã gửi; không hứa gì về người khác. | tr. 163 |
| **Logical timestamp** | Mốc thời gian logic | Giá trị chỉ thứ tự ghi (như log sequence number), không phụ thuộc đồng hồ. | tr. 164 |
| **Cross-device read-after-write consistency** | Đọc sau ghi trên nhiều thiết bị | Ghi trên thiết bị này thì xem trên thiết bị khác cũng thấy; cần metadata tập trung. | tr. 164 |
| **Monotonic reads** | Đọc đơn điệu | Đọc liên tiếp không thấy dữ liệu cũ hơn lần trước; mạnh hơn eventual, yếu hơn strong consistency. | tr. 165 |
| **Consistent prefix reads** | Đọc tiền tố nhất quán | Các lần ghi xảy ra theo thứ tự nào thì người đọc thấy theo thứ tự đó. | tr. 166 |
| **Causality** **(G)** | Quan hệ nhân quả | Phụ thuộc giữa các sự kiện khi sự kiện này "happens before" sự kiện kia, ví dụ câu trả lời sau câu hỏi. | tr. 165, 176, 554 |

### Multi-leader replication

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Multi-leader replication** (*master–master*, *active/active*) | Nhân bản nhiều leader | Nhiều node nhận ghi; mỗi leader đồng thời là follower của leader khác. | tr. 168 |
| **Multi-datacenter operation** | Vận hành nhiều datacenter | Mỗi datacenter một leader; ghi cục bộ, nhân bản bất đồng bộ giữa các datacenter. | tr. 168–169 |
| **Offline operation / sync** | Làm việc offline / đồng bộ | Mỗi thiết bị có database cục bộ làm leader; lag có thể hàng ngày. | tr. 170 |
| **Collaborative editing** | Cộng tác chỉnh sửa | Nhiều người sửa đồng thời một tài liệu; không khóa thì là multi-leader. | tr. 170–171 |
| **Write conflict** | Xung đột ghi | Hai lần ghi đồng thời trên hai leader sửa cùng dữ liệu theo cách khác nhau. | tr. 171 |
| **Conflict resolution** | Giải quyết xung đột | Quyết định giá trị cuối cùng khi có xung đột. | tr. 171–173 |
| **Conflict avoidance** | Tránh xung đột | Mọi lần ghi cho một bản ghi đi qua cùng một leader. Hỏng khi phải đổi leader nhà. | tr. 172 |
| **Convergent conflict resolution** | Giải quyết hội tụ | Mọi replica đi tới cùng giá trị cuối khi mọi thay đổi đã được nhân bản. | tr. 172–173 |
| **Last write wins (LWW)** | Ghi sau cùng thắng | Giữ lần ghi có timestamp lớn nhất, bỏ các lần ghi khác. Hội tụ nhưng âm thầm mất dữ liệu. | tr. 173, 186 |
| **Conflict handler (on write / on read)** | Bộ xử lý xung đột (khi ghi / khi đọc) | Code ứng dụng giải quyết xung đột: chạy nền khi phát hiện (on write), hoặc nhận mọi phiên bản ở lần đọc sau (on read). | tr. 173 |
| **CRDT** (*conflict-free replicated datatype*) | Kiểu dữ liệu nhân bản không xung đột | Cấu trúc dữ liệu (set, map, list, counter…) tự gộp các sửa đổi đồng thời theo cách hợp lý; dùng two-way merge. | tr. 174, 191 |
| **Mergeable persistent data structures** | Cấu trúc dữ liệu bền vững gộp được | Theo dõi lịch sử như Git, dùng three-way merge. | tr. 174 |
| **Operational transformation (OT)** | Biến đổi thao tác | Thuật toán của Etherpad, Google Docs cho sửa đồng thời danh sách có thứ tự. | tr. 174 |
| **Replication topology** | Topology nhân bản | Các đường truyền mà ghi lan giữa các node: circular, star (tree), all-to-all. | tr. 175 |
| **Single point of failure** | Điểm hỏng đơn | Một node hỏng làm gián đoạn cả luồng nhân bản; gặp ở circular và star. | tr. 176 |

### Leaderless replication và ghi đồng thời

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Leaderless replication / Dynamo-style** | Nhân bản không leader | Mọi replica nhận ghi trực tiếp; client hoặc coordinator gửi ghi và đọc tới nhiều replica. Riak, Cassandra, Voldemort. | tr. 177 |
| **Coordinator node** | Nút điều phối | Node gửi ghi tới các replica thay client, nhưng không áp đặt thứ tự ghi. | tr. 177 |
| **Read repair** | Sửa khi đọc | Client thấy replica stale khi đọc song song và ghi giá trị mới vào nó. | tr. 179 |
| **Anti-entropy process** | Tiến trình chống sai lệch | Tiến trình nền so dữ liệu giữa các replica và chép phần thiếu, không theo thứ tự. | tr. 179 |
| **Quorum** **(G)** | Túc số | Số node tối thiểu phải "bỏ phiếu" để một thao tác được coi là thành công. | tr. 179, 556 |
| **n, w, r** | — | Số replica, số xác nhận cần cho ghi, số node cần hỏi khi đọc. w + r > n thì tập ghi và tập đọc giao nhau. | tr. 179–181 |
| **Quorum reads and writes / strict quorum** | Đọc ghi theo túc số / túc số chặt | Đọc và ghi tuân theo r, w với w + r > n trên đúng n node "nhà". | tr. 179 |
| **Sloppy quorum** | Túc số lỏng | Cho tính cả node ngoài n node nhà để đủ w; tăng write availability nhưng không bảo đảm đọc thấy ghi mới nhất. | tr. 183–184 |
| **Hinted handoff** | Chuyển giao kèm gợi ý | Node tạm nhận ghi thay node khác chuyển dữ liệu về node nhà khi mạng phục hồi. | tr. 184 |
| **Staleness monitoring** | Giám sát độ lỗi thời | Đo xem replica cũ tới đâu; dễ với leader-based (vị trí log), khó với leaderless. | tr. 182–183 |
| **Clock skew** **(G, skew nghĩa 2)** | Lệch đồng hồ | Đồng hồ các máy lệch nhau, khiến LWW có thể làm mất ghi. | tr. 181, 557 |
| **Concurrent** | Đồng thời | Hai thao tác không thao tác nào biết về thao tác kia, bất kể thời gian vật lý. | tr. 186–187 |
| **Happens-before** | Xảy ra trước | A happens before B nếu B biết về, phụ thuộc vào hay xây trên A. | tr. 186–187 |
| **Causally dependent** | Phụ thuộc nhân quả | B xây trên A, nên phải xảy ra sau A. | tr. 186 |
| **Version number** | Số phiên bản | Server tăng mỗi lần ghi key; client ghi kèm version đã đọc để server biết giá trị nào được ghi đè. | tr. 189–190 |
| **Siblings** | Giá trị anh em | Các giá trị được ghi đồng thời mà server giữ lại để client gộp (tên gọi của Riak). | tr. 190 |
| **Tombstone** | Dấu xóa | Dấu kèm version báo món đã bị xóa, để khi gộp siblings món đó không xuất hiện lại. | tr. 191 |
| **Version vector** | Vector phiên bản | Tập version number của mọi replica cho một key; phân biệt ghi đè với ghi đồng thời. | tr. 191 |
| **Dotted version vector** | — | Biến thể version vector dùng trong Riak 2.0. | tr. 191 |
| **Causal context** | Ngữ cảnh nhân quả | Tên Riak đặt cho version vector được mã hóa thành chuỗi. | tr. 191 |
| **Vector clock** | Đồng hồ vector | Hay bị gọi lẫn với version vector nhưng không hoàn toàn giống; so sánh trạng thái replica thì dùng version vector. | tr. 191 |

### Bổ sung ngoài sách (liên quan chương 5)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **Merkle tree** | Cây hash; hai replica so hash từng nhánh để tìm nhanh phần dữ liệu lệch. Dynamo dùng cho anti-entropy. | Anti-entropy |
| **G-Counter** | CRDT bộ đếm chỉ tăng: mỗi replica một ô đếm, gộp bằng max từng ô, giá trị là tổng. | CRDT |
| **Nguyên lý chuồng bồ câu (pigeonhole principle)** | Lý do w + r > n buộc tập ghi và tập đọc giao nhau: \|W ∩ R\| ≥ w + r − n ≥ 1. | Quorum |
| **Majority quorum** | Với n lẻ, w = r = (n + 1)/2 chịu được (n − 1)/2 node hỏng. | Quorum |

<a id="chuong-6"></a>

## Chương 6 — Partitioning

### Khái niệm chung

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Partitioning / sharding** **(G)** | Phân vùng | Cố ý chia dataset (hoặc phép tính) quá lớn cho một máy thành các phần nhỏ và trải ra nhiều máy. Mỗi bản ghi thuộc đúng một partition; mỗi partition như một database nhỏ. | tr. 199, 556 |
| **Partition** | Phân vùng (một phần) | Một tập con của dataset. Tên gọi khác theo sản phẩm: *shard* (MongoDB, Elasticsearch, SolrCloud), *region* (HBase), *tablet* (Bigtable), *vnode* (Cassandra, Riak), *vBucket* (Couchbase). | tr. 199 |
| **Network partition / netsplit** | Phân mảnh mạng | Một loại lỗi mạng giữa các node; không liên quan tới partitioning dữ liệu (Chương 8). | tr. 199 |
| **Partitioning kết hợp replication** | Phân vùng kèm nhân bản | Mỗi partition có bản sao trên nhiều node; với leader–follower, mỗi node có thể là leader của một số partition và follower của partition khác. | tr. 200–201 |

### Phân vùng dữ liệu key-value

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Skew** **(G, nghĩa 1)** | Độ lệch tải | Phân vùng không công bằng: một số partition có nhiều dữ liệu hay truy vấn hơn hẳn, làm partitioning kém hiệu quả. | tr. 201, 557 |
| **Hot spot** | Điểm nóng | Partition (hay node) chịu tải cao bất thường; trường hợp cực đoan là mọi tải dồn vào một partition. Glossary coi là tên khác của skew nghĩa 1. | tr. 201, 557 |
| **Key range partitioning** | Phân vùng theo khoảng key | Mỗi partition giữ một khoảng key liên tục, như các tập bách khoa toàn thư; ranh giới thích nghi với dữ liệu. Range scan dễ, nhưng dễ hot spot (ví dụ key là timestamp). | tr. 202–203 |
| **Partition boundary** | Ranh giới phân vùng | Giá trị key (hoặc hash) phân chia hai partition kề nhau; do quản trị viên hoặc database chọn. | tr. 202, 204 |
| **Range scan / range query** | Quét theo khoảng | Đọc mọi key giữa hai giá trị; hiệu quả với key range partitioning, mất hiệu quả với hash partitioning. | tr. 202, 204 |
| **Hash partitioning** | Phân vùng theo hash của key | Mỗi partition giữ một khoảng giá trị hash; chia key đều hơn nhưng phá thứ tự key. Hàm hash phải tất định giữa các tiến trình, không cần mạnh về mật mã (MD5, Fowler–Noll–Vo). | tr. 203–204 |
| **Hash** **(G)** | Hàm băm | Hàm biến đầu vào thành số trông ngẫu nhiên, cùng đầu vào cho cùng số; ở chương này dùng để trải đều key lên các partition. | tr. 203, 555 |
| **Consistent hashing** | Băm nhất quán | Theo Karger và cộng sự: cách chia tải cho cache quy mô internet (CDN) bằng ranh giới chọn ngẫu nhiên, không cần điều khiển trung tâm. "Consistent" không liên quan tới consistency. Sách khuyên gọi là hash partitioning. | tr. 204, 213 |
| **Compound primary key** | Khóa chính ghép | Cassandra: chỉ băm phần đầu để chọn partition, phần sau dùng như concatenated index để sắp xếp. Range scan được trên phần sau khi cố định phần đầu, hợp với quan hệ một-nhiều. | tr. 204–205 |
| **Hot key splitting** | Tách key nóng | Thêm số ngẫu nhiên (ví dụ hai chữ số) vào đầu hoặc cuối key nóng để chia ghi ra nhiều key; đọc phải gộp mọi key con, và cần sổ sách theo dõi key bị tách. | tr. 205 |

### Secondary index trong database phân vùng

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Secondary index** **(G)** | Chỉ mục phụ | Tìm các lần xuất hiện của một giá trị, thường không định danh duy nhất bản ghi; không ánh xạ gọn vào partition vì bản ghi cùng giá trị rải khắp nơi. | tr. 206, 557 |
| **Document ID** | Định danh document | ID duy nhất của mỗi document (ví dụ mỗi tin rao xe), dùng làm khóa phân vùng trong ví dụ của sách. | tr. 206 |
| **Document-partitioned index / local index** | Index phân vùng theo document / index cục bộ | Mỗi partition giữ secondary index cho riêng document của nó. Ghi chỉ chạm một partition; đọc phải scatter/gather. | tr. 206–207 |
| **Scatter/gather** | Rải rồi gom | Gửi truy vấn tới mọi partition rồi gộp kết quả; tốn kém và dễ bị tail latency amplification. | tr. 207 |
| **Term-partitioned index / global index** | Index phân vùng theo term / index toàn cục | Index phủ mọi partition, được phân vùng theo term (hoặc hash của term). Đọc chỉ hỏi một partition; ghi một document có thể chạm nhiều partition; thường cập nhật bất đồng bộ. | tr. 208–209 |
| **Term** | Term (mục tra cứu) | Giá trị được tìm trong index, ví dụ color:red. Tên lấy từ full-text index (Glossary: một loại secondary index, tr. 555), nơi term là mọi từ trong document. | tr. 208 |
| **Distributed transaction** | Giao dịch phân tán | Transaction trải trên nhiều partition; cần để global index luôn cập nhật đồng bộ, nhưng không phải database nào cũng hỗ trợ (Chương 7, 9). | tr. 209 |

### Rebalancing

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Rebalancing** **(G)** | Tái cân bằng | Chuyển dữ liệu và request giữa các node để chia tải công bằng, khi throughput tăng, dataset lớn lên hay máy hỏng. Phải vẫn phục vụ đọc ghi và chuyển ít dữ liệu nhất có thể. | tr. 209, 556 |
| **Hash mod N** | Băm chia lấy dư | Gán key cho node bằng hash(key) mod N. Không nên dùng: đổi số node là phần lớn key phải chuyển. | tr. 210 |
| **Fixed number of partitions** | Số partition cố định | Tạo trước nhiều partition hơn số node, chỉ chuyển nguyên partition khi thêm bớt node. Số partition là số node tối đa; kích thước partition tăng theo dataset. Riak, Elasticsearch, Couchbase, Voldemort. | tr. 210–211 |
| **Dynamic partitioning** | Phân vùng động | Tách partition vượt ngưỡng (HBase mặc định 10 GB), gộp partition quá nhỏ; số partition thích nghi với lượng dữ liệu. HBase, RethinkDB, MongoDB. | tr. 212 |
| **Pre-splitting** | Tách trước | Cấu hình sẵn một tập partition trên database rỗng, để không phải dồn mọi lần ghi vào một node lúc đầu; với key range cần biết trước phân phối key. | tr. 212 |
| **Partitioning proportionally to nodes** | Phân vùng tỷ lệ với số node | Mỗi node có số partition cố định (Cassandra mặc định 256); node mới tách ngẫu nhiên partition cũ và nhận một nửa. Cần hash partitioning; gần nghĩa gốc của consistent hashing. Cassandra, Ketama. | tr. 212–213 |
| **Automatic / manual rebalancing** | Tái cân bằng tự động / thủ công | Một dải liên tục; Couchbase, Riak, Voldemort sinh đề xuất tự động nhưng cần quản trị viên xác nhận. Tự động hoàn toàn tiện nhưng khó lường. | tr. 213 |
| **Cascading failure** | Lỗi dây chuyền | Node quá tải bị coi là chết, rebalance tự động đặt thêm tải lên các node và mạng, làm thêm node hỏng theo. | tr. 214 |

### Request routing

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Request routing** | Định tuyến request | Xác định node nào giữ partition của một key để gửi request tới; khó vì cách gán partition thay đổi khi rebalance. | tr. 214 |
| **Service discovery** | Khám phá dịch vụ | Bài toán tổng quát mà request routing là một trường hợp: tìm địa chỉ IP và cổng của thành phần cần gọi. | tr. 214 |
| **Round-robin load balancer** | Bộ cân bằng tải xoay vòng | Chia request lần lượt cho các node; dùng trong cách gửi tới node bất kỳ, node đó chuyển tiếp nếu cần. | tr. 214 |
| **Routing tier / partition-aware load balancer** | Tầng định tuyến | Thành phần nhận mọi request và chuyển tới đúng node, bản thân không xử lý request. Ví dụ mongos (MongoDB), moxi (Couchbase). | tr. 214, 216 |
| **Partition-aware client** | Client biết phân vùng | Client tự biết cách gán partition cho node và kết nối thẳng tới đúng node. | tr. 214 |
| **Coordination service / ZooKeeper** | Dịch vụ điều phối | Giữ ánh xạ chính thức partition → node; node đăng ký, routing tier hay client đăng ký theo dõi và được báo khi có thay đổi. Espresso (qua Helix), HBase, SolrCloud, Kafka. | tr. 215–216 |
| **Gossip protocol** | Giao thức tin đồn | Các node lan truyền thay đổi trạng thái cluster cho nhau; Cassandra, Riak dùng thay cho coordination service ngoài. | tr. 216 |
| **MPP (massively parallel processing)** | Xử lý song song quy mô lớn | Database quan hệ cho phân tích; optimizer chia truy vấn phức tạp thành nhiều giai đoạn và partition chạy song song trên các node. | tr. 216 |

### Bổ sung ngoài sách (liên quan chương 6)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **Hash ring** | Mô hình vòng tròn của consistent hashing: cache (node) và key được băm lên cùng một vòng; key thuộc về node đầu tiên gặp theo chiều quy ước. | Consistent hashing |
| **Key salting** | Tên thường gặp của kỹ thuật thêm số ngẫu nhiên vào key nóng. | Hot key splitting |
| **Jump consistent hash** | Hàm hash của Lamping và Veach (tài liệu [8] của chương): ánh xạ key thẳng sang bucket 0…N − 1 không cần metadata; tăng N lên N + 1 chỉ chuyển khoảng 1/(N + 1) số key. | Proportional partitioning, hash mod N |
| **Tỷ lệ key phải chuyển với hash mod N** | Đổi từ N sang N + 1 node thì khoảng N/(N + 1) số key đổi node (từ 10 lên 11 node: khoảng 91%). | Hash mod N |

<a id="chuong-7"></a>

## Chương 7 — Transactions

### Transaction và ACID

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Transaction** **(G)** | Giao dịch | Gom nhiều lần đọc và ghi thành một đơn vị logic: hoặc commit toàn bộ, hoặc abort. Tạo ra để đơn giản hóa xử lý lỗi và vấn đề đồng thời cho ứng dụng. | tr. 222, 558 |
| **Commit / abort / rollback** | Xác nhận / hủy / hoàn tác | Commit: transaction thành công, mọi lần ghi có hiệu lực. Abort (rollback): transaction thất bại, mọi lần ghi bị bỏ, ứng dụng có thể thử lại. | tr. 222 |
| **Partial failure** (ngữ cảnh transaction) | Lỗi một phần | Một số thao tác thành công còn số khác thất bại; transaction giúp ứng dụng khỏi phải lo trường hợp này. | tr. 222 |
| **Safety guarantee** | Đảm bảo an toàn | Những kịch bản lỗi và đồng thời mà database lo thay ứng dụng khi dùng transaction. | tr. 222 |
| **Race condition** | Tình trạng tranh chấp | Kết quả sai do thứ tự thời gian xui xẻo giữa các client truy cập cùng dữ liệu. | tr. 221, 225 |
| **ACID** | — | Atomicity, Consistency, Isolation, Durability (Härder và Reuter, 1983). Trên thực tế ACID của mỗi database một khác, phần lớn đã thành thuật ngữ marketing. | tr. 223 |
| **BASE** | — | *Basically Available, Soft state, Eventual consistency*: tên gọi hệ không đạt ACID; còn mơ hồ hơn ACID, nghĩa hợp lý duy nhất là "không phải ACID". | tr. 223 |
| **Atomic / atomicity** **(G)** | Tính nguyên tử | Glossary có hai nghĩa: (1) trong đồng thời, thao tác không bao giờ bị thấy ở trạng thái làm dở; (2) trong transaction, nhóm lần ghi phải cùng commit hoặc cùng rollback dù có fault. ACID dùng nghĩa 2. | tr. 223–224, 553 |
| **Abortability** | Khả năng hủy | Tên mà tác giả cho là mô tả đúng hơn atomicity của ACID: lỗi thì abort và bỏ mọi lần ghi, nên retry an toàn. | tr. 224 |
| **Consistency** (nghĩa ACID) | Tính nhất quán | Database ở "trạng thái tốt" theo invariant của ứng dụng. Là thuộc tính của ứng dụng, không phải của database; một trong ít nhất bốn nghĩa của chữ *consistency*. | tr. 224–225 |
| **Invariant** | Bất biến | Phát biểu về dữ liệu luôn phải đúng, ví dụ ghi có và ghi nợ cân bằng. | tr. 225 |
| **Isolation** **(G)** | Tính cô lập | Mức độ các transaction chạy đồng thời có thể can thiệp lẫn nhau. Theo sách giáo khoa là serializability; thực tế thường dùng mức yếu hơn. | tr. 225, 555 |
| **Durability / durable** **(G)** | Tính bền vững | Dữ liệu của transaction đã commit không bị quên dù có fault: ghi vào bộ nhớ không bay hơi kèm WAL, hoặc nhân bản tới đủ node, trước khi báo commit. Không có durability tuyệt đối. | tr. 226–227, 554 |
| **Nonvolatile storage** | Bộ nhớ không bay hơi | Ổ cứng, SSD: giữ dữ liệu khi mất điện. | tr. 226 |
| **Correlated fault** | Lỗi tương quan | Mất điện hay bug hạ mọi replica cùng lúc; lý do ghi đĩa vẫn cần cho in-memory database. | tr. 227 |

### Single-object, multi-object và xử lý lỗi

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Multi-object transaction** | Transaction đa đối tượng | Transaction sửa nhiều object (hàng, document) cần giữ đồng bộ: foreign key, dữ liệu denormalized, secondary index. | tr. 228, 231 |
| **Single-object write** | Ghi một đối tượng | Storage engine hầu như luôn cho atomicity (bằng log) và isolation (bằng khóa) trên một object ở một node. | tr. 230 |
| **Denormalize** **(G)** | Phi chuẩn hóa | Thêm dư thừa (như bộ đếm thư chưa đọc) để đọc nhanh; buộc phải cập nhật nhiều chỗ cùng lúc. | tr. 228, 231, 554 |
| **Multi-put** | Ghi nhiều key | API ghi nhiều key một lần của key-value store; không nhất thiết có ngữ nghĩa transaction. | tr. 230 |
| **Read-modify-write cycle** | Chu trình đọc–sửa–ghi | Đọc giá trị, tính giá trị mới ở ứng dụng, ghi lại; nguồn gốc của lost update. | tr. 230, 243 |
| **Compare-and-set (CAS)** | So sánh rồi gán | Chỉ cho ghi nếu giá trị chưa đổi kể từ lần đọc; không an toàn nếu điều kiện đọc từ snapshot cũ. | tr. 230, 245 |
| **Lightweight transaction** | "Transaction nhẹ" | Tên marketing cho compare-and-set và thao tác single-object khác; gây hiểu lầm vì chúng không phải transaction. | tr. 230 |
| **Exponential backoff** | Lùi thời gian theo cấp số nhân | Giãn dần khoảng chờ giữa các lần retry để retry không làm quá tải tệ hơn. | tr. 232 |
| **Deduplication** | Khử trùng lặp | Cơ chế ở ứng dụng để transaction đã commit mà client tưởng thất bại không bị chạy hai lần khi retry. | tr. 232 |
| **Best effort** | Cố gắng hết mức | Kiểu làm việc của datastore leaderless: làm được bao nhiêu thì làm, gặp lỗi không hoàn tác phần đã làm. | tr. 232 |

### Isolation yếu và các anomaly

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Weak (nonserializable) isolation level** | Mức cô lập yếu | Mức chỉ chống một số vấn đề đồng thời; khó hiểu, gây bug tinh vi, nhưng phổ biến vì serializable tốn hiệu năng. | tr. 233 |
| **Read uncommitted** | Đọc chưa commit | Mức yếu hơn read committed: chặn dirty write nhưng không chặn dirty read. | tr. 234 |
| **Read committed** | Đọc đã commit | Không dirty read, không dirty write. Mặc định ở Oracle 11g, PostgreSQL, SQL Server 2012, MemSQL (lúc viết sách). | tr. 234–237 |
| **Dirty read** | Đọc bẩn | Thấy lần ghi chưa commit của transaction khác. | tr. 234 |
| **Dirty write** | Ghi bẩn | Ghi đè giá trị chưa commit của transaction khác; gần như mọi bản cài transaction đều chặn. | tr. 235 |
| **Row-level lock** | Khóa mức hàng | Khóa giữ trên một hàng hay document tới khi transaction commit hay abort; cách thường dùng để chặn dirty write. | tr. 236 |
| **Read skew / nonrepeatable read** **(G, skew nghĩa 2)** | Đọc lệch thời điểm | Thấy các phần database ở các thời điểm khác nhau (Alice thấy 900 thay vì 1.000). Read committed cho phép, snapshot isolation chặn. Glossary xếp vào nghĩa "timing anomaly" của *skew*. | tr. 238, 557 |
| **Snapshot isolation** | Cô lập theo ảnh chụp | Mỗi transaction đọc từ consistent snapshot lúc bắt đầu; hữu ích cho backup, analytics. | tr. 238–239 |
| **Consistent snapshot** | Ảnh chụp nhất quán | Mọi dữ liệu đã commit tại một thời điểm, đóng băng cho transaction. | tr. 238 |
| **Repeatable read** | Đọc lặp lại được | Tên trong chuẩn SQL; PostgreSQL, MySQL dùng cho snapshot isolation, DB2 dùng cho serializability. "Không ai thật sự biết nó nghĩa là gì". | tr. 242 |
| **Lost update** | Mất cập nhật | Hai chu trình read-modify-write đồng thời, lần ghi sau đè mất lần sửa trước. | tr. 243 |
| **Clobber** | Đè nát | Lần ghi sau ghi đè lần ghi trước mà không gộp thay đổi của nó. | tr. 243 |
| **Atomic write operation** | Thao tác ghi nguyên tử | Database tự làm read-modify-write (ví dụ tăng counter trong UPDATE); thường là cách chống lost update tốt nhất. | tr. 243 |
| **Cursor stability** | — | Cài thao tác atomic bằng khóa độc quyền trên object khi đọc. | tr. 244 |
| **Explicit locking / SELECT FOR UPDATE** | Khóa tường minh | Ứng dụng khóa các hàng truy vấn trả về trước khi read-modify-write. | tr. 244, 248 |
| **Automatic lost update detection** | Tự phát hiện mất cập nhật | Cho chạy song song, abort transaction gây lost update; có ở PostgreSQL RR, Oracle serializable, SQL Server SI, không có ở MySQL/InnoDB RR. | tr. 245 |
| **Commutative operation** | Thao tác giao hoán | Áp dụng theo thứ tự khác nhau vẫn ra cùng kết quả (tăng counter, thêm vào set); chống lost update khi nhân bản. | tr. 246, 262 |
| **Write skew** **(G, skew nghĩa 2)** | Ghi lệch | Hai transaction đọc cùng tập object rồi ghi object khác nhau, mỗi bên dựa trên tiền đề bên kia vừa phá. Tổng quát hóa của lost update; chỉ serializable chặn tự động. | tr. 247–248, 557 |
| **Phantom** | Bóng ma | Lần ghi của transaction này làm đổi kết quả truy vấn tìm kiếm của transaction khác; khi kiểm tra sự vắng mặt của hàng thì không có gì để khóa. | tr. 251 |
| **Materializing conflicts** | Vật chất hóa xung đột | Tạo bảng khóa nhân tạo (phòng × khung giờ) để phantom thành xung đột khóa trên hàng có thật; phương án cuối. | tr. 251 |

### MVCC và snapshot

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **MVCC** (*multi-version concurrency control*) | Kiểm soát đồng thời đa phiên bản | Giữ nhiều phiên bản đã commit của object để các transaction thấy database ở các thời điểm khác nhau. | tr. 239 |
| **Transaction ID (txid)** | Mã transaction | Số duy nhất, luôn tăng, gán cho transaction khi bắt đầu; ở PostgreSQL là 32 bit, tràn sau khoảng 4 tỷ transaction (vacuum xử lý). | tr. 239 |
| **created_by / deleted_by** | — | Hai trường của mỗi hàng trong MVCC của PostgreSQL: txid đã tạo và txid đã đánh dấu xóa. Update = delete + create. | tr. 240 |
| **Visibility rules** | Quy tắc hiển thị | Bỏ qua lần ghi của transaction đang chạy lúc bắt đầu, đã abort, hoặc có txid lớn hơn; mọi lần ghi khác thấy được. | tr. 240–241 |
| **Garbage collection** (MVCC) | Thu gom rác | Xóa các phiên bản không còn transaction nào thấy, kèm entry index tương ứng. | tr. 240–241 |
| **Append-only / copy-on-write B-tree** | B-tree chỉ ghi nối thêm | Không ghi đè page mà sao chép page bị sửa và các page tổ tiên; mỗi gốc mới là một snapshot (CouchDB, Datomic, LMDB). | tr. 242 |
| **Readers never block writers, writers never block readers** | — | Nguyên tắc hiệu năng then chốt của snapshot isolation (và SSI), khác 2PL. | tr. 239, 257 |

### Serializability

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Serializable / serializability** **(G)** | Khả tuần tự | Transaction đồng thời cho kết quả như chạy lần lượt theo một thứ tự nào đó; mức isolation mạnh nhất, chặn mọi race condition. Khác *serialization* (mã hóa, Chương 4). | tr. 225, 252, 557 |
| **Actual serial execution** | Thực thi tuần tự thật sự | Chạy mỗi lúc một transaction trên một thread; khả thi khi dữ liệu trong bộ nhớ, transaction ngắn (VoltDB/H-Store, Redis, Datomic). | tr. 252–256 |
| **Interactive transaction** | Transaction tương tác | Ứng dụng gửi từng câu lệnh qua mạng, đọc kết quả rồi gửi tiếp; không hợp với thực thi tuần tự. | tr. 253–254 |
| **Stored procedure** **(G)** | Thủ tục lưu trữ | Logic của transaction gửi trước cho database và chạy trọn ở server, không qua lại với client. | tr. 254–255, 557 |
| **Deterministic** **(G)** | Tất định | Cùng input luôn cho cùng output; VoltDB đòi stored procedure tất định để chạy lại trên mọi replica. | tr. 255, 554 |
| **Cross-partition transaction** | Transaction xuyên partition | Phải phối hợp lock-step trên mọi partition; VoltDB khoảng 1.000 ghi mỗi giây, không tăng khi thêm máy. | tr. 256 |
| **Anti-caching** | — | Transaction cần dữ liệu không ở trong bộ nhớ thì abort, nạp dữ liệu bất đồng bộ, rồi chạy lại. | tr. 256 |
| **Two-phase locking (2PL)** **(G)** | Khóa hai pha | Đọc lấy shared lock, ghi lấy exclusive lock, giữ mọi khóa tới hết transaction; người đọc và người ghi chặn nhau. | tr. 257–258, 558 |
| **SS2PL** (*strong strict two-phase locking*) | — | Tên gọi khác của dạng 2PL mà sách mô tả (giữ mọi khóa tới hết transaction), để phân biệt với các biến thể 2PL khác. | tr. 257 |
| **Two-phase commit (2PC)** **(G)** | Commit hai pha | Thuật toán để nhiều node cùng commit hay cùng abort; hoàn toàn khác 2PL (Chương 9). | tr. 232, 257, 558 |
| **Lock** **(G)** | Khóa | Cơ chế bảo đảm mỗi lúc chỉ một thread, node hay transaction truy cập một thứ; người khác phải chờ. | tr. 236, 258, 556 |
| **Shared lock / exclusive lock** | Khóa chia sẻ / khóa độc quyền | Nhiều transaction cùng giữ được shared lock để đọc; exclusive lock để ghi, không ai khác được giữ khóa cùng lúc. | tr. 258 |
| **Deadlock** | Khóa chết | Hai transaction chờ khóa của nhau; database phát hiện và abort một bên, ứng dụng phải retry. | tr. 258 |
| **Predicate lock** | Khóa vị từ | Khóa trên mọi object khớp một điều kiện tìm kiếm, kể cả object chưa tồn tại; chặn phantom. | tr. 259–260 |
| **Index-range locking / next-key locking** | Khóa khoảng index | Xấp xỉ predicate lock bằng khóa một entry hay khoảng giá trị trên index; rộng hơn nhưng rẻ hơn. Không có index thì khóa cả bảng. | tr. 260–261 |
| **Pessimistic concurrency control** | Kiểm soát đồng thời bi quan | Có gì có thể sai thì chờ tới khi an toàn (2PL; thực thi tuần tự là bi quan cực độ). | tr. 261 |
| **Optimistic concurrency control** | Kiểm soát đồng thời lạc quan | Cứ chạy tiếp, kiểm tra lúc commit, abort nếu vi phạm; kém khi tranh chấp cao, tốt khi còn dư năng lực. | tr. 261–262 |
| **Mutual exclusion** | Loại trừ lẫn nhau | Cơ chế bảo vệ cấu trúc dữ liệu trong lập trình đa luồng; 2PL giống nó. | tr. 261 |
| **Contention** | Tranh chấp | Nhiều transaction cùng truy cập một object; làm optimistic concurrency control abort nhiều. | tr. 262 |
| **Serializable snapshot isolation (SSI)** | Cô lập ảnh chụp khả tuần tự | Thuật toán optimistic trên nền snapshot isolation (2008); PostgreSQL serializable từ 9.1, FoundationDB. | tr. 261–266 |
| **Premise / outdated premise** | Tiền đề / tiền đề lỗi thời | Sự thật đúng lúc transaction bắt đầu mà quyết định ghi dựa vào; SSI abort transaction có thể đã hành động trên tiền đề lỗi thời. | tr. 262–263 |
| **Stale MVCC read** | Đọc phiên bản MVCC lỗi thời | Lần đọc đã bỏ qua lần ghi chưa commit mà về sau commit; SSI kiểm tra lúc commit. | tr. 263–264 |
| **Tripwire** (SSI) | Dây báo động | Khóa của SSI trên index không chặn ai, chỉ báo cho transaction đã đọc rằng dữ liệu của chúng có thể lỗi thời. | tr. 265 |

### Bổ sung ngoài sách (liên quan chương 7)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **Idempotency key** | Mã yêu cầu duy nhất client gửi kèm; server ghi lại và bỏ qua yêu cầu trùng mã, để retry không làm thao tác chạy hai lần. Glossary của sách định nghĩa *idempotent* ở tr. 555. | Retry, deduplication |
| **First-updater-wins** | Quy tắc của PostgreSQL ở repeatable read: transaction sửa một hàng đã bị transaction đồng thời sửa và commit thì nhận lỗi serialization. | Tự phát hiện lost update |
| **HOT** (*heap-only tuple*) | Tối ưu của PostgreSQL: phiên bản mới nằm cùng page và không cột được index nào đổi thì index không cần entry mới. | Index trong MVCC |
| **Exclusion constraint** | Ràng buộc của PostgreSQL dùng với range type: cấm hai hàng có khoảng chồng lấn, ví dụ hai lượt đặt cùng phòng. | Write skew đặt phòng |
| **Bloat** | Bảng phình to vì phiên bản cũ chưa được dọn, thường do transaction mở quá lâu giữ chân garbage collection. | MVCC |

<a id="chuong-8"></a>

## Chương 8 — The Trouble with Distributed Systems

### Partial failure và chịu lỗi

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Distributed** **(G)** | Phân tán | Chạy trên nhiều node nối với nhau qua mạng. Đặc trưng bởi partial failure: một phần hỏng trong khi phần khác vẫn chạy, và phần mềm thường không biết chính xác cái gì hỏng. | tr. 274–275, 554 |
| **Deterministic / nondeterministic** **(G)** | Tất định / không tất định | Tất định: cùng input luôn cho cùng output. Một máy chạy đúng thì tất định; partial failure trong hệ phân tán thì không tất định. | tr. 274–275, 554 |
| **Partial failure** | Hỏng một phần | Một số phần của hệ thống hỏng theo cách khó lường trong khi phần khác chạy tốt; có thể không biết một thao tác đã thành công hay chưa. Là đặc trưng định nghĩa của hệ phân tán. | tr. 275, 310 |
| **High-performance computing (HPC)** | Tính toán hiệu năng cao | Siêu máy tính hàng nghìn CPU cho tính toán khoa học. Xử lý lỗi bằng checkpoint, dừng cả cluster và chạy lại, tức biến partial failure thành total failure. | tr. 275 |
| **Cloud computing** | Điện toán đám mây | Thường gắn với datacenter nhiều khách thuê, máy phổ thông nối bằng IP, cấp tài nguyên co giãn, tính tiền theo mức dùng. | tr. 275 |
| **Multi-tenant** | Nhiều khách thuê | Tài nguyên (đường truyền, switch, card mạng, CPU) được chia sẻ giữa nhiều khách hàng; dẫn tới độ trễ biến động. | tr. 275, 284 |
| **Checkpoint** | Điểm lưu | Trạng thái tính toán được ghi định kỳ xuống storage bền vững để chạy lại từ đó khi có lỗi. | tr. 275 |
| **Commodity machine** | Máy phổ thông | Máy rẻ, cùng hiệu năng với giá thấp hơn nhờ quy mô, nhưng tỷ lệ hỏng cao hơn phần cứng chuyên dụng. | tr. 276 |
| **RDMA** (*remote direct memory access*) | Truy cập bộ nhớ từ xa trực tiếp | Cách các node siêu máy tính giao tiếp, bên cạnh shared memory. | tr. 276 |
| **Clos topology / bisection bandwidth** | Topology Clos / băng thông chia đôi | Cách xếp mạng datacenter nhiều tầng switch để có băng thông cao giữa hai nửa bất kỳ của mạng. | tr. 276 |
| **Fault-tolerant** **(G)** | Chịu lỗi | Tự phục hồi khi có sự cố (máy crash, link mạng hỏng). Chương 8: phải đưa chịu lỗi vào phần mềm, xây hệ tin cậy từ thành phần không tin cậy. | tr. 276, 555 |
| **Error-correcting code** | Mã sửa lỗi | Cho truyền dữ liệu chính xác qua kênh đôi khi làm sai bit; ví dụ về việc xây hệ tin cậy hơn từ nền kém tin cậy, nhưng có giới hạn. | tr. 277 |

### Mạng không tin cậy

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Asynchronous packet network** / **Asynchronous** **(G)** | Mạng gói bất đồng bộ | Mạng không đảm bảo khi nào gói tin tới hay có tới không (internet, Ethernet). Glossary: không chờ việc gì hoàn tất và không giả định nó mất bao lâu. | tr. 278, 553 |
| **Network partition / netsplit** | Phân mảnh mạng | Một phần mạng bị cắt khỏi phần còn lại. Sách thường gọi chung là *network fault* để tránh nhầm với partition (shard) ở Chương 6. | tr. 279 |
| **Network fault** | Sự cố mạng | Thuật ngữ chung sách dùng cho mọi sự cố mạng; khá phổ biến kể cả trong datacenter (khoảng 12 sự cố mỗi tháng trong một nghiên cứu). | tr. 279–280 |
| **Timeout** **(G)** | Hết thời gian chờ | Cách thường dùng (và gần như duy nhất chắc chắn có) để phát hiện fault, nhưng không phân biệt lỗi mạng với lỗi node và không có giá trị "đúng". | tr. 279, 281, 287, 557 |
| **Out of rotation** | Rút khỏi vòng phục vụ | Load balancer ngừng gửi request tới node đã chết. | tr. 280 |
| **RST / FIN, ICMP Destination Unreachable** | Gói từ chối kết nối / gói báo không với tới | Tín hiệu lỗi tường minh từ hệ điều hành hay router; có ích nhưng không thể trông cậy. | tr. 280–281 |
| **Cascading failure** | Lỗi dây chuyền | Khai tử nhầm node chỉ chậm vì quá tải làm dồn tải sang node khác, cực đoan là mọi node khai tử lẫn nhau. | tr. 281 |
| **Bounded / unbounded delay** **(G)** | Độ trễ có giới hạn / không giới hạn | Có hay không có giới hạn trên biết trước. Mạng gói có độ trễ không giới hạn; mạng circuit có độ trễ có giới hạn. | tr. 282, 285, 553, 558 |
| **Network congestion** | Nghẽn mạng | Gói tin phải chờ mới có chỗ trên đường truyền bận; hàng đợi switch đầy thì gói bị bỏ. | tr. 282 |
| **Queueing** | Xếp hàng | Nguyên nhân chính khiến độ trễ mạng biến động: ở switch, ở hệ điều hành máy đích, ở hypervisor, ở bên gửi. | tr. 282–284 |
| **Flow control / congestion avoidance / backpressure** **(G)** | Kiểm soát luồng | Bên gửi tự giới hạn tốc độ để không làm quá tải mạng hay bên nhận; tạo thêm hàng đợi ở bên gửi. | tr. 282–283, 553, 555 |
| **TCP retransmission** | Gửi lại của TCP | Gói không được xác nhận trong một timeout (tính từ round-trip time) thì được gửi lại; ứng dụng không thấy mất gói nhưng thấy độ trễ. | tr. 283 |
| **UDP** | — | Không flow control, không gửi lại; hợp khi dữ liệu trễ là vô giá trị (VoIP, hội nghị video). | tr. 283 |
| **Noisy neighbor** | Hàng xóm ồn ào | Khách thuê khác dùng nhiều tài nguyên chung làm độ trễ của bạn biến động. | tr. 284 |
| **Jitter** | Độ biến động độ trễ | Mức dao động của response time, được đo để tự điều chỉnh timeout. | tr. 284 |
| **Phi Accrual failure detector** | Bộ phát hiện lỗi Phi Accrual | Đo liên tục response time và jitter, tự điều chỉnh ngưỡng khai tử theo phân phối quan sát được (Akka, Cassandra). | tr. 284 |
| **Circuit / circuit-switched network** | Mạch / mạng chuyển mạch kênh | Giữ chỗ băng thông cố định dọc toàn tuyến trong suốt cuộc gọi (ISDN: 16 bit mỗi 250 µs); không xếp hàng nên độ trễ có giới hạn. | tr. 284–285 |
| **Synchronous network** / **Synchronous** **(G)** | Mạng đồng bộ | Mạng có độ trễ tối đa cố định nhờ giữ chỗ tài nguyên. Glossary: ngược với asynchronous. | tr. 285, 557 |
| **Packet-switched** | Chuyển mạch gói | Ethernet và IP: gói tận dụng băng thông động, tối ưu cho lưu lượng bùng nổ, nhưng xếp hàng và độ trễ không giới hạn. | tr. 285 |
| **Bursty traffic** | Lưu lượng bùng nổ | Web, email, truyền file: không có yêu cầu băng thông cố định, chỉ cần xong càng nhanh càng tốt. | tr. 285 |
| **ATM, InfiniBand** | — | Mạng lai giữa circuit và packet switching; InfiniBand có flow control đầu cuối ở tầng liên kết. | tr. 285–286 |
| **Quality of service (QoS) / admission control** | Chất lượng dịch vụ / kiểm soát nhận vào | Ưu tiên và lập lịch gói tin, giới hạn tốc độ bên gửi; giả lập được circuit switching hoặc độ trễ có giới hạn theo thống kê. Chưa bật trên cloud và internet. | tr. 286–287 |
| **Static / dynamic resource partitioning** | Chia tài nguyên tĩnh / động | Tĩnh: đảm bảo độ trễ nhưng tận dụng thấp, đắt. Động: tận dụng cao, rẻ, nhưng độ trễ biến động. | tr. 286 |

### Đồng hồ không tin cậy

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Time-of-day clock / wall-clock time** | Đồng hồ giờ trong ngày | Trả ngày giờ theo lịch (số giây từ epoch, không tính leap second), đồng bộ bằng NTP; có thể bị đặt lại và nhảy lùi. Dùng cho thời điểm. | tr. 288 |
| **Epoch** | Mốc thời gian | Nửa đêm UTC ngày 1/1/1970 theo lịch Gregory, mốc tính của nhiều time-of-day clock. | tr. 288 |
| **Monotonic clock** | Đồng hồ đơn điệu | Đảm bảo chỉ tiến lên; hiệu hai lần đọc trên cùng máy là thời gian đã trôi qua; giá trị tuyệt đối vô nghĩa, không so sánh giữa máy. Dùng cho timeout, response time. | tr. 288–289 |
| **NTP** (*Network Time Protocol*) | Giao thức thời gian mạng | Chỉnh đồng hồ theo giờ của một nhóm server; độ chính xác bị giới hạn bởi độ trễ mạng. | tr. 288–290 |
| **Slewing** | Chỉnh tốc độ đồng hồ | NTP tăng hoặc giảm tốc độ đồng hồ (mặc định tối đa 0,05%) thay vì làm nó nhảy. | tr. 289 |
| **Clock drift** | Đồng hồ trôi | Thạch anh chạy nhanh hay chậm hơn đúng, thay đổi theo nhiệt độ; Google giả định 200 ppm (6 ms nếu đồng bộ mỗi 30 giây, 17 giây nếu mỗi ngày). | tr. 289 |
| **Leap second** | Giây nhuận | Làm một phút dài 59 hoặc 61 giây; đã làm crash nhiều hệ thống lớn. | tr. 290 |
| **Smearing** | Rải giây nhuận | NTP server "nói dối", điều chỉnh giây nhuận dần trong suốt một ngày. | tr. 290 |
| **PTP** (*Precision Time Protocol*) | Giao thức thời gian chính xác | Cùng với bộ thu GPS, giúp đạt độ chính xác rất cao (như yêu cầu 100 µs của MiFID II). | tr. 290 |
| **Clock skew** **(G, skew nghĩa 2)** | Lệch đồng hồ | Đồng hồ các node lệch nhau, khiến sự kiện trông như xảy ra theo thứ tự sai; LWW có thể bỏ mất ghi. | tr. 291–292, 557 |
| **Clock offset monitoring** | Giám sát độ lệch đồng hồ | Theo dõi độ lệch giữa mọi máy và loại node lệch quá xa, vì đồng hồ sai gây mất dữ liệu âm thầm. | tr. 291 |
| **Last write wins (LWW)** | Ghi sau cùng thắng | Giữ giá trị có timestamp lớn nhất. Chương 8: lần ghi biến mất khi lệch đồng hồ, không phân biệt tuần tự với đồng thời, trùng timestamp. | tr. 292–293 |
| **Tiebreaker** | Giá trị phân xử | Giá trị phụ (có thể là số ngẫu nhiên lớn) để phân xử hai lần ghi trùng timestamp; cũng có thể vi phạm causality. | tr. 293 |
| **Logical clock** | Đồng hồ logic | Dựa trên bộ đếm tăng dần, chỉ đo thứ tự tương đối của sự kiện; an toàn hơn đồng hồ vật lý để sắp thứ tự. | tr. 293 |
| **Physical clock** | Đồng hồ vật lý | Time-of-day clock và monotonic clock: đo thời gian thực đã trôi qua. | tr. 293 |
| **Confidence interval** | Khoảng tin cậy | Một lần đọc đồng hồ là một khoảng thời gian chứ không phải một điểm; độ bất định gồm độ trôi, độ bất định của server và round-trip time. | tr. 293–294 |
| **TrueTime** | — | API của Spanner trả về [earliest, latest], báo tường minh khoảng tin cậy của đồng hồ cục bộ. | tr. 294 |
| **Global snapshot / monotonically increasing transaction ID** | Snapshot toàn cục / transaction ID tăng đơn điệu | Snapshot isolation phân tán cần transaction ID phản ánh causality; khó sinh mà không phối hợp. Spanner dùng khoảng tin cậy không chồng nhau và cố ý chờ trước khi commit. | tr. 294–295 |
| **Snowflake** | — | Bộ sinh ID phân tán của Twitter, cấp khối ID cho từng node; chỉ xấp xỉ đơn điệu, không đảm bảo causality. | tr. 295 |

### Process pause và real-time

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Lease** | Hợp đồng thuê có thời hạn | Giống một lock có timeout: chỉ một node giữ tại một thời điểm, phải gia hạn định kỳ. Không an toàn nếu không chịu được process pause. | tr. 295–296 |
| **Process pause** | Tiến trình bị dừng | Thread bị dừng một khoảng đáng kể ở bất kỳ điểm nào mà không biết: GC, VM suspend, context switch, I/O, swap, SIGSTOP. | tr. 295–298 |
| **Stop-the-world GC pause** | Dừng toàn bộ để dọn rác | GC dừng mọi thread, đôi khi tới vài phút; ngay cả GC "concurrent" cũng thỉnh thoảng phải dừng. | tr. 296–297 |
| **Live migration** | Di chuyển máy ảo đang chạy | Chuyển máy ảo sang host khác không reboot; gây pause dài tùy tốc độ ghi bộ nhớ. | tr. 297 |
| **Steal time** | Thời gian bị lấy | Thời gian CPU dành cho máy ảo khác khi lõi CPU được chia sẻ. | tr. 297 |
| **Page fault / thrashing** | Lỗi trang / quá tải swap | Truy cập bộ nhớ phải nạp page từ đĩa; cực đoan là hệ điều hành chỉ lo swap. Server thường tắt paging. | tr. 297 |
| **SIGSTOP / SIGCONT** | — | Tín hiệu Unix dừng và tiếp tục một tiến trình (Ctrl-Z); có thể bị gửi nhầm. | tr. 297 |
| **Preempt** | Giành quyền chạy | Thread bị dừng ở bất kỳ điểm nào và chạy tiếp sau đó mà không hay biết. | tr. 298 |
| **Hard real-time / deadline** | Thời gian thực cứng / hạn chót | Hệ phải phản hồi trước deadline, nếu không có thể hỏng cả hệ (máy bay, túi khí). Cần RTOS, thời gian thực thi xấu nhất, hạn chế cấp phát động; rất đắt. | tr. 298–299 |
| **RTOS** (*real-time operating system*) | Hệ điều hành thời gian thực | Lập lịch tiến trình với lượng CPU được đảm bảo trong các khoảng quy định. | tr. 298 |

### Tri thức, sự thật và lời nói dối

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Asymmetric fault** | Lỗi bất đối xứng | Node nhận được message nhưng message đi ra bị mất hay trễ; bị khai tử dù chạy bình thường. | tr. 300 |
| **Quorum** **(G)** / **majority quorum** | Túc số / túc số đa số | Quyết định (kể cả khai tử một node) cần phiếu của nhiều node, thường hơn một nửa; chỉ có một đa số tại một thời điểm. 3 node chịu được 1 node hỏng, 5 node chịu được 2. | tr. 301, 556 |
| **Split brain** **(G)** | Hai leader cùng lúc | Hai node cùng tin mình là leader; lý do hệ thống cần "chỉ có một" leader hay người giữ lock. | tr. 301, 557 |
| **Lock** **(G)** | Khóa | Bảo đảm chỉ một thread, node hay transaction truy cập được một thứ. Trong hệ phân tán, node tin mình giữ lock có thể đã mất nó. | tr. 301–302, 556 |
| **Fencing** / **fencing token** | Rào chắn / token rào chắn | Mỗi lần cấp lock kèm một số tăng dần; tài nguyên từ chối mọi lần ghi mang token cũ hơn token đã thấy. Chặn được node vô tình sai sau pause. | tr. 303–304 |
| **zxid / cversion** | — | Transaction ID và version của node trong ZooKeeper, tăng đơn điệu nên dùng được làm fencing token. | tr. 303 |
| **Byzantine fault** **(G)** | Lỗi Byzantine | Node hành xử sai tùy ý, kể cả "nói dối" hay gửi message mâu thuẫn, độc hại. Sách giả định không có loại lỗi này. | tr. 304, 553 |
| **Byzantine Generals Problem / Two Generals Problem** | Bài toán các vị tướng Byzantine / hai vị tướng | Đạt đồng thuận khi người đưa tin có thể chậm hay mất (hai tướng) và khi có kẻ phản bội gửi tin giả (Byzantine). | tr. 304 |
| **Consensus** **(G)** | Đồng thuận | Khiến nhiều node đồng ý về một điều; bài toán trong môi trường không tin nhau là Byzantine Generals Problem (Chương 9). | tr. 304, 554 |
| **Byzantine fault-tolerant (BFT)** | Chịu lỗi Byzantine | Chạy đúng dù một số node không tuân giao thức hay bị tấn công; cần ở hàng không vũ trụ, nhiều tổ chức không tin nhau (blockchain). | tr. 305 |
| **Supermajority** | Siêu đa số | Hơn hai phần ba số node chạy đúng, điều kiện của phần lớn thuật toán BFT (4 node chịu được 1 node lỗi). | tr. 305 |
| **Weak forms of lying** | Nói dối yếu | Message sai do lỗi phần cứng, bug, cấu hình sai; phòng bằng checksum tầng ứng dụng, kiểm tra input, nhiều NTP server. | tr. 306 |

### System model và tính đúng

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **System model** | Mô hình hệ thống | Trừu tượng mô tả những gì thuật toán được phép giả định về thời gian và lỗi; thuật toán được chứng minh đúng trong một model. | tr. 300, 306–307 |
| **Synchronous / partially synchronous / asynchronous model** | Mô hình đồng bộ / đồng bộ một phần / bất đồng bộ | Về thời gian: mọi thứ có giới hạn / thường có giới hạn nhưng đôi khi vượt tùy ý / không giả định gì, không có đồng hồ. Partially synchronous là thực tế nhất. | tr. 307, 553 |
| **Crash-stop / crash-recovery / Byzantine (arbitrary) faults** | Crash rồi dừng / crash rồi phục hồi / lỗi tùy ý | Về lỗi node: chết là mất hẳn / có thể quay lại, giữ stable storage, mất bộ nhớ / làm bất cứ điều gì. | tr. 307 |
| **Stable storage** | Lưu trữ ổn định | Đĩa không bay hơi giữ được qua crash trong crash-recovery model; thực tế đôi khi vẫn mất. | tr. 307, 309 |
| **Property (correctness)** | Thuộc tính (tính đúng) | Cách định nghĩa thuật toán đúng; ví dụ fencing token cần uniqueness, monotonic sequence, availability. | tr. 308 |
| **Safety property** | Thuộc tính an toàn | "Điều xấu không xảy ra"; vi phạm thì chỉ ra được thời điểm và không hoàn tác được; phải luôn đúng. | tr. 308–309 |
| **Liveness property** | Thuộc tính sống | "Điều tốt rốt cuộc xảy ra", thường có chữ *eventually*; được kèm điều kiện. Eventual consistency là một liveness property. | tr. 308–309 |
| **Degraded (limping) node** | Node suy giảm | Node chưa chết nhưng chạy rất kém (card mạng Gigabit tụt còn 1 Kb/s); khó xử lý hơn node hỏng hẳn. | tr. 311 |

### Bổ sung ngoài sách (liên quan chương 8)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **Jepsen** | Loạt kiểm thử của Kyle Kingsbury, tạo sự cố mạng để kiểm tra database phân tán có giữ đúng đảm bảo đã hứa; nhiều bài mang tên "Call Me Maybe". | Đề từ, tài liệu [20], [53], [55] |
| **Lamport clock** | Mỗi node giữ bộ đếm, tăng trước mỗi sự kiện, gửi kèm trong message, khi nhận thì lấy max rồi cộng 1; A happens before B thì bộ đếm của A nhỏ hơn. | Logical clock, tài liệu [56] |
| **Hybrid logical clock** | Kết hợp đồng hồ vật lý với bộ đếm logic, đề xuất trong tài liệu [57]. | Logical clock, global snapshot |
| **Commit wait** | Tên trong bài báo Spanner cho bước chờ tới khi độ bất định qua đi rồi mới cho commit hiển thị; thời gian chờ trung bình xấp xỉ hai lần độ bất định. | TrueTime, Spanner |
| **Hàng đợi M/M/1** | Thời gian trong hệ bằng thời gian phục vụ chia cho (1 − ρ): gấp 2 lần ở 50% tận dụng, 10 lần ở 90%, 100 lần ở 99%. | Queueing, độ trễ gần hết công suất |
| **n = 3f + 1** | Số node tối thiểu để chịu được f node Byzantine; giải thích con số "hơn hai phần ba". | Byzantine fault tolerance |
| **Trace ID** | ID theo dõi truyền qua các service để biết request nào gây ra request nào, thay vì dựa vào timestamp của nhiều máy. | Log, logical clock |

<a id="chuong-9"></a>

## Chương 9 — Consistency and Consensus

### Consistency guarantees và linearizability

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Convergence** | Hội tụ | Tên gọi tốt hơn cho eventual consistency: mọi replica cuối cùng có cùng giá trị, nhưng không nói khi nào. | tr. 322 |
| **Consistency model** | Mô hình nhất quán | Đảm bảo mà hệ dữ liệu phân tán đưa ra về giá trị đọc được khi có nhiều replica. Mô hình mạnh hơn dễ dùng đúng hơn nhưng chậm hơn hoặc kém chịu lỗi hơn. | tr. 323 |
| **Linearizability / linearizable** **(G)** (*atomic consistency, strong consistency, immediate consistency, external consistency*) | Tính tuyến tính hóa | Hệ thống hành xử như chỉ có một bản sao dữ liệu, mọi thao tác atomic; ghi xong thì mọi lần đọc sau phải thấy giá trị mới. | tr. 324–329, 555 |
| **Recency guarantee** | Đảm bảo độ mới | Giá trị đọc được là giá trị mới nhất, không từ cache hay replica stale; bản chất của linearizability. | tr. 324 |
| **Atomic** **(G, nghĩa 1)** | Nguyên tử | Thao tác dường như có hiệu lực tại một thời điểm duy nhất, không ai thấy nó ở trạng thái "làm dở". | tr. 324, 327, 553 |
| **Register** | Thanh ghi | Trong lý thuyết hệ phân tán: một object đọc ghi được, ví dụ một key, một hàng, một document. | tr. 325 |
| **Regular register** | Thanh ghi thường | Register mà lần đọc đồng thời với ghi có thể trả giá trị cũ hoặc mới; yếu hơn linearizable. | tr. 326 |
| **Compare-and-set (cas)** | So sánh rồi đặt | Thao tác atomic: đặt x = v_new chỉ khi x đang bằng v_old, nếu không thì giữ nguyên và báo lỗi. Bản linearizable của nó tương đương consensus. | tr. 327, 335, 350 |
| **Strict serializability / strong one-copy serializability (strong-1SR)** | Khả tuần tự nghiêm ngặt | Có cả serializability (transaction) lẫn linearizability (recency). 2PL và chạy tuần tự thật thường đạt được; SSI thì không. | tr. 329 |
| **Serializable** **(G)** | Khả tuần tự | Transaction đồng thời hành xử như chạy lần lượt theo một thứ tự nào đó; khác linearizability vì là đảm bảo về transaction nhiều object. | tr. 329, 557 |
| **Lock** **(G)** / **leader election** | Khóa / bầu leader | Cơ chế chỉ cho một node giữ quyền; dùng lock để bầu leader thì lock phải linearizable. | tr. 330, 556 |
| **Coordination service** | Dịch vụ phối hợp | Dịch vụ như ZooKeeper, etcd dùng consensus để cung cấp lock, leader election và thao tác linearizable chịu lỗi. | tr. 330, 370 |
| **Uniqueness constraint** | Ràng buộc duy nhất | Ví dụ username, email, đường dẫn file không trùng; áp dụng ngay lúc ghi thì cần linearizability. | tr. 330–331 |
| **Cross-channel timing dependency** | Phụ thuộc thời gian xuyên kênh | Race giữa hai kênh liên lạc (ví dụ file storage và message queue) khi storage không linearizable. | tr. 331–332 |
| **Quorum read (etcd) / sync() (ZooKeeper)** | Đọc quorum / đồng bộ trước khi đọc | Cách yêu cầu đọc linearizable, vì mặc định hai hệ này có thể đọc stale từ bất kỳ replica nào. | tr. 330, 351 |
| **CAP theorem** **(G)** | Định lý CAP | Khi bị network partition, hệ phải chọn linearizability hoặc availability. Phạm vi hẹp, nhiều hiểu lầm, nay chủ yếu mang ý nghĩa lịch sử. | tr. 336–338, 554 |
| **CP / AP** | — | Cách gọi hai lựa chọn khi partition (consistent hoặc available); sách khuyên tránh vì cách phân loại này có nhiều lỗi. | tr. 336 |
| **Network partition** | Phân hoạch mạng | Loại fault mạng: node còn sống nhưng mất kết nối với nhau. Khác "partitioning" (sharding) ở Chương 6. | tr. 337 |
| **Memory barrier / fence** | Rào bộ nhớ | Lệnh buộc CPU đa lõi làm thay đổi bộ nhớ hiển thị giữa các lõi; không có nó thì RAM đa lõi không linearizable. | tr. 338 |

### Ordering và causality

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Causality** **(G)** | Quan hệ nhân quả | Phụ thuộc giữa sự kiện khi cái này "happens before" cái kia: nguyên nhân trước kết quả, câu hỏi trước câu trả lời. | tr. 339–341, 554 |
| **Causal dependency / causal order** | Phụ thuộc nhân quả / thứ tự nhân quả | Chuỗi thao tác đọc rồi ghi dựa trên cái đã đọc; định nghĩa cái gì xảy ra trước cái gì. | tr. 339–340 |
| **Causally consistent / causal consistency** | Nhất quán nhân quả | Hệ tuân theo thứ tự nhân quả; mô hình mạnh nhất không chậm vì network delay và vẫn available khi mạng hỏng. Snapshot isolation là một ví dụ. | tr. 341–342 |
| **Total order** **(G)** / **partial order** | Thứ tự toàn phần / thứ tự bộ phận | Total order so sánh được mọi cặp (linearizability); partial order có cặp không so sánh được (causality). | tr. 341, 557–558 |
| **Incomparable** | Không so sánh được | Hai phần tử không cái nào lớn hơn cái nào, như {a, b} và {b, c}, hay hai thao tác đồng thời. | tr. 341 |
| **Sequence number** | Số thứ tự | Số gọn định danh thao tác và cho total order; nhất quán với causality nếu do một leader cấp. | tr. 343–344 |
| **Logical clock** | Đồng hồ logic | Thuật toán sinh dãy số định danh thao tác (thường là counter), không dựa vào đồng hồ vật lý. | tr. 343 |
| **Noncausal sequence number generator** | Bộ sinh số không nhân quả | Lẻ chẵn hay bit node ID, timestamp đồng hồ vật lý, cấp trước khối số: nhanh nhưng không nhất quán với causality. | tr. 344–345 |
| **Lamport timestamp** | Mốc thời gian Lamport | Cặp (counter, node ID); mọi node và client mang theo counter lớn nhất đã thấy. Total order nhất quán với causality, không phân biệt được đồng thời. | tr. 345–347 |
| **Total order broadcast / atomic broadcast / total order multicast** | Phát tán thứ tự toàn phần | Giao thức gửi message với reliable delivery và totally ordered delivery; thứ tự chốt khi deliver. Tương đương consensus. | tr. 348–352 |
| **Reliable delivery / totally ordered delivery** | Giao tin cậy / giao đúng thứ tự | Hai safety property của total order broadcast: không mất message; mọi node nhận cùng thứ tự. | tr. 348 |
| **State machine replication** | Nhân bản máy trạng thái | Mọi replica xử lý cùng các lần ghi theo cùng thứ tự nên nhất quán; dựa trên total order broadcast. | tr. 349 |
| **Fencing token** / **zxid**, **cversion** | Thẻ rào chắn | Số tăng đơn điệu mỗi lần cấp lock; tài nguyên từ chối thao tác mang token cũ. ZooKeeper dùng zxid, cversion. | tr. 349, 370 |
| **Sequential consistency / timeline consistency** | Nhất quán tuần tự | Mọi node thấy cùng một thứ tự toàn phần nhưng không bắt buộc khớp thời gian thực; hơi yếu hơn linearizability. | tr. 351 |
| **Increment-and-get** | Tăng rồi lấy | Thao tác atomic trên register số nguyên; bản linearizable cấp được dãy số không lỗ hổng, tương đương consensus. | tr. 351–352 |

### Distributed transaction và atomic commit

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Consensus** **(G)** | Đồng thuận | Khiến nhiều node đồng ý một giá trị và không đổi ý; khó hơn nhiều so với vẻ ngoài. | tr. 352, 364, 554 |
| **Atomic commit problem** / **atomic** **(G, nghĩa 2)** | Bài toán commit nguyên tử | Mọi node của distributed transaction cùng commit hoặc cùng abort; quy về được consensus. | tr. 353–355, 553 |
| **FLP result** | Kết quả FLP | Fischer, Lynch, Paterson: trong mô hình asynchronous không timeout, không thuật toán tất định nào luôn đạt consensus nếu một node có thể crash. | tr. 353 |
| **Asynchronous system model** **(G: asynchronous)** | Mô hình hệ bất đồng bộ | Thuật toán không được giả định gì về thời gian, không có đồng hồ hay timeout. | tr. 353, 553 |
| **Commit record** | Bản ghi commit | Bản ghi append vào log sau dữ liệu; đĩa ghi xong nó là thời điểm transaction một node được commit. | tr. 354 |
| **Irrevocable commit** | Commit không thể thu hồi | Đã commit thì không được abort hồi tố, vì transaction khác có thể đã dựa vào dữ liệu đó. | tr. 355 |
| **Compensating transaction** | Transaction bù trừ | Transaction riêng hoàn tác hiệu ứng của transaction đã commit; tính đúng xuyên transaction là việc của ứng dụng. | tr. 355 |
| **Two-phase commit (2PC)** **(G)** | Commit hai pha | Pha 1 prepare (participant hứa), pha 2 commit hoặc abort theo quyết định đã ghi log của coordinator. | tr. 355–359, 558 |
| **Two-phase locking (2PL)** **(G)** | Khóa hai pha | Thuật toán serializable isolation; hoàn toàn khác 2PC, chỉ giống tên. | tr. 356, 558 |
| **Coordinator / transaction manager** | Bộ điều phối | Thành phần điều khiển 2PC, thường là thư viện trong tiến trình ứng dụng (Narayana, JOTM, BTM, MSDTC). | tr. 356 |
| **Participant** | Bên tham gia | Node database tham gia distributed transaction. | tr. 356 |
| **Prepare request** | Yêu cầu chuẩn bị | Coordinator hỏi participant có commit được không; trả lời yes nghĩa là hứa commit được trong mọi hoàn cảnh. | tr. 356–357 |
| **Commit point** | Điểm commit | Lúc coordinator ghi quyết định commit hay abort vào log trên đĩa; từ đó quyết định không đổi được. | tr. 357, 359 |
| **In doubt / uncertain** | Đang nghi vấn | Participant đã bỏ phiếu yes nhưng chưa biết quyết định; không được tự commit hay abort, phải chờ coordinator, trong lúc đó giữ lock. | tr. 358, 362 |
| **Blocking / nonblocking atomic commit** | Commit nguyên tử chặn / không chặn | 2PC là blocking vì có thể kẹt chờ coordinator; nonblocking cần perfect failure detector. | tr. 359 |
| **Three-phase commit (3PC)** | Commit ba pha | Đề xuất nonblocking nhưng giả định delay và thời gian phản hồi có giới hạn, nên không bảo đảm atomicity trong mạng thực tế. | tr. 359 |
| **Perfect failure detector** | Bộ phát hiện lỗi hoàn hảo | Cơ chế biết chắc node đã crash hay chưa; timeout trong mạng delay không giới hạn không phải là nó. | tr. 359 |
| **Database-internal distributed transaction** | Transaction phân tán nội bộ | Mọi participant là node của cùng một database (VoltDB, MySQL Cluster NDB); thường chạy khá tốt. | tr. 360 |
| **Heterogeneous distributed transaction** | Transaction phân tán không đồng nhất | Participant là các công nghệ khác nhau (database nhiều hãng, message broker). | tr. 360 |
| **Exactly-once message processing** | Xử lý message đúng một lần | Commit nguyên tử việc xác nhận message và side effect, nên thử lại an toàn; chỉ được khi mọi hệ cùng tham gia atomic commit. | tr. 360–361 |
| **XA (X/Open XA)** / **JTA** | Chuẩn XA | Chuẩn 1991 cài 2PC xuyên công nghệ; là C API (Java dùng JTA với driver JDBC, JMS), không phải network protocol. | tr. 361–362 |
| **Orphaned in-doubt transaction** | Transaction nghi vấn mồ côi | Transaction coordinator không quyết được (log mất hay hỏng); giữ lock mãi, cần admin xử lý tay. | tr. 363 |
| **Heuristic decision** | Quyết định heuristic | Lối thoát khẩn cấp cho participant tự quyết commit hay abort; "heuristic" là cách nói giảm cho có lẽ phá atomicity. | tr. 363 |

### Fault-tolerant consensus và coordination service

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Uniform agreement, integrity, validity, termination** | Đồng thuận đồng nhất, toàn vẹn, hợp lệ, kết thúc | Bốn thuộc tính của consensus: không hai node quyết khác nhau; không quyết hai lần; giá trị quyết phải được đề xuất; node không crash cuối cùng quyết được. | tr. 365 |
| **Uniform consensus** | Đồng thuận đồng nhất | Biến thể consensus sách dùng; tương đương consensus thông thường với failure detector không tin cậy. | tr. 365 |
| **Safety / liveness property** | Thuộc tính an toàn / sống | Agreement, integrity, validity là safety; termination là liveness và hình thức hóa fault tolerance. | tr. 365 |
| **Byzantine fault** **(G)** | Lỗi Byzantine | Node hành xử sai tùy ý (gửi message mâu thuẫn); hầu hết thuật toán consensus giả định không có. Chịu được nếu ít hơn một phần ba node Byzantine. | tr. 366, 553 |
| **Viewstamped Replication (VSR), Paxos, Multi-Paxos, Raft, Zab** | — | Các thuật toán consensus chịu lỗi nổi tiếng; thực chất là total order broadcast (Multi-Paxos là bản tối ưu của Paxos). | tr. 366–367 |
| **Epoch number** (*ballot number, view number, term number*) | Số kỷ nguyên | Số tăng mỗi lần bầu leader; bảo đảm leader duy nhất trong mỗi epoch, epoch cao hơn thắng. | tr. 368 |
| **Quorum** **(G)** (trong consensus) | Túc số | Leader phải có phiếu của một quorum (thường là đa số) cho mỗi quyết định; quorum bầu leader và quorum bỏ phiếu phải giao nhau. | tr. 366, 368, 556 |
| **Dynamic membership** | Thành viên động | Mở rộng cho phép tập node bỏ phiếu thay đổi theo thời gian; kém được hiểu rõ hơn static membership. | tr. 369 |
| **ZooKeeper / etcd / Chubby** | — | Dịch vụ phối hợp giữ dữ liệu nhỏ, thay đổi chậm, nhân bản bằng total order broadcast chịu lỗi; ZooKeeper mô phỏng Chubby của Google. | tr. 330, 370 |
| **Lease** | Hợp đồng thuê | Lock có thời hạn, tự hết để được nhả khi client hỏng. | tr. 370 |
| **Session / heartbeat / ephemeral node** | Phiên / nhịp tim / node tạm | Client giữ session bằng heartbeat; quá session timeout thì session chết và ephemeral node (ví dụ lock) tự biến mất. | tr. 371 |
| **Watch / change notification** | Theo dõi / thông báo thay đổi | Client đăng ký nhận thông báo khi giá trị đổi, thay vì poll. | tr. 371 |
| **Service discovery / service registry** | Khám phá dịch vụ | Tìm địa chỉ IP của service; không cần consensus (DNS đọc stale vẫn ổn). | tr. 372 |
| **Read-only caching replica** | Replica đệm chỉ đọc | Nhận log quyết định của consensus bất đồng bộ, không bỏ phiếu; phục vụ đọc không cần linearizable. | tr. 372 |
| **Membership service** | Dịch vụ thành viên | Xác định node nào đang là thành viên sống của cluster bằng failure detection cộng consensus. | tr. 372–373 |
| **Split brain** **(G)** | Hai leader cùng lúc | Hai node cùng tin mình là leader; consensus dùng để tránh nó khi bầu leader. | tr. 322, 352, 557 |

### Bổ sung ngoài sách (liên quan chương 9)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **ABD algorithm** | Thuật toán của Attiya, Bar-Noy, Dolev (tài liệu [23]): đọc quorum rồi ghi lại giá trị mới nhất vào một quorum trước khi trả kết quả, làm register đọc ghi linearizable. | Linearizability and quorums |
| **PACELC** | Cách nhìn của Abadi (tài liệu [46]): khi Partition chọn A hoặc C, Else chọn Latency hoặc Consistency. | CAP, chi phí của linearizability |
| **Knossos** | Công cụ kiểm tra linearizability của Jepsen; bài toán tổng quát là NP-complete. | Kiểm tra linearizability |
| **Saga** | Chuỗi transaction cục bộ, bước lỗi thì chạy compensating transaction cho các bước đã commit (tài liệu [74]). | Compensating transaction |
| **2f + 1 và 3f + 1** | Số node tối thiểu để chịu f lỗi crash (đa số) và f lỗi Byzantine. Số node chẵn không tăng khả năng chịu lỗi crash. | Consensus cần đa số |
| **Observer (ZooKeeper)** | Tên ZooKeeper đặt cho replica nhận log nhưng không bỏ phiếu. | Read-only caching replica |

<a id="chuong-10"></a>

## Chương 10 — Batch Processing

Gồm cả thuật ngữ trong phần giới thiệu Part III (tr. 385–387).

### Dữ liệu dẫn xuất (phần giới thiệu Part III)

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **System of record / source of truth** **(G)** | Hệ thống gốc / nguồn sự thật | Giữ phiên bản có thẩm quyền của dữ liệu; dữ liệu mới ghi vào đây trước, mỗi sự kiện một lần (thường normalized). Lệch với hệ khác thì nó đúng theo định nghĩa. | tr. 386, 557 |
| **Derived data / derived data system** **(G)** | Dữ liệu dẫn xuất | Kết quả biến đổi dữ liệu có sẵn từ hệ khác qua một quy trình lặp lại được; mất thì dựng lại được. Ví dụ: cache, index, materialized view, dữ liệu cho recommendation. | tr. 386, 554 |
| **Redundant** | Dư thừa | Derived data nhân bản thông tin đã có, nhưng thường thiết yếu cho hiệu năng đọc. | tr. 386 |
| **Denormalized** **(G)** | Phi chuẩn hóa | Có trùng lặp có chủ đích để đọc nhanh; derived data thường ở dạng này. | tr. 386, 554 |
| **Cache** **(G)** | Bộ đệm | Ví dụ kinh điển của derived data: thiếu dữ liệu thì quay về database bên dưới. | tr. 386, 553–554 |
| **Dataflow** | Luồng dữ liệu | Phần nào có input, output nào và phụ thuộc nhau ra sao; phân biệt system of record với derived data làm nó rõ ràng. | tr. 386 |

### Ba kiểu hệ thống và công cụ Unix

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Service / online system** | Dịch vụ / hệ thống trực tuyến | Chờ request, xử lý nhanh nhất có thể rồi trả response; đo bằng response time và availability. | tr. 389 |
| **Batch processing system / batch job / batch process** **(G)** | Hệ thống xử lý theo lô (offline) | Chạy job trên một lượng lớn input cố định, sinh output mà không sửa input; thường chạy định kỳ; đo bằng throughput. | tr. 390, 553 |
| **Stream processing / near-real-time / nearline** **(G: stream process)** | Xử lý luồng | Ở giữa online và batch: tiêu thụ sự kiện ngay sau khi xảy ra, latency thấp hơn batch (Chương 11). | tr. 390, 557 |
| **Throughput** | Thông lượng | Thước đo chính của batch: thời gian xử lý xong một input có kích thước cho trước. | tr. 390 |
| **Commodity hardware** | Phần cứng thông dụng | Máy thường, không chuyên dụng; MapReduce là bước tiến về quy mô xử lý trên loại phần cứng này. | tr. 390 |
| **Working set** | Tập làm việc | Lượng bộ nhớ job cần truy cập ngẫu nhiên; khi đếm URL, chỉ phụ thuộc số URL khác nhau. | tr. 393 |
| **In-memory aggregation** | Tổng hợp trong bộ nhớ | Đếm hay gộp bằng hash table trong bộ nhớ; tốt khi working set vừa bộ nhớ. | tr. 393 |
| **Mergesort / spilling to disk** | Sắp xếp trộn / tràn ra đĩa | Sắp từng khối trong bộ nhớ, ghi segment ra đĩa, rồi merge tuần tự; GNU sort tự làm việc này và song song hóa. | tr. 394 |
| **Unix philosophy** | Triết lý Unix | Mỗi chương trình làm tốt một việc; output là input của chương trình chưa biết; thử sớm; dùng công cụ. | tr. 394 |
| **Pipe** | Ống | Gắn stdout của tiến trình này vào stdin của tiến trình khác qua bộ đệm nhỏ trong bộ nhớ, không ghi toàn bộ dữ liệu trung gian ra đĩa. | tr. 394, 396 |
| **Uniform interface** | Giao diện thống nhất | Mọi chương trình dùng cùng giao diện I/O (trong Unix là file, tức chuỗi byte) nên nối được với nhau. | tr. 395 |
| **File descriptor** | Bộ mô tả file | Giao diện I/O của Unix; biểu diễn được file, socket, stdin, stdout, thiết bị. | tr. 395 |
| **Record separator** | Ký tự ngăn bản ghi | Công cụ Unix dùng ký tự xuống dòng (0x0A); ASCII có ký tự riêng 0x1E cho việc này. | tr. 395 |
| **Balkanization of data** | Dữ liệu bị chia cắt | Dữ liệu nằm trong các hệ không dễ chuyển qua lại, kể cả khi cùng mô hình dữ liệu. | tr. 396 |
| **stdin / stdout** | Đầu vào / đầu ra chuẩn | Cho phép shell nối dây input, output thay cho chương trình. | tr. 396 |
| **Separation of logic and wiring** | Tách logic khỏi nối dây | Chương trình không quan tâm input từ đâu, output đi đâu; một dạng loose coupling, late binding, inversion of control. | tr. 396, 414 |
| **Immutable input** | Input bất biến | Chạy lại lệnh hay job bao nhiêu lần cũng không làm hỏng input; nền tảng cho thử nghiệm, retry và rollback. | tr. 397, 413–414 |

### MapReduce và HDFS

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **MapReduce** | — | Mô hình lập trình batch (Google, 2004): mapper, sắp xếp ngầm, reducer; cài trong Hadoop, CouchDB, MongoDB. | tr. 390, 397–403 |
| **HDFS** (*Hadoop Distributed File System*) | Hệ thống file phân tán của Hadoop | Bản cài lại mã nguồn mở của Google File System (GFS); shared-nothing, daemon trên mỗi máy và NameNode trung tâm. | tr. 398 |
| **Distributed filesystem** | Hệ thống file phân tán | Filesystem trải trên đĩa của nhiều máy; giao diện chung của các MapReduce job. | tr. 398 |
| **Object storage** | Lưu trữ đối tượng | S3, Azure Blob Storage, OpenStack Swift; giống HDFS nhưng thường tách lưu trữ khỏi tính toán. | tr. 398 |
| **NameNode** | — | Server trung tâm của HDFS, theo dõi block nào của file nằm trên máy nào. | tr. 398 |
| **Erasure coding / Reed–Solomon** | Mã xóa | Khôi phục dữ liệu mất với chi phí lưu trữ thấp hơn nhân bản toàn phần; làm mất lợi thế đọc cục bộ. | tr. 398 |
| **Mapper** | Hàm map | Gọi một lần cho mỗi bản ghi input, trích key và value, không giữ trạng thái giữa các bản ghi. | tr. 399 |
| **Reducer** | Hàm reduce | Nhận một key và iterator trên mọi value cùng key, sinh output. | tr. 400 |
| **Map task / reduce task** | Tác vụ map / reduce | Mỗi block input một map task; số reduce task do người viết job cấu hình, reducer chọn theo hash của key. | tr. 400–401 |
| **Putting the computation near the data** | Đưa tính toán tới gần dữ liệu | Chạy map task trên máy giữ replica của input để khỏi chép qua mạng. | tr. 400 |
| **Locality** **(G)** | Tính cục bộ | Đặt dữ liệu hay tính toán ở cùng chỗ với nơi cần nó. | tr. 400, 421, 555 |
| **Shuffle** | Xáo (thực ra không ngẫu nhiên) | Chia output của mapper theo reducer, sắp trên đĩa cục bộ, reducer tải về và merge. | tr. 402 |
| **Workflow / workflow scheduler** | Luồng công việc / bộ lập lịch | Chuỗi job nối qua thư mục HDFS; scheduler (Oozie, Azkaban, Luigi, Airflow, Pinball) chỉ chạy job khi job trước thành công. | tr. 402–403 |

### Join, grouping và skew

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Join** **(G)** | Phép kết | Ghép bản ghi có liên kết; trong batch là giải quyết mọi lần xuất hiện của liên kết trong dataset. | tr. 403–404, 555 |
| **Equi-join** | Kết bằng | Ghép bản ghi có giá trị giống hệt ở một trường; loại join sách bàn. | tr. 403 |
| **Full table scan** | Quét toàn bảng | Đọc toàn bộ input; đắt cho vài bản ghi nhưng hợp lý cho analytics. | tr. 403 |
| **Activity events / clickstream data** | Sự kiện hoạt động / dữ liệu luồng nhấp | Log những gì người dùng đã đăng nhập làm trên website; đóng vai fact table. | tr. 404 |
| **ETL** **(G)** | Trích, biến đổi, nạp | Ở đây dùng để lấy bản sao database (từ backup) đưa vào HDFS; Hadoop cũng làm ETL cho MPP warehouse. | tr. 405, 416, 554 |
| **Sort-merge join** | Kết bằng sắp xếp và trộn | Mapper trích join key, sắp theo key, reducer merge bản ghi hai phía của cùng key. | tr. 405–406 |
| **Secondary sort** | Sắp xếp phụ | Sắp để reducer thấy bản ghi của một phía trước (ví dụ hồ sơ người dùng), rồi các bản ghi khác theo thời gian. | tr. 406 |
| **Reduce-side join** | Kết phía reduce | Join làm trong reducer; không cần giả định về input nhưng sort, chép, merge tốn kém. | tr. 408 |
| **GROUP BY / aggregation** | Gom nhóm / tổng hợp | Gom bản ghi theo key rồi đếm, cộng, lấy top k; trên MapReduce giống join. | tr. 406–407 |
| **Sessionization** | Gom theo phiên | Gom mọi sự kiện của một phiên người dùng để biết chuỗi hành động. | tr. 407 |
| **A/B testing** | Thử nghiệm A/B | So hành vi người dùng thấy phiên bản mới với người thấy phiên bản cũ. | tr. 407 |
| **Skew / hot spot** **(G, skew nghĩa 1)** | Lệch tải / điểm nóng | Một reducer phải xử lý nhiều bản ghi hơn hẳn; cả job chờ reducer chậm nhất. | tr. 407, 557 |
| **Linchpin object / hot key** | Đối tượng then chốt / khóa nóng | Bản ghi hoạt động lệch hẳn, ví dụ người nổi tiếng có hàng triệu người theo dõi. | tr. 407 |
| **Skewed join (Pig) / sharded join (Crunch)** | — | Rải bản ghi hot key ngẫu nhiên cho nhiều reducer, nhân bản input kia tới các reducer đó; Pig lấy mẫu để tìm hot key, Crunch cần chỉ định. | tr. 407–408 |
| **Two-stage aggregation** | Tổng hợp hai giai đoạn | Stage một rải ngẫu nhiên và tổng hợp từng phần, stage hai gộp thành một giá trị mỗi key. | tr. 408 |
| **Map-side join** | Kết phía map | Job rút gọn không reducer, không sort; cần giả định về kích thước, partition, thứ tự của input. | tr. 408 |
| **Broadcast hash join** (*replicated join*, *MapJoin*) | Kết băm phát tán | Mỗi mapper nạp trọn input nhỏ vào hash table rồi quét một block của input lớn. | tr. 409 |
| **Partitioned hash join** (*bucketed map join*) | Kết băm theo phân vùng | Hai input partition cùng cách; hash join độc lập cho từng cặp partition. | tr. 409–410 |
| **Map-side merge join** | Kết trộn phía map | Hai input partition và sắp cùng cách; mapper merge như reducer. | tr. 410 |
| **HCatalog / Hive metastore** | — | Nơi lưu metadata về cách partition và sắp xếp dataset trong Hadoop. | tr. 410 |

### Output của batch và so sánh với MPP database

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Term dictionary / postings list** | Từ điển thuật ngữ / danh sách xuất hiện | Cấu trúc của full-text index: tra từ khóa ra danh sách document ID chứa nó. | tr. 411 |
| **Full-text search** **(G)** | Tìm kiếm toàn văn | Dựng index cho tập document cố định là cách dùng gốc của MapReduce ở Google. | tr. 411, 555 |
| **Bulk loading** | Nạp hàng loạt | Batch job dựng file database bất biến, server chỉ đọc nạp vào rồi chuyển nguyên tử (Voldemort, Terrapin, ElephantDB, HBase). | tr. 413 |
| **All-or-nothing output** | Output tất cả hoặc không | Job thành công thì output như mỗi task chạy đúng một lần; job thất bại thì không có output. | tr. 413 |
| **Speculative execution / task attempt** | Thực thi dự phòng / lần thử của task | Hadoop có thể chạy nhiều attempt cho một task; ghi ra hệ thống ngoài thì các attempt đều lộ side effect (giải thích cơ chế là bổ sung ngoài sách). Summary nói nhiều task của một partition có thể thành công nhưng chỉ một task công bố output (tr. 430). | tr. 413, 430 |
| **Human fault tolerance** | Chịu lỗi do con người | Phục hồi được sau khi deploy code lỗi: rollback code và chạy lại, hoặc chuyển về output cũ. | tr. 414 |
| **Minimizing irreversibility** | Giảm điều không đảo ngược được | Rollback dễ giúp phát triển tính năng nhanh hơn; có lợi cho Agile. | tr. 414 |
| **MPP database** (*massively parallel processing*) | Database xử lý song song quy mô lớn | Thực thi song song truy vấn SQL analytics trên cụm máy; Gamma, Teradata, Tandem NonStop SQL. | tr. 415 |
| **Data lake / enterprise data hub** | Hồ dữ liệu | Gom dữ liệu thô từ khắp tổ chức về một chỗ, lo thiết kế schema sau. | tr. 415 |
| **Schema-on-read** | Schema khi đọc | Người tiêu thụ diễn giải dữ liệu thô, thay vì người tạo phải đưa về định dạng chuẩn. | tr. 415–416 |
| **Sushi principle** | Nguyên tắc sushi | "Dữ liệu thô thì tốt hơn": giữ dạng thô để có nhiều cách biến đổi. | tr. 416 |
| **Preemption** | Dừng ưu tiên | Dừng task ưu tiên thấp để nhường tài nguyên; ở Google khoảng 5% mỗi giờ cho task MapReduce. | tr. 418 |
| **Overcommitting** | Cấp vượt mức | Cấp tài nguyên ưu tiên thấp nhiều hơn thực có vì thu hồi được; tận dụng máy tốt hơn. | tr. 418 |
| **YARN** | — | Cluster scheduler của Hadoop; CapacityScheduler có preemption giữa hàng đợi, nhưng (lúc viết sách) không có preemption tổng quát theo ưu tiên. | tr. 418, 422 |

### Vượt qua MapReduce

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Materialization / materialize** **(G)** | Vật chất hóa | Tính sẵn kết quả và ghi ra, thay vì tính khi cần; ở đây là ghi trạng thái trung gian ra file. | tr. 420, 556 |
| **Intermediate state** | Trạng thái trung gian | Dữ liệu chỉ dùng để chuyển từ job này sang job sau. | tr. 420 |
| **Straggler** | Task tụt lại | Task chạy lâu hơn hẳn các task khác do skew hay tải không đều. | tr. 420 |
| **Dataflow engine** | Engine luồng dữ liệu | Spark, Tez, Flink: xử lý cả workflow như một job, nối operator linh hoạt, chỉ sắp khi cần. | tr. 421 |
| **Operator** | Toán tử | Hàm do người dùng định nghĩa trong dataflow engine; tổng quát hóa map và reduce. | tr. 421 |
| **Resilient distributed dataset (RDD)** | — | Abstraction của Spark theo dõi dòng dõi dữ liệu để tính lại khi lỗi. | tr. 422 |
| **Checkpoint** | Điểm kiểm tra | Ghi trạng thái (operator ở Flink, đỉnh ở Pregel) ra storage bền vững để phục hồi. | tr. 422, 426 |
| **Deterministic** **(G)** | Tất định | Cùng input luôn cho cùng output; cần để tính lại an toàn. | tr. 423, 554 |
| **Pipelined execution** | Thực thi kiểu ống | Chuyển dần output của operator sang operator sau, không chờ input đủ. | tr. 423 |
| **Directed acyclic graph (DAG)** | Đồ thị có hướng không chu trình | Cách xếp operator trong dataflow engine; không phải xử lý graph. | tr. 424 |
| **Graph** **(G)** | Đồ thị | Đỉnh và cạnh; trong batch là phân tích offline toàn bộ graph. | tr. 424, 555 |
| **PageRank** | — | Ước lượng mức phổ biến của trang web theo các trang liên kết tới nó. | tr. 424 |
| **Transitive closure** | Bao đóng bắc cầu | Lặp lại việc đi theo cạnh tới khi hết, như tìm mọi địa điểm ở Bắc Mỹ (Hình 2-6). | tr. 424 |
| **Iterative processing** | Xử lý lặp | Scheduler ngoài chạy từng bước rồi kiểm tra điều kiện dừng. | tr. 424 |
| **Bulk synchronous parallel (BSP) / Pregel model** | Song song đồng bộ theo khối | Đỉnh gửi message dọc cạnh; mỗi iteration gọi hàm cho từng đỉnh; trạng thái đỉnh trong bộ nhớ. Giraph, GraphX, Gelly. | tr. 425 |
| **Thinking like a vertex** | Nghĩ như một đỉnh | Mô hình lập trình chỉ xử lý một đỉnh mỗi lần, nên framework partition graph tùy ý. | tr. 426 |
| **Cost-based query optimizer** | Bộ tối ưu truy vấn theo chi phí | Hive, Spark, Flink tự chọn thuật toán join và thứ tự join. | tr. 427 |
| **Declarative** **(G)** | Khai báo | Chỉ nói cần gì; ở batch giúp optimizer chọn join, đọc theo cột, vectorized. | tr. 427–428, 554 |
| **Function callback / user-defined function** | Hàm gọi lại / hàm do người dùng định nghĩa | Mapper, reducer chạy code tùy ý; UDF trong database thường cồng kềnh. | tr. 427–428 |
| **Vectorized execution** | Thực thi vector hóa | Vòng lặp trong chặt, thân thiện CPU cache, tránh gọi hàm (Hive, Spark DataFrames, Impala). | tr. 428 |
| **k-nearest neighbors** | k láng giềng gần nhất | Tìm phần tử gần một phần tử trong không gian nhiều chiều; một dạng tìm kiếm tương tự. | tr. 429 |
| **Bounded / unbounded** **(G)** | Có giới hạn / không giới hạn | Input của batch có kích thước biết trước nên job hoàn tất; input của stream không bao giờ dứt. | tr. 430, 553, 558 |

### Bổ sung ngoài sách (liên quan chương 10)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **External merge sort / k-way merge** | Sắp từng khối vừa bộ nhớ thành các run, rồi merge k run cùng lúc; chỉ dùng I/O tuần tự. | Sort ngoài bộ nhớ, shuffle |
| **DataNode** | Tên của daemon HDFS trên mỗi máy, giữ block và phục vụ đọc ghi. | HDFS, NameNode |
| **Combiner** | Tổng hợp sơ bộ ở phía mapper trước shuffle; chỉ đúng với phép gộp được từng phần (đếm, cộng, max). | Two-stage aggregation |
| **Superstep** | Tên gọi một iteration trong bài báo Pregel. | BSP, Pregel |
| **Damping factor** | Hệ số trong PageRank thật, mô phỏng người dùng nhảy ngẫu nhiên sang trang bất kỳ. | PageRank |

<a id="chuong-11"></a>

## Chương 11 — Stream Processing

### Event stream và messaging system

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Stream processing / stream process** **(G)** | Xử lý luồng | Tính toán chạy liên tục, tiêu thụ một stream event không bao giờ kết thúc và dẫn xuất output từ nó; xử lý từng event khi nó xảy ra thay vì gom theo ngày, theo giờ. | tr. 439–440, 557 |
| **Bounded / unbounded** **(G)** | Có giới hạn / không giới hạn | Bounded: kích thước biết trước, hữu hạn (input của batch). Unbounded: dữ liệu đến dần mãi, dataset không bao giờ "đầy đủ". | tr. 439, 553, 558 |
| **Stream** | Luồng | Dữ liệu được cung cấp dần theo thời gian: stdin/stdout, lazy list, kết nối TCP, video qua internet. | tr. 440 |
| **Event** | Sự kiện | Object nhỏ, tự chứa, bất biến, mô tả một việc đã xảy ra, thường kèm timestamp theo đồng hồ time-of-day. Tương đương bản ghi trong batch. | tr. 440 |
| **Producer / publisher / sender** | Bên sinh event | Bên sinh event một lần và gửi đi. | tr. 440 |
| **Consumer / subscriber / recipient** | Bên tiêu thụ | Bên xử lý event; một event có thể được nhiều consumer xử lý. | tr. 440 |
| **Topic / stream** | Chủ đề / luồng | Nhóm các event liên quan, giống tên file gom các bản ghi liên quan. | tr. 441 |
| **Polling** | Thăm dò định kỳ | Consumer định kỳ hỏi datastore có event mới không; poll càng dày thì tỷ lệ request vô ích càng cao. | tr. 441 |
| **Messaging system** | Hệ thống truyền message | Producer gửi message chứa event, message được đẩy tới consumer; cho nhiều producer và nhiều consumer trên một topic. | tr. 441 |
| **Publish/subscribe** | Xuất bản/đăng ký | Mô hình nhiều producer gửi vào topic, nhiều consumer nhận từ topic; mỗi hệ thống làm một kiểu. | tr. 441 |
| **Backpressure / flow control** **(G)** | Áp lực ngược / điều khiển luồng | Chặn hoặc làm chậm bên gửi vì bên nhận không theo kịp; Unix pipe và TCP dùng buffer nhỏ cố định. Một trong ba lựa chọn cùng drop và buffer. | tr. 441–442, 553, 555 |
| **Direct messaging** | Truyền message trực tiếp | Producer gửi thẳng tới consumer qua mạng, không qua node trung gian; giả định hai bên luôn online. | tr. 442–443 |
| **UDP multicast** | — | Dùng trong tài chính cho stream giá chứng khoán vì latency thấp; giao thức tầng ứng dụng gửi lại gói mất. | tr. 442 |
| **ZeroMQ / nanomsg** | — | Thư viện messaging không cần broker, publish/subscribe qua TCP hoặc IP multicast. | tr. 442 |
| **StatsD / Brubeck** | — | Thu thập metric qua UDP không tin cậy; bộ đếm chỉ đúng khi nhận đủ message, nên tối đa là xấp xỉ. | tr. 442 |
| **Webhook** | — | URL callback của một service được đăng ký với service khác, và service kia gửi request tới URL đó mỗi khi có event. | tr. 443 |
| **Message broker / message queue** | Trung gian tin nhắn / hàng đợi tin nhắn | Về bản chất là một loại database tối ưu cho message stream; chạy như server, chịu được client đến đi; consumer thường bất đồng bộ. | tr. 443 |
| **JMS / AMQP** | — | Hai chuẩn của broker truyền thống (RabbitMQ, ActiveMQ, HornetQ, Qpid, IBM MQ…): giao từng message, xóa khi được xác nhận. | tr. 444 |
| **Load balancing (messaging)** | Cân bằng tải | Mỗi message được giao tới một trong các consumer để chia việc. JMS gọi là shared subscription. | tr. 444 |
| **Fan-out (messaging)** | Phát tán | Mỗi message được giao tới mọi consumer; JMS: topic subscription, AMQP: exchange binding. | tr. 445 |
| **Acknowledgment** | Xác nhận | Consumer báo broker đã xử lý xong message để broker xóa nó. | tr. 445 |
| **Redelivery** | Giao lại | Không có ack (kết nối đóng, timeout) thì broker giao message cho consumer khác; cùng load balancing sẽ làm đảo thứ tự. | tr. 445–446 |

### Partitioned log

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Log** **(G)** | Nhật ký | File chỉ nối thêm; event log có thể biểu diễn một data stream. | tr. 447, 556 |
| **Log-based message broker** | Broker dựa trên log | Producer nối message vào cuối log, consumer đọc tuần tự; lai giữa lưu trữ durable của database và thông báo độ trễ thấp của messaging. Kafka, Kinesis Streams, DistributedLog. | tr. 447–448 |
| **Partition (của log)** | Phân vùng | Mỗi partition là một log riêng trên máy khác nhau; topic là nhóm partition cùng loại message. | tr. 447 |
| **Offset** | Vị trí | Số thứ tự tăng đơn điệu broker gán cho mỗi message trong partition. Thứ tự toàn phần trong partition, không có giữa các partition. | tr. 447 |
| **Total order** **(G)** | Thứ tự toàn phần | So sánh được mọi cặp; message trong một partition có thứ tự toàn phần. | tr. 447, 557–558 |
| **Consumer group** | Nhóm consumer | Các node chia nhau tiêu thụ một topic; broker gán cả partition cho từng node. | tr. 448 |
| **Head-of-line blocking** | Nghẽn đầu hàng | Một message chậm chặn các message sau nó trong cùng partition. | tr. 449 |
| **Consumer offset** | Vị trí của consumer | Mọi message có offset nhỏ hơn đã được xử lý; broker chỉ cần định kỳ ghi offset thay vì theo dõi ack từng message. Giống log sequence number. | tr. 449 |
| **Segment** | Đoạn log | Log được chia thành segment; segment cũ bị xóa hoặc chuyển sang lưu trữ để thu hồi đĩa. | tr. 450 |
| **Circular buffer / ring buffer** | Bộ đệm vòng | Buffer kích thước cố định bỏ dữ liệu cũ khi đầy; log trên đĩa là một ring buffer lớn (6 TB ở 150 MB/s ≈ 11 giờ). | tr. 450 |
| **Replay** | Phát lại | Chỉnh offset để xử lý lại message cũ, ví dụ từ offset hôm qua; tiêu thụ log là thao tác chỉ đọc. | tr. 451 |

### Database và stream

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **State machine replication** | Nhân bản máy trạng thái | Mọi replica xử lý cùng các event ghi theo cùng thứ tự, xử lý tất định, thì đi tới cùng trạng thái. | tr. 452 |
| **ETL** **(G)** | Trích xuất, biến đổi, nạp | Cách đồng bộ data warehouse truyền thống: chép toàn bộ database, biến đổi, nạp hàng loạt. | tr. 452, 554 |
| **Dual writes** | Ghi kép | Code ứng dụng tự ghi vào từng hệ thống khi dữ liệu đổi; gặp race condition và ghi thất bại một phần. | tr. 452–453 |
| **Atomic commit** (*atomic* **(G)**) | Commit nguyên tử | Bảo đảm nhiều lần ghi cùng thành công hoặc cùng thất bại; giải cho dual writes thì tốn kém. | tr. 453, 477, 553 |
| **Change data capture (CDC)** | Thu thập thay đổi dữ liệu | Quan sát mọi thay đổi ghi vào database và trích ra thành stream để nhân bản sang hệ thống khác; biến database thành leader, hệ khác thành follower. | tr. 454–455 |
| **System of record / source of truth** **(G)** | Hệ thống gốc / nguồn sự thật | Hệ thống giữ phiên bản có thẩm quyền của dữ liệu; thay đổi được ghi ở đây trước. | tr. 454, 557 |
| **Derived data system / derived data** **(G)** | Hệ thống dữ liệu dẫn xuất | Dữ liệu tạo từ dữ liệu khác qua quá trình lặp lại được: search index, cache, data warehouse, materialized view. | tr. 439, 454, 554 |
| **Changelog table** | Bảng changelog | Bảng do trigger ghi thay đổi vào, một cách cài CDC; dễ vỡ, tốn hiệu năng. | tr. 455 |
| **Binlog / oplog** | — | Log nhân bản của MySQL và MongoDB, được Maxwell, Debezium, Mongoriver parse để làm CDC. | tr. 455 |
| **Initial snapshot** | Ảnh chụp ban đầu | Snapshot nhất quán ứng với một offset đã biết trong change log, dùng khi log không còn đủ lịch sử. | tr. 455 |
| **Log compaction** | Nén log | Chỉ giữ bản ghi mới nhất cho mỗi key (tombstone để xóa); dung lượng chỉ phụ thuộc nội dung hiện tại; đọc từ offset 0 là có bản sao đầy đủ. Kafka hỗ trợ. | tr. 456 |
| **Tombstone** | Dấu xóa | Bản cập nhật với giá trị null báo key đã bị xóa, khiến key bị loại khi compaction. | tr. 456 |
| **Change stream / change feed** | Luồng thay đổi | Database phơi thay đổi như giao diện chính thức: RethinkDB, Firebase, CouchDB, Meteor, VoltDB. | tr. 456–457 |
| **Kafka Connect** | — | Nỗ lực tích hợp công cụ CDC của nhiều database với Kafka; sink xuất dữ liệu từ Kafka sang database và index khác. | tr. 457, 461 |
| **Event sourcing** | Lưu trữ theo sự kiện | Lưu mọi thay đổi trạng thái ứng dụng thành log event bất biến ở mức ứng dụng (ý định người dùng), không phải thay đổi trạng thái mức thấp. | tr. 457 |
| **Domain-driven design (DDD)** | Thiết kế hướng miền | Cộng đồng phát triển kỹ thuật event sourcing. | tr. 457 |
| **Event store** | Kho event | Kho chỉ nối thêm cho event sourcing; có database chuyên dụng như Event Store. | tr. 457–458 |
| **Chronicle data model** | Mô hình dữ liệu biên niên | Mô hình dữ liệu mà event sourcing giống. | tr. 458 |
| **Deterministic** **(G)** | Tất định | Cùng input luôn cho cùng output; phép dẫn xuất state từ event log phải tất định để chạy lại được. | tr. 458, 554 |
| **Command** | Lệnh | Request mới đến, còn có thể bị từ chối; phải được kiểm tra đồng bộ trước khi thành event. | tr. 459 |
| **Event (là sự thật)** | Sự kiện | Command đã được chấp nhận, durable và bất biến; consumer không được từ chối; hủy là một event mới. | tr. 459 |
| **Tentative reservation** | Đặt tạm | Tách việc đặt chỗ thành event đặt tạm và event xác nhận để kiểm tra bất đồng bộ. | tr. 459 |
| **Immutability** | Tính bất biến | Dữ liệu không bị sửa hay xóa tại chỗ; giúp batch và event log chạy lại an toàn. | tr. 459 |
| **Changelog** | Nhật ký thay đổi | Log mọi thay đổi, biểu diễn sự tiến hóa của state theo thời gian; state là tích phân của stream, stream là đạo hàm của state. | tr. 460 |
| **Ledger / compensating transaction** | Sổ cái / giao dịch bù trừ | Sổ kế toán chỉ ghi thêm; sửa sai bằng giao dịch bù trừ chứ không xóa. | tr. 460–461 |
| **Read-optimized view** | View tối ưu cho đọc | Biểu diễn dẫn xuất từ event log cho một kiểu truy vấn; có thể dựng nhiều view từ một log. | tr. 461 |
| **CQRS** (*command query responsibility segregation*) | Tách trách nhiệm ghi và đọc | Tách dạng dữ liệu được ghi khỏi dạng được đọc, cho phép nhiều view đọc. | tr. 462 |
| **Denormalize** **(G)** | Phi chuẩn hóa | Thêm dư thừa để tăng tốc đọc; hợp lý trong view đọc vì quá trình dịch từ event log giữ chúng nhất quán. | tr. 462, 554 |
| **Churn** | Mức biến động | Tỷ lệ update và delete; churn cao trên dataset nhỏ làm lịch sử bất biến phình to. | tr. 463 |
| **Excision / shunning** | Cắt bỏ | Viết lại lịch sử để dữ liệu như chưa từng được ghi: excision của Datomic, shunning của Fossil. | tr. 463 |

### Xử lý stream, thời gian và join

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Operator / job** | Toán tử / công việc | Đoạn code xử lý input stream để sinh output stream; đọc chỉ đọc, ghi chỉ nối thêm. | tr. 464 |
| **Complex event processing (CEP) / complex event** | Xử lý sự kiện phức hợp | Đặt quy tắc tìm mẫu event trong stream (như regex); truy vấn lưu lâu dài, event chảy qua; khớp thì phát complex event. Esper, StreamBase. | tr. 465–466 |
| **Stream analytics** | Phân tích luồng | Tổng hợp và thống kê trên nhiều event: tần suất, trung bình trượt, so với kỳ trước. | tr. 466 |
| **Rolling average** | Trung bình trượt | Trung bình một giá trị trong một khoảng thời gian gần nhất. | tr. 466 |
| **Window** | Cửa sổ | Khoảng thời gian dùng để tổng hợp. | tr. 466, 472 |
| **Probabilistic algorithm / HyperLogLog** | Thuật toán xác suất | Cho kết quả xấp xỉ, tốn ít bộ nhớ: Bloom filter, HyperLogLog (ước lượng cardinality), ước lượng percentile. Chỉ là tối ưu hóa, stream processing không vốn dĩ xấp xỉ. | tr. 466 |
| **Materialize / materialized view** **(G)** | Vật chất hóa / view vật chất hóa | Tính trước và lưu kết quả; duy trì view từ stream thay đổi cần window kéo về đầu thời gian. | tr. 467, 556 |
| **Search on streams / percolator** | Tìm kiếm trên luồng | Lưu truy vấn tìm kiếm trước, cho tài liệu chạy qua; có thể index cả truy vấn. Percolator của Elasticsearch. | tr. 467 |
| **Distributed RPC (Storm)** | — | Truy vấn người dùng được phân ra các node đang xử lý stream, đan xen với event, rồi tổng hợp kết quả. | tr. 468 |
| **Event time** | Thời gian sự kiện | Timestamp nhúng trong event, thời điểm việc thật sự xảy ra. | tr. 468–469 |
| **Processing time** | Thời gian xử lý | Đồng hồ cục bộ của máy xử lý; dùng để chia window thì sai khi có processing lag (đỉnh giả sau redeploy). | tr. 469 |
| **Processing lag** | Độ trễ xử lý | Khoảng trễ giữa lúc event xảy ra và lúc được xử lý: xếp hàng, lỗi mạng, khởi động lại, xử lý lại. | tr. 469 |
| **Straggler** | Event đến muộn | Event tới sau khi window đã được tuyên bố xong; bỏ qua (kèm metric) hoặc công bố correction. | tr. 470 |
| **Correction / retraction** | Bản sửa / rút lại | Công bố giá trị mới của window kèm straggler, có thể rút lại output cũ. | tr. 470 |
| **Tumbling window** | Cửa sổ lật | Độ dài cố định, mỗi event thuộc đúng một window. | tr. 472 |
| **Hopping window** | Cửa sổ nhảy | Độ dài cố định, chồng lên nhau theo bước nhảy để làm mượt. | tr. 472 |
| **Sliding window** | Cửa sổ trượt | Chứa mọi event trong một khoảng cách với nhau, không có ranh giới cố định. | tr. 472 |
| **Session window / sessionization** | Cửa sổ phiên | Không độ dài cố định; gom event của một người dùng, kết thúc khi người dùng nghỉ một lúc (ví dụ 30 phút). | tr. 472 |
| **Join** **(G)** | Kết nối | Ghép các bản ghi có điểm chung; trên stream khó hơn vì event mới đến bất cứ lúc nào. | tr. 472–473, 555 |
| **Stream-stream join (window join)** | Join luồng-luồng | Ghép event liên quan của hai stream trong một window, ví dụ tìm kiếm và click cùng session ID. | tr. 473 |
| **Click-through rate** | Tỷ lệ click | Tỷ lệ kết quả được click; cần cả event tìm kiếm không có click. | tr. 473 |
| **Stream-table join (stream enrichment)** | Join luồng-bảng / làm giàu | Bổ sung thông tin từ bảng (hồ sơ) vào event; giữ bản sao cục bộ của bảng cập nhật bằng CDC. | tr. 473–474 |
| **Table-table join (materialized view maintenance)** | Join bảng-bảng | Hai changelog; mỗi thay đổi bên này join với trạng thái mới nhất bên kia; ví dụ timeline Twitter. | tr. 474–475 |
| **Time-dependence of joins** | Phụ thuộc thời gian của join | Join với state thay đổi theo thời gian thì phải chọn thời điểm; thứ tự giữa các stream không xác định làm join không tất định. | tr. 475–476 |
| **Slowly changing dimension (SCD)** | Chiều thay đổi chậm | Cấp ID cho mỗi phiên bản bản ghi được join (ví dụ thuế suất); join tất định nhưng không log compaction được. | tr. 476 |

### Fault tolerance

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Exactly-once semantics / effectively-once** | Ngữ nghĩa đúng một lần | Output trông như mỗi bản ghi được xử lý đúng một lần dù thực tế có thể xử lý lại; "effectively-once" mô tả đúng hơn. | tr. 476 |
| **Microbatching** | Lô siêu nhỏ | Chia stream thành batch khoảng một giây (Spark Streaming); ngầm tạo tumbling window theo processing time. | tr. 477 |
| **Checkpointing / barrier** | Điểm kiểm tra / rào | Định kỳ chụp state ra kho durable, kích hoạt bằng barrier trong stream (Flink); crash thì quay về checkpoint cuối. | tr. 477 |
| **Two-phase commit (2PC) / XA** **(G)** | Commit hai pha | Atomic commit giữa nhiều hệ; XA xuyên công nghệ dị chủng có nhiều vấn đề; stream framework làm atomic commit nội bộ hiệu quả hơn. | tr. 443, 477, 558 |
| **Idempotent / idempotence** **(G)** | Lũy đẳng | Làm nhiều lần cùng hiệu quả như một lần; làm thao tác thành idempotent bằng cách ghi kèm offset Kafka. Cần replay cùng thứ tự, xử lý tất định, một người ghi. | tr. 478, 555 |
| **Fencing** | Rào chắn | Ngăn node bị tưởng đã chết nhưng còn sống ghi chen vào khi failover việc xử lý. | tr. 478 |
| **Rebuilding state** | Khôi phục state | Snapshot định kỳ (Flink), changelog vào topic log compaction (Samza, Kafka Streams), xử lý dư thừa (VoltDB), hoặc dựng lại từ input. | tr. 478–479 |

### Bổ sung ngoài sách (liên quan chương 11)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **At-least-once / at-most-once delivery** | At-least-once: không mất message nhưng có thể xử lý lặp (ack và redelivery). At-most-once: không lặp nhưng có thể mất. | Acknowledgment, exactly-once |
| **Consumer lag** | Khoảng cách giữa offset mới nhất của partition và offset của consumer; metric giám sát chính của broker dựa trên log. | Consumer offset, ring buffer |
| **Partition key** | Trong Kafka, chọn partition theo hash của key (ví dụ user ID) để mọi message cùng key giữ thứ tự. | Partitioned log |
| **Watermark** | Tên thường dùng (MillWheel, Dataflow, Flink) cho message báo "không còn event nào trước t"; thường lấy mức tối thiểu của mọi input. | Straggler, window |
| **Transactional outbox** | Ghi thay đổi nghiệp vụ và một hàng "outbox" trong cùng transaction, rồi CDC đọc bảng outbox; tránh dual writes. | Dual writes, CDC |
| **Crypto-shredding** | Mã hóa dữ liệu cá nhân bằng khóa riêng mỗi người dùng; hủy khóa để "xóa" mọi bản sao trong log bất biến. | Limitations of immutability |
| **Chandy–Lamport snapshot** | Thuật toán chụp snapshot phân tán nhất quán; barrier của Flink là một biến thể. | Checkpointing |
| **Kafka transactions (KIP-98)** | Phát hành trong Kafka 0.11 (2017, sau khi sách in): producer idempotent, ghi output và consumer offset trong cùng transaction, chỉ bên trong Kafka. | Atomic commit revisited |

<a id="chuong-12"></a>

## Chương 12 — The Future of Data Systems

Chương tổng hợp, nhiều thuật ngữ đã gặp ở chương trước; ở đây ghi nghĩa theo ngữ cảnh chương 12.

### Data integration và derived data

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Data integration** | Tích hợp dữ liệu | Bảo đảm dữ liệu ở đúng dạng, ở mọi nơi cần, khi phải ghép nhiều công cụ chuyên dụng. | tr. 490–495 |
| **System of record** **(G)** | Hệ ghi gốc, nguồn sự thật | Hệ giữ phiên bản gốc, có thẩm quyền của dữ liệu; thay đổi được ghi ở đây trước, các dataset khác dẫn xuất từ nó. | tr. 491, 543, 557 |
| **Derived data** **(G)** | Dữ liệu dẫn xuất | Dataset tạo ra từ dữ liệu khác qua một quy trình lặp lại được (index, cache, materialized view, model). | tr. 491, 495, 554 |
| **Change data capture (CDC)** | Thu thập thay đổi dữ liệu | Bắt thay đổi của system of record và áp lên hệ derived theo đúng thứ tự; nên là cách duy nhất cập nhật hệ derived. | tr. 491 |
| **Dual writes** | Ghi kép | Ứng dụng tự ghi vào nhiều hệ; hai lần ghi đồng thời có thể được áp dụng theo thứ tự khác nhau ở mỗi hệ, gây bất nhất vĩnh viễn. | tr. 491, 507 |
| **Total order** **(G)** | Thứ tự toàn phần | Cách sắp xếp luôn so sánh được hai phần tử bất kỳ. Dồn mọi ghi qua một hệ quyết định total order giúp dẫn xuất nhất quán. | tr. 491, 493, 557–558 |
| **Total order broadcast** | Phát quảng bá theo thứ tự toàn phần | Mọi node nhận message theo cùng thứ tự; tương đương consensus. Log cung cấp đảm bảo này trong mỗi partition. | tr. 491, 493, 522 |
| **State machine replication** | Nhân bản máy trạng thái | Mọi bản sao xử lý cùng chuỗi input theo cùng thứ tự nên đi tới cùng trạng thái; nền của việc dẫn xuất từ log. | tr. 491 |
| **Event sourcing** | Lưu trữ theo sự kiện | Ghi input thành event bất biến, mọi state dẫn xuất từ event; giúp integrity và auditability. | tr. 491, 497, 519, 526, 531 |
| **Deterministic** **(G)** | Tất định | Hàm luôn cho cùng output với cùng input; điều kiện để retry và reprocess an toàn. | tr. 492, 495, 554 |
| **Idempotent** **(G)** | Lũy đẳng | Thao tác chạy nhiều lần có cùng hiệu ứng như chạy một lần; thay atomic commit trong hệ dựa trên log. | tr. 492, 517, 555 |
| **Distributed transaction / XA** | Transaction phân tán | Giữ nhiều hệ nhất quán bằng lock và atomic commit; tác giả cho rằng XA chịu lỗi và hiệu năng kém, khuếch đại lỗi. | tr. 492, 502, 528 |
| **Two-phase commit (2PC)** **(G)** | Commit hai pha | Thuật toán để nhiều node cùng commit hoặc cùng abort; không phủ chặng mạng từ người dùng tới server. | tr. 492, 517–518, 558 |
| **Two-phase locking (2PL)** **(G)** | Khóa hai pha | Cách distributed transaction quyết định thứ tự ghi bằng lock. | tr. 492, 558 |
| **Linearizable** **(G)** | Tuyến tính hóa được | Hành xử như chỉ có một bản dữ liệu; transaction thường có tính chất này, hệ derived bất đồng bộ thì không. | tr. 492, 524, 555 |
| **Consensus** **(G)** | Đồng thuận | Cần để quyết định total order và thực thi uniqueness; thuật toán hiện có giả định một node đủ xử lý cả stream. | tr. 493, 521, 554 |
| **Causality** **(G)** | Quan hệ nhân quả | Phụ thuộc giữa các event; có thể mất khi event nằm ở các hệ khác nhau (ví dụ unfriend rồi gửi tin). | tr. 493–494, 514, 554 |
| **Logical timestamp** | Mốc thời gian logic | Cho total ordering không cần phối hợp, nhưng bên nhận vẫn phải xử lý event tới sai thứ tự. | tr. 494 |
| **Microservices** | Vi dịch vụ | Mỗi service với state bền riêng; event từ hai service không có thứ tự xác định. | tr. 493, 508 |

### Batch, stream và tiến hóa ứng dụng

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Batch process** **(G)** | Xử lý theo lô | Tính toán trên input hữu hạn, không sửa input; dùng để reprocess lịch sử. | tr. 494–496, 553 |
| **Stream process** **(G)** | Xử lý luồng | Tính toán chạy liên tục trên stream event không giới hạn; phản ánh thay đổi với độ trễ thấp. | tr. 494–496, 557 |
| **Bounded / unbounded** **(G)** | Có giới hạn / không giới hạn | Input của batch hữu hạn, của stream không giới hạn; khác biệt căn bản giữa hai loại xử lý. | tr. 495, 553, 558 |
| **Microbatch** | Lô siêu nhỏ | Spark làm stream processing bằng cách chia stream thành các lô nhỏ; có thể kém với hopping hay sliding window. | tr. 495 |
| **Pure function** | Hàm thuần | Output chỉ phụ thuộc input, không side effect ngoài output tường minh; tinh thần functional của batch. | tr. 495 |
| **Reprocessing** | Xử lý lại | Chạy lại phép dẫn xuất trên dữ liệu lịch sử để tạo view mới; cho phép đổi sang mô hình dữ liệu khác hẳn. | tr. 496–497, 544 |
| **Dual gauge / mixed gauge** | Đường ray hai khổ | Thêm ray thứ ba để tàu hai khổ cùng chạy trong lúc chuyển đổi; ví dụ cho migration dần dần. | tr. 496 |
| **Gradual migration** | Chuyển đổi dần | Giữ view cũ và mới song song, chuyển người dùng dần, mỗi bước đảo ngược được. | tr. 497 |
| **Lambda architecture** | Kiến trúc lambda | Chạy song song stream (xấp xỉ, nhanh) và batch (chính xác, chậm) trên cùng event bất biến; phải duy trì logic hai nơi và gộp output. | tr. 497–498 |
| **Exactly-once / effectively-once semantics** | Ngữ nghĩa đúng một lần | Hiệu ứng cuối cùng như không có lỗi dù có retry; là cơ chế giữ integrity. | tr. 498, 516–517, 525 |
| **Event time / processing time** | Thời điểm sự kiện / thời điểm xử lý | Hợp nhất batch và stream cần window theo event time, vì processing time vô nghĩa khi reprocess. | tr. 498 |
| **Straggler** | Event tới muộn | Event tới sau khi window đã đóng; vấn đề khi batch chạy tăng dần. | tr. 498 |
| **Sessionization** | Gom phiên | Nhóm event theo phiên người dùng; làm việc gộp output trong lambda khó hơn. | tr. 498 |

### Unbundling database và dataflow

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Information management system** | Hệ quản lý thông tin | Cách nhìn chung database, Hadoop và hệ điều hành: đều lưu, xử lý và truy vấn dữ liệu. | tr. 499 |
| **Unbundling** | Tháo rời (database) | Đưa tính năng duy trì index, view của database ra ngoài, ghép các hệ chuyên dụng bằng CDC và event log; thống nhất ghi. Thuật ngữ của Jay Kreps. | tr. 501–504 |
| **Federated database / polystore** | Database liên hợp | Một giao diện truy vấn thống nhất trên nhiều storage engine; thống nhất đọc, không giải quyết đồng bộ ghi. | tr. 501 |
| **Foreign data wrapper** | Bộ bọc dữ liệu ngoài | Tính năng PostgreSQL cho truy vấn dữ liệu ở nguồn khác; ví dụ của federated database. | tr. 501 |
| **Meta-database of everything** | Siêu database của mọi thứ | Coi dataflow của cả tổ chức như một database lớn; batch, stream, ETL như subsystem duy trì index. | tr. 500–501 |
| **Secondary index** **(G)** | Chỉ mục phụ | Ví dụ derived dataset có hàm dẫn xuất đơn giản, được database tích hợp sẵn qua `CREATE INDEX`. | tr. 499–500, 505, 557 |
| **Materialize / materialized view** **(G materialize)** | Tính sẵn / view tính sẵn | Cache tính sẵn kết quả truy vấn; cùng với cache và index, dịch ranh giới write path và read path. | tr. 500, 504, 510–511, 556 |
| **Full-text search** **(G)** | Tìm kiếm toàn văn | Tìm theo từ khóa bất kỳ; index dẫn xuất qua xử lý ngôn ngữ tự nhiên rồi dựng inverted index. | tr. 490, 505, 510, 555 |
| **Loose coupling** | Liên kết lỏng | Lợi thế của tích hợp qua log: lỗi được khoanh vùng (mức hệ thống), đội làm việc độc lập (mức con người). | tr. 502, 543–544 |
| **Premature optimization** | Tối ưu hóa quá sớm | Xây cho quy mô chưa cần; lý do không nên unbundle khi một sản phẩm đã đủ. | tr. 503 |
| **Differential dataflow** | — | Hướng nghiên cứu cho phép khai báo materialized view phức tạp được cập nhật tăng dần. | tr. 504 |
| **Database inside-out** | Database lộn trong ra ngoài | Tên gọi khác của cách unbundle, theo bài nói năm 2014 của tác giả; tác giả coi là design pattern. | tr. 504 |
| **Functional reactive programming (FRP)** | Lập trình phản ứng hàm | Mô hình lập trình như Elm, nơi giá trị tự cập nhật theo thay đổi; một nguồn ý tưởng của dataflow. | tr. 504 |
| **Derivation function** | Hàm dẫn xuất | Hàm biến đổi tạo derived dataset từ dataset gốc; code ứng dụng đóng vai này. | tr. 505 |
| **Feature engineering** | Kỹ thuật đặc trưng | Bước dẫn xuất đặc trưng cho machine learning, nổi tiếng là đặc thù ứng dụng. | tr. 505 |
| **Inverted index / posting list** | Chỉ mục ngược / danh sách vị trí | Cấu trúc tra cứu của full-text index; search index trả posting list cho search query executor (Hình 12-1). | tr. 505, 509 |
| **Stored procedure** **(G)** | Thủ tục lưu sẵn | Cách chạy code trong database; hàm dẫn xuất tất định giống stored procedure. | tr. 505, 526, 557 |
| **Stateless service** | Dịch vụ không trạng thái | Server quên request sau khi trả lời; state nằm trong database. | tr. 506 |
| **Observer pattern** | Mẫu quan sát | Tự cài thông báo khi giá trị đổi; hầu hết ngôn ngữ và database không có sẵn, nên phải poll. | tr. 506 |
| **Tuple spaces** | Không gian bộ | Mô hình thập niên 1980: tiến trình quan sát thay đổi state và phản ứng. | tr. 507 |
| **Stream operator** | Toán tử luồng | Code ứng dụng chạy trong stream processor, nhận stream thay đổi và sinh stream thay đổi khác; ghép như pipe Unix. | tr. 507–508 |
| **Stream-table join (stream enrichment)** | Join luồng với bảng | Thay RPC: subscribe stream tỷ giá rồi join với event mua hàng; phục vụ request đọc cũng là kiểu join này. | tr. 509, 514 |
| **Cache** **(G)** | Bộ đệm | Thường chứa dữ liệu ở dạng hiển thị trên UI; state trên thiết bị người dùng cũng là cache của state server. | tr. 505, 510–512, 553–554 |

### Observing derived state

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Write path / read path** | Đường ghi / đường đọc | Phần hành trình dữ liệu tính sẵn khi ghi, và phần chỉ chạy khi có người hỏi; gặp nhau ở derived dataset. | tr. 509–510 |
| **Eager / lazy evaluation** | Tính ngay / tính khi cần | Write path giống eager evaluation, read path giống lazy evaluation. | tr. 510 |
| **Offline-first application** | Ứng dụng ưu tiên offline | Làm nhiều nhất có thể bằng database cục bộ, đồng bộ nền khi có mạng. | tr. 511 |
| **Server-sent events (EventSource) / WebSockets** | — | Giao thức giữ kết nối mở để server chủ động đẩy thay đổi tới trình duyệt. | tr. 512 |
| **Consumer offset** | Vị trí đọc của consumer | Cho thiết bị offline kết nối lại và đọc tiếp, không bỏ lỡ thay đổi. | tr. 512 |
| **End-to-end event stream** | Luồng sự kiện đầu cuối | Write path kéo dài từ thao tác trên một thiết bị tới UI của người khác, độ trễ dưới một giây. | tr. 512–513 |
| **Publish/subscribe dataflow** | Dataflow xuất bản/đăng ký | Hướng thay cho request/response để kéo write path tới người dùng. | tr. 513 |
| **Reads are events** | Đọc cũng là sự kiện | Biểu diễn request đọc thành event đi qua stream processor; request một lần đi qua join, subscribe là join bền. | tr. 513–514 |
| **Data provenance** | Nguồn gốc dữ liệu | Biết dữ liệu được dẫn xuất từ đâu; log đọc và dataflow tường minh giúp truy nguồn. | tr. 514, 531 |
| **Distributed RPC (Storm)** | — | Tính năng Storm cho truy vấn kết hợp nhiều partition, ví dụ đếm người đã thấy một URL. | tr. 514 |

### Correctness

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Correctness** | Tính đúng | Chương trình có ngữ nghĩa định nghĩa rõ và hiểu được, kể cả khi có lỗi. | tr. 515 |
| **Jepsen** | — | Loạt thí nghiệm của Kyle Kingsbury cho thấy khác biệt giữa đảm bảo sản phẩm tuyên bố và hành vi thật. | tr. 515 |
| **Serializable** **(G)** | Khả tuần tự | Đảm bảo transaction mạnh nhất, nhưng không ngăn được bug ứng dụng hay request trùng. | tr. 516, 557 |
| **Atomic** **(G)** | Nguyên tử | Ghi một object là nguyên tử ở hầu hết data system; dùng để tránh atomic commit nhiều partition. | tr. 515, 523, 553 |
| **Duplicate suppression** | Chặn bản trùng | Loại xử lý trùng; TCP làm trong một kết nối, cần request ID cho end-to-end. | tr. 517–519 |
| **Operation identifier / request ID** | Mã thao tác | ID duy nhất sinh ở client (UUID hay hash các trường form), truyền tới database để chỉ áp dụng một lần. | tr. 518–519, 523 |
| **Post/Redirect/Get** | — | Mẫu web tránh cảnh báo gửi lại form khi bình thường, nhưng không giúp khi POST timeout. | tr. 518 |
| **End-to-end argument** | Lập luận đầu cuối | Saltzer, Reed, Clark 1984: một chức năng chỉ làm đầy đủ và đúng được với sự tham gia của ứng dụng ở hai đầu. | tr. 519–520, 531 |
| **End-to-end checksum / encryption** | Checksum / mã hóa đầu cuối | Bắt mọi nguồn hỏng và chống mọi kẻ tấn công, điều checksum TCP hay TLS từng chặng không làm được. | tr. 519–520 |
| **Uniqueness constraint** | Ràng buộc duy nhất | Username, email, ghế ngồi…; cần consensus, thực thi được bằng log partition theo giá trị. | tr. 521–522 |
| **Multi-partition request processing** | Xử lý request nhiều partition | Ghi request thành một message, dẫn xuất debit và credit tất định, khử trùng theo request ID; không cần atomic commit. | tr. 522–524 |
| **Checkpoint** | Điểm lưu | Stream processor crash thì tiếp tục từ đây; có thể xử lý lại nhưng không bỏ sót. | tr. 523 |
| **Timeliness** | Tính kịp thời | Người dùng thấy trạng thái cập nhật; vi phạm là tạm thời ("eventual consistency"). | tr. 524 |
| **Integrity** | Tính toàn vẹn | Không mất, không mâu thuẫn, dẫn xuất đúng; vi phạm là vĩnh viễn ("perpetual inconsistency"). Quan trọng hơn timeliness. | tr. 524–526 |
| **CAP theorem** **(G)** | Định lý CAP | Dùng consistency theo nghĩa linearizability, một cách mạnh để có timeliness. | tr. 524, 554 |
| **Compensating transaction** | Transaction bù | Thay đổi sửa một vi phạm ràng buộc sau khi nó xảy ra (xin lỗi, hoàn tiền, đổi lựa chọn). | tr. 526 |
| **Loosely interpreted constraint** | Ràng buộc diễn giải lỏng | Cho tạm vi phạm rồi sửa bằng xin lỗi, khi chi phí xin lỗi chấp nhận được. | tr. 526–527 |
| **Overbooking** | Bán vượt chỗ | Hãng bay, khách sạn cố ý vi phạm "một người một chỗ" và có quy trình bồi thường. | tr. 526–527 |
| **Coordination-avoiding data system** | Hệ dữ liệu tránh phối hợp | Giữ integrity mạnh mà không phối hợp đồng bộ; timeliness yếu, chạy được multi-leader nhiều datacenter. | tr. 527–528 |
| **System model** | Mô hình hệ thống | Tập giả định về điều có thể hỏng; thực tế là xác suất chứ không nhị phân. | tr. 528 |
| **Bit flip / rowhammer** | Lật bit / búa hàng | Bit bộ nhớ đổi ngẫu nhiên; rowhammer là mẫu truy cập làm lật bit ở hàng nhớ lân cận. | tr. 529 |
| **Auditing** | Kiểm toán | Kiểm tra integrity của dữ liệu để phát hiện và sửa hỏng hóc. | tr. 530–533 |
| **Trust, but verify** | Tin, nhưng kiểm chứng | Tinh thần tự kiểm toán liên tục thay vì tin mù quáng vào đảm bảo tuyệt đối. | tr. 530 |
| **Self-validating / self-auditing system** | Hệ tự kiểm tra | Hệ liên tục kiểm tra integrity của chính mình. | tr. 530 |
| **Time-travel debugging** | Debug du hành thời gian | Tái tạo chính xác hoàn cảnh dẫn tới sự kiện bất ngờ nhờ dataflow tất định. | tr. 531 |
| **Audit table** | Bảng kiểm toán | Bảng log mọi thay đổi; khó bảo đảm integrity của cả bảng này lẫn state. | tr. 532 |
| **Hardware security module** | Mô-đun bảo mật phần cứng | Dùng để ký định kỳ transaction log, chống giả mạo. | tr. 532 |
| **Distributed ledger / blockchain** | Sổ cái phân tán | Database phân tán, replica do tổ chức không tin nhau vận hành, kiểm tra integrity lẫn nhau và dùng consensus. | tr. 532 |
| **Proof of work** | Bằng chứng công việc | Kỹ thuật đào Bitcoin; tác giả thấy lãng phí phi thường. | tr. 532 |
| **Byzantine fault** **(G)** | Lỗi Byzantine | Node hành xử sai tùy ý; tác giả hoài nghi khía cạnh chịu lỗi Byzantine của blockchain. | tr. 532, 553 |
| **Merkle tree** | Cây Merkle | Cây các hash, chứng minh hiệu quả một bản ghi có trong dataset. | tr. 532 |
| **Certificate transparency** | Minh bạch chứng chỉ | Công nghệ dựa trên Merkle tree để kiểm tra tính hợp lệ của chứng chỉ TLS/SSL. | tr. 532–533 |

### Đạo đức: dự đoán, quyền riêng tư, theo dõi

| Thuật ngữ | Tiếng Việt | Giải thích | Trang |
| --- | --- | --- | --- |
| **Predictive analytics** | Phân tích dự đoán | Dùng dữ liệu dự đoán về con người (tái phạm, vỡ nợ, đòi bồi thường); ảnh hưởng trực tiếp đời sống từng người. | tr. 533–536 |
| **Algorithmic decision-making** | Ra quyết định bằng thuật toán | Quyết định tự động về con người; đặt ra câu hỏi trách nhiệm và giải trình. | tr. 534–535 |
| **Algorithmic prison** | Nhà tù thuật toán | Bị loại có hệ thống khỏi việc làm, đi lại, bảo hiểm, nhà ở, tài chính vì bị gắn nhãn rủi ro, không cần bằng chứng. | tr. 534 |
| **Bias / discrimination** | Thiên kiến / phân biệt đối xử | Input thiên lệch thì mô hình học và khuếch đại thiên lệch. | tr. 534–535 |
| **Protected traits** | Đặc điểm được bảo vệ | Sắc tộc, tuổi, giới tính, xu hướng tính dục, khuyết tật, niềm tin; luật cấm đối xử khác nhau theo chúng. | tr. 534 |
| **Proxy (biến thay thế)** | Biến đại diện | Đặc điểm tương quan mạnh với đặc điểm được bảo vệ, như mã bưu chính, địa chỉ IP, nơi ở. | tr. 534–535 |
| **Moral imagination** | Trí tưởng tượng đạo đức | Điều chỉ con người có, cần để tương lai tốt hơn quá khứ mà mô hình chỉ ngoại suy. | tr. 535 |
| **Responsibility / accountability** | Trách nhiệm / giải trình | Ai chịu trách nhiệm khi thuật toán sai; có kháng nghị và giải thích được không. | tr. 535 |
| **Credit rating agency / credit score** | Cơ quan xếp hạng tín dụng / điểm tín dụng | Dựa trên lịch sử vay của chính người đó, sửa được lỗi; khác chấm điểm bằng ML mờ đục. | tr. 535 |
| **Stereotyping** | Rập khuôn | Đánh giá một người theo hành vi của "người giống họ". | tr. 535 |
| **Echo chamber** | Buồng vang | Hệ gợi ý chỉ cho thấy ý kiến người dùng đã đồng ý; nuôi rập khuôn, tin sai, phân cực. | tr. 536 |
| **Self-reinforcing feedback loop** | Vòng phản hồi tự củng cố | Ví dụ điểm tín dụng giảm → khó tìm việc → nghèo hơn → điểm giảm thêm. | tr. 536 |
| **Systems thinking** | Tư duy hệ thống | Xét cả hệ, gồm con người tương tác, để đoán hậu quả như vòng phản hồi. | tr. 536 |
| **Behavioral data / tracking** | Dữ liệu hành vi / theo dõi | Hoạt động bị ghi log như side effect; có lợi cho tính năng nhưng dễ vượt quá khi sống nhờ quảng cáo. | tr. 536–537 |
| **Surveillance** | Giám sát | Cách tác giả gọi việc theo dõi phục vụ nhà quảng cáo chứ không phục vụ người dùng. | tr. 537–538 |
| **Consent** | Sự đồng ý | Thường không có ý nghĩa vì người dùng không hiểu, quan hệ một chiều, và không dùng thì có chi phí xã hội. | tr. 538–539 |
| **Network effect** | Hiệu ứng mạng | Làm việc từ chối dùng một dịch vụ có chi phí xã hội. | tr. 539 |
| **Privacy (decision right)** | Quyền riêng tư (quyền quyết định) | Tự do chọn tiết lộ gì cho ai; giám sát chuyển quyền này sang công ty. | tr. 539–540 |
| **Privacy settings** | Cài đặt quyền riêng tư | Kiểm soát người dùng khác thấy gì; không giới hạn chính dịch vụ. | tr. 540 |
| **Data exhaust** | Khí thải dữ liệu | Cách gọi dữ liệu hành vi như chất thải; thực chất là tài sản cốt lõi của dịch vụ quảng cáo. | tr. 540 |
| **Data broker** | Môi giới dữ liệu | Ngành mua, gộp, suy luận và bán lại dữ liệu cá nhân, chủ yếu cho marketing. | tr. 541 |
| **Toxic asset / hazardous material** | Tài sản độc hại / vật liệu nguy hiểm | Dữ liệu có thể rò rỉ, bị chính phủ, kẻ trộm, ban lãnh đạo mới lấy; phải cân nhắc khi thu thập. | tr. 541 |
| **European Data Protection Directive (1995)** | Chỉ thị bảo vệ dữ liệu châu Âu | Dữ liệu cá nhân phải thu cho mục đích cụ thể, rõ ràng, hợp pháp và không quá mức; đi ngược triết lý Big Data. | tr. 542 |
| **Self-regulation** | Tự điều chỉnh | Ngành tự hạn chế thu thập và xử lý dữ liệu để giữ niềm tin người dùng. | tr. 543 |
| **Tragedy of the commons** | Bi kịch của mảnh đất chung | Quyền kiểm soát dữ liệu như vườn quốc gia: không bảo vệ thì bị phá hủy. | tr. 543 |
| **Purging data** | Xóa dữ liệu | Không giữ dữ liệu mãi; đi ngược ý tưởng bất biến nhưng giải được. | tr. 543 |
| **Cryptographic access control** | Kiểm soát truy cập bằng mật mã | Thực thi quyền truy cập bằng giao thức mật mã thay vì chỉ bằng chính sách. | tr. 543 |

### Bổ sung ngoài sách (liên quan chương 12)

| Thuật ngữ | Giải thích | Liên quan đến |
| --- | --- | --- |
| **Kappa architecture** | Tên Jay Kreps đặt cho cách dùng một stream engine, replay log để reprocess, thay cho lambda. | Lambda architecture, hợp nhất batch/stream |
| **Canary release / dark launch** | Cho phiên bản mới nhận một phần nhỏ traffic thật rồi mở rộng dần. | Gradual migration |
| **Saga** | Chia transaction dài thành chuỗi transaction cục bộ, mỗi bước có compensating transaction (Garcia-Molina và Salem 1987, tài liệu [60] của sách). | Compensating transaction |
| **CALM theorem** | *Consistency As Logical Monotonicity*: chương trình đơn điệu chạy nhất quán không cần phối hợp; kiểm tra "chưa ai lấy" thì không đơn điệu. | Coordination-avoiding data system |
| **Merkle inclusion proof** | Chứng minh bản ghi có trong dataset bằng khoảng log₂(n) hash; 1 triệu bản ghi cần khoảng 20 hash. | Merkle tree, certificate transparency |
| **GDPR** | Quy định bảo vệ dữ liệu của EU (2016/679), áp dụng từ 25/5/2018, thay Chỉ thị 95/46/EC. | European Data Protection Directive |
| **Crypto-shredding** | Mã hóa dữ liệu mỗi người bằng khóa riêng; xóa khóa để "quên" dữ liệu trong log bất biến. | Purging data, cryptographic access control |
