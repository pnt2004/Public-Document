# Thuật ngữ DDIA (Glossary)

Thuật ngữ tiếng Anh kèm giải thích tiếng Việt, bổ sung dần theo từng chương đã học. Số trang là trang in của ấn bản thứ nhất (2017). Thuật ngữ đánh dấu **(G)** cũng có trong Glossary cuối sách (tr. 553–558). Phần giải thích ở đây là diễn giải, đôi khi kèm ngữ cảnh từ chương đang học.

Đã cập nhật đến: **Chương 5** (kèm phần giới thiệu Part II).

- [Chương 1 — Reliable, Scalable, and Maintainable Applications](#chuong-1)
- [Chương 2 — Data Models and Query Languages](#chuong-2)
- [Chương 3 — Storage and Retrieval](#chuong-3)
- [Chương 4 — Encoding and Evolution](#chuong-4)
- [Chương 5 — Replication](#chuong-5)

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
