# Thuật ngữ DDIA (Glossary)

Thuật ngữ tiếng Anh kèm giải thích tiếng Việt, bổ sung dần theo từng chương đã học. Số trang là trang in của ấn bản thứ nhất (2017). Thuật ngữ đánh dấu **(G)** cũng có trong Glossary cuối sách (tr. 553–558). Phần giải thích ở đây là diễn giải, đôi khi kèm ngữ cảnh từ chương đang học.

Đã cập nhật đến: **Chương 1**.

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
