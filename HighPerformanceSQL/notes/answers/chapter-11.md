# Đáp án — Chương 11: Scaling MySQL

Quay lại ghi chú: [../chapter-11.md](../chapter-11.md). "tr. N" là số trang in; trang PDF = N + 22. Phần không nhãn diễn giải sách; **Bổ sung ngoài sách**, **Ví dụ minh họa do AI xây dựng**, **Kết quả đã đo** dùng như trong ghi chú chương. Kết quả đo lấy từ [thực hành chương 11](../experiments/chapter-11.md).

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Maximum throughput và actual capacity

*Vì sao sách tách maximum throughput khỏi actual capacity? Nếu một benchmark báo 50.000 QPS thì còn thiếu thông tin gì để nói capacity của hệ thống?*

- Benchmark thường đẩy hệ thống tới **throughput tối đa**. Hệ thống thật không thể chạy ở mức đó: hiệu năng giảm, thời gian phản hồi vừa lớn vừa dao động (tr. 288).
- Sách định nghĩa **actual capacity** là throughput đạt được **khi vẫn giữ hiệu năng chấp nhận được** (tr. 288). Vậy capacity luôn gắn với một điều kiện về latency.
- Con số 50.000 QPS thiếu: (1) latency, nhất là percentile cao, ở mức tải đó và ở các mức thấp hơn; (2) ngưỡng "chấp nhận được", tức SLO (**Bổ sung ngoài sách**, nối với [Chương 2](../chapter-02.md#sli-slo-sla)); (3) hỗn hợp truy vấn của benchmark có giống workload thật không, vì không phải truy vấn nào cũng như nhau (tr. 289–290); (4) tài nguyên nào chạm trần trước (CPU, IOPS, mạng, tr. 290).
- Kết luận: capacity là "throughput lớn nhất mà latency còn đạt SLO, với đúng hỗn hợp truy vấn của mình", thường thấp hơn nhiều so với đỉnh benchmark.

[về mục 2.1](../chapter-11.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Nhanh có nghĩa là scalable không

*Theo phép so sánh xa lộ, một server MySQL có latency truy vấn rất thấp có chắc scalable không? Nêu hai yếu tố trong MySQL tương ứng với "nút giao" hay "xe đổi làn" làm giảm scalability.*

- **Không chắc.** Sách nói capacity và scalability độc lập với performance; scalability phụ thuộc nút giao, tai nạn, xe chạy tốc độ khác nhau, đổi làn, chứ không phụ thuộc động cơ (tr. 288). Hệ thống có thể scalable mà không high performance, và ngược lại.
- Hai yếu tố trong MySQL (**Bổ sung ngoài sách**, diễn giải):
  - **Nút giao:** chỉ có một source nhận ghi. Truy vấn nhanh đến đâu thì mọi lần ghi vẫn phải đi qua một máy, và replica không chia được tải ghi (tr. 291; Chương 9, tr. 228).
  - **Hàng nóng, lock:** nhiều transaction cùng sửa một hàng phải xếp hàng chờ lock dù mỗi câu rất nhanh; thêm node ứng dụng chỉ làm hàng chờ dài hơn (tr. 298).
  - **Đổi làn:** truy vấn xuyên shard phải hỏi mọi node và chờ node chậm nhất (tr. 302).
- Vì thế muốn biết hệ thống có scalable không, phải hỏi "thêm tài nguyên thì capacity có tăng tương ứng không" (tr. 288), không phải "truy vấn chạy bao nhiêu mili giây".

[về mục 2.1](../chapter-11.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Số user gấp đôi, tải tăng hơn gấp đôi

*Số user của một mạng xã hội tăng gấp đôi. Vì sao lượng dữ liệu, số transaction và độ phức tạp truy vấn có thể tăng nhiều hơn gấp đôi?*

- **Dữ liệu:** sách nói dữ liệu có thể tăng **nhanh hơn tỷ lệ** so với số user, và nhiều ứng dụng web không bao giờ xóa dữ liệu (tr. 288–289). User cũ vẫn tiếp tục tạo nội dung trong khi user mới gia nhập.
- **Transaction:** nhiều user nghĩa là nhiều transaction, nhưng không tỷ lệ thuận (tr. 289). Một user mới kết bạn tạo ra hoạt động ở cả phía người kia.
- **Độ phức tạp truy vấn:** truy vấn phụ thuộc số quan hệ giữa user; số quan hệ bị chặn trên bởi N × (N − 1) / 2 (tr. 289). **Bổ sung ngoài sách (phép tính):** gấp đôi N thì chặn trên tăng khoảng gấp bốn (1.000 user → 499.500 cặp; 2.000 user → 1.999.000 cặp).
- **Hoạt động và dữ liệu liên quan:** user hoạt động mạnh, nhóm phổ biến, user có rất nhiều bạn khiến truy vấn trên "nhóm liên quan" nặng hơn hẳn trung bình (tr. 289).

[về mục 2.1](../chapter-11.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — MySQL scalable nhưng ứng dụng thì không

*Cho một ví dụ trong đó kiến trúc MySQL scalable nhưng ứng dụng thì không. Sách đánh giá cả hệ thống thế nào trong trường hợp đó?*

- **Ví dụ minh họa do AI xây dựng:** đội vận hành đã có read pool sẵn sàng thêm replica, nhưng ứng dụng gắn cứng một hostname cho mọi truy vấn, đọc lẫn ghi. Thêm replica không giúp gì vì ứng dụng không biết dùng chúng. Ví dụ khác: ứng dụng mở một kết nối mới cho mỗi request và không giới hạn số kết nối, nên khi scale out tầng ứng dụng thì database bị ngập kết nối (tr. 310).
- Sách: nếu tăng capacity khó vì **bất kỳ** lý do gì, cả ứng dụng **không scalable** (tr. 288). Scalability là thuộc tính của toàn hệ thống, không phải của riêng MySQL.

[về mục 2.1](../chapter-11.md#s2-1)

<a id="q2-2-1"></a>

### Câu 2.2.1 — 1.000 QPS ở 20% CPU

*Hệ thống đang chạy 1.000 QPS ở 20% CPU. Khi nào thì thêm được 4.000 QPS nữa, và hai lý do nào khiến điều đó thường không xảy ra?*

- Sách nói chỉ thêm được 4.000 QPS **nếu latency của chúng giữ nguyên** (tr. 290), tức 4.000 truy vấn mới có cùng chi phí với hỗn hợp hiện tại.
- Lý do thứ nhất: **latency tăng khi CPU tiến gần 100%** (chú thích 3, tr. 290). **Bổ sung ngoài sách:** với xấp xỉ hàng đợi M/M/1, thời gian phản hồi khoảng S / (1 − ρ); ở 100% thì về lý thuyết vô hạn.
- Lý do thứ hai: **tài nguyên khác chạm trần trước CPU**. Nếu thêm 4.000 truy vấn mà chạm trần IOPS đĩa, latency của mọi truy vấn đọc tăng (tr. 290). Mạng cũng có giới hạn throughput (tr. 290).
- Thêm một lý do ngầm: **không phải truy vấn nào cũng như nhau** (tr. 289). 4.000 QPS mới có thể nặng hơn hẳn 1.000 QPS hiện tại.

[về mục 2.2](../chapter-11.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Vì sao chỉ số hệ thống không đủ

*Nếu chỉ có chỉ số CPU, bộ nhớ, đĩa, vì sao gần như không thể biết hệ thống read-bound hay write-bound? Bạn sẽ thu thêm số liệu gì từ MySQL?*

- CPU cao có thể do đọc (truy vấn quét nhiều hàng) hay do ghi (cập nhật nhiều hàng, kiểm tra ràng buộc). IOPS đĩa cao có thể do đọc dữ liệu không có trong bộ nhớ hay do flush trang bẩn, ghi redo và binary log. Chỉ số hệ thống không gắn tài nguyên với **loại** truy vấn, nên sách nói gần như không thể biết (tr. 290).
- Sách bảo đo **latency đọc so với ghi** bằng ví dụ ở Chương 3, tr. 62, và **theo dõi xu hướng** (tr. 290).
- Số liệu cụ thể (**Bổ sung ngoài sách**): `events_statements_summary_global_by_event_name` (số câu và **`SUM_TIMER_WAIT`** theo loại câu); `events_statements_summary_by_digest` (câu đắt nhất); `table_io_waits_summary_by_table` (đọc, ghi theo bảng); `Innodb_os_log_written`, `Innodb_data_fsyncs`, `Innodb_row_lock_waits` cho đường ghi; `Innodb_buffer_pool_reads` so với `Innodb_buffer_pool_read_requests` cho đọc từ đĩa.
- Lưu ý (**Kết quả đã đo**): dùng `SUM_TIMER_WAIT`, không dùng `LOCK_TIME` như truy vấn ở tr. 62; trên workload A, xếp hạng theo `LOCK_TIME` đặt SELECT lên đầu trong khi tổng thời gian đặt UPDATE lên đầu.

[về mục 2.2](../chapter-11.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Dấu hiệu read-bound và thứ tự xử lý

*Hai dấu hiệu của read-bound là gì? Trước khi thêm replica, sách bảo làm những việc gì, và vì sao thứ tự đó hợp lý?*

- Dấu hiệu (tr. 290): **CPU cao** (chỉ báo chính; CPU càng cao latency càng tăng) và **IOPS hay throughput đọc đĩa cao** (xuống đĩa thường xuyên hoặc đọc rất nhiều hàng).
- Trước khi thêm replica: **thêm index, tối ưu truy vấn, cache** dữ liệu cache được (tr. 290). Chỉ khi hết cách cải thiện mới dùng replica.
- Vì sao hợp lý:
  - Một truy vấn thiếu index tốn tài nguyên như nhau trên mọi replica; thêm replica chỉ **nhân** chi phí đó lên, không loại bỏ nó.
  - Read pool có chi phí vận hành: load balancer, service discovery, health check, chấp nhận dữ liệu cũ (tr. 293–296). Scaling được đo bằng chi phí và độ đơn giản (tr. 287).
  - Tối ưu trước còn **mua thời gian** để lập chiến lược dài hạn (tr. 311).

[về mục 2.2](../chapter-11.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Xếp theo số câu hay theo thời gian

*Dự đoán: một workload có 99% số câu là SELECT nhưng 50 câu UPDATE mỗi câu sửa 2.000 hàng. Nếu xếp hạng theo số câu và theo tổng thời gian, kết quả khác nhau thế nào? Đối chiếu với kết quả đo.*

- **Dự đoán:** theo số câu, SELECT áp đảo (99%). Theo thời gian, mỗi UPDATE sửa 2.000 hàng phải tìm hàng, khóa, ghi undo, redo, và commit ghi bền xuống đĩa, nên mỗi câu đắt hơn SELECT theo primary key hàng chục, hàng trăm lần; tổng thời gian UPDATE có thể vượt SELECT.
- **Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định, bảng 200.000 hàng, trung vị 3 lần): workload A có 5.003 SELECT mất 871 ms, 50 UPDATE mất 1.451 ms, tức UPDATE chiếm khoảng 62% thời gian dù chỉ khoảng 1% số câu. Trung bình mỗi UPDATE khoảng 29 ms, mỗi SELECT khoảng 0,17 ms. Đối chứng workload B (UPDATE một hàng): UPDATE chỉ chiếm khoảng 10%.
- Ý của sách: workload là **hỗn hợp truy vấn và latency** của chúng, không phải số câu (tr. 289–290).

[về mục 2.2](../chapter-11.md#s2-2)

<a id="q2-2-5"></a>

### Câu 2.2.5 — Cả đọc và ghi cùng tăng

*Khi cả đọc lẫn ghi cùng tăng, sách khuyên làm gì trước, và vì sao đó là điều kiện tiên quyết cho read pool?*

- Sách khuyên soi schema để tìm **nhóm bảng** tăng nhanh về đọc và **nhóm khác** tăng về ghi, rồi **tách chúng vào các cluster chức năng khác nhau** để scale đọc và ghi độc lập. Scale một cluster cho cả hai cùng lúc là "tự chuốc đau đớn và sự cố" (tr. 291).
- Sách gọi đó là điều kiện tiên quyết để read pool hiệu quả hơn nhiều (tr. 291). **Bổ sung ngoài sách (diễn giải):** read pool chỉ tốt khi replica theo kịp source. Nhóm bảng ghi nặng khiến replica phải áp lượng ghi lớn và dễ lag; lag thì health check loại replica hoặc người dùng thấy dữ liệu cũ, kể cả với những bảng vốn chỉ cần đọc. Tách ra thì cluster nặng đọc có replica ít lag, còn cluster nặng ghi được xử lý bằng queue hay shard.

[về mục 2.2](../chapter-11.md#s2-2)

<a id="q2-3-1"></a>

### Câu 2.3.1 — Không chia theo cấu trúc đội

*Vì sao sách khuyên không chia database theo cấu trúc đội kỹ thuật? Điều gì xảy ra hai năm sau nếu bạn làm vậy?*

- Sách nói cấu trúc đội **luôn** thay đổi vào lúc nào đó (tr. 291). Ranh giới dữ liệu thì rất đắt để đổi: phải di chuyển dữ liệu, sửa ứng dụng, đổi API.
- **Ví dụ minh họa do AI xây dựng:** hai năm sau, đội "Growth" bị giải tán, nhiệm vụ chia cho đội "Payments" và đội "Accounts". Database "growth" giờ chứa bảng của hai chức năng, do hai đội cùng sửa, và các truy vấn phải `JOIN` chéo qua ba database. Ranh giới dữ liệu không còn khớp với chức năng kinh doanh, nên cả lợi ích scale độc lập lẫn kiểm soát truy cập đều mất.
- Chức năng kinh doanh (đăng ký, thiết lập khách hàng, thanh toán) ổn định hơn nhiều so với sơ đồ tổ chức (tr. 292).

[về mục 2.3](../chapter-11.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Mỗi bảng một database

*"Mỗi bảng một database" sai ở đâu? Nêu ít nhất hai chi phí cụ thể với MySQL.*

- Sách dùng nó làm ví dụ cường điệu: phân mảnh quá mức chắc chắn làm mọi thứ tệ hơn (tr. 291).
- Chi phí cụ thể (**Bổ sung ngoài sách**):
  - Mọi `JOIN` giữa các bảng thành gọi API hoặc gộp trong ứng dụng; optimizer không còn chọn được thứ tự join hay access path chung.
  - Mất transaction và foreign key giữa các bảng; ứng dụng phải tự giữ nhất quán.
  - Số cluster nhân lên: mỗi cluster cần replica, backup, monitoring, failover, nâng cấp. Chi phí và độ phức tạp vận hành tăng, trái với tiêu chí "chi phí và độ đơn giản" (tr. 287).
  - Mỗi request phải mở kết nối tới nhiều nơi, tăng latency và số kết nối.

[về mục 2.3](../chapter-11.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Tách danh tính và thanh toán

*Bảng danh tính khách hàng và bảng thanh toán đang được JOIN trong nhiều truy vấn. Muốn tách chúng sang hai cluster, ngoài việc di chuyển dữ liệu bạn còn phải làm gì?*

- Sách nói đây là ví dụ hay gặp về các mối quan tâm kinh doanh bị trộn lẫn, và phải vận động **không chỉ tách dữ liệu** mà cả **refactor ứng dụng** và **đưa vào truy cập qua API** giữa các ranh giới (tr. 292).
- Cụ thể (**Bổ sung ngoài sách**): liệt kê mọi truy vấn `JOIN` hai nhóm bảng (ví dụ từ digest của Performance Schema); viết lại thành hai bước hoặc gọi API của dịch vụ kia; quyết định dữ liệu nào được nhân bản sang phía kia (ví dụ tên khách hàng trên hóa đơn) và cách đồng bộ; bỏ foreign key xuyên hai nhóm; tạo database user riêng cho từng ứng dụng (tr. 309, Chương 13).
- Lúc đầu những bảng có chức năng và kiểu truy cập rõ ràng dễ tách; càng về sau càng tinh tế (tr. 292).

[về mục 2.3](../chapter-11.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — Lợi ích ngoài hiệu năng

*Ngoài hiệu năng, functional sharding giúp gì cho phục hồi sau thảm họa và compliance? Dẫn trang ở chương 2 và 13.*

- **Phục hồi:** Chương 2 nói nếu database lớn tới mức phục hồi từ backup lâu hơn mức doanh nghiệp chấp nhận cho chức năng quan trọng, có thể cần đưa dữ liệu của chức năng đó sang cluster riêng để mục tiêu phục hồi khả thi; tập dữ liệu phục hồi lâu nhất quyết định thời gian hoàn tất (tr. 35). Tách theo chức năng làm từng phần nhỏ hơn, phục hồi nhanh hơn.
- **Quản lý riêng:** định nghĩa functional sharding ở Chương 2 nêu rõ mục đích là quản lý riêng uptime, hiệu năng, thậm chí quyền truy cập của tập dữ liệu (tr. 34).
- **Compliance:** Chương 13 nêu "sharding for compliance reasons": khi một sản phẩm mới chịu yêu cầu khác hẳn (ví dụ dữ liệu sức khỏe chịu HIPAA), nên đặt nó trong kho dữ liệu riêng ngay từ đầu để áp kiểm soát phù hợp mà không đè gánh nặng lên dữ liệu cũ (tr. 332). Tr. 309 của chương 11 cũng nối việc tách database user với mục tiêu compliance.

[về mục 2.3](../chapter-11.md#s2-3)

<a id="q2-4-1"></a>

### Câu 2.4.1 — Replica ngoài pool

*Vì sao hình 11-1 để một replica ngoài read pool? Sách đưa ra cách nào khác để đạt cùng mục đích mà vẫn giữ node đó trong pool?*

- Để **các loại workload đọc không ảnh hưởng nhau**. Tiến trình báo cáo hay backup có thể ngốn hết I/O đĩa và gây replication lag; dành riêng một hay vài replica cho các việc đó và loại chúng khỏi pool phục vụ khách hàng (tr. 293).
- Cách khác: thêm **kiểm tra replication** vào health check của load balancer, để node backup bị tụt lại **tự động bị loại** khỏi pool, rồi **tự được đưa lại** khi đã theo kịp (tr. 293).
- Đánh đổi (**Bổ sung ngoài sách**): cách thứ nhất đơn giản và dễ đoán nhưng tốn một máy chỉ cho việc nền; cách thứ hai tận dụng máy hơn nhưng phụ thuộc health check phức tạp, đúng loại logic mà cảnh báo ở tr. 296 bảo phải cẩn thận.

[về mục 2.4](../chapter-11.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — Hai mặt của leastconn

*Sách khuyên leastconn thay vì roundrobin, nhưng cũng nói leastconn có thể gây hại. Giải thích cả hai mặt và nêu hai cách giảm rủi ro.*

- **Mặt lợi** (tr. 294): `leastconn` là cách được khuyên cho MySQL. Khi tải tăng cao, cân bằng kiểu `roundrobin` vẫn chia đều theo lượt nên không giúp dồn tải sang host chưa quá tải. **Bổ sung ngoài sách:** server đang chậm giữ kết nối lâu hơn nên có nhiều kết nối hơn, và `leastconn` tự động tránh nó.
- **Mặt hại** (tr. 297): `leastconn` có thể **dội tải vào server mới** vừa thêm vào pool khi cache của nó **chưa ấm**. Server mới có ít kết nối nhất nên nhận nhiều kết nối mới nhất, đúng lúc nó chậm nhất vì mọi truy vấn phải đọc đĩa.
- Cách giảm rủi ro (**Bổ sung ngoài sách**): (1) tăng trọng số dần, ví dụ `slowstart` của HAProxy; (2) làm ấm buffer pool trước khi đưa vào pool, với `innodb_buffer_pool_load_at_startup` và kiểm tra `Innodb_buffer_pool_load_status`, và đưa điều này vào định nghĩa "sẵn sàng" mà sách bảo phải cụ thể (tr. 293).
- Sách kết luận không có thuật toán tốt nhất chung: phải thử nghiệm, xét cả lúc bất thường (tr. 297).

[về mục 2.4](../chapter-11.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Cả cluster cùng lag

*Một migration lớn làm mọi replica lag 3 phút. Health check lag với ngưỡng 30 giây sẽ gây ra điều gì? Sách đề xuất chuẩn bị gì, và quyết định cuối cùng nên dựa vào đâu?*

- Mọi replica trượt check cùng lúc, load balancer loại **toàn bộ** pool, và **mọi truy vấn đọc** qua pool thất bại. Một sự cố "dữ liệu hơi cũ" biến thành sự cố "không đọc được".
- Sách cảnh báo đúng tình huống này: cẩn thận với health check có logic phức tạp như check lag, và phải có kế hoạch khi **mọi** replica trượt check; ví dụ một pool **fallback tĩnh** đưa mọi node trở lại khi có lỗi toàn cục như cả cluster lag (tr. 296).
- Quyết định dựa vào câu trả lời của đội ứng dụng cho các câu hỏi ở tr. 296 (dữ liệu cũ bao nhiêu thì chấp nhận được, hệ thống cư xử ra sao khi thiếu dữ liệu, chấp nhận trong bao lâu) và vào **SLI, SLO** đã thỏa thuận (tr. 295). Với catalog, dữ liệu cũ 3 phút thường tốt hơn lỗi; với số dư tài khoản thì có thể ngược lại.

[về mục 2.4](../chapter-11.md#s2-4)

<a id="q2-4-4"></a>

### Câu 2.4.4 — read_only và super_read_only

*Vì sao sách khuyên kiểm tra cả read_only lẫn super_read_only? Dự đoán chuyện gì xảy ra nếu sau một failover, source mới bị service discovery đưa nhầm vào read pool.*

- Mẹo ở tr. 295: kiểm tra hai biến để chắc mọi thành viên của read pool **thực sự là replica**.
- Vì sao cả hai (**Bổ sung ngoài sách + Kết quả đã đo**): `read_only=ON` chặn user thường (lỗi 1290) nhưng user có `CONNECTION_ADMIN` hay `SUPER`, ví dụ root, **vẫn ghi được**. Chỉ `super_read_only=ON` mới chặn cả họ. Thí nghiệm 2 xác nhận cả hai hành vi trên MySQL 8.0.42. Một replica được cấu hình đúng nên bật cả hai; một node chỉ bật `read_only` vẫn có thể bị ghi nhầm bởi công cụ chạy bằng tài khoản quản trị.
- **Dự đoán** khi source mới lọt vào read pool: source phải phục vụ thêm toàn bộ phần đọc chia cho nó; CPU tăng, latency của **ghi** tăng, đúng thứ read pool sinh ra để tránh. Nếu một phần lưu lượng "đọc" thật ra có ghi (ví dụ ứng dụng nhầm endpoint), phần đó sẽ thành công trên source nhưng lỗi trên replica, gây hành vi khó hiểu. Check `read_only`, `super_read_only` (cả hai đều tắt trên source) loại source ngay.
- Kết quả đo thêm: một check chỉ nhìn heartbeat trả "khỏe" cả trên source, vì heartbeat trên source luôn mới.

[về mục 2.4](../chapter-11.md#s2-4)

<a id="q2-4-5"></a>

### Câu 2.4.5 — Service discovery sập

*Service discovery bị sập 10 phút. Những câu hỏi nào ở tr. 295 quyết định mức thiệt hại? Vì sao sách nói đừng nhắm tới 100% uptime?*

- Câu hỏi then chốt (tr. 295): khi service discovery sập, nó **chặn mọi kết nối database mới** hay **chỉ chặn việc đổi thành viên** của load balancer? Có **sửa tay** được không? Ngoài ra: thay đổi thành viên diễn ra ở nền hay phải cắt kết nối đang có; cấu hình load balancer được làm mới thế nào khi instance hỏng.
- Nếu chỉ mất khả năng đổi thành viên, pool vẫn phục vụ với danh sách cũ; rủi ro chỉ là không loại được node hỏng trong 10 phút. Nếu mọi kết nối mới phụ thuộc service discovery, đó là sự cố toàn hệ thống.
- Sách nói linh hoạt luôn kéo theo phức tạp; quyết định phải **neo vào SLI và SLO**, không nhắm tới 100% uptime huyền thoại (tr. 295). **Bổ sung ngoài sách:** mỗi lớp thêm vào để tăng uptime (service discovery, health check phức tạp) lại là một thành phần có thể hỏng; đuổi theo 100% thường làm hệ thống phức tạp hơn và có khi kém tin cậy hơn.

[về mục 2.4](../chapter-11.md#s2-4)

<a id="q2-5-1"></a>

### Câu 2.5.1 — Nhiều writer, nhiều deadlock

*Vì sao thêm node ứng dụng cùng ghi vào một source lại làm tăng lock timeout và deadlock, dù CPU source chưa cạn?*

- Sách: kho dữ liệu ưu tiên **nhất quán** hơn sẵn sàng; càng nhiều node ứng dụng ghi vào một source, càng dễ gặp lock timeout, deadlock và ghi phải thử lại, dẫn tới lỗi và latency với khách hàng (tr. 298).
- Cơ chế (**Bổ sung ngoài sách**, liên hệ [Chương 1](../chapter-01.md)): InnoDB khóa hàng khi sửa. Nhiều transaction cùng đụng những hàng nóng phải **xếp hàng chờ lock**; thời gian chờ không tốn CPU nên CPU vẫn thấp. Càng nhiều writer đồng thời, xác suất hai transaction khóa hàng theo thứ tự ngược nhau càng cao, sinh deadlock. Ghi thất bại được thử lại lại cộng thêm tải đúng lúc hệ thống đang căng.
- Vì vậy hàng đợi giúp: số worker giới hạn số writer đồng thời trên source.

[về mục 2.5](../chapter-11.md#s2-5)

<a id="q2-5-2"></a>

### Câu 2.5.2 — 200 và 202

*Khác biệt giữa 200 và 202 về mặt cam kết với client là gì? Vì sao sách nói khác biệt đó bằng "toàn bộ công sức shard"?*

- 200 ngụ ý yêu cầu **đã được thực hiện xong ngay**. 202 Accepted chỉ nói yêu cầu **đã được nhận** và sẽ được xử lý sau (tr. 298).
- Đây là thay đổi **hợp đồng API**, nên phải thương lượng với đội sản phẩm để cam kết khả thi (tr. 298).
- Câu "khác biệt giữa 200 và 202 là toàn bộ công sức shard" (tr. 298) nghĩa là: muốn giữ 200 cho mọi lệnh xóa dù lượng xóa tăng mạnh, database phải chịu được toàn bộ lượng ghi đó **ngay lập tức**, tức phải shard dữ liệu để có nhiều source ghi song song. Chấp nhận 202 cho phép dàn đều lượng ghi theo thời gian trên hạ tầng hiện có và hoãn được dự án sharding.

[về mục 2.5](../chapter-11.md#s2-5)

<a id="q2-5-3"></a>

### Câu 2.5.3 — Chỉ số của hàng đợi

*Chỉ số nào cho biết chiến lược hàng đợi đã hết tác dụng? Vì sao không dùng độ dài hàng đợi đơn thuần?*

- Sách: phải quyết định **trước** khung thời gian yêu cầu được hoàn tất sau khi vào hàng đợi, và **theo dõi sự tăng của thời gian một yêu cầu nằm trong hàng đợi**; đó là chỉ số cho biết chiến lược đã hết tác dụng và cần shard (tr. 298–299).
- Vì sao thời gian chứ không phải độ dài (**Bổ sung ngoài sách**): cam kết với khách hàng là về **thời gian** ("xong trong 24 giờ"), nên chỉ số phải đo đúng thứ đã cam kết. Độ dài hàng đợi phụ thuộc cả tốc độ xử lý: 10.000 yêu cầu có thể là bình thường nếu worker xử lý 1.000 yêu cầu mỗi phút, nhưng là nghiêm trọng nếu mỗi phút chỉ xử lý 10. Thời gian chờ **tăng đều** qua nhiều tuần cho thấy tốc độ đến trung bình đã vượt tốc độ phục vụ, và hàng đợi chỉ đang hoãn vấn đề.

[về mục 2.5](../chapter-11.md#s2-5)

<a id="q2-5-4"></a>

### Câu 2.5.4 — Ghi không nên đưa vào hàng đợi

*Nêu một loại ghi không nên đưa vào hàng đợi và giải thích bằng tiêu chí của sách ở tr. 298.*

- Tiêu chí của sách: có phải mọi lần ghi **thật sự cần lưu vào database ngay**, hay một số có thể vào hàng đợi và ghi **trong một khung thời gian chấp nhận được** (tr. 298)?
- **Ví dụ minh họa do AI xây dựng:** trừ tồn kho khi khách thanh toán, hoặc ghi một khoản chuyển tiền. Bước tiếp theo của chính request đó (xác nhận đơn, hiển thị số dư) phụ thuộc kết quả ghi, và nếu hai request cùng mua món hàng cuối cùng thì phải có một bên bị từ chối **ngay**. Đưa vào hàng đợi nghĩa là trả lời khách trước khi biết ghi có thành công, có thể bán quá số hàng. Những lần ghi như vậy không có "khung thời gian chấp nhận được" nào ngoài "ngay bây giờ".
- Ngược lại, xóa dữ liệu lịch sử như ví dụ của sách (tr. 298) hay ghi log hoạt động thì có thể hoãn.

[về mục 2.5](../chapter-11.md#s2-5)

<a id="q2-6-1"></a>

### Câu 2.6.1 — Băm primary key

*Hệ thống blog shard bảng posts bằng cách băm primary key như NDB Cluster. Dự đoán chi phí của truy vấn "mọi bài của user 3" và truy vấn "bài có ID 12345". Đối chiếu với kết quả đo ở thí nghiệm 3.*

- **Dự đoán theo sách** (tr. 300): bài của user 3 bị rải đều khắp các shard vì được chia theo primary key, không theo user, nên phải hỏi **mọi** shard. Bài có ID 12345 thì biết ngay shard nào, chỉ hỏi **một** shard. Băm primary key giúp biết **lưu** ở đâu nhưng khó **lấy**, tùy bạn có biết primary key hay không.
- **Kết quả đã đo** (4 shard mô phỏng bằng 4 schema trên một instance): layout P (`post_id MOD 4`) cần 4 truy vấn cho bài của user 3, mỗi shard xét 340–365 hàng, tổng 1.397; bài 12345 cần 1 truy vấn, xét 1 hàng. Layout U (`user_id MOD 4`) thì ngược lại: 1 truy vấn cho bài của user 3, nhưng 4 truy vấn cho bài 12345 (1 trúng, 3 rỗng).
- Nhận xét: tổng số hàng xét không đổi; cái đổi là **số shard phải hỏi**. Với hệ thật, mỗi shard thêm một vòng mạng, và latency bằng shard chậm nhất (**Bổ sung ngoài sách**). Vì vậy partitioning key nên phục vụ truy vấn **quan trọng và thường xuyên nhất** (tr. 300).

[về mục 2.6](../chapter-11.md#s2-6)

<a id="q2-6-2"></a>

### Câu 2.6.2 — Bang hay quận

*Vì sao sách khuyên không shard theo bang mà theo quận hay mã vùng khi cần 20 shard? Nguyên tắc chung là gì, và nó liên quan gì tới việc "gộp số shard khác nhau"?*

- Shard theo bang thì shard của California quá lớn so với phần còn lại (tr. 301). Không có cách chia 50 bang thành 20 nhóm đều nhau khi một bang đã quá lớn.
- Quận hay mã vùng cũng không đều, nhưng có **rất nhiều**, nên chọn được 20 nhóm có tổng dân số gần bằng nhau, và chọn nhóm theo **tính gần gũi** để tránh truy vấn xuyên shard (tr. 301).
- Nguyên tắc chung (tr. 301): chọn khóa vừa tránh truy vấn xuyên shard vừa làm shard **đủ nhỏ**; lý tưởng là đều nhỏ, nếu không thì đủ nhỏ để **cân bằng bằng cách gộp số shard khác nhau** lại. Đơn vị nhỏ và nhiều cho phép gộp linh hoạt; đơn vị lớn và ít thì không.
- Liên hệ đo được: ở thí nghiệm 3, một user chiếm 8% bài làm shard của nó lớn hơn trung bình khoảng 21%. Một "California" trong dữ liệu luôn làm lệch shard nếu đơn vị shard quá thô.

[về mục 2.6](../chapter-11.md#s2-6)

<a id="q2-6-3"></a>

### Câu 2.6.3 — Bình luận ở hai kho

*Trong ví dụ câu lạc bộ sách, vì sao lưu bình luận đầy đủ ở kho user và chỉ lưu tiêu đề, ID ở kho sách? Cái giá về tính nhất quán là gì, và sách xử lý nó thế nào?*

- Ứng dụng nhìn dữ liệu theo **hai chiều**: mọi bình luận của một cuốn sách, và mọi sách một user đã bình luận (tr. 302). Bình luận vắt qua ranh giới hai kho.
- Thay vì nhân đôi toàn bộ, sách lưu bình luận đầy đủ cùng dữ liệu user và chỉ giữ **tiêu đề và ID** ở kho sách. Vậy đủ để hiển thị phần lớn màn hình bình luận của một cuốn sách mà chỉ chạm một kho; cần toàn văn thì lấy ở kho user (tr. 302). Lưu ít hơn và chỉ nhân bản phần cần cho truy vấn phổ biến.
- Cái giá (**Bổ sung ngoài sách**): hai bản ghi ở hai kho không được ghi trong cùng một transaction cục bộ, nên có lúc lệch: bình luận đã xóa ở kho user nhưng tiêu đề còn ở kho sách, hoặc ngược lại.
- Sách xử lý (tr. 303): không dùng foreign key xuyên shard; XA dùng được nhưng hiếm vì chi phí; thay vào đó dùng **job dọn dẹp định kỳ** (ví dụ xóa bình luận của tài khoản hết hạn khỏi các shard theo sách) và **script kiểm tra** định kỳ bảo đảm dữ liệu nhất quán giữa các shard.

[về mục 2.6](../chapter-11.md#s2-6)

<a id="q2-6-4"></a>

### Câu 2.6.4 — Gộp top 10 xuyên shard

*Cần "top 10 user hoạt động nhất". Giải thích vì sao lấy top 10 của mỗi shard rồi gộp lại cho kết quả đúng khi shard theo user_id nhưng có thể sai khi shard theo post_id. Nêu hai cách sách gợi ý để giảm chi phí loại truy vấn này.*

- Shard theo `user_id`: mọi bài của một user nằm trọn một shard, nên số đếm ở shard đó **đã là** số đếm cuối cùng của user. Top 10 toàn cục chắc chắn nằm trong hợp các top 10 của từng shard.
- Shard theo `post_id`: bài của một user rải trên mọi shard, mỗi shard chỉ có một phần số đếm. Một user có tổng cao có thể không lọt top 10 ở shard nào đó, phần đếm ở shard ấy bị bỏ mất khi gộp, và xếp hạng sai.
- **Kết quả đã đo** (thí nghiệm 3, 7 ngày, 2.317 bài): gộp ngây thơ đúng 10/10 hạng với layout U, **sai 6/10** với layout P. Muốn đúng với P phải lấy **mọi** nhóm từ mọi shard (1.580 hàng thay vì 40) rồi cộng; khi làm vậy cả 1.039 user khớp.
- Cách giảm chi phí theo sách (tr. 302–303): **cache mạnh tay** kết quả truy vấn xuyên shard; dùng **summary table** dựng bằng cách duyệt mọi shard, lưu dư thừa trên mỗi shard hoặc gom về một kho khác; dữ liệu không shard đặt ở **global node** có cache. Sách cũng khuyên đặt phần gộp xuyên shard trong logic ứng dụng.

[về mục 2.6](../chapter-11.md#s2-6)

<a id="q2-6-5"></a>

### Câu 2.6.5 — ProxySQL hay Vitess trước

*Một công ty có nhiều dịch vụ cũ không dám sửa mã, đang gặp sự cố mỗi khi latency tăng vì ứng dụng mở thêm kết nối. Nên chọn ProxySQL hay Vitess trước, và vì sao? Khi nào Vitess mới đáng giá?*

- **ProxySQL trước** hợp với tình huống này, theo lập luận của sách:
  - Nó **vô hình** với ứng dụng vì nói giao thức MySQL và mở kết nối thay ứng dụng; dùng được tính năng mà không sửa mã của các ứng dụng không còn được đầu tư (tr. 307).
  - **Connection pooling** tách kết nối phía client khỏi kết nối tới database, chỉ mở tới database số kết nối làm được việc và tái sử dụng chúng; đúng cách chặn vòng xoáy "mở thêm kết nối" dẫn tới thundering herd và lỗi dây chuyền (tr. 307, 310).
  - Nó dễ triển khai, ít đầu tư hạ tầng hơn (tr. 311). Nhớ chạy nhiều instance theo cluster, không để SPoF (tr. 308).
- **Kết quả đã đo** minh họa vấn đề gốc: vượt `max_connections` thì 50/200 client bị từ chối và bộ đếm kết nối hỏng tăng lên 500 vì client thử lại.
- **Vitess** đáng giá khi cần những thứ ProxySQL không có: **tự động shard** tập dữ liệu, **quản lý thay đổi schema**, **VReplication** cho ETL và luồng thay đổi (tr. 311), cùng failover ghi tự động và durability bằng semisync (tr. 305). Cái giá là độ phức tạp, latency thêm vào, và kế hoạch chuyển đổi ba bước vì nó không drop-in (tr. 306). Mọi nhận định này theo trạng thái năm 2021.

[về mục 2.6](../chapter-11.md#s2-6)

<a id="q2-7-1"></a>

### Câu 2.7.1 — Thứ tự các bước

*Sắp xếp theo đúng thứ tự sách khuyên: sharding, tối ưu truy vấn và index, xác định read-bound hay write-bound, read pool, hàng đợi. Giải thích vì sao bước đầu tiên được đặt ở đầu.*

- Thứ tự: **tối ưu truy vấn và index (cùng cấu hình)** → **xác định read-bound hay write-bound** → nếu đọc: **read pool**; nếu ghi: **hàng đợi** cho phần hoãn được → **sharding** khi lag không chấp nhận được, hoặc ghi vẫn vượt sức (tr. 298–299, 311).
- Vì sao tối ưu đầu tiên (tr. 290, 311): một truy vấn kém tốn tài nguyên trên mọi máy, nên scale out chỉ nhân chi phí lên. Tối ưu thường rẻ và đơn giản nhất, đúng tiêu chí "chi phí và độ đơn giản" (tr. 287), và **mua thời gian** để lên chiến lược dài hạn (tr. 311). Nó cũng làm bước phân loại workload chính xác hơn: chưa tối ưu thì một truy vấn thiếu index có thể làm workload trông read-bound trong khi thực chất chỉ là một lỗi.

[về mục 2.7](../chapter-11.md#s2-7)

<a id="q2-7-2"></a>

### Câu 2.7.2 — Read-bound mà vẫn shard

*Workload rõ ràng là read-bound, nhưng sách vẫn có thể khuyên sharding. Khi nào, và vì sao read pool không giải quyết được?*

- Sách: với read-bound, chuyển sang read pool **trừ khi replication lag là vấn đề không thể vượt qua**; nếu lag là vấn đề, cân nhắc sharding (tr. 311).
- Vì sao (**Bổ sung ngoài sách**, diễn giải): read pool chỉ phục vụ được những truy vấn chịu được dữ liệu cũ bằng mức lag thực tế. Nếu ứng dụng cần đọc dữ liệu rất mới, hoặc lượng ghi đủ lớn khiến replica thường xuyên lag vượt ngưỡng, các replica liên tục bị health check loại và đọc dồn về source. Khi đó cách giảm tải đọc trên mỗi source là chia dữ liệu: mỗi shard có source riêng, nhỏ hơn, phục vụ cả đọc mới lẫn ghi cho phần dữ liệu của nó. [Chương 9](../chapter-09.md#active-passive) cũng nói khi chạm giới hạn đọc, có thể phải dùng sharding để giảm đọc trên source (tr. 242).

[về mục 2.7](../chapter-11.md#s2-7)

<a id="q2-7-3"></a>

### Câu 2.7.3 — Mua thời gian

*"Mua thời gian" ở tr. 311 nghĩa là gì trong vận hành? Cho ví dụ một quyết định dài hạn cần thời gian đó.*

- Sách nói tối ưu truy vấn, index và cấu hình **có thể mua đủ thời gian** để lên chiến lược dài hạn tốt hơn (tr. 311). Trong vận hành, đó là khoảng thời gian trước khi lại chạm giới hạn SLO, tính từ xu hướng tăng trưởng (Chương 2 nói đo dài hạn để biết khi nào cần đầu tư scale, tr. 39).
- **Ví dụ minh họa do AI xây dựng:** tối ưu ba truy vấn đắt nhất làm CPU source giảm từ 80% xuống 45%. Với tốc độ tăng 5% mỗi tháng, đó là vài tháng trước khi lại căng. Thời gian này dùng để: chọn partitioning key dựa trên phân tích truy vấn thật (tr. 300); tách nhóm bảng thanh toán sang cluster riêng (tr. 291–292); hoặc chạy giai đoạn canary để chuyển kết nối sang Vitess (tr. 306). Những việc này cần nhiều tuần hay nhiều tháng và rất rủi ro nếu làm vội giữa sự cố. Chương 2 cũng cảnh báo: nếu chỉ nghĩ tới sharding khi sự cố capacity đã xảy ra thì có lẽ đã quá muộn (tr. 35).

[về mục 2.7](../chapter-11.md#s2-7)
