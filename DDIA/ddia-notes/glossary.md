# Thuật ngữ DDIA (Glossary)

Thuật ngữ tiếng Anh kèm giải thích tiếng Việt, bổ sung dần theo từng chương đã học. Số trang là trang in của ấn bản thứ nhất (2017). Thuật ngữ đánh dấu **(G)** cũng có trong Glossary cuối sách (tr. 553–558). Phần giải thích ở đây là diễn giải, đôi khi kèm ngữ cảnh từ chương đang học.

Đã cập nhật đến: **Chương 2**.

- [Chương 1 — Reliable, Scalable, and Maintainable Applications](#chuong-1)
- [Chương 2 — Data Models and Query Languages](#chuong-2)

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
| **Shared-nothing architecture** **(G)** | Kiến trúc không chia sẻ | Các node độc lập, mỗi node có CPU, bộ nhớ, ổ đĩa riêng, nối với nhau qua mạng thông thường. | tr. 17, 557 |
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
