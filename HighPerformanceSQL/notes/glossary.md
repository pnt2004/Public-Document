# Thuật ngữ — High Performance MySQL

Thuật ngữ Anh–Việt của *High Performance MySQL* (Fourth Edition, 2021), xếp theo chương. Mỗi mục có định nghĩa ngắn và liên kết tới phần giải thích sâu trong ghi chú chương. "tr. N" là số trang in; trang PDF = N + 22. Mục ghi **Bổ sung ngoài sách** là thuật ngữ thêm để giải thích, không có trong chương tương ứng.

Một thuật ngữ có thể xuất hiện ở nhiều chương với góc nhìn khác nhau (ví dụ buffer pool ở chương 1, 4, 5); hãy dùng tìm kiếm trong trình xem (Ctrl+F) để thấy mọi nơi nó được bàn.

## Mục lục

- [Chương 1 — MySQL Architecture](#chuong-1)
- [Chương 2 — Monitoring in a Reliability Engineering World](#chuong-2)
- [Chương 3 — Performance Schema](#chuong-3)
- [Chương 4 — Operating System and Hardware Optimization](#chuong-4)
- [Chương 5 — Optimizing Server Settings](#chuong-5)
- [Chương 6 — Schema Design and Management](#chuong-6)
- [Chương 7 — Indexing for High Performance](#chuong-7)
- [Chương 8 — Query Performance Optimization](#chuong-8)
- [Chương 9 — Replication](#chuong-9)
- [Chương 10 — Backup and Recovery](#chuong-10)
- [Chương 11 — Scaling MySQL](#chuong-11)
- [Chương 12 — MySQL in the Cloud](#chuong-12)
- [Chương 13 — Compliance with MySQL](#chuong-13)
- [Phụ lục A — Upgrading MySQL](#phu-luc-a)
- [Phụ lục B — MySQL on Kubernetes](#phu-luc-b)

<a id="chuong-1"></a>

## Chương 1 — MySQL Architecture

### Kiến trúc server

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| logical architecture | kiến trúc logic (ba tầng) | Cách sách chia MySQL: tầng kết nối và bảo mật, tầng server (parse, tối ưu, hàm, view), tầng storage engine (tr. 1–2) | [§2.1](chapter-01.md#logical-architecture) |
| storage engine | storage engine, bộ máy lưu trữ | Thành phần lưu và lấy dữ liệu; mỗi engine có cách khóa, transaction, tổ chức file riêng (tr. 1, 5) | [§2.1](chapter-01.md#logical-architecture) |
| storage engine API | giao diện storage engine | Vài chục hàm mức thấp (bắt đầu transaction, lấy hàng theo primary key…) mà server gọi; che khác biệt giữa các engine (tr. 2, 18) | [§2.1](chapter-01.md#storage-engine-api) |
| thread per connection | một thread cho mỗi kết nối | Mặc định mỗi connection có một thread; truy vấn của nó chạy trên một core tại một thời điểm (tr. 2) | [§2.1.1](chapter-01.md#connection-management) |
| thread cache | cache thread | Các thread sẵn sàng được giữ lại để không phải tạo, hủy cho mỗi connection (tr. 2) | [§2.1.1](chapter-01.md#connection-management) |
| thread pooling | gom nhóm thread | Một nhóm thread phục vụ nhiều connection; MySQL có API plug-in từ 5.5, ít dùng (tr. 2) | [§2.1.1](chapter-01.md#thread-pooling-check) |
| authentication | xác thực | Kiểm tra danh tính dựa trên username, host nguồn, password; có thể dùng chứng chỉ X.509 qua TLS (tr. 2) | [§2.1.1](chapter-01.md#connection-management) |
| parse tree | cây phân tích cú pháp | Cấu trúc nội bộ server dựng từ câu SQL trước khi tối ưu (tr. 3) | [§2.1.2](chapter-01.md#optimizer-execution) |
| optimizer | bộ tối ưu truy vấn | Thành phần ở tầng server viết lại truy vấn, chọn thứ tự bảng và index; hỏi engine về khả năng, chi phí, thống kê (tr. 3) | [§2.1.2](chapter-01.md#optimizer-execution) |
| optimizer hint | gợi ý cho optimizer | Từ khóa đặc biệt trong câu truy vấn để tác động quyết định của optimizer (tr. 3) | [§2.1.2](chapter-01.md#optimizer-execution) |
| query cache | bộ đệm kết quả truy vấn | Cache kết quả trong server của bản cũ; nghẽn khi concurrency cao, deprecated 5.7.20, bị gỡ trong 8.0 (tr. 3) | [§2.1.2](chapter-01.md#query-cache) |

### Khóa và concurrency

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| concurrency control | kiểm soát truy cập đồng thời | Cơ chế giữ dữ liệu đúng khi nhiều truy vấn cùng đổi dữ liệu; ở MySQL có hai mức: server và engine (tr. 3) | [§2.2](chapter-01.md#concurrency-control) |
| shared lock (read lock) | khóa chia sẻ (khóa đọc) | Nhiều người cùng giữ được, không chặn nhau, nhưng bị khóa ghi chặn (tr. 4) | [§2.2.1](chapter-01.md#read-write-locks) |
| exclusive lock (write lock) | khóa độc quyền (khóa ghi) | Chặn cả khóa đọc lẫn khóa ghi khác; một người ghi tại một thời điểm (tr. 4) | [§2.2.1](chapter-01.md#read-write-locks) |
| lock granularity | độ mịn của khóa | Kích thước phần dữ liệu bị khóa; khóa nhỏ tăng concurrency nhưng tăng overhead (tr. 4–5) | [§2.2.2](chapter-01.md#lock-granularity) |
| locking strategy | chiến lược khóa | Thỏa hiệp giữa overhead của khóa và an toàn dữ liệu; ở MySQL mỗi engine tự chọn (tr. 5) | [§2.2.2](chapter-01.md#lock-granularity) |
| table lock | khóa bảng | Khóa cả bảng; overhead thấp nhất; hàng đợi ghi ưu tiên hơn hàng đợi đọc (tr. 5) | [§2.2.2](chapter-01.md#table-locks) |
| READ LOCAL | khóa bảng READ LOCAL | Biến thể table lock cho phép một số kiểu ghi đồng thời (tr. 5) | [§2.2.2](chapter-01.md#table-locks) |
| row lock | khóa hàng | Concurrency cao nhất, overhead lớn nhất; cài trong engine, không trong server (tr. 5–6) | [§2.2.2](chapter-01.md#row-locks) |
| intention lock | khóa ý định | Khóa mức bảng (IS, IX) báo hiệu sắp khóa hàng; sách chỉ khuyên đọc tài liệu (chú thích 3, tr. 5) | [§2.2.2](chapter-01.md#row-locks) |
| metadata lock | khóa metadata | Khóa ở tầng server khi đổi tên bảng hay đổi schema (chú thích 4, tr. 6) | [§2.2.2](chapter-01.md#row-locks) |
| two-phase locking | khóa hai pha | Giao thức của InnoDB: xin khóa bất cứ lúc nào trong transaction, nhả tất cả cùng lúc khi `COMMIT` hay `ROLLBACK` (tr. 12) | [§2.3.4](chapter-01.md#two-phase-locking) |
| implicit locking | khóa ngầm | Khóa InnoDB tự đặt theo isolation level (tr. 12) | [§2.3.4](chapter-01.md#two-phase-locking) |
| explicit locking | khóa tường minh | `SELECT ... FOR SHARE`, `SELECT ... FOR UPDATE`; hay bị lạm dụng (tr. 12) | [§2.3.4](chapter-01.md#two-phase-locking) |
| `LOCK TABLES` | lệnh khóa bảng | Khóa bảng cài ở server; không cần với InnoDB; chỉ dùng khi đang trong transaction và autocommit tắt (tr. 12–13) | [§2.3.4](chapter-01.md#lock-tables) |
| deadlock | khóa chết | Hai hay nhiều transaction cùng giữ và cùng xin khóa trên cùng tài nguyên, tạo vòng phụ thuộc (tr. 9) | [§2.3.2](chapter-01.md#deadlocks) |
| deadlock detection | phát hiện deadlock | InnoDB nhận ra vòng phụ thuộc và trả lỗi ngay, rollback transaction dễ hoàn tác nhất (tr. 10) | [§2.3.2](chapter-01.md#deadlock-victim) |
| lock wait timeout | thời gian chờ khóa tối đa | Cách phá deadlock thô: bỏ cuộc khi chờ quá lâu; không phải lúc nào cũng tốt (tr. 10) | [§2.3.2](chapter-01.md#deadlock-victim) |
| next-key locking | khóa next-key | InnoDB khóa cả bản ghi và khoảng trống trong index để chặn phantom ở REPEATABLE READ (tr. 16) | [§2.7](chapter-01.md#next-key-locking) |

### Transaction và isolation

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| transaction | giao dịch | Nhóm câu SQL xử lý như một đơn vị nguyên tử: tất cả hoặc không gì cả (tr. 6) | [§2.3](chapter-01.md#transactions) |
| ACID | ACID | Atomicity, consistency, isolation, durability: bốn tiêu chí của hệ xử lý transaction an toàn (tr. 7) | [§2.3](chapter-01.md#acid) |
| atomicity | tính nguyên tử | Transaction áp dụng hết hoặc không gì; không có transaction làm dở (tr. 7) | [§2.3](chapter-01.md#acid) |
| consistency | tính nhất quán | Database luôn đi từ trạng thái nhất quán sang trạng thái nhất quán (tr. 7) | [§2.3](chapter-01.md#acid) |
| isolation | tính cô lập | Kết quả của transaction thường vô hình với transaction khác cho tới khi xong (tr. 7) | [§2.3](chapter-01.md#acid) |
| durability | tính bền vững | Đã commit thì không mất khi crash; khái niệm "mờ", có nhiều mức, không gì bền 100% (tr. 7) | [§2.3](chapter-01.md#acid) |
| two-phase commit | commit hai pha | Hệ phối hợp commit phức tạp, chậm, để chống nhiều kiểu hỏng; khác two-phase locking (tr. 7) | [§2.3](chapter-01.md#acid) |
| isolation level | mức cô lập | Quy tắc thay đổi nào được thấy trong và ngoài transaction; ANSI có bốn mức (tr. 8) | [§2.3.1](chapter-01.md#isolation-levels) |
| READ UNCOMMITTED | đọc chưa commit | Thấy kết quả chưa commit; hiếm dùng (tr. 8) | [§2.3.1](chapter-01.md#isolation-levels) |
| READ COMMITTED | đọc đã commit | Mặc định của phần lớn database (không phải MySQL); có nonrepeatable read (tr. 8) | [§2.3.1](chapter-01.md#isolation-levels) |
| REPEATABLE READ | đọc lặp lại được | Hàng đã đọc trông như cũ trong cùng transaction; mặc định của MySQL (tr. 8–9) | [§2.3.1](chapter-01.md#isolation-levels) |
| SERIALIZABLE | tuần tự hóa | Mức cao nhất; khóa mọi hàng nó đọc; nhiều tranh chấp (tr. 9) | [§2.3.1](chapter-01.md#isolation-levels) |
| dirty read | đọc bẩn | Đọc dữ liệu chưa commit (tr. 8) | [§2.3.1](chapter-01.md#isolation-levels) |
| nonrepeatable read | đọc không lặp lại được | Chạy cùng câu hai lần trong một transaction thấy dữ liệu khác (tr. 8) | [§2.3.1](chapter-01.md#isolation-levels) |
| phantom read | đọc bóng ma | Chọn lại một khoảng thấy hàng mới do transaction khác chèn (tr. 8) | [§2.3.1](chapter-01.md#phantom-read) |
| locking read | đọc có khóa | Cột "Locking reads" của Bảng 1-1; ở InnoDB là `FOR SHARE`, `FOR UPDATE` (tr. 9, 12) | [§2.3.1](chapter-01.md#phantom-read) |
| transaction logging | ghi log transaction | Đổi dữ liệu trong bộ nhớ, ghi bản ghi thay đổi vào log trên đĩa, cập nhật bảng sau (tr. 10) | [§2.3.3](chapter-01.md#transaction-logging) |
| write-ahead logging (WAL) | ghi log trước | Kỹ thuật trên: log tuần tự trước, dữ liệu sau; ghi xuống đĩa hai lần (tr. 10) | [§2.3.3](chapter-01.md#wal) |
| AUTOCOMMIT | tự động commit | Chế độ mặc định: mỗi câu đơn lẻ là một transaction tự commit (tr. 11) | [§2.3.4](chapter-01.md#autocommit) |
| implicit commit | commit ngầm | DDL như `ALTER TABLE`, và `LOCK TABLES`, commit transaction đang mở trước khi chạy (tr. 11) | [§2.3.4](chapter-01.md#autocommit) |
| nontransactional table | bảng không hỗ trợ transaction | Bảng (ví dụ MyISAM) mà thay đổi không rollback được; trộn với InnoDB gây mất nhất quán (tr. 12) | [§2.3.4](chapter-01.md#mixing-engines) |

### MVCC

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| multiversion concurrency control (MVCC) | kiểm soát đồng thời đa phiên bản | Dùng snapshot dữ liệu để đọc không khóa; kết hợp với row lock; không có chuẩn chung (tr. 13, 15) | [§2.4](chapter-01.md#mvcc) |
| snapshot | ảnh chụp dữ liệu | Dữ liệu như tại một thời điểm; transaction thấy góc nhìn nhất quán dù chạy lâu (tr. 13) | [§2.4](chapter-01.md#mvcc) |
| optimistic / pessimistic concurrency control | kiểm soát đồng thời lạc quan / bi quan | Hai biến thể cách cài MVCC mà sách nhắc tên (tr. 13) | [§2.4](chapter-01.md#mvcc) |
| transaction ID | mã transaction | ID InnoDB gán cho transaction; sách nói gán ở lần đọc đầu (tr. 13) | [§2.4](chapter-01.md#read-view) |
| undo log, undo record | nhật ký hoàn tác, bản ghi hoàn tác | Mô tả cách hoàn tác một thay đổi; dùng để rollback và dựng bản cũ cho người đọc (tr. 13–14) | [§2.4](chapter-01.md#undo-log) |
| rollback pointer | con trỏ rollback | Trỏ từ bản ghi tới undo record của thay đổi gần nhất (tr. 13) | [§2.4](chapter-01.md#undo-log) |
| read view | khung nhìn đọc | Thông tin quyết định phiên bản nào session được thấy; so với transaction ID của bản ghi (tr. 14) | [§2.4](chapter-01.md#read-view) |
| delete mark | dấu xóa | Xóa bằng cách bật bit "deleted" trong info flags; undo ghi là "remove delete mark" (tr. 14) | [§2.4](chapter-01.md#undo-log) |

### Replication, file dữ liệu và InnoDB

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| replication | nhân bản | Phân phối các lần ghi của node nhận ghi tới node khác (tr. 15) | [§2.5](chapter-01.md#replication) |
| source, replica | nguồn, bản sao | Node nhận ghi và node nhận bản sao; source có một thread cho mỗi replica (tr. 15) | [§2.5](chapter-01.md#replication) |
| topology tree | cây topology | Sơ đồ source và các replica (Hình 1-3, tr. 15) | [§2.5](chapter-01.md#replication) |
| binary log | nhật ký nhị phân | Log mà source gửi tới replica trong Hình 1-3 (tr. 15) | [§2.5](chapter-01.md#replication) |
| data dictionary | từ điển dữ liệu | Metadata của bảng trong 8.0, lưu trong bảng InnoDB, hỗ trợ transaction và DDL nguyên tử (tr. 16–17) | [§2.6](chapter-01.md#data-dictionary) |
| dictionary object cache | cache đối tượng từ điển | Cache LRU trong bộ nhớ cho định nghĩa partition, bảng, stored program, charset, collation (tr. 16) | [§2.6](chapter-01.md#data-dictionary) |
| serialized dictionary information (SDI) | thông tin từ điển tuần tự hóa | Bản định nghĩa bảng dạng tuần tự; sách nói thay cho `.frm` (tr. 16) | [§2.6](chapter-01.md#sdi) |
| tablespace | không gian bảng | Loạt datafile InnoDB tự quản lý như một hộp đen (tr. 16) | [§2.7](chapter-01.md#innodb) |
| clustered index | index cụm | Bảng InnoDB được xây trên nó; tra theo primary key rất nhanh (tr. 16) | [§2.7](chapter-01.md#clustered-index) |
| secondary index | index phụ | Index không phải primary key; chứa các cột primary key, nên primary key lớn làm nó lớn (tr. 16–17) | [§2.7](chapter-01.md#clustered-index) |
| predictive read-ahead | đọc trước dự đoán | Tối ưu InnoDB đọc trước dữ liệu từ đĩa (tr. 17) | [§2.7](chapter-01.md#internal-optimizations-check) |
| adaptive hash index | hash index thích nghi | InnoDB tự dựng hash index trong bộ nhớ cho tra cứu rất nhanh (tr. 17) | [§2.7](chapter-01.md#internal-optimizations-check) |
| insert buffer | bộ đệm insert | Tối ưu InnoDB để tăng tốc insert (tr. 17) | [§2.7](chapter-01.md#internal-optimizations-check) |
| hot backup | backup nóng | Backup khi server đang chạy; InnoDB hỗ trợ qua MySQL Enterprise Backup, Percona XtraBackup (tr. 17) | [§2.7](chapter-01.md#innodb) |
| online DDL | DDL trực tuyến | Từ 5.6: một số thay đổi schema tại chỗ không cần khóa cả bảng (tr. 17) | [§2.7](chapter-01.md#online-ddl) |
| JSON data type | kiểu dữ liệu JSON | Từ 5.7: tự kiểm tra hợp lệ, lưu tối ưu cho đọc, có hàm SQL (tr. 17) | [§2.7.1](chapter-01.md#json) |
| multivalued index | index đa giá trị | Index trên mảng JSON, mỗi phần tử một mục (tr. 17) | [§2.7.1](chapter-01.md#json) |
| atomic DDL | DDL nguyên tử | Từ 8.0: câu DDL hoàn tất trọn vẹn hoặc rollback trọn vẹn, nhờ undo, redo log riêng cho DDL (tr. 18) | [§2.7.3](chapter-01.md#atomic-ddl) |

### Bổ sung ngoài sách (liên quan chương 1)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| connection pooling | gom kết nối | Proxy hay thư viện giữ ít connection thật tới database và chia cho nhiều client (chương 11 bàn ProxySQL, Vitess) | [§2.1.1](chapter-01.md#thread-pooling-check) |
| consistent read | đọc nhất quán (không khóa) | `SELECT` thường ở RC, RR: đọc từ read view qua undo, không khóa | [§2.3.1](chapter-01.md#phantom-read) |
| gap lock | khóa khoảng trống | Khóa khoảng giữa hai bản ghi index (`X,GAP` trong `data_locks`), chặn chèn vào khoảng | [§2.7](chapter-01.md#next-key-locking) |
| insert intention lock | khóa ý định chèn | Khóa mà câu `INSERT` xin trên khoảng trống; phải chờ nếu khoảng đang bị gap lock | [§2.7](chapter-01.md#next-key-locking) |
| lost update | mất cập nhật | Hai transaction đọc cùng giá trị cũ rồi cùng ghi đè; một thay đổi bị mất | [§2.3](chapter-01.md#acid) |
| redo log | nhật ký redo | Transaction log của InnoDB, dùng cho crash recovery | [§2.3.3](chapter-01.md#wal) |
| `DB_TRX_ID`, `DB_ROLL_PTR` | cột ẩn transaction ID, rollback pointer | Hai cột ẩn InnoDB thêm vào mỗi hàng để phục vụ MVCC (thấy được qua `ibd2sdi`) | [§2.4](chapter-01.md#read-view) |
| purge, history list length | dọn undo, độ dài danh sách lịch sử | Purge xóa undo không còn read view nào cần; history list length đo lượng undo chờ dọn | [§2.4](chapter-01.md#purge-history) |
| change buffer | bộ đệm thay đổi | Tên mới (từ 5.5) của insert buffer, đệm cả delete-mark và purge cho secondary index | [§2.7](chapter-01.md#internal-optimizations-check) |
| `performance_schema.data_locks` | bảng khóa dữ liệu | Bảng 8.0 liệt kê khóa InnoDB đang giữ hay chờ; đi cùng `data_lock_waits`, `sys.innodb_lock_waits` | [§4](chapter-01.md#measure) |

<a id="chuong-2"></a>

## Chương 2 — Monitoring in a Reliability Engineering World

### Reliability engineering và mục tiêu dịch vụ

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Site Reliability Engineering (SRE) | Kỹ thuật độ tin cậy hệ thống | Bộ nguyên lý vận hành giúp trả lời: trải nghiệm khách hàng có chấp nhận được không, có nên tập trung vào độ tin cậy không, cân tính năng mới với toil thế nào (tr. 19) | [§2.0](chapter-02.md#s2-0) |
| Database reliability engineer (DBRE) | Kỹ sư độ tin cậy database | Vai trò DBA mới: dùng service level để chia thời gian giữa hiệu năng, scaling và công cụ nội bộ, thay cho việc "gác cổng" database (tr. 20) | [§2.1](chapter-02.md#dbre) |
| Toil | Việc vặt vận hành | Việc vận hành thủ công, lặp lại, tự động hóa được, không tạo giá trị lâu dài; sách nhắc (tr. 19, 29) nhưng không định nghĩa, định nghĩa theo tài liệu SRE | [§2.0](chapter-02.md#s2-0) |
| Outcomes, not outputs | Đo kết quả, không đo sản lượng | Thước đo nên là kết quả khách hàng nhận được (trang nhanh, checkout thành công), không phải chỉ số nội bộ (QPS, CPU) (tr. 20, 25) | [§2.0](chapter-02.md#outcomes-vs-outputs) |
| Service level indicator (SLI) | Chỉ báo mức dịch vụ | Cách đo xem khách hàng có hài lòng không, từ góc nhìn người dùng (tr. 21) | [§2.2](chapter-02.md#sli-slo-sla) |
| Service level objective (SLO) | Mục tiêu mức dịch vụ | Mức tối thiểu của một SLI để khách hàng còn hài lòng, luôn định nghĩa trên một khoảng thời gian (tr. 21) | [§2.2](chapter-02.md#sli-slo-sla) |
| Service level agreement (SLA) | Thỏa thuận mức dịch vụ | SLO đưa vào hợp đồng với khách hàng trả tiền, có chế tài; là tùy chọn (tr. 21) | [§2.2](chapter-02.md#sli-slo-sla) |
| Degraded state / failed state | Trạng thái suy giảm / trạng thái hỏng | Hai mức cần thống nhất trước khi đo: lúc nào coi là suy giảm, lúc nào hỏng hẳn và phải khắc phục ngay (tr. 20) | [§2.2](chapter-02.md#service-level-goals) |
| Nines of availability | Số "9" của availability | Cách nói mức availability (99,9% là "ba số 9"); mỗi số 9 thêm vào cắt ngân sách downtime khoảng 10 lần (tr. 22, Table 2-1) | [§2.2.1](chapter-02.md#nines) |

### Đo gì và cảnh báo

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Availability | Tính sẵn sàng | Trả lời được request của khách hàng mà không lỗi: thành công (200) hoặc nhận request kèm hứa xử lý bất đồng bộ (202) (tr. 26) | [§2.3.3](chapter-02.md#availability) |
| Synthetic test / synthetic transaction | Kiểm thử tổng hợp / giao dịch tổng hợp | Thao tác giả lập chạy định kỳ từ nơi khác trong mạng (tải trang, `SELECT 1`, ghi rồi đọc) để kiểm dịch vụ còn dùng được (tr. 24, 26) | [§2.3.3](chapter-02.md#availability-probes) |
| Degraded functionality | Chức năng suy giảm | Chế độ chạy giảm chất lượng khi có sự cố, ví dụ gợi ý chung thay gợi ý cá nhân hóa (tr. 26) | [§2.3.3](chapter-02.md#availability) |
| Mean time to recovery (MTTR) | Thời gian khôi phục trung bình | Thời gian từ khi hỏng tới khi tính năng lõi hoạt động lại; dùng làm mục tiêu cho failover và restore (tr. 26, 35) | [§2.3.3](chapter-02.md#availability) |
| Testing in production | Kiểm thử trong production | Quan sát thay đổi với lưu lượng thật, có vòng phản hồi nhanh gắn với từng thay đổi để rollback sớm (tr. 25) | [§2.3.2](chapter-02.md#testing-in-production) |
| Leading indicator | Chỉ báo sớm | Tín hiệu xấu đi trước khi khách hàng gặp lỗi, ví dụ `Threads_running` (tr. 27, 29) | [§2.3.3](chapter-02.md#threads-running) |
| Steady state monitoring | Giám sát trạng thái ổn định | Dashboard và script cảnh báo dựa trên baseline, báo điều bất thường dù có thay đổi hay không (tr. 29) | [§2.3.6](chapter-02.md#steady-state) |
| Proactive monitoring | Giám sát chủ động | Theo dõi tín hiệu chưa gắn trực tiếp với khách hàng (đĩa, kết nối, lag, I/O, auto-increment, restore) để ngăn sự cố (tr. 29–35) | [§2.3.6](chapter-02.md#proactive-monitoring) |
| Actionable alert | Cảnh báo hành động được | Cảnh báo mà người nhận có việc cụ thể để làm; cảnh báo gửi người, nhất là ngoài giờ, phải luôn như vậy (tr. 29, 32) | [§2.3.6](chapter-02.md#actionable-alerts) |
| Client-side latency | Latency phía client | Thời gian tới khi query hoàn tất do ứng dụng đo, gần trải nghiệm khách hàng hơn latency đo trong server (tr. 27) | [§2.3.4](chapter-02.md#client-side-latency) |

### Chỉ số và lỗi của MySQL

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `Threads_running` | Số thread đang chạy | Biến trạng thái: số query đang chạy (thread không ngủ); vượt số core hoặc khoảng 100 là dấu hiệu quá tải (tr. 27, 31) | [§2.3.3](chapter-02.md#threads-running) |
| `Threads_connected` | Số kết nối đang mở | Biến trạng thái: số kết nối hiện có; cao mà `Threads_running` thấp nghĩa là ứng dụng giữ nhiều kết nối không dùng (tr. 31) | [§2.3.6](chapter-02.md#connection-growth) |
| `max_connections` | Số kết nối tối đa | Biến hệ thống giới hạn số kết nối; chạm trần thì kết nối mới bị từ chối; theo dõi tỉ lệ `Threads_connected` / `max_connections` (tr. 27, 30–31) | [§2.3.6](chapter-02.md#connection-growth) |
| `Connection_errors_xxx` | Bộ đếm lỗi kết nối | Nhóm biến trạng thái đếm lỗi kết nối theo loại; tăng đột ngột là dấu hiệu có thứ mới đang hỏng (tr. 28) | [§2.3.5](chapter-02.md#errors) |
| Lock wait timeout | Hết thời gian chờ khóa | Lỗi khi chờ row lock quá lâu; tốc độ tăng vọt báo tranh chấp lock leo thang, có thể dẫn tới mất ghi (tr. 28) | [§2.3.5](chapter-02.md#errors-rate-vs-single) |
| Aborted connections | Kết nối bị hủy | Kết nối bị cắt bất thường; tăng đột ngột gợi ý vấn đề ở access layer, kéo theo nhiều retry (tr. 28) | [§2.3.5](chapter-02.md#errors-rate-vs-single) |
| Read-only error | Lỗi chế độ chỉ đọc | Lỗi khi ghi vào server read-only; một lần là đủ để xử lý, vì thường là source mới chưa tắt read-only hoặc ghi bị gửi nhầm sang replica (tr. 28) | [§2.3.5](chapter-02.md#errors-rate-vs-single) |
| Too many connections | Quá nhiều kết nối | Lỗi khi chạm `max_connections` (hoặc không tạo được thread mới); thành lỗi 5xx ngay cho ứng dụng (tr. 28) | [§2.3.5](chapter-02.md#errors-rate-vs-single) |
| Connection storm | Bão kết nối | Ứng dụng thấy latency tăng nên mở thêm kết nối, làm database thêm tải và nhanh cạn `max_connections` (tr. 31) | [§2.3.6](chapter-02.md#connection-storm) |
| Replication lag | Độ trễ nhân bản | Độ trễ giữa lúc ghi ở source và lúc có trên replica; vừa là SLI cấp tính vừa là dấu hiệu năng lực ghi dài hạn (tr. 31–32) | [§2.3.6](chapter-02.md#replication-lag) |
| `IOwait` / `IOutil` | Chờ I/O / mức dùng I/O | Thread đang xếp hàng chờ đĩa / phần trăm năng lực truy cập đĩa đang dùng; `IOutil` gần 100% ngoài giờ backup gợi ý full table scan (tr. 32–33) | [§2.3.6](chapter-02.md#io-utilization) |
| Auto-increment space | Không gian auto-increment | Phần giá trị còn lại của kiểu số nguyên làm primary key auto-increment (mặc định có dấu); cạn thì mọi insert lỗi (tr. 33) | [§2.3.6](chapter-02.md#auto-increment-space) |

### Công cụ giám sát

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Percona Monitoring and Management (PMM) | Công cụ giám sát PMM | Bộ giám sát mã nguồn mở dạng client và server, có dashboard theo kinh nghiệm cộng đồng Percona (tr. 24) | [§2.3.2](chapter-02.md#monitoring-solutions) |
| `pt-query-digest` | Công cụ tổng hợp query | Công cụ của Percona Toolkit phân tích slow log, cho biết instance tiêu thời gian vào đâu; mang tính điều tra sau sự việc (tr. 24) | [§2.3.2](chapter-02.md#monitoring-solutions) |
| `pt-kill` | Công cụ giết query | Công cụ của Percona Toolkit tự động hóa lệnh `KILL` để giảm tải chiến thuật (tr. 31) | [§2.3.6](chapter-02.md#connection-growth) |
| Tracing | Truy vết request | Theo dõi một request qua nhiều thành phần (sách nêu Honeycomb, Lightstep) để soi các outlier latency (tr. 27) | [§2.3.4](chapter-02.md#client-side-latency) |

### Đo dài hạn và kiến trúc

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Business cadence | Nhịp kinh doanh | Chu kỳ lưu lượng của doanh nghiệp; đỉnh có thể lớn hơn trung bình nhiều bậc độ lớn và là lúc SLO bị thử thách nhất (tr. 36) | [§2.4.1](chapter-02.md#business-cadence) |
| Percentile | Phân vị | Giá trị sau khi sắp xếp và bỏ phần cao nhất (p95 bỏ 5% cao nhất); khớp với cách đặt SLO (tr. 38) | [§2.4.3](chapter-02.md#percentiles) |
| Say no to averages | Nói không với trung bình | Gộp dữ liệu dài hạn thành trung bình san phẳng đỉnh, cho cảm giác an toàn giả; xem đỉnh khi xem nhiều tháng (tr. 38) | [§2.4.3](chapter-02.md#say-no-to-averages) |
| Retention | Thời gian lưu giữ chỉ số | Công cụ phải hiển thị nhanh dữ liệu khoảng dài, không chỉ nạp nhanh hay lưu lâu (tr. 38) | [§2.4.3](chapter-02.md#retention) |
| Functional sharding | Phân mảnh theo chức năng | Tách bảng của một chức năng kinh doanh sang cluster riêng để quản lý riêng uptime, hiệu năng, quyền (tr. 34) | [§2.3.6](chapter-02.md#functional-horizontal-sharding) |
| Horizontal sharding | Phân mảnh ngang | Chia một data set quá lớn ra nhiều cluster, dùng cơ chế tra cứu để tìm phần cần (tr. 35) | [§2.3.6](chapter-02.md#functional-horizontal-sharding) |
| Backup restore time | Thời gian khôi phục từ backup | Thời gian từ file backup tới database chạy và đã bắt kịp replication; đo tự động, lưu ít nhất một năm; tăng dần là tiền thân của nhu cầu shard (tr. 35) | [§2.3.6](chapter-02.md#backup-restore-time) |

### Bổ sung ngoài sách (liên quan chương 2)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Error budget | Ngân sách lỗi | 1 − SLO: lượng thất bại được phép trong cửa sổ SLO; còn ngân sách thì được thay đổi có rủi ro | [§2.2](chapter-02.md#error-budget) |
| Little's law | Định luật Little | Số việc đang dở dang = tốc độ đến × thời gian xử lý; giải thích vì sao `Threads_running` tăng khi latency tăng | [§2.3.3](chapter-02.md#threads-running) |
| `CONNECTION_ADMIN` (kết nối dự phòng) | Quyền quản trị kết nối | MySQL cho thêm một kết nối ngoài `max_connections` cho tài khoản có quyền này (8.0) hoặc `SUPER`; đã đo trong TN2 | [§2.3.6](chapter-02.md#connection-storm) |
| `super_read_only` | Chỉ đọc kể cả với quản trị | Biến (từ 5.7.8) chặn ghi cả với user có quyền quản trị; probe `SELECT 1` không phát hiện được (TN3) | [§2.3.5](chapter-02.md#errors) |
| `events_errors_summary_global_by_error` | Bảng tổng hợp lỗi theo mã | Bảng Performance Schema (8.0) đếm số lần mỗi mã lỗi được phát sinh phía server | [§2.3.5](chapter-02.md#errors) |
| `QUANTILE_95`, `QUANTILE_99` | Phân vị latency theo digest | Cột của `events_statements_summary_by_digest` (8.0) tính từ histogram; là cận trên của bucket nên có thể lớn hơn max | [§2.3.4](chapter-02.md#query-latency) |
| `information_schema_stats_expiry` | Thời hạn cache thống kê | Biến 8.0 (mặc định 86.400 giây) quyết định thống kê như `AUTO_INCREMENT` trong `information_schema.TABLES` được cache bao lâu | [§2.3.6](chapter-02.md#auto-increment-space) |
| `innodb_autoinc_lock_mode` | Chế độ khóa auto-increment | Mặc định 2 ở 8.0 (1 ở 5.7); với insert không biết trước số hàng, giá trị được cấp theo lô và có thể bỏ phí, làm không gian khóa cạn nhanh hơn số hàng | [§2.3.6](chapter-02.md#auto-increment-space) |

<a id="chuong-3"></a>

## Chương 3 — Performance Schema

### Khái niệm nền

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Performance Schema (`performance_schema`) | Lược đồ hiệu năng | Database có sẵn trong MySQL, cung cấp số đo mức thấp về các thao tác bên trong server, lưu trong các bảng ở bộ nhớ (tr. 41) | [§2.1](chapter-03.md#performance-schema) |
| Instrument | Điểm đo, "cảm biến" | Một đoạn mã MySQL được gắn để thu thông tin, đặt tên dạng `loại/hệ con/.../tên`, ví dụ `wait/lock/metadata/sql/mdl` (tr. 41–42) | [§2.1](chapter-03.md#instrument) |
| Consumer | Bảng nhận dữ liệu | Bảng lưu thông tin do instrument sinh ra; bật tắt trong `setup_consumers` (tr. 41, 44) | [§2.1](chapter-03.md#consumer) |
| Event (`_current`, `_history`, `_history_long`) | Sự kiện hiện tại, lịch sử ngắn, lịch sử dài | Ba dạng bảng cho waits, statements, stages, transactions: đang diễn ra; 10 event gần nhất mỗi thread; 10.000 event gần nhất (tr. 44) | [§2.1.2](chapter-03.md#consumer-organization) |
| Summary table | Bảng tổng hợp | Bảng giữ số liệu đã gộp theo tiêu chí ghi trong tên, ví dụ `memory_summary_by_thread_by_event_name` (tr. 44) | [§2.1.2](chapter-03.md#consumer-organization) |
| Digest | Mẫu truy vấn chuẩn hóa | Dạng câu lệnh đã thay hằng số bằng `?`, dùng để gộp số đo của mọi biến thể; bị cắt ở 1.024 byte (tr. 44–45, 53) | [§2.1.2](chapter-03.md#digest) |
| Instance table | Bảng thể hiện đối tượng | Bảng liệt kê đối tượng đang có, ví dụ `file_instances` (tr. 45) | [§2.1.2](chapter-03.md#consumer-organization) |
| Setup table | Bảng cấu hình | Các bảng `setup_*` dùng để cấu hình Performance Schema lúc chạy (tr. 45) | [§2.2](chapter-03.md#s2-2) |
| Autoscaling / autosized | Tự điều chỉnh kích thước | Bảng cấp ít bộ nhớ lúc đầu rồi tăng khi cần; bộ nhớ đã cấp không trả lại tới khi restart (tr. 45–46, 52) | [§2.1.3](chapter-03.md#resource-consumption) |
| sys schema | Lược đồ sys | Tập view và stored routine dựng trên `performance_schema` để dùng dễ hơn; tự nó không lưu số đo (tr. 46) | [§2.1.5](chapter-03.md#sys-schema) |
| THREAD_ID | Mã thread nội bộ | Định danh thread của MySQL dùng trong hầu hết bảng Performance Schema; khác `PROCESSLIST_ID` (tr. 47) | [§2.1.6](chapter-03.md#thread-id) |
| PROCESSLIST_ID | Mã kết nối | Mã của thread tiền cảnh, thấy trong `SHOW PROCESSLIST`; là số dùng cho `KILL`; chỉ có trong bảng `threads` (tr. 47) | [§2.1.6](chapter-03.md#thread-id) |

### Cấu hình

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `performance_schema` (biến) | Bật/tắt Performance Schema | Biến chỉ đọc, chỉ đặt được lúc khởi động (tr. 48) | [§2.2.1](chapter-03.md#enable-pfs) |
| `setup_instruments` | Bảng cấu hình instrument | Liệt kê mọi instrument với `ENABLED`, `TIMED`, `PROPERTIES`, `DOCUMENTATION`; sửa bằng `UPDATE`, không bền qua restart (tr. 42–43, 48–49) | [§2.2.2](chapter-03.md#enable-instruments) |
| ENABLED / TIMED / COUNTED | Bật / đo thời gian / chỉ đếm | `ENABLED` quyết định có thu event; `TIMED` quyết định có đo thời gian; `COUNTED` là bật nhưng không đo thời gian (tr. 50) | [§2.2.2](chapter-03.md#enable-instruments) |
| `performance-schema-instrument` | Tùy chọn khởi động cho instrument | Cú pháp `tên=giá trị`, lặp được, hỗ trợ wildcard; chuỗi dài hơn được ưu tiên bất kể thứ tự (tr. 49–50) | [§2.2.2](chapter-03.md#enable-instruments) |
| `setup_consumers`, `performance-schema-consumer` | Cấu hình consumer | Bật tắt consumer lúc chạy hoặc lúc khởi động; sách nêu 15 consumer, 8.0.42 có 16 (tr. 50) | [§2.2.3](chapter-03.md#enable-consumers) |
| `global_instrumentation` | Instrument toàn cục | Consumer gốc; tắt thì không thu gì (Bảng 3-1, tr. 50) | [§2.2.3](chapter-03.md#enable-consumers) |
| `thread_instrumentation` | Instrument theo thread | Chỉ được xét khi `global_instrumentation` bật; tắt thì không thu dữ liệu theo thread hay event riêng (tr. 50) | [§2.2.3](chapter-03.md#enable-consumers) |
| `statements_digest` | Consumer digest | Bật việc cộng dồn số đo theo digest (tr. 50) | [§2.2.3](chapter-03.md#enable-consumers) |
| `setup_objects` | Lọc theo đối tượng | Bật tắt theo `OBJECT_TYPE` (EVENT, FUNCTION, PROCEDURE, TABLE, TRIGGER), schema, tên; luật cụ thể thắng (tr. 51) | [§2.2.4](chapter-03.md#setup-objects) |
| `setup_threads` | Lọc thread nền | Cột `ENABLED`, `HISTORY` cho từng loại thread nền (tr. 51) | [§2.2.5](chapter-03.md#setup-threads-actors) |
| `setup_actors` | Lọc thread người dùng | Luật theo `HOST`, `USER` với `ENABLED`, `HISTORY`; cột `ROLE` không dùng (Bảng 3-2, tr. 52) | [§2.2.5](chapter-03.md#setup-threads-actors) |
| `init_file` | File SQL chạy lúc khởi động | Cách duy nhất để giữ thay đổi của `setup_objects`, `setup_actors`, `setup_threads` qua restart (tr. 51–52) | [§2.2.4](chapter-03.md#setup-objects) |
| `performance_schema_<object>_[size\|instances\|classes\|length\|handles]` | Biến kích thước | Biến khởi động quy định kích thước consumer, bảng setup, số thể hiện; `-1` là tự tính (tr. 52) | [§2.2.6](chapter-03.md#memory-sizing) |

### Phân tích câu lệnh

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `events_statements_[current\|history\|history_long]` | Bảng sự kiện câu lệnh | Ba bảng cùng cấu trúc chứa số đo từng câu: thời gian, hàng đọc/gửi, bảng tạm, sort, lỗi… (tr. 54) | [§2.3.1](chapter-03.md#regular-statements) |
| `TIMER_WAIT` | Thời gian chạy | Thời gian của event, đơn vị picosecond (`TIMER_END − TIMER_START`) | [§2.3.1](chapter-03.md#timer-units) |
| `LOCK_TIME` | Thời gian chờ khóa | Thời gian chờ khóa bảng của câu lệnh, đơn vị picosecond; không phải latency của câu | [§2.3.2](chapter-03.md#read-write) |
| `ROWS_EXAMINED`, `ROWS_SENT`, `ROWS_AFFECTED` | Hàng đã đọc, đã gửi, bị ảnh hưởng | So `ROWS_EXAMINED` với hai cột kia để phát hiện câu đọc thừa (tr. 56) | [§2.3.1](chapter-03.md#statement-indicators) |
| `NO_INDEX_USED`, `NO_GOOD_INDEX_USED` | Không dùng index, index không tốt | Cờ cho biết câu không dùng index hoặc dùng index không tối ưu; mức nghiêm trọng cao (Bảng 3-4, tr. 56) | [§2.3.1](chapter-03.md#statement-indicators) |
| `CREATED_TMP_TABLES`, `CREATED_TMP_DISK_TABLES` | Bảng tạm trong bộ nhớ, trên đĩa | Số bảng tạm câu đã tạo; bảng tạm trên đĩa là dấu hiệu nặng (Bảng 3-4, tr. 55) | [§2.3.1](chapter-03.md#statement-indicators) |
| `SELECT_FULL_JOIN`, `SELECT_RANGE_CHECK`, `SELECT_SCAN` | Join quét toàn bảng, kiểm tra key từng hàng, quét bảng đầu | Chỉ số về cách join truy cập bảng (Bảng 3-4, tr. 55) | [§2.3.1](chapter-03.md#statement-indicators) |
| `SORT_MERGE_PASSES`, `SORT_ROWS`, `SORT_SCAN`, `SORT_RANGE` | Chỉ số sắp xếp | Số lượt merge, số hàng sắp xếp, sắp xếp bằng quét bảng hay range (Bảng 3-4, tr. 55–56) | [§2.3.1](chapter-03.md#statement-indicators) |
| `prepared_statements_instances` | Bảng prepared statement | Thống kê cộng dồn của mỗi prepared statement còn tồn tại; mất khi `DROP PREPARE` (tr. 58–59) | [§2.3.1](chapter-03.md#prepared-statements) |
| `statement/sp/*` | Instrument stored routine | `stmt` ghi câu trong routine; `hpush_jump`, `hpop`, `hreturn`… ghi sự kiện điều khiển và handler (tr. 59–60) | [§2.3.1](chapter-03.md#stored-routines) |
| Stage (`events_stages_*`) | Giai đoạn thực thi | Thông tin profile theo giai đoạn của câu lệnh; chỉ có stage chung của server, không có bên trong storage engine (tr. 61–62) | [§2.3.1](chapter-03.md#stages) |
| `NESTING_EVENT_ID` | Mã event cha | Nối event con (stage) với event cha (statement); nên join kèm `THREAD_ID` | [§2.3.1](chapter-03.md#stages) |
| `sys.statement_analysis` | View phân tích câu lệnh | Câu chuẩn hóa với thống kê gộp, sắp theo tổng thời gian (Bảng 3-5, tr. 57) | [§2.3.1](chapter-03.md#examining-statements) |
| `sys.statements_with_full_table_scans` | View câu quét toàn bảng | Câu chuẩn hóa từng quét toàn bảng (Bảng 3-5, tr. 57); trên 8.0.42 không bắt được `UPDATE` quét toàn bảng | [§2.3.1](chapter-03.md#examining-statements) |

### Khóa, bộ nhớ, biến, lỗi

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Metadata lock (MDL) | Khóa siêu dữ liệu | Khóa bảo vệ định nghĩa đối tượng; mọi câu SQL giữ shared MDL tới hết transaction, chặn DDL (tr. 63) | [§2.3.3](chapter-03.md#metadata-lock) |
| `metadata_locks` | Bảng metadata lock | Liệt kê lock đang giữ (`GRANTED`) và đang chờ (`PENDING`) cùng `LOCK_TYPE` như `SHARED_READ`, `EXCLUSIVE` (tr. 63–64) | [§2.3.3](chapter-03.md#metadata-locks) |
| `memory_summary_*` | Bảng tổng hợp bộ nhớ | Bộ nhớ gộp toàn cục, theo thread, account, host, user (Bảng 3-8, tr. 64) | [§2.3.4](chapter-03.md#memory-usage) |
| `CURRENT_NUMBER_OF_BYTES_USED`, `HIGH_NUMBER_OF_BYTES_USED` | Bộ nhớ hiện dùng, mức đỉnh | Hai cột chính khi đọc bảng bộ nhớ (tr. 64–65) | [§2.3.4](chapter-03.md#memory-usage) |
| `sys.memory_global_total` | Tổng bộ nhớ được instrument | Một giá trị duy nhất; không bằng RSS của tiến trình (tr. 65) | [§2.3.4](chapter-03.md#memory-usage) |
| `sys.memory_by_thread_by_current_bytes` | Bộ nhớ theo thread | Sắp thread theo bộ nhớ hiện dùng giảm dần; cột `user` là `NAME@HOST` hoặc thread hệ thống (tr. 65–66) | [§2.3.4](chapter-03.md#memory-usage) |
| `variables_by_thread` | Biến server theo thread | Giá trị biến session của mọi thread, để tìm session đặt giá trị khác thường (tr. 67) | [§2.3.5](chapter-03.md#variables) |
| `status_by_thread` | Biến trạng thái theo thread | Status như `Handler_write` cho từng thread, để tìm kết nối gây áp lực tài nguyên (tr. 68) | [§2.3.5](chapter-03.md#variables) |
| `user_variables_by_thread` | Biến người dùng theo thread | Các biến `@x` của mọi session, kể cả kích thước giá trị (tr. 68–69) | [§2.3.5](chapter-03.md#variables) |
| `variables_info`, `VARIABLE_SOURCE` | Nguồn gốc biến | Biến lấy giá trị từ đâu (`COMPILED`, `COMMAND_LINE`, `PERSISTED`, `DYNAMIC`…), ai đặt, lúc nào (tr. 69) | [§2.3.5](chapter-03.md#variables) |
| `events_errors_summary_*` | Bảng tổng hợp lỗi | Lỗi gộp toàn cục theo mã, hoặc theo user, host, account, thread (tr. 70) | [§2.3.6](chapter-03.md#errors) |
| `SUM_ERROR_RAISED`, `SUM_ERROR_HANDLED` | Số lần lỗi phát ra, được xử lý | Lỗi bị handler bắt chỉ tăng `SUM_ERROR_HANDLED` (đã đo trên 8.0.42) (tr. 70) | [§2.3.6](chapter-03.md#errors) |
| `SHOW ENGINE PERFORMANCE_SCHEMA STATUS` | Trạng thái engine Performance Schema | Kích thước, số lượng của từng consumer; dòng cuối là số byte Performance Schema đang chiếm (tr. 72–73) | [§2.3.7](chapter-03.md#pfs-itself) |

### Bổ sung ngoài sách (liên quan chương 3)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Picosecond | Pico giây | 10⁻¹² giây; đơn vị của `TIMER_*`, `LOCK_TIME`, `SUM_TIMER_*` (MySQL Reference Manual; kiểm chứng bằng `SLEEP(1)`) | [§2.3.1](chapter-03.md#timer-units) |
| `Handler_*` | Bộ đếm lời gọi handler | Biến trạng thái đếm lời gọi đọc/ghi tới storage engine, gồm cả thao tác trên bảng tạm nội bộ | [§2.3.2](chapter-03.md#read-write) |
| `events_statements_cpu` | Consumer thời gian CPU | Consumer thứ 16 trên 8.0.42 (tài liệu MySQL ghi có từ 8.0.28), mặc định tắt | [§2.2.3](chapter-03.md#enable-consumers) |
| `PS_CURRENT_THREAD_ID()` | THREAD_ID của session hiện tại | Hàm từ 8.0.16; `PS_THREAD_ID(id)` đổi `PROCESSLIST_ID` sang `THREAD_ID` | [§2.1.6](chapter-03.md#threads) |
| `lock_wait_timeout` | Thời gian chờ metadata lock tối đa | Mặc định 31.536.000 s; đặt nhỏ cho session chạy DDL để tránh làm tắc bảng | [§2.3.3](chapter-03.md#mdl-queue) |
| `QUERY_SAMPLE_TEXT` | Câu mẫu của digest | Cột trong `events_statements_summary_by_digest` (8.0.3+) chứa một câu cụ thể để chạy `EXPLAIN` | [Tình huống 1](chapter-03.md#th-deploy) |
| `Performance_schema_digest_lost` | Digest bị mất | Biến trạng thái đếm digest không có chỗ khi bảng digest đầy | [§2.1.2](chapter-03.md#digest) |
| `table_io_waits_summary_by_index_usage` | Thống kê dùng index | Nền của `sys.schema_unused_indexes`; bị đặt lại khi restart và khi bảng bị DDL (đã đo) | [§4](chapter-03.md#measure) |
| Cây phụ thuộc consumer | Consumer hierarchy | `global_instrumentation` → `thread_instrumentation`, `statements_digest` → `_current` → `_history`, `_history_long` (MySQL Reference Manual) | [§2.2.3](chapter-03.md#consumer-hierarchy) |

<a id="chuong-4"></a>

## Chương 4 — Operating System and Hardware Optimization

### Tài nguyên và nút thắt

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| CPU exhaustion / CPU saturation | Cạn CPU, CPU bão hòa | Nút thắt hay gặp nhất theo sách: quá nhiều truy vấn chạy song song, hoặc ít truy vấn chạy quá lâu trên CPU (tr. 75) | [§2.1](chapter-04.md#cpu-exhaustion) |
| I/O saturation | Bão hòa I/O | Thiết bị lưu trữ không theo kịp yêu cầu; ít gặp hơn từ khi chuyển sang SSD (tr. 75) | [§2.1](chapter-04.md#s2-1) |
| Memory exhaustion | Cạn bộ nhớ | Thường do cấp phát quá nhiều bộ nhớ cho MySQL (tr. 76) | [§2.1](chapter-04.md#s2-1) |
| CPU-bound workload | Workload bị giới hạn bởi CPU | Workload mà CPU là tài nguyên quyết định tốc độ; nhận biết qua cân bằng CPU và I/O của truy vấn quan trọng và độ đều tải giữa các CPU (tr. 76) | [§2.2](chapter-04.md#s2-2) |
| Low latency / High throughput | Độ trễ thấp / Thông lượng cao | Hai mục tiêu chọn CPU: latency cần CPU nhanh vì mỗi truy vấn dùng một CPU; throughput cần nhiều CPU (tr. 76) | [§2.2](chapter-04.md#latency-throughput) |
| Commodity server | Server phổ thông | Máy giá vừa phải, dễ rút ra chẩn đoán hay hoán đổi; tác giả ưa dùng cho MySQL (tr. 96) | [§2.8](chapter-04.md#s2-8) |

### Bộ nhớ, cache và working set

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Logical read / Physical read | Đọc logic / Đọc vật lý | Đọc từ bộ nhớ so với đọc từ đĩa; cache ấm và đủ lớn khử được đọc vật lý, không khử được đọc logic (tr. 76) | [§2.3.1](chapter-04.md#caching-writes) |
| Cache hit | Trúng cache | Yêu cầu được phục vụ từ bộ nhớ, không chạm đĩa (tr. 76) | [§2.3.1](chapter-04.md#caching-writes) |
| Many writes, one flush | Nhiều lần ghi, một lần flush | Cùng một dữ liệu đổi nhiều lần trong bộ nhớ, chỉ ghi xuống đĩa một lần (tr. 77) | [§2.3.1](chapter-04.md#caching-writes) |
| I/O merging | Gộp I/O | Nhiều thay đổi khác nhau được gom để ghi bằng ít thao tác đĩa hơn (tr. 77) | [§2.3.1](chapter-04.md#caching-writes) |
| Write-ahead logging (WAL) | Ghi log trước | Ghi bản ghi thay đổi tuần tự vào log, flush trang dữ liệu về sau bằng luồng nền (tr. 77) | [§2.3.1](chapter-04.md#wal) |
| Buffered (asynchronous) write | Ghi có đệm, bất đồng bộ | Hệ điều hành nhận và gom lại rồi mới flush (tr. 77) | [§2.3.1](chapter-04.md#caching-writes) |
| Unbuffered (synchronous) write | Ghi không đệm, đồng bộ | Phải xuống đĩa xong mới hoàn tất; hưởng lợi từ write-back cache có pin (tr. 77) | [§2.3.1](chapter-04.md#caching-writes) |
| Working set | Tập dữ liệu làm việc | Dữ liệu cần thường xuyên nhất trong một khoảng thời gian; chỉ phần này cần vừa bộ nhớ (tr. 77, 96) | [§2.3.2](chapter-04.md#working-set) |
| Memory-to-disk ratio | Tỷ lệ bộ nhớ/đĩa | Quan trọng thời HDD, kém quan trọng với SSD (tr. 77) | [§2.3.2](chapter-04.md#working-set) |
| IOPS | Số thao tác I/O mỗi giây | Input/output operations per second; HDD có IOPS thấp (tr. 77) | [§2.3.2](chapter-04.md#working-set) |

### Lưu trữ: SSD và RAID

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Solid-state drive (SSD), flash memory | Ổ thể rắn, bộ nhớ flash | Lưu trữ không mất dữ liệu, không bộ phận chuyển động; mạnh ở random I/O và concurrency; chuẩn cho OLTP (tr. 78) | [§2.4](chapter-04.md#ssd) |
| Hard disk drive (HDD) | Ổ đĩa cứng | Đĩa từ quay; latency cao, IOPS thấp; câu "faster than SSH" ở tr. 75 thực chất là HDD | [§2.1](chapter-04.md#ssd-typo) |
| NVRAM | RAM không mất dữ liệu | Tr. 78 dùng cho thiết bị flash; tr. 82 dùng cho bộ nhớ của flash-backed cache | [§2.4](chapter-04.md#ssd) |
| Erase block | Khối xóa | Đơn vị xóa lớn của flash (ví dụ 512 KB); xóa chậm và làm mòn khối (tr. 78) | [§2.4.1](chapter-04.md#write-amplification) |
| Wear leveling | San đều độ mòn | Phân bổ ghi để các khối flash mòn đều (tr. 79) | [§2.4.1](chapter-04.md#write-amplification) |
| Write amplification | Khuếch đại ghi | Lượng ghi phát sinh thêm do dời dữ liệu, ghi lại dữ liệu và metadata (tr. 79) | [§2.4.1](chapter-04.md#write-amplification) |
| Garbage collection (SSD) | Thu gom rác của SSD | Thu hồi khối để có chỗ ghi mới; càng đầy càng tốn, làm thiết bị chậm dần (tr. 79) | [§2.4.2](chapter-04.md#garbage-collection) |
| RAID | Mảng đĩa dư thừa | Redundant Array of Inexpensive Disks; giúp dư thừa, dung lượng, cache, tốc độ (tr. 77, 79) | [§2.5](chapter-04.md#s2-5) |
| RAID 0, 1, 5, 6, 10, 50 | Các mức RAID | Từ strip không dư thừa (0) tới mirror (1), parity đơn (5), parity kép (6), mirror strip (10), RAID 5 strip (50); so sánh ở Bảng 4-1 (tr. 79–81) | [§2.5](chapter-04.md#raid-levels) |
| Parity | Dữ liệu chẵn lẻ | Thông tin dư để dựng lại dữ liệu khi một ổ hỏng (RAID 5) hoặc hai ổ (RAID 6) (tr. 80–81) | [§2.5](chapter-04.md#raid-levels) |
| Concatenated mirror | Mirror nối tiếp | Cách cài RAID 10 không strip, dữ liệu nóng dồn vào một cặp ổ (tr. 81) | [§2.5](chapter-04.md#raid-levels) |
| Degraded array | Mảng suy giảm | Mảng đang thiếu ổ; cần cảnh báo ngay (tr. 82) | [§2.5.1](chapter-04.md#raid-monitoring) |
| Background Patrol Read | Đọc tuần tra nền | Tính năng controller kiểm tra và sửa phương tiện hỏng khi các ổ vẫn online (tr. 82) | [§2.5.1](chapter-04.md#raid-monitoring) |
| Hot spare | Ổ dự phòng nóng | Ổ để trống, controller tự dùng khi khôi phục (tr. 82) | [§2.5.1](chapter-04.md#raid-monitoring) |
| Battery backup unit (BBU) | Bộ pin dự phòng | Pin giữ dữ liệu trong cache ghi của controller khi mất điện (tr. 82, 85) | [§2.5.1](chapter-04.md#bbu) |
| Write-back / Write-through | Ghi trễ / Ghi xuyên | Cache xác nhận ghi ngay rồi ghi sau, so với chờ ghi xuống đĩa; pin hỏng thì controller chuyển sang write-through (tr. 82) | [§2.5.1](chapter-04.md#bbu) |
| Learning cycle | Chu kỳ hiệu chuẩn pin | Lúc controller cho pin xả nạp để hiệu chuẩn, cache ghi bị tắt (tr. 82) | [§2.5.1](chapter-04.md#bbu) |
| Flash-backed cache | Cache dựa trên flash | Cache ghi dùng NVRAM thay pin, không cần learning cycle (tr. 82) | [§2.5.1](chapter-04.md#bbu) |
| Stripe chunk size | Kích thước chunk của stripe | Lượng dữ liệu liên tục trên một ổ trong mảng strip; tối ưu tùy workload và controller (tr. 83) | [§2.5.2](chapter-04.md#stripe-chunk) |
| RAID cache | Cache của controller RAID | Bộ nhớ nhỏ trên controller; nên dành cho ghi, cache đọc thường lãng phí (tr. 84–85) | [§2.5.2](chapter-04.md#raid-cache) |
| Read-ahead | Đọc trước | Nạp trước dữ liệu dự đoán sắp cần; controller, filesystem và InnoDB đều có thể làm (tr. 84, 88) | [§2.5.2](chapter-04.md#raid-cache) |

### Mạng

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Network latency / bandwidth | Độ trễ / băng thông mạng | Với ứng dụng nhiều lần truyền nhỏ, latency cộng dồn là vấn đề lớn nhất (tr. 86) | [§2.6](chapter-04.md#network) |
| Packet loss | Mất gói | Chỉ 1% đã làm hiệu năng giảm đáng kể vì phải chờ rồi gửi lại (tr. 86) | [§2.6](chapter-04.md#network) |
| `skip_name_resolve` | Bỏ phân giải tên | Tắt tra DNS xuôi và ngược khi kết nối; tài khoản phải dùng IP, `localhost` hoặc wildcard IP (tr. 86) | [§2.6](chapter-04.md#skip-name-resolve) |
| `ip_local_port_range`, `tcp_max_syn_backlog`, `tcp_fin_timeout` | Tham số TCP của Linux | Khoảng cổng cục bộ, hàng đợi kết nối đến, thời gian chờ sau khi đóng socket; chỉ đổi khi bất thường (tr. 86–87) | [§2.6](chapter-04.md#s2-6) |

### Filesystem và hệ điều hành

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Journaling filesystem | Filesystem có nhật ký | ext4, XFS, ZFS; tránh kiểm tra filesystem lâu sau crash (tr. 87) | [§2.7](chapter-04.md#filesystem) |
| `data=writeback` / `data=ordered` / `data=journal` | Ba chế độ journal của ext3/ext4 | Chỉ metadata không đồng bộ / metadata sau dữ liệu / cả dữ liệu (tr. 87–88) | [§2.7](chapter-04.md#data-journaling) |
| `noatime`, `nodiratime` | Tắt ghi thời gian truy cập | Tùy chọn mount tránh ghi khi chỉ đọc; có thể tăng 5–10% (tr. 88) | [§2.7](chapter-04.md#data-journaling) |
| Direct I/O, `O_DIRECT` | I/O trực tiếp | I/O không qua page cache; giảm áp lực file cache; với InnoDB đặt bằng `innodb_flush_method` (tr. 88, 91) | [§2.7.2](chapter-04.md#swapping) |
| LVM snapshot | Snapshot của LVM | Ảnh chụp volume dùng để khởi tạo replica hay backup; cần kiểm tra tương thích filesystem (tr. 88) | [§2.7](chapter-04.md#data-journaling) |
| XFS | XFS | Filesystem tác giả khuyên dùng trên Linux (tr. 89, 97) | [§2.7](chapter-04.md#filesystem) |
| Disk queue scheduler (`cfq`, `noop`, `deadline`) | Bộ lập lịch hàng đợi đĩa | Quyết định thứ tự gửi yêu cầu xuống thiết bị; dùng bất cứ thứ gì trừ `cfq` cho server (tr. 89–90) | [§2.7.1](chapter-04.md#io-scheduler) |
| SAN | Mạng lưu trữ | Storage area network; tự lập lịch, hợp với `noop` (tr. 90) | [§2.7.1](chapter-04.md#io-scheduler) |
| `tcmalloc`, `jemalloc` | Bộ cấp phát bộ nhớ thay thế | Thay `glibc` malloc để tăng hiệu năng, giảm phân mảnh (tr. 90) | [§2.7.2](chapter-04.md#swapping) |
| Swapping | Hoán đổi bộ nhớ | Hệ điều hành ghi bộ nhớ ảo xuống đĩa khi thiếu RAM; tác giả khuyên chạy database không dùng swap (tr. 90–91) | [§2.7.2](chapter-04.md#swapping) |
| `swappiness` | Mức ưu tiên swap | `/proc/sys/vm/swappiness`, mặc định 60; sách khuyên server đặt 0 (tr. 91) | [§2.7.2](chapter-04.md#swapping) |
| OOM killer, `oom_score_adj` | Bộ giết tiến trình khi hết bộ nhớ | Tiến trình kernel giết một tiến trình khi thiếu bộ nhớ; chỉnh điểm để bảo vệ MySQL và SSH (tr. 91) | [§2.7.2](chapter-04.md#oom-killer) |
| NUMA | Truy cập bộ nhớ không đồng nhất | Nonuniform memory access; có thể góp phần khiến Linux swap MySQL (tr. 91) | [§2.7.2](chapter-04.md#swapping) |
| `memlock` | Khóa bộ nhớ | Tùy chọn khóa MySQL trong RAM; tránh swap nhưng có thể làm crash (tr. 92) | [§2.7.2](chapter-04.md#swapping) |

### Công cụ quan sát

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `vmstat` | Thống kê bộ nhớ ảo | Báo tiến trình, bộ nhớ, swap, I/O, ngắt, CPU; dùng với khoảng lấy mẫu và bỏ dòng đầu (tr. 92–93) | [§2.7.3](chapter-04.md#vmstat) |
| `r`, `b` (vmstat) | Tiến trình chờ CPU, chờ I/O | `r` chờ CPU; `b` ngủ không ngắt được, thường chờ I/O (tr. 93) | [§2.7.3](chapter-04.md#vmstat) |
| `si`, `so` / `swpd` | Hoạt động swap / dung lượng swap | Nên nhìn hoạt động (`si`, `so`), không nhìn dung lượng (tr. 90, 93) | [§2.7.3](chapter-04.md#vmstat) |
| `us`, `sy`, `id`, `wa` | Phần trăm thời gian CPU | Mã người dùng, kernel, rảnh, chờ I/O (tr. 93) | [§2.7.3](chapter-04.md#vmstat) |
| Steal time (`st`) | Thời gian bị đánh cắp | Máy ảo có việc mà hypervisor chạy thứ khác (tr. 93) | [§2.7.3](chapter-04.md#vmstat) |
| Context switch (`cs`) | Chuyển ngữ cảnh | Số lần chuyển tiến trình mỗi giây (tr. 93) | [§2.7.3](chapter-04.md#vmstat) |
| `iostat` | Thống kê I/O thiết bị | Thống kê thiết bị khối; dùng `-dxk` với khoảng lấy mẫu (tr. 93–95) | [§2.7.3](chapter-04.md#iostat) |
| `await`, `r_await`, `w_await` | Thời gian chờ I/O | Thời gian trung bình (ms) yêu cầu được phục vụ; `r_await`, `w_await` gồm cả chờ và phục vụ (tr. 94) | [§2.7.3](chapter-04.md#iostat) |
| `svctm` | Thời gian phục vụ | Ms phục vụ không tính chờ (tr. 94); bản sysstat mới đã bỏ | [§2.7.3](chapter-04.md#iostat) |
| `avgqu-sz`, `avgrq-sz` | Độ dài hàng đợi, kích thước yêu cầu | Số yêu cầu đang chờ; kích thước yêu cầu tính bằng sector (tr. 94) | [§2.7.3](chapter-04.md#iostat) |
| `%util` | Tỷ lệ thời gian bận | Phần trăm thời gian có ít nhất một yêu cầu; không phải mức bão hòa trừ với một ổ đơn (tr. 95) | [§2.7.3](chapter-04.md#iostat) |
| Little's law | Định luật Little | Số yêu cầu đồng thời = tốc độ đến × thời gian trong hệ thống; sách dùng để tính concurrency của thiết bị (tr. 95) | [§2.7.3](chapter-04.md#little-law) |
| `mpstat`, `sar`, `dstat`, `collectl`, `blktrace` | Công cụ quan sát bổ trợ | Công cụ bổ trợ: từng CPU, lịch sử, thay thế vmstat/iostat, chi tiết I/O đĩa (tr. 92, 95) | [§2.7.4](chapter-04.md#other-tools) |
| `pt-diskstats` | Thống kê đĩa của Percona Toolkit | Công cụ của Percona Toolkit thay `iostat`, tách đọc ghi, thấy concurrency, tương tác (tr. 95–96) | [§2.7.4](chapter-04.md#other-tools) |
| `perf` | Profiler của Linux | Profiler của Linux, xem toàn hệ thống hay một tiến trình (tr. 96) | [§2.7.4](chapter-04.md#other-tools) |

### Bổ sung ngoài sách (liên quan chương 4)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| USE method | Phương pháp USE | Utilization, Saturation, Errors của Brendan Gregg: ba câu hỏi cho mỗi tài nguyên | [§2.1](chapter-04.md#cpu-exhaustion) |
| blk-mq: `none`, `mq-deadline`, `bfq`, `kyber` | Scheduler đa hàng đợi | Thay `cfq`, `deadline`, `noop` từ Linux 5.0 | [§2.7.1](chapter-04.md#io-scheduler) |
| `relatime` | Cập nhật atime tương đối | Mặc định mount của Linux từ 2.6.30, giảm ghi atime | [§2.7](chapter-04.md#data-journaling) |
| TRIM / discard | Báo khối đã bỏ | Lệnh để SSD biết khối nào không còn dùng, giúp garbage collection | [§2.4.2](chapter-04.md#garbage-collection) |
| Ephemeral port, TIME_WAIT | Cổng tạm, trạng thái chờ đóng | Cổng phía mở kết nối; kết nối đóng còn giữ cổng một lúc | [§2.6](chapter-04.md#s2-6) |
| InnoDB linear read-ahead (`innodb_read_ahead_threshold`) | Đọc trước tuyến tính của InnoDB | Thấy truy cập tuần tự trong một extent thì đọc trước extent kế; đã đo ở thí nghiệm 1 | [Thí nghiệm 1](experiments/chapter-04.md#exp1) |
| LRU hai đoạn (young/old), `innodb_old_blocks_time` | Danh sách LRU hai đoạn | Trang mới vào đoạn old, chỉ lên young khi được dùng lại sau một khoảng thời gian; bảo vệ dữ liệu nóng khỏi quét | [Thí nghiệm 2](experiments/chapter-04.md#exp2) |
| Adaptive hash index (đo được) | Chỉ mục băm thích ứng | Đo được: giảm số lần lấy trang mỗi tra cứu từ khoảng 3 xuống khoảng 2; khái niệm ở chương 7 | [Thí nghiệm 2](experiments/chapter-04.md#exp2) |
| Doublewrite buffer | Bộ đệm ghi kép | Mỗi trang flush được ghi thêm một lần vào vùng doublewrite; một dạng write amplification ở tầng database | [Thí nghiệm 3](experiments/chapter-04.md#exp3) |
| Undo tablespace, rollback segment | Tablespace undo, rollback segment | Mỗi transaction để lại trang undo bẩn; nhiều transaction nhỏ sinh I/O nền | [Thí nghiệm 3](experiments/chapter-04.md#exp3) |

<a id="chuong-5"></a>

## Chương 5 — Optimizing Server Settings

### Cơ chế cấu hình

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Option file (`my.cnf`, `my.ini`) | File cấu hình | File dạng INI chia section (`[mysqld]`, `[client]`); server đọc cùng tham số dòng lệnh. Nên hỏi `mysqld --verbose --help` để biết file nào được đọc, theo thứ tự nào (tr. 100–101) | [§2.1](chapter-05.md#config-sources) |
| Option / variable | Tùy chọn / biến | Phần lớn tùy chọn dòng lệnh tương ứng một biến server cùng tên; có ngoại lệ như `--memlock` đặt `locked_in_memory` (tr. 101) | [§2.1](chapter-05.md#config-sources) |
| Global scope | Phạm vi toàn server | Giá trị áp dụng cho cả server, và là mặc định để khởi tạo giá trị session khi kết nối mở (tr. 102) | [§2.1.1](chapter-05.md#scope) |
| Session scope | Phạm vi phiên | Giá trị riêng của một kết nối; đổi chỉ ảnh hưởng kết nối đó và mất khi kết nối đóng (tr. 102) | [§2.1.1](chapter-05.md#scope) |
| Dynamic configuration variable | Biến cấu hình động | Biến đổi được bằng `SET` khi server đang chạy; mất khi tắt nếu không ghi vào file (tr. 102) | [§2.1.1](chapter-05.md#dynamic-variables) |
| Compiled-in default | Giá trị biên dịch sẵn | Giá trị mặc định trong mã server; `SET GLOBAL x = DEFAULT` trả về giá trị này, không phải giá trị lúc khởi động (tr. 102–103) | [§2.1.1](chapter-05.md#set-default) |
| Persisted system variable (`SET PERSIST`) | Biến hệ thống được lưu bền | Tính năng MySQL 8.0: đặt giá trị lúc chạy và ghi xuống đĩa để dùng ở lần khởi động sau (tr. 103) | [§2.1.2](chapter-05.md#persisted-variables) |
| Side effect of setting variables | Tác dụng phụ khi đặt biến | Đổi biến online có thể bắt server làm nhiều việc (ví dụ flush block bẩn) hoặc chỉ có hiệu lực ở thời điểm sau (tr. 103–104) | [§2.1.3](chapter-05.md#side-effects) |
| Status variable | Biến trạng thái | Bộ đếm và chỉ số của server (`SHOW GLOBAL STATUS`); nên xem cả giá trị tuyệt đối lẫn thay đổi theo thời gian (tr. 108) | [§2.3.2](chapter-05.md#status-variables) |
| Tuning by ratio | Tune theo tỉ lệ | Thói quen chỉnh cache theo hit ratio; sách gọi là lời khuyên rất sai vì hit ratio phụ thuộc workload và không phản ánh kích thước đúng (tr. 105–106) | [§2.2](chapter-05.md#tuning-by-ratio) |
| Cache hit ratio | Tỉ lệ trúng cache | Tỉ lệ yêu cầu được phục vụ từ cache; tương quan với kích thước cache chỉ là ngẫu nhiên (tr. 105) | [§2.2](chapter-05.md#tuning-by-ratio) |
| Tuning script | Script tuning | Script phổ biến trên internet mã hóa các thói quen tune theo tỉ lệ; sách khuyên bỏ qua (tr. 106) | [§2.2](chapter-05.md#s2-2) |
| Memory consumption formula | Công thức tính bộ nhớ | Công thức MySQL in ra khi crash; lỗi thời, vì không thể đặt cận trên cho bộ nhớ MySQL (tr. 106) | [§2.2](chapter-05.md#memory-formula) |
| `innodb_dedicated_server` | Chế độ server chuyên dụng | Tùy chọn MySQL 8.0 tự đặt buffer pool, redo log và flush method theo RAM của máy chỉ chạy MySQL; thường dùng 50–75% RAM (tr. 108–109, 124) | [§2.3.1](chapter-05.md#innodb-dedicated-server) |
| Minimal configuration | Cấu hình tối thiểu | File mẫu của sách: vài thiết lập chung, InnoDB, log, kết nối; nhiều hơn mức nhiều người cần (tr. 107–108) | [§2.3.1](chapter-05.md#sample-config) |

### Bộ nhớ

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Per-connection memory | Bộ nhớ theo kết nối | Bộ nhớ để giữ kết nối và chạy truy vấn; khó dự đoán (prepared statement, bảng tạm lớn, stored procedure) nhưng không cần tính kịch bản xấu nhất (tr. 109) | [§2.4.1](chapter-05.md#per-connection-memory) |
| InnoDB buffer pool | Vùng đệm InnoDB | Bộ nhớ lớn nhất của InnoDB, chứa index, dữ liệu hàng, adaptive hash index, change buffer, lock; giúp trì hoãn và gộp ghi (tr. 110) | [§2.4.3](chapter-05.md#buffer-pool) |
| Dirty page | Trang bẩn | Trang đã sửa trong buffer pool, chưa ghi xuống datafile; nhiều trang bẩn làm shutdown lâu (tr. 110) | [§2.4.3](chapter-05.md#dirty-pages) |
| `innodb_max_dirty_pages_pct` | Ngưỡng tỉ lệ trang bẩn | Ngưỡng để InnoDB thôi flush "lười" và flush nhanh hết mức; không phải trần cứng (tr. 110) | [§2.4.3](chapter-05.md#dirty-pages) |
| Lazy flushing, page cleaner | Flush lười, luồng dọn trang | Flush trang bẩn bằng luồng nền, gộp và ghi tuần tự, chỉ gấp khi cần chỗ hay vượt ngưỡng (tr. 110) | [§2.4.3](chapter-05.md#dirty-pages) |
| Cold cache | Cache nguội | Buffer pool rỗng sau khi khởi động lại (tr. 110) | [§2.4.3](chapter-05.md#buffer-pool-warmup) |
| Buffer pool dump/load | Lưu và nạp lại buffer pool | `innodb_buffer_pool_dump_at_shutdown` và `innodb_buffer_pool_load_at_startup`, bật mặc định, làm ấm server khi khởi động (tr. 110–111) | [§2.4.3](chapter-05.md#buffer-pool-warmup) |
| Thread cache | Bộ đệm thread | Giữ thread rảnh sẵn sàng phục vụ kết nối mới; kích thước `thread_cache_size`, kiểm bằng `Threads_created` (tr. 111) | [§2.4.4](chapter-05.md#thread-cache) |
| Adaptive hash index | Index băm thích ứng | Cấu trúc InnoDB giữ trong buffer pool; chương 5 chỉ nhắc, chương 7 giải thích (tr. 110) | [§2.4.3](chapter-05.md#buffer-pool) |
| Change buffer | Vùng đệm thay đổi | Cấu trúc InnoDB nằm trong buffer pool và tablespace; tên mới của insert buffer; chương 5 chỉ nhắc (tr. 110, 116) | [§2.4.3](chapter-05.md#buffer-pool) |

### I/O và durability

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Transaction log (redo log) | Nhật ký transaction | Log kích thước cố định, ghi tuần tự và vòng tròn, giúp commit rẻ và phục hồi sau crash; biến random I/O thành sequential I/O (tr. 113) | [§2.5.1](chapter-05.md#redo-log) |
| `innodb_log_file_size` | Kích thước log file | Cùng `innodb_log_files_in_group` quyết định tổng kích thước redo log theo sách; rất quan trọng cho ghi (tr. 113, 124) | [§2.5.1](chapter-05.md#redo-log) |
| Log buffer | Bộ đệm log | Vùng nhớ giữ bản ghi redo trước khi ghi ra log file; flush khi đầy, khi commit hoặc mỗi giây (tr. 113–114) | [§2.5.2](chapter-05.md#log-buffer) |
| Group commit | Commit theo nhóm | Gom log của nhiều transaction sẵn sàng flush vào một thao tác I/O (tr. 114) | [§2.5.2](chapter-05.md#group-commit) |
| `innodb_flush_log_at_trx_commit` | Chế độ flush log khi commit | 1: ghi và flush mỗi commit; 2: ghi vào cache hệ điều hành mỗi commit, flush mỗi giây; 0: ghi và flush mỗi giây (tr. 114) | [§2.5.2](chapter-05.md#flush-log-at-trx-commit) |
| Write và flush | Ghi và đẩy xuống đĩa | Ghi chỉ chuyển dữ liệu vào cache của hệ điều hành; flush mới đưa tới bộ nhớ bền vững (tr. 114) | [§2.5.2](chapter-05.md#flush-log-at-trx-commit) |
| Fake flush | Flush giả | Controller hay hệ điều hành báo đã flush nhưng dữ liệu còn trong cache khác; có thể gây hỏng dữ liệu (tr. 115) | [§2.5.2](chapter-05.md#flush-log-at-trx-commit) |
| `innodb_flush_method` | Phương thức flush | Cách InnoDB mở, đọc, ghi, flush log file và datafile; sách khuyên `O_DIRECT` trên Unix có RAID với write cache dùng pin (tr. 115) | [§2.5.2](chapter-05.md#flush-method) |
| Tablespace | Không gian bảng | Filesystem ảo trên một hay nhiều tệp, chứa bảng, index, undo log, change buffer, doublewrite buffer (tr. 115–116) | [§2.5.3](chapter-05.md#tablespace) |
| `innodb_data_file_path` | Đường dẫn tệp dữ liệu | Khai báo các tệp của system tablespace, nối đuôi nhau, có thể tự giãn và đặt trần (tr. 116) | [§2.5.3](chapter-05.md#tablespace) |
| `innodb_file_per_table` | Mỗi bảng một tệp | Lưu mỗi bảng trong tệp `.ibd` riêng; dễ thu hồi dung lượng, dễ quan sát (tr. 116–117) | [§2.5.3](chapter-05.md#file-per-table) |
| Doublewrite buffer | Vùng ghi kép | Cấu trúc InnoDB mà trang được ghi qua trước khi vào vị trí thật (Hình 5-1, tr. 112, 116) | [§2.5](chapter-05.md#figure-5-1) |
| Undo log, old row versions | Undo log, phiên bản cũ của hàng | Thông tin để dựng lại phiên bản cũ; transaction mở lâu ở REPEATABLE READ ngăn xóa chúng (tr. 115, 117) | [§2.5.3](chapter-05.md#old-row-versions) |
| Purge | Dọn phiên bản cũ | Quá trình nhiều luồng xóa phiên bản cũ không còn cần; chỉnh bằng `innodb_purge_threads`, `innodb_purge_batch_size` (tr. 117) | [§2.5.3](chapter-05.md#old-row-versions) |
| History list length | Độ dài history list | Chỉ số trong phần TRANSACTIONS của InnoDB status cho biết lượng undo chờ purge (tr. 118) | [§2.5.3](chapter-05.md#history-list) |
| `innodb_max_purge_lag` | Ngưỡng trễ purge | Số transaction chờ purge tối đa trước khi InnoDB trì hoãn truy vấn cập nhật; "cái ác nhỏ hơn" (tr. 118) | [§2.5.3](chapter-05.md#history-list) |
| `sync_binlog` | Đồng bộ binary log | 1 (mặc định) flush binary log để bền; khác 1 có thể làm binlog lệch dữ liệu sau crash, hỏng replication (tr. 118) | [§2.5.4](chapter-05.md#sync-binlog) |
| Battery-backed write cache | Write cache dùng pin | Cache của RAID controller có pin, chế độ write-back, cho ghi nhanh mà vẫn bền (tr. 115, 118–119) | [§2.5.4](chapter-05.md#s2-5-4) |

### Concurrency

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `innodb_thread_concurrency` | Giới hạn thread trong InnoDB | Số thread tối đa trong kernel InnoDB cùng lúc; 0 là không giới hạn; gợi ý bắt đầu bằng số lõi CPU (tr. 119) | [§2.6](chapter-05.md#innodb-thread-concurrency) |
| `innodb_thread_sleep_delay` | Thời gian ngủ trước khi vào hàng | Pha 1 khi thread không vào được kernel: ngủ bấy nhiêu micro giây (mặc định 10.000) rồi thử lại (tr. 119) | [§2.6](chapter-05.md#innodb-thread-concurrency) |
| Concurrency tickets | Vé concurrency | Số lần thread được quay lại kernel InnoDB không qua kiểm tra; cấp theo truy vấn (`innodb_concurrency_tickets`, tr. 120) | [§2.6](chapter-05.md#innodb-thread-concurrency) |
| `innodb_commit_concurrency` | Giới hạn commit đồng thời | Số thread được commit cùng lúc; giai đoạn commit chủ yếu bị chặn bởi I/O (tr. 120) | [§2.6](chapter-05.md#s2-6) |

### Safety settings

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Safety settings | Thiết lập an toàn | Thiết lập giúp server an toàn, đáng tin hơn; thường không tăng hiệu năng (tr. 100, 120) | [§2.7](chapter-05.md#s2-7) |
| `max_connections` | Số kết nối tối đa | "Phanh khẩn cấp": từ chối kết nối thừa để thất bại nhanh và rẻ; theo dõi `Max_used_connections` (tr. 120–121) | [§2.7](chapter-05.md#max-connections) |
| `max_connect_errors` | Số lỗi kết nối tối đa | Số lỗi trước khi một host bị chặn tới khi flush host cache; vô tác dụng khi bật `skip_name_resolve` (tr. 120) | [§2.7](chapter-05.md#safety-table) |
| `skip_name_resolve` | Bỏ phân giải tên | Tắt tra DNS khi xác thực; phải chuyển grant theo hostname sang IP, wildcard hay `localhost` (tr. 121) | [§2.7](chapter-05.md#safety-table) |
| `sql_mode` | Chế độ SQL | Các tùy chọn thay đổi hành vi SQL của server; đổi trên ứng dụng đang chạy có rủi ro, kiểm khi nâng cấp (tr. 121–122) | [§2.7](chapter-05.md#safety-table) |
| `sysdate_is_now` | SYSDATE như NOW | Làm `SYSDATE()` tất định, bảo vệ replication và point-in-time recovery (tr. 122) | [§2.7](chapter-05.md#safety-table) |
| `read_only`, `super_read_only` | Chỉ đọc, chỉ đọc cả với SUPER | Chặn ghi trên replica; bản `super_` chặn cả user có `SUPER`, chỉ replication được ghi (tr. 122) | [§2.7](chapter-05.md#safety-table) |

### InnoDB nâng cao

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `innodb_autoinc_lock_mode` | Chế độ khóa auto-increment | Cách sinh giá trị auto-increment; có thể là nút thắt khi insert đồng thời cao (tr. 123) | [§2.8](chapter-05.md#s2-8) |
| `innodb_buffer_pool_instances` | Số phần của buffer pool | Chia buffer pool để giảm tranh chấp mutex toàn cục trên máy nhiều lõi (tr. 123) | [§2.8](chapter-05.md#s2-8) |
| `innodb_io_capacity` | Năng lực I/O khai báo | Cho InnoDB biết năng lực I/O để flush trang bẩn đều đặn; trên flash có thể cần rất cao (tr. 123) | [§2.8](chapter-05.md#s2-8) |
| `innodb_read_io_threads`, `innodb_write_io_threads` | Số luồng I/O đọc, ghi | Số luồng nền cho I/O, mặc định 4 và 4 (tr. 123) | [§2.8](chapter-05.md#s2-8) |
| `innodb_strict_mode` | Chế độ nghiêm ngặt của InnoDB | Biến một số cảnh báo `CREATE TABLE` thành lỗi; cẩn thận khi restore backup (tr. 123) | [§2.8](chapter-05.md#s2-8) |
| Two-part LRU list | Danh sách LRU hai phần | LRU của buffer pool chia hai phần để truy vấn một lần không đẩy trang dùng lâu dài ra (tr. 124) | [§2.8](chapter-05.md#old-blocks-time) |
| `innodb_old_blocks_time` | Thời gian giữ ở phần old | Số mili giây phải trôi qua trước khi trang được chuyển phần trong LRU; sách nói mặc định 0 và khuyên khoảng 1000 (tr. 124). Đo trên 8.0.42: mặc định đã là 1000, và hướng chuyển là từ old lên young (xem đính chính) | [§2.8](chapter-05.md#old-blocks-time) |

### Bổ sung ngoài sách (liên quan chương 5)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `innodb_redo_log_capacity` | Dung lượng redo log | Từ MySQL 8.0.30 thay `innodb_log_file_size` và `innodb_log_files_in_group`; đổi được online; dung lượng thật đọc ở status `Innodb_redo_log_capacity_resized` | [§2.5.1](chapter-05.md#redo-log-capacity) |
| `mysqld-auto.cnf` | Tệp biến đã persist | Tệp JSON trong datadir nơi `SET PERSIST` ghi giá trị; bị bỏ qua khi khởi động với `--no-defaults` (đã đo) | [§2.1.2](chapter-05.md#persisted-variables) |
| `SET PERSIST_ONLY`, `RESET PERSIST` | Chỉ ghi persist, xóa persist | Ghi giá trị cho biến read-only để dùng ở lần khởi động sau; xóa mọi giá trị đã persist | [§2.1.2](chapter-05.md#persisted-variables) |
| `performance_schema.variables_info` | Bảng nguồn gốc biến | Cho biết biến lấy giá trị từ đâu (`COMPILED`, `COMMAND_LINE`, `DYNAMIC`, `PERSISTED`...), ai đổi, lúc nào | [§2.1.1](chapter-05.md#set-default) |
| `--validate-config` | Kiểm tra cấu hình | Tùy chọn của `mysqld` (từ 8.0.16) kiểm cấu hình khởi động rồi thoát, không mở dữ liệu | [§2.3.1](chapter-05.md#sample-config) |
| Thread pool | Nhóm thread | Plugin server phục vụ nhiều kết nối bằng số thread có hạn; không có trong bản Community 8.0.42 | [§2.4.4](chapter-05.md#thread-pool-vs-cache) |
| Connection pooling (tầng truy cập) | Gộp kết nối ở proxy | Proxy như ProxySQL giữ ít kết nối tới database cho nhiều kết nối ứng dụng (chương 11) | [§2.4.4](chapter-05.md#thread-pool-vs-cache) |
| Midpoint insertion | Chèn ở điểm giữa | Trang mới đọc vào được chèn ở đầu phần old của LRU, chỉ lên phần new khi được truy cập lại sau `innodb_old_blocks_time` | [§2.8](chapter-05.md#old-blocks-time) |
| Checkpoint | Điểm kiểm | Vị trí trong redo log mà mọi thay đổi trước đó đã vào datafile; phần log trước nó được ghi đè | [§2.5.1](chapter-05.md#redo-log) |
| Undo tablespace | Tablespace undo | Ở MySQL 8.0, undo log nằm trong tablespace riêng (`undo_001`, `undo_002`) có thể truncate, không nằm trong `ibdata1` | [§2.5.3](chapter-05.md#file-per-table) |
| `O_DIRECT_NO_FSYNC` | O_DIRECT bỏ bớt fsync | Giá trị `innodb_flush_method` mà `innodb_dedicated_server` đặt nếu nền tảng hỗ trợ (theo `--help` của 8.0.42) | [§2.3.1](chapter-05.md#innodb-dedicated-server) |
| Double buffering | Đệm hai lần | Cùng dữ liệu nằm cả trong buffer pool lẫn page cache của hệ điều hành; `O_DIRECT` tránh điều này | [§2.5.2](chapter-05.md#flush-method) |

<a id="chuong-6"></a>

## Chương 6 — Schema Design and Management

### Nguyên tắc chọn kiểu dữ liệu

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Smaller is usually better | Nhỏ hơn thường tốt hơn | Dùng kiểu nhỏ nhất vẫn lưu đúng dữ liệu: ít chỗ trên đĩa, trong bộ nhớ và CPU cache; nhưng đừng đánh giá thấp miền giá trị (tr. 126) | [§2.1](chapter-06.md#smaller-is-better) |
| Simple is good | Đơn giản là tốt | Kiểu đơn giản tốn ít CPU hơn; số nguyên so sánh rẻ hơn chuỗi vì chuỗi phải đi qua character set và collation (tr. 126) | [§2.1](chapter-06.md#simple-is-good) |
| Nullable column | Cột cho phép NULL | Cột khai báo không có `NOT NULL`; làm index, thống kê và phép so sánh phức tạp hơn; nên `NOT NULL` trừ khi thật sự cần NULL (tr. 126) | [§2.1](chapter-06.md#avoid-null) |
| Character set | Bộ ký tự | Cách mã hóa ký tự thành byte, ví dụ latin1 (1 byte), utf8mb4 (tối đa 4 byte) | [§2.1](chapter-06.md#simple-is-good) |
| Collation | Quy tắc so sánh và sắp xếp | Quy tắc so sánh, sắp xếp chuỗi của một bộ ký tự; làm so sánh chuỗi tốn hơn so sánh số (tr. 126, 128) | [§2.1](chapter-06.md#simple-is-good) |
| Data type alias | Tên đồng nghĩa của kiểu | Tên tương thích như `INTEGER`, `BOOL`, `NUMERIC`; `SHOW CREATE TABLE` hiện kiểu gốc, không ảnh hưởng hiệu năng (tr. 127) | [§2.1](chapter-06.md#guidelines) |

### Số

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Integer types | Kiểu số nguyên | `TINYINT`, `SMALLINT`, `MEDIUMINT`, `INT`, `BIGINT`: 8, 16, 24, 32, 64 bit (tr. 127) | [§2.1.1](chapter-06.md#whole-numbers) |
| UNSIGNED | Không dấu | Cấm giá trị âm, gần gấp đôi giới hạn trên; cùng dung lượng và hiệu năng với kiểu có dấu (tr. 127) | [§2.1.1](chapter-06.md#whole-numbers) |
| Display width | Độ rộng hiển thị | Số trong `INT(11)`: chỉ ảnh hưởng hiển thị, không giới hạn giá trị; deprecated từ 8.0.17 (tr. 127) | [§2.1.1](chapter-06.md#display-width) |
| FLOAT / DOUBLE | Số dấu phẩy động | Kiểu xấp xỉ 4 và 8 byte; MySQL tính nội bộ bằng `DOUBLE`; không nên ghi precision (tr. 128) | [§2.1.2](chapter-06.md#real-numbers) |
| DECIMAL | Số thập phân chính xác | Kiểu chính xác cho phần lẻ, ví dụ tiền; tốn chỗ và tính toán hơn dấu phẩy động (tr. 127–128) | [§2.1.2](chapter-06.md#real-numbers) |
| Money as BIGINT | Lưu tiền bằng BIGINT | Nhân số tiền với bội của đơn vị nhỏ nhất (ví dụ 10⁶) và lưu số nguyên, tránh cả sai số lẫn chi phí `DECIMAL` (tr. 128) | [§2.1.2](chapter-06.md#money-bigint) |

### Chuỗi

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| VARCHAR | Chuỗi độ dài thay đổi | Chỉ dùng chỗ cần thiết cộng 1–2 byte độ dài; hàng có thể dài ra khi update (tr. 129) | [§2.1.3](chapter-06.md#varchar-char) |
| CHAR | Chuỗi độ dài cố định | Luôn cấp đủ chỗ cho số ký tự khai báo; bỏ khoảng trắng cuối; hợp với giá trị cùng độ dài như MD5 (tr. 129) | [§2.1.3](chapter-06.md#varchar-char) |
| BINARY / VARBINARY | Chuỗi nhị phân | Lưu byte thay vì ký tự, so sánh theo byte nên đơn giản và nhanh; `BINARY` đệm bằng `\0` (tr. 130) | [§2.1.3](chapter-06.md#varchar-char) |
| Generosity (over-sized VARCHAR) | Khai báo dư độ dài | `VARCHAR(200)` chứa `'hello'` tốn như `VARCHAR(5)` trên đĩa nhưng có thể tốn nhiều bộ nhớ hơn khi sort và dùng bảng tạm (tr. 131) | [§2.1.3](chapter-06.md#generosity) |
| BLOB / TEXT | Đối tượng lớn nhị phân / văn bản | Họ kiểu chuỗi lớn; mỗi giá trị là một đối tượng riêng, InnoDB có thể lưu ngoài hàng; không index hết độ dài (tr. 131) | [§2.1.3](chapter-06.md#blob-text) |
| max_sort_length | Số byte dùng khi sắp xếp | Biến server giới hạn số byte đầu được dùng khi sắp xếp `BLOB`/`TEXT` (tr. 131) | [§2.1.3](chapter-06.md#blob-text) |
| ENUM | Kiểu liệt kê | Một giá trị trong tập chuỗi định sẵn, lưu bằng số vị trí 1–2 byte; sắp theo vị trí; đổi tập giá trị là thay đổi schema (tr. 132–134) | [§2.1.3](chapter-06.md#enum) |
| Lookup table | Bảng tra cứu | Bảng từ điển có khóa chính số nguyên, thay cho giá trị ký tự trong join hay cho ENUM lạm dụng (tr. 134, 145) | [§2.1.3](chapter-06.md#enum) |

### Thời gian và kiểu bit

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| DATETIME | Ngày giờ | Năm 1000–9999, độ chính xác tới micro giây, không phụ thuộc múi giờ; sách ghi 8 byte, 8.0 thực tế 5 byte (tr. 135) | [§2.1.4](chapter-06.md#datetime-timestamp) |
| TIMESTAMP | Dấu thời gian | Số giây từ 1970 UTC, 4 byte, tới 19/01/2038, hiển thị theo múi giờ (tr. 135–136) | [§2.1.4](chapter-06.md#datetime-timestamp) |
| Year 2038 problem | Vấn đề năm 2038 | Giới hạn của số giây kiểu có dấu 32 bit, áp dụng cho `TIMESTAMP` và epoch trong `INT` có dấu (tr. 136) | [§2.1.4](chapter-06.md#date-as-integer) |
| Unix epoch | Mốc thời gian Unix | Số giây từ 01/01/1970 UTC; lưu trong `INT` để tự xử lý thời gian ở code (tr. 136) | [§2.1.4](chapter-06.md#date-as-integer) |
| BIT | Kiểu bit | Một hoặc nhiều bit đúng/sai, tối đa 64; là kiểu chuỗi nên dễ nhầm; nên tránh (tr. 137) | [§2.1.5](chapter-06.md#bit-type) |
| SET | Kiểu tập hợp | Tập bit đóng gói cho nhiều giá trị đúng/sai, có `FIND_IN_SET()` (tr. 137–138) | [§2.1.5](chapter-06.md#set-type) |
| Bitwise operations | Phép toán bit | Dùng số nguyên làm tập bit; đổi ý nghĩa không cần `ALTER TABLE` nhưng truy vấn khó đọc (tr. 137–138) | [§2.1.5](chapter-06.md#set-type) |

### JSON

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| JSON data type | Kiểu dữ liệu JSON | Kiểu native cho phép thao tác từng phần của tài liệu JSON; tốn chỗ hơn và chậm hơn cột thường (tr. 139–142) | [§2.1.6](chapter-06.md#json-data) |
| Generated column | Cột sinh | Cột có giá trị tính từ biểu thức trên cột khác; dùng để trích trường JSON và đánh index (tr. 141) | [§2.1.6](chapter-06.md#generated-column) |
| Virtual generated column | Cột sinh ảo | Cột sinh không lưu trong hàng, tính khi đọc; giá trị được lưu trong index nếu có (tr. 141) | [§2.1.6](chapter-06.md#generated-column) |
| JSON path operator `->` | Toán tử đường dẫn JSON | Trích một phần tài liệu JSON theo đường dẫn như `'$.designation'`; kết quả là giá trị JSON (tr. 141) | [§2.1.6](chapter-06.md#generated-column) |

### Identifier

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Identifier | Định danh | Cách tham chiếu một hàng, thường làm nó duy nhất; thường là một phần hay toàn bộ khóa chính (tr. 142) | [§2.1.7](chapter-06.md#identifiers) |
| AUTO_INCREMENT | Tự tăng | Thuộc tính cột sinh số nguyên mới cho mỗi hàng; cần đủ kích thước để không hết số (tr. 143) | [§2.1.7](chapter-06.md#integer-identifiers) |
| UUID | Định danh duy nhất toàn cục | Giá trị 128 bit; dạng chuỗi tốn chỗ và rải rác; nên lưu `BINARY(16)` (tr. 143–144) | [§2.1.7](chapter-06.md#uuid) |
| Page split | Tách page | Page index đầy khi chèn nên bị tách; với giá trị ngẫu nhiên gây phân mảnh và truy cập đĩa ngẫu nhiên (tr. 143) | [§2.1.7](chapter-06.md#string-identifiers) |
| Locality of reference | Tính cục bộ của tham chiếu | Dữ liệu dùng gần nhau nằm gần nhau, là điều kiện để cache hiệu quả; giá trị ngẫu nhiên phá nó (tr. 143) | [§2.1.7](chapter-06.md#string-identifiers) |
| Hot spot | Điểm nóng | Vùng index nhiều luồng cùng ghi; giá trị giả ngẫu nhiên đôi khi giúp tránh (chú thích 6, tr. 143) | [§2.1.7](chapter-06.md#string-identifiers) |
| Autogenerated schema | Schema tự sinh | Schema do công cụ migration hay ORM tạo, hay dùng `VARCHAR` lớn và kiểu lệch nhau (tr. 144) | [§2.1.7](chapter-06.md#autogenerated-schemas) |
| ORM (object-relational mapping) | Ánh xạ đối tượng – quan hệ | Lớp ánh xạ đối tượng sang bảng; tiện cho developer nhưng thường không tận dụng thế mạnh của database (tr. 144) | [§2.1.7](chapter-06.md#autogenerated-schemas) |
| INET_ATON() / INET_NTOA() | Hàm chuyển địa chỉ IPv4 | Chuyển giữa dạng bốn số chấm và số nguyên không dấu 32 bit, 4 byte thay vì khoảng 16 (tr. 144) | [§2.1.8](chapter-06.md#special-types) |

### Bẫy thiết kế schema của MySQL

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Row buffer | Bộ đệm hàng | Định dạng hàng chép giữa server và storage engine; giải mã tốn CPU theo số cột (tr. 145) | [§2.2.1](chapter-06.md#too-many-columns) |
| Entity-attribute-value (EAV) | Mô hình thực thể – thuộc tính – giá trị | Lưu mỗi thuộc tính thành một hàng; cần nhiều self-join, dễ chạm giới hạn 61 bảng mỗi join (tr. 145) | [§2.2.2](chapter-06.md#too-many-joins) |
| 61-table join limit | Giới hạn 61 bảng mỗi join | Số bảng tối đa trong một join của MySQL; nên giữ khoảng một tá trở xuống (tr. 145) | [§2.2.2](chapter-06.md#too-many-joins) |
| Magic constant | Hằng ma thuật | Một giá trị hợp lệ của kiểu (như −1) dùng để biểu diễn "chưa biết"; thường tệ hơn NULL (tr. 146) | [§2.2.5](chapter-06.md#null-not-invented) |
| Zero date | Ngày toàn số 0 | Giá trị `'0000-00-00 00:00:00'` giả tạo; cấm được bằng `SQL_MODE` (tr. 146) | [§2.2.5](chapter-06.md#null-not-invented) |

### Quản lý schema

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Data store platform | Nền tảng dữ liệu | Cách nhìn coi quản lý schema là dịch vụ tự phục vụ cho các đội, không phải cổng kiểm soát (tr. 147) | [§2.3.1](chapter-06.md#s2-3-1) |
| Schema source control | Quản lý phiên bản schema | Lưu định nghĩa schema mong muốn trong source control; thường là yêu cầu compliance (tr. 147) | [§2.3.1](chapter-06.md#source-control) |
| Skeema | Skeema | Công cụ mã nguồn mở quản lý schema trong version control theo cluster và môi trường; không tự chạy thay đổi ở production (tr. 148) | [§2.3.1](chapter-06.md#skeema) |
| Schema linting | Kiểm tra quy ước schema | Kiểm tra tự động trong pull request: charset, cấm khóa ngoại… (tr. 149, 153) | [§2.3.1](chapter-06.md#skeema) |
| Native DDL / online DDL | DDL native, DDL trực tuyến | Thay đổi schema bằng chính MySQL không chặn ghi, qua thuật toán `INPLACE` hoặc `INSTANT` (tr. 149–150) | [§2.3.1](chapter-06.md#native-ddl) |
| ALGORITHM=INSTANT / INPLACE | Thuật toán tức thì / tại chỗ | Hai thuật toán DDL native sách khuyên dùng (tr. 150); `INSTANT` chỉ đổi metadata, `INPLACE` không chép bảng qua tầng server nhưng có thể rebuild trong InnoDB (Bổ sung ngoài sách) | [§2.3.1](chapter-06.md#ddl-check) |
| pt-online-schema-change | Công cụ đổi schema online của Percona | Chép bảng và dùng trigger để theo dõi thay đổi; ổn định lâu năm, hỗ trợ khóa ngoại có đánh đổi (tr. 150–152) | [§2.3.1](chapter-06.md#pt-osc) |
| gh-ost | Công cụ đổi schema online của GitHub | Chép bảng, đọc binary log dạng row như một replica, không trigger; dừng nếu bảng có khóa ngoại (tr. 151–152) | [§2.3.1](chapter-06.md#gh-ost) |
| Throttling | Điều tiết tốc độ | Kiểm soát tốc độ chép của thay đổi schema để giảm ảnh hưởng; lý do dùng công cụ ngoài (tr. 150) | [§2.3.1](chapter-06.md#native-ddl) |
| CI/CD pipeline for schema changes | Pipeline CI/CD cho thay đổi schema | Repo mỗi cluster, pull request, test tự động, cấu hình an toàn nền, phê duyệt tùy đội (tr. 152–153) | [§2.3.1](chapter-06.md#cicd-pipeline) |
| Baseline configuration | Cấu hình nền | Ngưỡng an toàn mặc định cho công cụ đổi schema online, như số thread đang chạy hay tải hệ thống tối đa (tr. 153) | [§2.3.1](chapter-06.md#cicd-pipeline) |

### Bổ sung ngoài sách (liên quan chương 6)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| explicit_defaults_for_timestamp | — | Mặc định ON từ 8.0.2: cột `TIMESTAMP` không tự lấy giờ hiện tại, cho phép NULL | [§2.1.4](chapter-06.md#datetime-8-0) |
| NULL bitmap | Bitmap NULL | Vùng bit trong mỗi hàng `COMPACT`/`DYNAMIC` đánh dấu cột nào là NULL | [§2.1](chapter-06.md#avoid-null) |
| PAD SPACE / NO PAD collation | Collation đệm khoảng trắng / không đệm | PAD SPACE bỏ qua khoảng trắng cuối khi so sánh và dùng sort key cố định; NO PAD (như `utf8mb4_0900_ai_ci`) cho sort key độ dài thay đổi | [§2.1.3](chapter-06.md#generosity-measured) |
| TempTable / MEMORY (internal temporary table) | Engine bảng tạm trong bộ nhớ | TempTable (mặc định 8.0) lưu `VARCHAR` theo độ dài thật; MEMORY dùng hàng cố định theo độ dài tối đa | [§2.1.3](chapter-06.md#generosity-measured) |
| `->>` (JSON_UNQUOTE) | Trích và bỏ nháy | Như `->` nhưng trả chuỗi không dấu nháy kép | [§2.1.6](chapter-06.md#generated-column) |
| JSON_STORAGE_SIZE() | — | Số byte của tài liệu JSON ở dạng nhị phân lưu trữ | [§2.1.6](chapter-06.md#json-binary) |
| UUID version 1, time_low | UUID phiên bản 1 | UUID dựa trên thời gian mà `UUID()` của MySQL sinh; đoạn đầu quay vòng khoảng 7,2 phút | [§2.1.7](chapter-06.md#uuid-mechanism) |
| UUID_TO_BIN(u, 1) / BIN_TO_UUID(b, 1) | Chuyển UUID sang nhị phân có đổi chỗ | Đổi chỗ phần thời gian để giá trị nhị phân tăng theo thời gian | [§2.1.7](chapter-06.md#uuid-mechanism) |
| Page fill factor | Độ đầy page | Tỷ lệ dữ liệu trên page; khoảng 92% khi chèn tăng dần, 67–69% khi chèn rải rác (E3) | [§2.1.7](chapter-06.md#uuid-measured) |
| innodb_online_alter_log_max_size | — | Giới hạn log tạm ghi DML đồng thời trong lúc ALTER online; vượt thì ALTER thất bại với lỗi 1799 | [§2.3.1](chapter-06.md#ddl-check) |
| Row version (INSTANT DDL) | Phiên bản hàng | Mỗi thay đổi `INSTANT` thêm một phiên bản định dạng hàng; số phiên bản có giới hạn | [§2.3.1](chapter-06.md#ddl-check) |
| INET6_ATON() / INET6_NTOA() | Hàm chuyển địa chỉ IPv6 | Chuyển IPv6 (và IPv4) sang `VARBINARY(16)` và ngược lại | [§2.1.8](chapter-06.md#special-types) |

<a id="chuong-7"></a>

## Chương 7 — Indexing for High Performance

### Cấu trúc index trong InnoDB

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Index (key) | chỉ mục | Cấu trúc dữ liệu mà storage engine dùng để tìm hàng nhanh; MySQL còn gọi là key. Cài ở tầng storage engine nên mỗi engine làm khác nhau | [§1](chapter-07.md#big-picture) |
| B-tree index | chỉ mục cây B | Loại index mặc định: giá trị lưu theo thứ tự, mọi lá cách gốc như nhau; dùng được cho tra bằng, khoảng, tiền tố và sắp xếp | [§2.1.1](chapter-07.md#b-tree) |
| B+tree | cây B+ | Biến thể B-tree mà InnoDB dùng: dữ liệu nằm ở lá, mỗi lá liên kết tới lá kế để duyệt khoảng nhanh | [§2.1.1](chapter-07.md#b-tree) |
| Node page, leaf page | trang nút, trang lá | Trang nút chứa khóa và con trỏ tới trang con; trang lá chứa giá trị cùng con trỏ tới dữ liệu (InnoDB: hàng hoặc PK). Page InnoDB mặc định 16 KB | [§2.1.1](chapter-07.md#b-tree) |
| Adaptive hash index | chỉ mục băm thích ứng | Hash index InnoDB tự dựng trong bộ nhớ, chồng lên B-tree, cho các giá trị được tra rất thường xuyên; chỉ bật hoặc tắt được | [§2.1.1](chapter-07.md#adaptive-hash-index) |
| Full-text index | chỉ mục toàn văn | Index tìm từ khóa trong văn bản cho `MATCH AGAINST`; không thay B-tree cho `WHERE` thường | [§2.1.1](chapter-07.md#full-text) |
| Clustered index | chỉ mục cụm | Cách lưu dữ liệu: hàng nằm ngay trong lá của B-tree; InnoDB cluster theo primary key; mỗi bảng chỉ một | [§2.2.4](chapter-07.md#clustered-index) |
| Index-organized table | bảng tổ chức theo index | Thuật ngữ Oracle cho cùng ý tưởng clustered index | [§2.2.4](chapter-07.md#clustered-index) |
| Hidden primary key | khóa chính ẩn | Khóa InnoDB tự tạo khi bảng không có PK hay unique index `NOT NULL`; bộ đếm dùng chung cho mọi bảng nên gây tranh chấp mutex | [§2.2.4](chapter-07.md#hidden-primary-key) |
| Secondary index | chỉ mục phụ | Index không phải clustered; trong InnoDB lá chứa giá trị PK làm "con trỏ", nên tra hàng cần hai lần duyệt B-tree | [§2.2.4](chapter-07.md#secondary-index) |
| Transaction ID, rollback pointer | mã giao dịch, con trỏ hoàn tác | Hai trường trong mỗi bản ghi lá của clustered index, phục vụ transaction và MVCC | [§2.2.4](chapter-07.md#innodb-layout) |
| Page split | tách trang | Khi khóa mới phải vào một page đã đầy, engine tách page làm hai; làm page thưa và bảng tốn chỗ | [§2.2.4](chapter-07.md#clustered-index) |
| Surrogate key | khóa thay thế | PK không suy ra từ dữ liệu ứng dụng, thường là `AUTO_INCREMENT`, giúp chèn tuần tự | [§2.2.4](chapter-07.md#pk-order) |
| Fill factor | hệ số lấp đầy | Mức đầy tối đa của page khi chèn tuần tự; InnoDB ban đầu lấp 15/16 page để chừa chỗ sửa đổi | [§2.2.4](chapter-07.md#pk-order) |

### Cách truy vấn dùng index

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Leftmost prefix | tiền tố trái nhất | Phần đầu liên tục của index nhiều cột; MySQL chỉ tìm hiệu quả trên phần này và dừng ở điều kiện khoảng đầu tiên | [§2.1.1](chapter-07.md#leftmost-prefix) |
| Match full value, leftmost prefix, column prefix, range | khớp đủ giá trị, khớp tiền tố cột, khớp khoảng | Các kiểu tra cứu B-tree hỗ trợ: cho mọi cột, cho cột đầu, cho phần đầu giá trị của cột, cho một khoảng | [§2.1.1](chapter-07.md#leftmost-prefix) |
| Index-only query | truy vấn chỉ đọc index | Truy vấn trả lời được chỉ từ index, không đọc hàng | [§2.1.1](chapter-07.md#leftmost-prefix) |
| Multicolumn index | chỉ mục nhiều cột | Index trên nhiều cột, sắp theo cột trái nhất trước; khác với nhiều index một cột | [§2.2.2](chapter-07.md#index-merge) |
| Index merge | gộp chỉ mục | Chiến lược dùng nhiều index của một bảng rồi hợp (`OR`) hoặc giao (`AND`) kết quả; thường là dấu hiệu đánh index kém | [§2.2.2](chapter-07.md#index-merge) |
| Covering index | chỉ mục phủ | Index chứa mọi cột truy vấn cần; `EXPLAIN` báo `Using index`; chỉ B-tree làm được | [§2.2.5](chapter-07.md#covering-index) |
| Access type (join type) | kiểu truy cập | Cột `type` của `EXPLAIN` (`ref`, `range`, `index`, `ALL`…); `type: index` là quét toàn index, khác `Using index` | [§2.2.5](chapter-07.md#covering-index) |
| Filesort | sắp xếp bằng thao tác sort | Thao tác sắp xếp khi không dùng được thứ tự index; chỉ xuống đĩa khi không sắp được trong bộ nhớ | [§2.2.6](chapter-07.md#index-sort) |
| Index scan for sort | quét index để sắp xếp | Đọc index theo thứ tự thay cho sắp xếp; rẻ khi index cover truy vấn hoặc có `LIMIT`, có thể đắt hơn quét bảng nếu phải tra từng hàng | [§2.2.6](chapter-07.md#index-sort) |

### Selectivity, thống kê và thiết kế

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Index selectivity | độ chọn lọc của index | Số giá trị phân biệt chia cho số hàng, từ 1/#T tới 1; unique index bằng 1 | [§2.2.1](chapter-07.md#selectivity) |
| Cardinality | số giá trị phân biệt | Số giá trị phân biệt (ước lượng) của index hay tiền tố của nó; hiển thị trong `SHOW INDEX` | [§2.3.2](chapter-07.md#index-statistics) |
| Prefix index | chỉ mục tiền tố | Index chỉ trên vài ký tự đầu của cột; nhỏ hơn nhưng kém chọn lọc, không cover được, không dùng cho `ORDER BY`, `GROUP BY` | [§2.2.1](chapter-07.md#selectivity) |
| Three-star system | hệ ba sao | Cách chấm index của Lahdenmaki và Leach: hàng liên quan nằm cạnh nhau, đúng thứ tự, đủ cột | [§2.1.2](chapter-07.md#three-star) |
| Column order | thứ tự cột | Thứ tự cột trong index nhiều cột; quyết định index có được sao hay không; quy tắc "chọn lọc nhất đặt đầu" chỉ là phụ | [§2.2.3](chapter-07.md#column-order) |
| Special value (outlier) | giá trị đặc biệt | Giá trị có nhiều hàng bất thường (guest, tài khoản hệ thống); có thể làm hỏng hiệu năng dù trung bình tốt | [§2.2.3](chapter-07.md#column-order) |
| Index statistics | thống kê index | Ước lượng phân bố (cardinality) mà optimizer dùng để tính chi phí; InnoDB lấy mẫu vài page | [§2.3.2](chapter-07.md#index-statistics) |
| `ANALYZE TABLE` | lệnh cập nhật thống kê | Lệnh tạo lại index statistics | [§2.3.2](chapter-07.md#index-statistics) |
| `innodb_stats_on_metadata` | lấy mẫu khi đọc metadata | Biến quyết định truy vấn metadata (`SHOW INDEX`…) có kích hoạt lấy mẫu statistics không; mặc định `OFF` trên 8.0 | [§2.3.2](chapter-07.md#index-statistics) |
| Duplicate index | chỉ mục trùng | Index cùng loại, cùng cột, cùng thứ tự với index khác; nên xóa | [§2.2.7](chapter-07.md#redundant-index) |
| Redundant index | chỉ mục thừa | Index là tiền tố của index B-tree khác, như `(A)` khi đã có `(A, B)`; đôi khi cần giữ vì hiệu năng | [§2.2.7](chapter-07.md#redundant-index) |
| Invisible index | chỉ mục vô hình | Index của MySQL 8.0 mà optimizer bỏ qua nhưng vẫn được bảo trì; dùng để thử trước khi xóa | [§2.2.7](chapter-07.md#invisible-index) |
| Unused index | chỉ mục không dùng | Index không truy vấn nào dùng; tìm bằng `sys.schema_unused_indexes` | [§2.2.8](chapter-07.md#unused-index) |

### Bảo trì bảng

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Table corruption | hỏng bảng | Dữ liệu hoặc index bị hỏng do phần cứng, thao tác file ngoài MySQL hay bug; với InnoDB là dấu hiệu vấn đề nghiêm trọng | [§2.3.1](chapter-07.md#corruption) |
| `CHECK TABLE`, `REPAIR TABLE` | kiểm tra bảng, sửa bảng | Kiểm tra lỗi bảng và index; sửa bảng với engine hỗ trợ | [§2.3.1](chapter-07.md#corruption) |
| No-op `ALTER` | `ALTER` không làm gì | `ALTER TABLE t ENGINE=<engine hiện tại>`, dựng lại bảng để sửa hoặc chống phân mảnh | [§2.3.1](chapter-07.md#corruption) |
| `innodb_force_recovery` | phục hồi cưỡng bức | Chế độ phục hồi cưỡng bức của InnoDB, dùng để cứu dữ liệu khi hỏng | [§2.3.1](chapter-07.md#corruption) |
| Index fragmentation | phân mảnh index | Trang lá đầy kém hoặc không tuần tự trên đĩa; làm quét khoảng và quét index chậm | [§2.3.3](chapter-07.md#fragmentation) |
| Row, intra-row, free space fragmentation | phân mảnh hàng, giữa các hàng, chỗ trống | Ba loại phân mảnh dữ liệu: hàng bị chia mảnh; page hay hàng liền nhau logic nhưng không liền trên đĩa; page nhiều chỗ trống | [§2.3.3](chapter-07.md#fragmentation) |
| `OPTIMIZE TABLE` | tối ưu (dựng lại) bảng | Dựng lại bảng để lấp đầy page; với InnoDB là tạo lại bảng rồi `ANALYZE` | [§2.3.3](chapter-07.md#fragmentation) |

### Bổ sung ngoài sách (liên quan chương 7)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Index condition pushdown (ICP) | đẩy điều kiện xuống index | Engine kiểm điều kiện trên các cột còn lại của mục index trước khi đọc hàng (`Using index condition`); không thu hẹp phạm vi quét | [§2.1.1](chapter-07.md#leftmost-prefix) |
| Skip scan | quét nhảy | Từ MySQL 8.0.13: dùng index khi điều kiện không chạm cột đầu, bằng cách nhảy qua từng giá trị của cột đầu; chỉ với truy vấn covering một bảng | [§2.1.1](chapter-07.md#skip-scan) |
| `key_len` | độ dài khóa | Số byte của phần đầu index được dùng; với `range` có thể tính cả cột không thu hẹp khoảng | [§2.1.1](chapter-07.md#leftmost-prefix) |
| Descending index | chỉ mục giảm dần | Từ MySQL 8.0, cột trong index có thể lưu theo chiều giảm, giải trường hợp `ORDER BY` hai chiều | [§2.2.6](chapter-07.md#index-sort) |
| Backward index scan | quét index ngược | Đọc index theo chiều ngược để phục vụ `ORDER BY ... DESC` | [§2.2.6](chapter-07.md#index-sort) |
| Index dive | lặn index | Optimizer đi xuống B-tree để đếm số mục trong một khoảng với hằng số cụ thể; giới hạn bởi `eq_range_index_dive_limit` | [§2.3.2](chapter-07.md#index-statistics) |
| Persistent statistics | thống kê bền vững | Statistics InnoDB lưu trong `mysql.innodb_index_stats`, tự tính lại khi khoảng 10% hàng đổi | [§2.3.2](chapter-07.md#index-statistics) |
| `information_schema_stats_expiry` | thời hạn cache thống kê | Thời gian cache statistics của `INFORMATION_SCHEMA` trong MySQL 8.0 (mặc định 86.400 giây) | [§2.3.2](chapter-07.md#index-statistics) |
| Histogram | biểu đồ tần suất | Thống kê phân bố cho cột không có index trên MySQL 8.0, tạo bằng `ANALYZE TABLE ... UPDATE HISTOGRAM` | [§2.3.2](chapter-07.md#index-statistics) |
| Generated invisible primary key (GIPK) | khóa chính vô hình tự sinh | Từ 8.0.30, server tự thêm PK `AUTO_INCREMENT` vô hình cho bảng thiếu PK nếu bật `sql_generate_invisible_primary_key` | [§2.2.4](chapter-07.md#hidden-primary-key) |
| `UUID_TO_BIN(uuid, 1)` | đổi UUID sang nhị phân có thứ tự | Hàm 8.0 đổi UUID sang 16 byte và đảo phần thời gian lên đầu, cho khóa gần tuần tự | [§2.2.4](chapter-07.md#pk-order) |
| `sys.schema_redundant_indexes` | view liệt kê index thừa | View của sys liệt kê index thừa hay trùng cùng index "trội" | [§2.2.7](chapter-07.md#redundant-index) |

<a id="chuong-8"></a>

## Chương 8 — Query Performance Optimization

### Thời gian phản hồi và truy cập dữ liệu

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Response time | Thời gian phản hồi | Thước đo chính của hiệu năng truy vấn; tổng thời gian của mọi nhiệm vụ con (mạng, CPU, lock, gọi storage engine, I/O) (tr. 191–192) | [§2.1](chapter-08.md#response-time) |
| Service time | Thời gian phục vụ | Phần response time server thực sự xử lý truy vấn (tr. 194) | [§2.2.2](chapter-08.md#examined-too-much) |
| Queue time | Thời gian chờ | Phần response time server không thực thi mà chờ, ví dụ chờ I/O hay row lock (tr. 194) | [§2.2.2](chapter-08.md#examined-too-much) |
| QUBE (quick upper-bound estimate) | Ước lượng cận trên nhanh | Ước lượng response time từ số I/O tuần tự và ngẫu nhiên nhân thời gian phần cứng, theo Lahdenmaki và Leach (tr. 194–195) | [§2.2.2](chapter-08.md#examined-too-much) |
| Rows examined | Số hàng được đọc | Số hàng MySQL truy cập để thực thi truy vấn; ghi trong slow query log (tr. 194–195) | [§2.2.2](chapter-08.md#rows-examined) |
| Rows returned (rows sent) | Số hàng trả về | Số hàng trong result set gửi cho client; tỉ lệ đọc so với trả thường 1:1 tới 10:1 (tr. 195) | [§2.2.2](chapter-08.md#rows-examined) |
| Slow query log | Nhật ký truy vấn chậm | Log ghi response time, rows examined, rows returned của truy vấn chậm; nơi tốt để tìm truy vấn đọc quá nhiều (tr. 194) | [§2.2.2](chapter-08.md#examined-too-much) |
| Access type (access method) | Kiểu truy cập | Cách tìm hàng của một bảng, hiện ở cột `type` của `EXPLAIN`: full table scan, index scan, range, tra unique index, hằng số (tr. 195) | [§2.2.2](chapter-08.md#access-type) |
| Full table scan (`ALL`) | Quét toàn bảng | Đọc mọi hàng của bảng; access type tệ nhất khi điều kiện chọn lọc cao (tr. 195–196) | [§2.2.2](chapter-08.md#access-type) |
| Index scan (`type: index`) | Quét toàn bộ index | Đọc mọi mục của một index; khác với `Extra: Using index` (tr. 197) | [§2.2.2](chapter-08.md#access-type) |
| `ref`, `eq_ref`, `const` | Tra index, tra unique, hằng | Tra index không unique; tra đúng một hàng qua unique index cho mỗi hàng bảng trước; tối đa một hàng, đọc lúc tối ưu (tr. 196, 208–209, 214) | [§2.2.2](chapter-08.md#access-type) |
| Using where | Lọc sau khi đọc hàng | Server đọc hàng từ bảng rồi mới bỏ hàng không khớp WHERE; cách áp WHERE tệ nhất (tr. 196–197) | [§2.2.2](chapter-08.md#where-apply) |
| Using index | Dùng covering index | Lọc trên mục index ở tầng server mà không đọc hàng từ bảng (tr. 197) | [§2.2.2](chapter-08.md#where-apply) |
| SELECT * | Lấy mọi cột | Lấy mọi cột; có thể ngăn covering index, thêm I/O, bộ nhớ, CPU (tr. 193) | [§2.2.1](chapter-08.md#select-star) |

### Tái cấu trúc truy vấn

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Complex query versus many queries | Một truy vấn phức tạp hay nhiều truy vấn đơn | Cân chi phí vòng hỏi đáp với chi phí xử lý bên trong; MySQL trả lời truy vấn nhỏ rất nhanh nhưng vẫn chậm hơn tốc độ duyệt hàng nội bộ (tr. 198–199) | [§2.3.1](chapter-08.md#complex-vs-many) |
| Chopping up a query | Chia nhỏ truy vấn | Chạy cùng truy vấn theo từng lô nhỏ, ví dụ DELETE ... LIMIT 10000, để giảm khóa, log, replication lag (tr. 199–200) | [§2.3.2](chapter-08.md#chopping) |
| Purge job | Job dọn dữ liệu cũ | Tác vụ định kỳ xóa dữ liệu cũ; ví dụ điển hình cần chia lô (tr. 199) | [§2.3.2](chapter-08.md#chopping) |
| Join decomposition | Tách join | Chạy nhiều truy vấn một bảng rồi join trong ứng dụng; lợi cho cache, khóa, phân tán, danh sách IN() (tr. 200) | [§2.3.3](chapter-08.md#join-decomposition) |

### Giao thức và trạng thái

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Half-duplex protocol | Giao thức bán song công | Mỗi lúc chỉ một bên gửi; không flow control, không cắt ngắn message; server đẩy hàng cho client (tr. 202) | [§2.4.1](chapter-08.md#client-server-protocol) |
| `max_allowed_packet` | Kích thước packet tối đa | Giới hạn kích thước một packet; quan trọng vì truy vấn được gửi trong một packet (tr. 202) | [§2.4.1](chapter-08.md#client-server-protocol) |
| Buffered result | Kết quả đệm | Thư viện client nhận hết result set vào bộ nhớ; server xong và nhả tài nguyên sớm (tr. 202–203) | [§2.4.1](chapter-08.md#buffered-result) |
| Unbuffered result (`mysql_use_result`) | Kết quả không đệm | Lấy từng hàng khi cần; ít bộ nhớ client nhưng server giữ lock và tài nguyên lâu hơn (tr. 203–204) | [§2.4.1](chapter-08.md#buffered-result) |
| `SQL_BUFFER_RESULT` | Hint đệm kết quả | Buộc server đưa kết quả vào bảng tạm để nhả lock sớm (tr. 203, 218) | [§2.4.1](chapter-08.md#buffered-result) |
| Query state | Trạng thái truy vấn | Trạng thái của thread trong `SHOW FULL PROCESSLIST`, cho biết "ai đang cầm bóng" (tr. 204) | [§2.4.2](chapter-08.md#query-states) |
| Locked (state) | Trạng thái chờ table lock | Chờ table lock ở tầng server; row lock của InnoDB không làm thread vào trạng thái này (tr. 204) | [§2.4.2](chapter-08.md#query-states) |

### Quá trình tối ưu

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Parser, parse tree | Bộ phân tích cú pháp, cây cú pháp | Tách truy vấn thành token, dựng cây và kiểm tra theo ngữ pháp SQL (tr. 205) | [§2.4.3](chapter-08.md#parser) |
| Preprocessor | Bộ tiền xử lý | Kiểm tra ngữ nghĩa (bảng, cột tồn tại, tên và alias không mơ hồ) và quyền (tr. 205) | [§2.4.3](chapter-08.md#parser) |
| Query execution plan | Kế hoạch thực thi | Cây chỉ thị (không phải bytecode) mà execution engine làm theo (tr. 212, 217) | [§2.4.3](chapter-08.md#execution-plan) |
| Cost-based optimizer | Optimizer dựa trên chi phí | Ước lượng chi phí các plan và chọn plan rẻ nhất; đơn vị gốc là một lần đọc ngẫu nhiên page 4 KB (tr. 205) | [§2.4.3](chapter-08.md#cost-based-optimizer) |
| `Last_query_cost` | Chi phí truy vấn vừa chạy | Biến status chứa chi phí ước lượng của truy vấn gần nhất (tr. 205–206) | [§2.4.3](chapter-08.md#last-query-cost) |
| Cardinality | Số giá trị khác nhau | Số giá trị phân biệt của index; một statistics optimizer dùng (tr. 206, 211) | [§2.4.3](chapter-08.md#statistics) |
| Table and index statistics | Thống kê bảng và index | Số page, cardinality, độ dài hàng và khóa, phân bố khóa; do storage engine cung cấp, không lưu ở tầng server (tr. 211) | [§2.4.3](chapter-08.md#statistics) |
| Static optimization | Tối ưu tĩnh | Tối ưu chỉ dựa trên parse tree, không phụ thuộc giá trị; làm một lần như lúc biên dịch (tr. 206–207) | [§2.4.3](chapter-08.md#static-dynamic) |
| Dynamic optimization | Tối ưu động | Tối ưu phụ thuộc ngữ cảnh (giá trị trong WHERE, số hàng); làm lại mỗi lần chạy (tr. 207) | [§2.4.3](chapter-08.md#static-dynamic) |
| Range checked for each record | Kiểm tra range cho từng hàng | Plan đánh giá lại index cho từng hàng của bảng trước trong join (tr. 207) | [§2.4.3](chapter-08.md#static-dynamic) |
| Early termination | Kết thúc sớm | Dừng truy vấn hay một bước khi đã đủ: LIMIT, `Impossible WHERE`, not-exists (tr. 209–210) | [§2.4.3](chapter-08.md#optimization-types) |
| Select tables optimized away | Bảng bị tối ưu bỏ | MIN, MAX hay COUNT được tính lúc tối ưu và thay bảng bằng hằng (tr. 207–208) | [§2.4.3](chapter-08.md#optimization-types) |
| Equality propagation | Lan truyền điều kiện bằng nhau | Áp điều kiện WHERE của một cột cho các cột bị buộc bằng nó qua JOIN, ON, USING (tr. 210, 220) | [§2.4.3](chapter-08.md#optimization-types) |
| Not exists (Distinct/not-exists) | Tối ưu not-exists | Trong LEFT JOIN ... IS NULL, dừng xử lý một hàng ngay khi tìm thấy một hàng khớp (tr. 210) | [§2.4.3](chapter-08.md#anti-join) |
| Index hint, optimizer hint | Gợi ý cho optimizer | Cách can thiệp khi biết optimizer sai và biết vì sao (tr. 211) | [§2.4.3](chapter-08.md#optimization-types) |

### Join và sắp xếp

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Join (theo nghĩa của MySQL) | Join | MySQL coi mọi truy vấn là một join, kể cả SELECT một bảng và subquery (tr. 212) | [§2.4.3](chapter-08.md#join-execution) |
| Nested-loop join | Join vòng lặp lồng | Với mỗi hàng bảng trước, tìm hàng khớp ở bảng sau; quay lui khi hết (tr. 212) | [§2.4.3](chapter-08.md#nested-loop) |
| Block nested-loop join | Join vòng lặp lồng theo khối | Thuật toán join không index trước 8.0.20, đã bị hash join thay thế (tr. 212) | [§2.4.3](chapter-08.md#hash-join) |
| Hash join | Join băm | Dựng bảng băm từ một đầu vào rồi dò bằng đầu vào kia; thay block nested-loop từ 8.0.20 (tr. 212) | [§2.4.3](chapter-08.md#hash-join) |
| Left-deep tree | Cây lệch trái | Dạng plan của MySQL: bắt đầu từ một bảng rồi join lần lượt từng bảng (tr. 213) | [§2.4.3](chapter-08.md#execution-plan) |
| Balanced tree | Cây cân bằng | Dạng plan join từng cặp bảng rồi join các kết quả; MySQL không dùng (tr. 213) | [§2.4.3](chapter-08.md#execution-plan) |
| Join optimizer | Bộ tối ưu join | Phần quan trọng nhất của optimizer, chọn thứ tự join rẻ nhất (tr. 213–216) | [§2.4.3](chapter-08.md#join-optimizer) |
| Driver table | Bảng dẫn | Bảng được đọc trước để dò vào bảng sau (cách nói của người dùng Oracle) (tr. 214) | [§2.4.3](chapter-08.md#join-optimizer) |
| `STRAIGHT_JOIN` | Ép thứ tự join | Buộc join theo thứ tự viết trong truy vấn; hiếm khi cần (tr. 214–216) | [§2.4.3](chapter-08.md#join-optimizer) |
| Search space | Không gian tìm kiếm | Tập mọi plan có thể; n bảng có n giai thừa thứ tự join (tr. 216) | [§2.4.3](chapter-08.md#search-space) |
| Greedy search, `optimizer_search_depth` | Tìm kiếm tham lam, độ sâu tìm kiếm | Lối tắt khi số bảng vượt `optimizer_search_depth` (tr. 216) | [§2.4.3](chapter-08.md#search-space) |
| Filesort | Sắp xếp bằng filesort | Mọi lần MySQL tự sắp xếp khi không dùng được index, trong bộ nhớ hay trên đĩa (tr. 216) | [§2.4.3](chapter-08.md#filesort) |
| Sort buffer | Bộ đệm sắp xếp | Vùng nhớ cho filesort; không vừa thì sắp theo chunk rồi merge (tr. 216) | [§2.4.3](chapter-08.md#filesort) |
| Two-pass filesort | Filesort hai lượt | Sắp con trỏ hàng và cột ORDER BY rồi đọc lại hàng; nhiều random I/O (tr. 217) | [§2.4.3](chapter-08.md#filesort) |
| Single-pass filesort | Filesort một lượt | Sắp mọi cột cần; tránh đọc hàng hai lần nhưng tốn chỗ hơn (tr. 217) | [§2.4.3](chapter-08.md#filesort) |
| Using temporary; Using filesort | Bảng tạm và filesort | Khi ORDER BY không chỉ trên bảng đầu của join: đổ kết quả join vào bảng tạm rồi sắp (tr. 217) | [§2.4.3](chapter-08.md#filesort) |

### Thực thi và trả kết quả

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Query execution engine | Bộ thực thi truy vấn | Làm theo plan, gọi storage engine qua handler API (tr. 217–218) | [§2.4.4](chapter-08.md#execution-engine) |
| Handler API | Giao diện handler | Giao diện storage engine; khoảng một tá thao tác cơ bản như đọc mục đầu, đọc mục kế của index (tr. 218) | [§2.4.4](chapter-08.md#execution-engine) |
| Handler instance | Thể hiện handler | Mỗi lần một bảng xuất hiện trong truy vấn có một instance, tạo từ giai đoạn tối ưu (tr. 218) | [§2.4.4](chapter-08.md#execution-engine) |
| Incremental result sending | Gửi kết quả dần | Server gửi mỗi hàng ngay khi tạo xong, mỗi hàng một packet (tr. 218) | [§2.4.5](chapter-08.md#returning-results) |

### Giới hạn của optimizer và mẫu truy vấn

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Push down (UNION) | Đẩy điều kiện xuống | Chép WHERE, LIMIT, ORDER BY từ ngoài vào từng nhánh UNION; MySQL không luôn tự làm (tr. 219, 225–226) | [§2.5.1](chapter-08.md#union-limitations) |
| Parallel execution | Thực thi song song | Chạy một truy vấn trên nhiều CPU; MySQL không có (tr. 220) | [§2.5.3](chapter-08.md#parallel-execution) |
| ERROR 1093 | Lỗi 1093 | Không được đọc bảng đích của UPDATE trong cùng câu; lách bằng derived table được materialize (tr. 220–221) | [§2.5.4](chapter-08.md#select-update-same-table) |
| `COUNT(*)` | Đếm hàng | Dạng đặc biệt đếm hàng, không bung `*` thành cột (tr. 221) | [§2.6.1](chapter-08.md#count) |
| `COUNT(expr)` | Đếm giá trị | Đếm số lần biểu thức khác NULL (tr. 221) | [§2.6.1](chapter-08.md#count) |
| Super aggregation, `WITH ROLLUP` | Gộp cấp trên | Thêm các hàng tổng của nhóm; có thể đổi cách gộp sang filesort hay bảng tạm (tr. 223) | [§2.6.3](chapter-08.md#rollup) |
| OFFSET | Độ dời | Số hàng bỏ qua; server vẫn sinh rồi vứt chúng (tr. 224) | [§2.6.4](chapter-08.md#limit-offset) |
| Deferred join | Join trì hoãn | Làm OFFSET trên covering index rồi mới join lấy cột đầy đủ (tr. 224) | [§2.6.4](chapter-08.md#deferred-join) |
| Positional query | Truy vấn theo vị trí | Dùng cột vị trí tính trước và có index để thay OFFSET (tr. 224) | [§2.6.4](chapter-08.md#limit-offset) |
| Cursor-style pagination | Phân trang nhớ vị trí | Nhớ khóa của hàng cuối và bắt đầu trang sau từ đó (tr. 224–225) | [§2.6.4](chapter-08.md#keyset) |
| `SQL_CALC_FOUND_ROWS` | Hint đếm tổng số hàng | Bắt server sinh hết kết quả để biết tổng số hàng; rất tốn (tr. 225) | [§2.6.5](chapter-08.md#sql-calc-found-rows) |
| UNION, UNION ALL | Hợp có loại trùng, hợp giữ trùng | UNION thêm bước loại trùng trên cả hàng trong bảng tạm; dùng UNION ALL khi không cần loại trùng (tr. 225–226) | [§2.6.6](chapter-08.md#union) |

### Bổ sung ngoài sách (liên quan chương 8)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Logical query processing | Xử lý truy vấn logic | Ngữ nghĩa của câu SQL: kết quả phải là gì, theo thứ tự FROM → WHERE → GROUP BY → ... → LIMIT | [§1](chapter-08.md#logical-physical) |
| Physical execution | Thực thi vật lý | Cách MySQL thật sự tạo kết quả: access path, thứ tự và thuật toán join, sort, bảng tạm | [§1](chapter-08.md#logical-physical) |
| Access path | Đường truy cập | Cách đọc một bảng: quét bảng, quét index, range, tra index | [§1](chapter-08.md#logical-physical) |
| Index Condition Pushdown (ICP) | Đẩy điều kiện xuống index | Từ 5.6, engine kiểm tra phần điều kiện trên cột index trước khi đọc hàng; `Using index condition` | [§2.2.2](chapter-08.md#where-apply) |
| `EXPLAIN FORMAT=TREE` | EXPLAIN dạng cây | Từ 8.0.16, in cây iterator kèm chi phí từng toán tử | [§2.4.3](chapter-08.md#execution-plan) |
| `EXPLAIN ANALYZE` | EXPLAIN có chạy thật | Từ 8.0.18, chạy truy vấn và in `actual time`, `rows`, `loops` của từng toán tử | [§2.4.3](chapter-08.md#execution-plan) |
| Iterator | Toán tử lặp | Đơn vị của execution engine 8.0; mỗi nút trong `FORMAT=TREE` kéo hàng từ nút con | [§2.4.4](chapter-08.md#execution-engine) |
| `Handler_read_*` | Bộ đếm handler | Biến status đếm từng loại lời gọi handler (read_key, read_next, read_rnd_next...) | [§2.4.4](chapter-08.md#execution-engine) |
| Cost model constants | Hằng số chi phí | `mysql.server_cost`, `mysql.engine_cost`; ví dụ `io_block_read_cost` 1, `memory_block_read_cost` 0,25 | [§2.4.3](chapter-08.md#cost-based-optimizer) |
| Optimizer trace | Vết optimizer | `information_schema.OPTIMIZER_TRACE`: lý do chọn plan, chế độ sort | [§2.4.3](chapter-08.md#filesort) |
| Priority queue (filesort) | Hàng đợi ưu tiên | Với LIMIT nhỏ, filesort chỉ giữ N hàng tốt nhất trong bộ nhớ | [§2.4.3](chapter-08.md#filesort) |
| Semijoin, antijoin | Nửa join, phản join | Chiến lược thực thi subquery IN, EXISTS và NOT EXISTS; `FirstMatch`, `Nested loop antijoin` | [§2.4.3](chapter-08.md#anti-join) |
| Histogram | Biểu đồ tần suất | Statistics phân bố cho cột không có index, tạo bằng `ANALYZE TABLE ... UPDATE HISTOGRAM` (8.0) | [§2.4.3](chapter-08.md#statistics) |
| `Select_range_check` | Biến đếm range check | Tăng khi plan "Range checked for each record" chạy; chú thích 6 của sách ghi nhầm là `Select_full_range_join` | [§2.4.3](chapter-08.md#static-dynamic) |
| Derived condition pushdown | Đẩy điều kiện vào derived table | Điều kiện ngoài được đẩy vào derived table, kể cả derived table chứa UNION ở các bản 8.0 gần đây | [§2.5.1](chapter-08.md#union-limitations) |
| `eq_range_index_dive_limit` | Ngưỡng lặn index | Từ ngưỡng này (mặc định 200) danh sách IN() dùng statistics thay vì đếm trong index | [§2.5.2](chapter-08.md#equality-propagation) |
| `innodb_parallel_read_threads` | Số thread đọc song song | Từ 8.0.14, InnoDB đọc clustered index song song cho `COUNT(*)` không WHERE và `CHECK TABLE` | [§2.5.3](chapter-08.md#parallel-execution) |
| Keyset pagination (seek method) | Phân trang theo khóa | Tên gọi phổ biến của kỹ thuật nhớ khóa hàng cuối; cần thứ tự duy nhất và ổn định | [§2.6.4](chapter-08.md#keyset) |
| N+1 query | Mẫu N+1 truy vấn | Mỗi phần tử danh sách sinh thêm một truy vấn; trường hợp của "quá nhiều truy vấn" | [§2.3.1](chapter-08.md#complex-vs-many) |
| Halloween problem | Bài toán Halloween | Câu lệnh vừa đọc vừa sửa cùng bảng có thể thấy hàng do chính nó sửa; một cách giải thích cho giới hạn kiểu lỗi 1093 | [§2.5.4](chapter-08.md#select-update-same-table) |

<a id="chuong-9"></a>

## Chương 9 — Replication

### Cơ chế cơ bản

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| replication | nhân bản dữ liệu | Giữ dữ liệu đồng bộ giữa các instance: source ghi event vào log, replica đọc và phát lại (tr. 227) | [§2.1](chapter-09.md#s2-1) |
| source / replica | máy nguồn / bản sao | Thuật ngữ mới thay cho master / slave; sách dùng thống nhất (tr. 227) | [§1](chapter-09.md#terminology) |
| scale-out | mở rộng theo chiều ngang | Thêm server (replica) thay vì làm một server mạnh hơn; replication là nền của kiến trúc này (tr. 227) | [§1](chapter-09.md#big-picture) |
| asynchronous replication | replication bất đồng bộ | Source không chờ replica; replica không được bảo đảm cập nhật ở thời điểm nào (tr. 227–228) | [§2.1](chapter-09.md#async-replication) |
| replica lag | độ trễ của replica | Khoảng cách giữa thời gian thực và trạng thái dữ liệu trên replica; không có cận trên (tr. 228) | [§2.1](chapter-09.md#async-replication) |
| binary log (binlog) | nhật ký nhị phân | Log trên source chứa các binary log event làm thay đổi dữ liệu hay cấu trúc (tr. 229) | [§2.1.1](chapter-09.md#how-replication-works) |
| binary log event | event của binlog | Một đơn vị thay đổi ghi trong binlog, được replica chép và phát lại (tr. 229) | [§2.1.1](chapter-09.md#how-replication-works) |
| relay log | log chuyển tiếp | Bản sao cục bộ trên replica của các event lấy từ binlog của source (tr. 229) | [§2.1.1](chapter-09.md#how-replication-works) |
| I/O thread | thread nhận | Thread của replica đọc event từ source và ghi vào relay log; tài liệu 8.0 gọi là receiver (tr. 229) | [§2.1.1](chapter-09.md#how-replication-works) |
| SQL thread | thread áp dụng | Thread của replica đọc relay log và phát lại thay đổi; tài liệu 8.0 gọi là applier (tr. 229) | [§2.1.1](chapter-09.md#how-replication-works) |

### Định dạng log và định danh transaction

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `binlog_format` | định dạng binlog | Tham số chọn statement, row hay mixed (tr. 230) | [§2.2.1](chapter-09.md#binlog-format) |
| statement-based replication | replication theo câu lệnh | Ghi câu SQL đã chạy; replica thực thi lại; gọn nhưng lỗi với truy vấn không tất định (tr. 230) | [§2.2.1](chapter-09.md#binlog-format) |
| row-based replication | replication theo hàng | Ghi từng hàng đã đổi ra sao; tất định nhưng binlog lớn; sách khuyên dùng (tr. 230–231) | [§2.2.1](chapter-09.md#binlog-format) |
| mixed | chế độ trộn | Mặc định statement, chuyển sang row khi cần; tác giả không thích vì khó dự đoán (tr. 231) | [§2.2.1](chapter-09.md#binlog-format) |
| nondeterministic query | truy vấn không tất định | Câu lệnh mà kết quả phụ thuộc thứ tự hàng hay yếu tố khác, ví dụ `DELETE ... LIMIT` không `ORDER BY` (tr. 230, 250) | [§2.2.1](chapter-09.md#binlog-format) |
| GTID (global transaction identifier) | định danh transaction toàn cục | `server_uuid` ghép số tăng dần, gán cho mỗi transaction source commit (tr. 231) | [§2.2.2](chapter-09.md#gtid) |
| `server_uuid` | UUID của server | Sinh lần khởi động đầu nếu không có `auto.cnf`; khác `server_id` (chú thích 2, tr. 231) | [§2.2.2](chapter-09.md#gtid) |
| `server_id` | mã server | Giá trị người quản trị đặt; phải duy nhất trong topology (tr. 231, 251–252) | [§2.6.2](chapter-09.md#server-id) |
| `SOURCE_AUTO_POSITION` | tự định vị | Tùy chọn của `CHANGE REPLICATION SOURCE TO` để replica gửi tập GTID đã có thay vì file và vị trí (chú thích 3, tr. 232) | [§2.2.2](chapter-09.md#gtid) |

### Crash safety và tính năng nâng cao

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| crash-safe replication | replication an toàn khi crash | Bộ thiết lập giúp replication không hỏng sau crash (tr. 232–233) | [§2.2.3](chapter-09.md#crash-safe) |
| `sync_binlog` | đồng bộ binlog | Bằng 1 thì đồng bộ binlog xuống đĩa trước mỗi transaction (tr. 233) | [§2.2.3](chapter-09.md#crash-safe) |
| `relay_log_info_repository` | nơi lưu vị trí applier | `TABLE` lưu vị trí trong bảng InnoDB, cập nhật cùng transaction (tr. 233) | [§2.2.3](chapter-09.md#crash-safe) |
| `relay_log_recovery` | phục hồi relay log | Khi crash thì bỏ relay log cục bộ và lấy lại từ source (tr. 233) | [§2.2.3](chapter-09.md#crash-safe) |
| delayed replication | replication trễ có chủ đích | Replica luôn chậm một khoảng cố định (`SOURCE_DELAY`) để cứu lỗi logic (tr. 233) | [§2.2.4](chapter-09.md#delayed-replication) |
| multithreaded replication | replication đa luồng | Nhiều SQL applier thread áp dụng relay log song song (tr. 234) | [§2.2.5](chapter-09.md#multithreaded-replication) |
| `replica_parallel_workers` | số worker áp dụng | Số applier thread; khác 0 là bật đa luồng (tr. 235) | [§2.2.5](chapter-09.md#multithreaded-replication) |
| `replica_parallel_type` (DATABASE, LOGICAL_CLOCK) | kiểu song song | Song song theo database, hay theo nhóm group commit trong cùng database (tr. 234–235) | [§2.2.5](chapter-09.md#multithreaded-replication) |
| binary log group commit | commit theo nhóm của binlog | Gom nhiều transaction vào một lần `fsync()` binlog; ví dụ chiếc phà (tr. 234–235) | [§2.2.5](chapter-09.md#group-commit) |
| coordinator thread | thread điều phối | Phân việc cho các worker; gây overhead (tr. 235) | [§2.2.5](chapter-09.md#multithreaded-replication) |
| `replica_preserve_commit_order` | giữ thứ tự commit | Bắt worker commit theo thứ tự của source để tránh "gaps" (tr. 235) | [§2.2.5](chapter-09.md#multithreaded-replication) |
| semisynchronous replication | replication bán đồng bộ | Commit chờ ít nhất một replica xác nhận đã ghi vào relay log; hết timeout thì rơi về bất đồng bộ (tr. 237) | [§2.2.6](chapter-09.md#semisync) |
| replication filter | bộ lọc replication | `binlog_do_db`, `binlog_ignore_db` trên source; `replicate_*` trên replica (tr. 237) | [§2.2.7](chapter-09.md#replication-filters) |
| default database | database mặc định | Database chọn bằng `USE`; `*_do_db`, `*_ignore_db` lọc theo nó, không theo database của đối tượng (tr. 238) | [§2.2.7](chapter-09.md#replication-filters) |
| `SQL_LOG_BIN` | bật/tắt ghi binlog cho session | `SET SQL_LOG_BIN=0` không ghi câu lệnh vào binlog; có rủi ro riêng (tr. 238, 252) | [§2.2.7](chapter-09.md#replication-filters) |

### Failover

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| failover / promoting a replica | chuyển đổi dự phòng / nâng replica | Hạ source khỏi vai nhận ghi và nâng một replica lên làm source; sách coi là đồng nghĩa (tr. 239) | [§2.3](chapter-09.md#failover) |
| planned (controlled) promotion | nâng cấp có kế hoạch | Failover chủ động khi bảo trì, theo bảy bước (tr. 239–240) | [§2.3.1](chapter-09.md#planned-promotion) |
| unplanned promotion | nâng cấp ngoài kế hoạch | Failover khi source hỏng; chọn đích theo dữ liệu đã replicate (tr. 240) | [§2.3.2](chapter-09.md#unplanned-promotion) |
| `read_only` | chỉ đọc | Chặn ghi của user không có quyền `SUPER` (tr. 250) | [§2.5.3](chapter-09.md#super-read-only) |
| `super_read_only` | chỉ đọc tuyệt đối | Chỉ cho replication ghi; bật nó thì `read_only` tự bật (chú thích 6, tr. 240; tr. 250) | [§2.5.3](chapter-09.md#super-read-only) |
| target | đích failover | Replica được chọn để promote, thường là cái chắc có đủ dữ liệu (tr. 240) | [§2.3.1](chapter-09.md#planned-promotion) |

### Topology

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| active/passive | chủ động/thụ động | Mọi đọc và ghi vào một source; replica passive chỉ chờ failover (tr. 241) | [§2.4.1](chapter-09.md#active-passive) |
| n+1, n+2 redundancy | dự phòng n+1, n+2 | Số server dư để failover; n+2 (ít nhất ba) cho phần cứng vật lý (tr. 242) | [§2.4.1](chapter-09.md#active-passive) |
| read pool | nhóm replica phục vụ đọc | Nhiều replica nhận đọc chịu được dữ liệu cũ (tr. 242) | [§2.4.2](chapter-09.md#read-pool) |
| traffic weighting | phân tải theo trọng số | Gửi traffic tỉ lệ với năng lực node (tr. 243) | [§2.4.2](chapter-09.md#read-pool) |
| depool / repool | rút / đưa lại node | Rút node khỏi pool khi tụt hay bảo trì rồi đưa lại (tr. 243–244) | [§2.4.2](chapter-09.md#read-pool) |
| stale read | đọc dữ liệu cũ | Đọc từ replica chưa nhận lần ghi mới nhất (tr. 243) | [§2.4.2](chapter-09.md#read-pool) |
| dual source (bidirectional) replication, cosource | replication hai chiều, cặp đồng source | Hai server là source và replica của nhau; không khuyến khích (tr. 244) | [§2.4.3](chapter-09.md#dual-source) |
| buffer pool churn | xáo trộn buffer pool | Buffer pool bị thay dữ liệu hàng loạt khi node nhận working set mới (tr. 244) | [§2.4.3](chapter-09.md#dual-source) |
| ring (circular) replication | replication vòng | Mỗi server là replica của server trước và source của server sau; không có ưu điểm (tr. 246) | [§2.4.3](chapter-09.md#ring-replication) |
| multisource replication | replication nhiều nguồn | Một replica nhận từ nhiều source qua các replication channel (tr. 246–247) | [§2.4.3](chapter-09.md#multisource) |
| replication channel | kênh replication | Một luồng replication riêng tới một source trên cùng replica (tr. 247) | [§2.4.3](chapter-09.md#multisource) |

### Quản trị và sự cố

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `SHOW REPLICA STATUS` | xem trạng thái replica | Lệnh xem thread, lỗi, GTID, độ trễ của replica (tr. 248) | [§2.5.2](chapter-09.md#replication-lag) |
| `Seconds_Behind_Source` | số giây tụt sau source | Ước lượng lag từ timestamp event; không đáng tin trong nhiều trường hợp (tr. 248–249) | [§2.5.2](chapter-09.md#replication-lag) |
| heartbeat record | bản ghi nhịp tim | Timestamp cập nhật mỗi giây trên source; lag = giờ replica trừ giá trị đó (tr. 249) | [§2.5.2](chapter-09.md#heartbeat) |
| `pt-heartbeat` | — | Công cụ heartbeat phổ biến nhất, thuộc Percona Toolkit (tr. 249) | [§2.5.2](chapter-09.md#heartbeat) |
| replica consistency | tính nhất quán của replica | Replica có đúng dữ liệu của source, trừ độ trễ (tr. 249–250) | [§2.5.3](chapter-09.md#replica-consistency) |
| rebuild a replica | dựng lại replica | Cách xử lý sách khuyên khi có lỗi replication hay binlog hỏng (tr. 250–251) | [§2.5.3](chapter-09.md#replica-consistency) |
| nonunique / undefined server ID | server ID trùng / chưa đặt | Lỗi cấu hình làm replica "đánh nhau" hoặc không khởi động được (tr. 251–252) | [§2.6.2](chapter-09.md#server-id) |
| `max_allowed_packet` | kích thước gói tối đa | Lệch giữa source và replica gây vòng lỗi hay hỏng relay log (tr. 254) | [§2.6.7](chapter-09.md#s2-6-7) |
| `log_replica_updates` | ghi binlog cho thay đổi replicate | Replica ghi lại thay đổi nhận được vào binlog của nó; tốn đĩa (tr. 254) | [§2.6.8](chapter-09.md#s2-6-8) |
| excessive replication lag | lag quá lớn | Xử lý bằng đa luồng, sharding hay tạm giảm durability (tr. 252–253) | [§2.6.6](chapter-09.md#excessive-lag) |

### Bổ sung ngoài sách (liên quan chương 9)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| errant transaction | transaction lạ | Transaction chạy trực tiếp trên replica, mang GTID với `server_uuid` của replica; lan ra khi replica được promote | [§2.3.1](chapter-09.md#errant-transaction) |
| `gtid_executed`, `GTID_SUBTRACT`, `GTID_SUBSET` | tập GTID đã chạy, phép trừ và kiểm tập con | Công cụ so sánh hai server trước khi failover | [§2.2.2](chapter-09.md#gtid) |
| `WAIT_FOR_EXECUTED_GTID_SET` | chờ tập GTID | Hàm chờ tới khi server đã thực thi một tập GTID | [§3](chapter-09.md#th1) |
| `START REPLICA UNTIL SQL_BEFORE_GTIDS` | chạy tới trước GTID | Cho SQL thread dừng ngay trước một transaction; dùng với delayed replica | [§2.2.4](chapter-09.md#delayed-replication) |
| `binlog_row_image` (FULL, MINIMAL) | ảnh hàng trong binlog | Ghi mọi cột hay chỉ khóa và cột thay đổi ở định dạng ROW | [§2.2.1](chapter-09.md#binlog-format) |
| `last_committed`, `sequence_number` | mốc phụ thuộc trong event GTID | Transaction cùng `last_committed` có thể áp dụng song song với `LOGICAL_CLOCK` | [§2.2.5](chapter-09.md#group-commit) |
| WRITESET dependency tracking | theo dõi phụ thuộc theo tập ghi | Coi hai transaction độc lập nếu không đụng cùng hàng; biến điều khiển đã deprecated trên 8.0.42 | [§2.2.5](chapter-09.md#multithreaded-replication) |
| `AFTER_SYNC` | điểm chờ sau đồng bộ | Mặc định của semisync 8.0: chờ ACK sau khi ghi binlog, trước khi commit trong engine | [§2.2.6](chapter-09.md#semisync) |
| receiver / applier / dump thread | thread nhận / áp dụng / gửi | Tên trong tài liệu 8.0 cho I/O thread, SQL thread và thread trên source phục vụ mỗi replica | [§2.1.1](chapter-09.md#how-replication-works) |
| `relay_log_space_limit` | giới hạn dung lượng relay log | Biến thực trên 8.0.42 (sách viết `relay_log_space`); mặc định 0 là không giới hạn | [§2.6.8](chapter-09.md#s2-6-8) |

<a id="chuong-10"></a>

## Chương 10 — Backup and Recovery

### Thuật ngữ nền

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Backup | Sao lưu, bản sao lưu | Bản sao dữ liệu (và những thứ cần để dựng lại server) ở một thời điểm, dùng được để phục hồi; phải được thử phục hồi mới tính là tốt | [§2.0](chapter-10.md#pham-vi) |
| Restore | Khôi phục dữ liệu | Lấy dữ liệu từ backup rồi nạp vào MySQL hoặc đặt file vào chỗ MySQL cần | [§2.0](chapter-10.md#restore-recover) |
| Recovery | Phục hồi | Toàn bộ quá trình cứu hệ thống sau sự cố: restore cộng khởi động lại, đổi cấu hình, phát lại binlog, kiểm tra, làm ấm cache | [§2.0](chapter-10.md#restore-recover) |
| Crash recovery | Phục hồi sau crash | Storage engine đối chiếu datafile với log để chỉ giữ thay đổi đã commit và áp lại thay đổi chưa vào datafile; không cứu được lệnh sai đã commit như `DROP TABLE` | [§2.0](chapter-10.md#crash-recovery) |
| Hot / warm / cold backup | Backup nóng, ấm, lạnh | Cách gọi mức tác động của backup lên server; nghĩa không thống nhất nên sách tránh dùng | [§2.0](chapter-10.md#pham-vi) |
| Raw backup (physical backup) | Backup file thô (backup vật lý) | Bản sao các file trên filesystem | [§2.0](chapter-10.md#raw-logical-def) |
| Logical backup (dump) | Backup logic, bản dump | Dữ liệu ở dạng MySQL hiểu được: câu SQL hay văn bản phân tách | [§2.0](chapter-10.md#raw-logical-def) |
| Disaster recovery | Phục hồi sau thảm họa | Việc phải làm khi phần cứng hỏng, bug làm hỏng dữ liệu, server hay dữ liệu không dùng được | [§2.1](chapter-10.md#why-backups) |

### Yêu cầu phục hồi

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Recovery point objective (RPO) | Mục tiêu điểm phục hồi | Lượng dữ liệu chấp nhận mất khi sự cố | [§2.2](chapter-10.md#rpo-rto) |
| Recovery time objective (RTO) | Mục tiêu thời gian phục hồi | Thời gian chấp nhận chờ để lấy lại dữ liệu và dịch vụ | [§2.2](chapter-10.md#rpo-rto) |
| Point-in-time recovery (PITR) | Phục hồi về một thời điểm | Restore backup rồi phát lại binary log tới thời điểm mong muốn, thường là ngay trước sự cố | [§2.3](chapter-10.md#pitr) |
| Soft / hard PITR | PITR "mềm" / "cứng" | Mềm: dựng lại "đủ gần" lúc sự cố; cứng: không bao giờ mất transaction đã commit, cần kỹ thuật như binlog trên SAN riêng hay DRBD | [§2.3](chapter-10.md#soft-hard-pitr) |
| DRBD (Distributed Replicated Block Device) | Thiết bị khối sao chép qua mạng | Sao chép đĩa ở tầng block sang máy khác; sách nêu như cách đạt PITR "cứng" | [§2.3](chapter-10.md#soft-hard-pitr) |
| "My replica is my backup" (Myth #1) | Ngộ nhận "replica là backup" | Replica và RAID sao chép trung thành cả lệnh phá hoại nên không qua phép thử `DROP DATABASE` | [§2.2](chapter-10.md#myth-replica) |
| DROP TABLE test | Phép thử `DROP TABLE` | Câu hỏi kiểm tra: cơ chế này có lấy lại được dữ liệu sau một lệnh xóa nhầm không? | [§2.8](chapter-10.md#summary) |

### Thiết kế giải pháp backup

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Online / offline backup | Backup khi server chạy / khi đã tắt | Offline dễ và an toàn nhất nhưng đắt về downtime; online có thể vẫn gián đoạn tùy yêu cầu nhất quán | [§2.3.1](chapter-10.md#online-offline) |
| Backup time | Thời gian backup | Thời gian tạo backup và chép tới đích | [§2.3.1](chapter-10.md#backup-time-load) |
| Backup load | Tải do backup | Mức backup làm giảm hiệu năng server; đánh đổi chính với backup time | [§2.3.1](chapter-10.md#backup-time-load) |
| Recovery time | Thời gian phục hồi | Thời gian chép backup về, restore, phát lại binlog | [§2.3.1](chapter-10.md#backup-time-load) |
| Throttling | Giới hạn tốc độ | Hạ tốc độ chép hay nén (`ionice`, `nice`, tùy chọn của công cụ, `pv --rate-limit`) để giữ backup load thấp | [§2.3.1](chapter-10.md#backup-time-load) |
| Backup generation | Thế hệ backup | Mỗi bản full backup cùng phần phụ thuộc; nên giữ nhiều thế hệ | [§2.3](chapter-10.md#khuyen-nghi) |
| Full backup | Backup đầy đủ | Bản chứa toàn bộ dữ liệu; nên làm ít nhất hằng tuần | [§2.3.4](chapter-10.md#incremental-differential) |
| Differential backup | Backup vi sai | Mọi thứ đổi từ lần full backup gần nhất | [§2.3.4](chapter-10.md#incremental-differential) |
| Incremental backup | Backup gia tăng | Mọi thứ đổi từ lần backup gần nhất bất kỳ loại nào; chuỗi phụ thuộc nhau | [§2.3.4](chapter-10.md#incremental-differential) |
| Partial backup | Backup một phần | Bản không chứa toàn bộ dữ liệu (differential, incremental) | [§2.3.4](chapter-10.md#incremental-differential) |
| Deduplication | Khử trùng lặp | Đích lưu chỉ giữ một bản của khối dữ liệu trùng (ví dụ ZFS filer) | [§2.3.4](chapter-10.md#incremental-differential) |
| Delayed replication | Replication trễ | Replica cố ý áp thay đổi trễ một khoảng để kịp chặn lệnh sai trước khi nó được áp | [§2.3.5](chapter-10.md#delayed-replica) |
| `super_read_only` | Chỉ đọc kể cả với tài khoản đặc quyền | Chỉ replication mới ghi được vào replica; cách phòng lệch dữ liệu trên replica dùng để backup | [§2.3.5](chapter-10.md#replica-drift) |
| `pt-table-checksum` | Công cụ so checksum bảng | Công cụ Percona Toolkit phát hiện replica lệch dữ liệu so với source | [§2.3.5](chapter-10.md#replica-drift) |

### Binary log

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Binary log (binlog) | Nhật ký nhị phân | Nhật ký mọi thay đổi đã commit; nền của PITR và replication | [§2.4](chapter-10.md#binlog-backup) |
| Roll forward | Tiến tới | Phát lại binlog lên một bản backup để đưa dữ liệu tới thời điểm sau | [§2.4](chapter-10.md#binlog-backup) |
| `binlog_expire_logs_seconds` | Thời hạn giữ binlog (giây) | MySQL tự xóa binlog cũ hơn thời hạn, chỉ khi khởi động hay xoay vòng binlog, xét theo thời gian sửa file | [§2.4](chapter-10.md#binlog-expire) |
| `expire_logs_days` | Thời hạn giữ binlog (ngày) | Biến cũ sách nhắc ở tr. 261; deprecated trong MySQL 8.0 | [§2.3](chapter-10.md#khuyen-nghi) |
| Binlog rotation | Xoay vòng binlog | Đóng file binlog hiện tại và mở file mới (khi đầy, khi `FLUSH LOGS`, khi khởi động lại) | [§2.4](chapter-10.md#binlog-expire) |
| `FLUSH LOGS` | Xoay vòng log | Bắt đầu binlog mới, ví dụ ngay sau mỗi lần backup để chỉ backup binlog mới | [§2.3.4](chapter-10.md#incremental-differential) |

### Công cụ

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| MySQL Enterprise Backup (MEB) | (giữ nguyên) | Công cụ raw backup "chính thức" của Oracle, có phí; không dừng, không khóa, có nén, incremental, streaming | [§2.5.1](chapter-10.md#meb) |
| Percona XtraBackup | (giữ nguyên) | Công cụ raw backup mã nguồn mở: chép datafile trong khi đọc đuổi redo log, rồi prepare bằng crash recovery | [§2.5.2](chapter-10.md#xtrabackup-tool) |
| mydumper | (giữ nguyên) | Công cụ logical backup và restore đa luồng, tác giả ưa dùng | [§2.5.3](chapter-10.md#mydumper) |
| `mysqldump` | (giữ nguyên) | Công cụ logical backup đi kèm MySQL, đơn luồng, phổ biến nhất | [§2.5.4](chapter-10.md#mysqldump-tool) |
| `mysqlimport` | (giữ nguyên) | Công cụ nạp logical backup dạng văn bản phân tách | [§2.3.2](chapter-10.md#logical-backup) |
| Log sequence number (LSN) | Số thứ tự trong log | Vị trí trong redo log của InnoDB; XtraBackup ghi lại LSN để biết phải áp redo từ đâu tới đâu | [§2.6.3](chapter-10.md#xtrabackup-how) |
| Prepare | Chuẩn bị (backup) | Bước chạy crash recovery trên bản sao XtraBackup để nó nhất quán tại điểm kết thúc chép | [§2.5.2](chapter-10.md#xtrabackup-tool) |
| Tailing the log | Đọc đuổi log | Luồng nền liên tục đọc phần redo log mới ghi trong lúc chép datafile | [§2.5.2](chapter-10.md#xtrabackup-tool) |
| xbstream | (giữ nguyên) | Định dạng một file của XtraBackup khi streaming; giải bằng `xbstream -x` | [§2.6.3](chapter-10.md#xtrabackup-usage) |
| `LOCK TABLES FOR BACKUP` | Khóa backup (Percona Server) | Khóa nhẹ XtraBackup dùng trong output mẫu của sách | [§2.6.3](chapter-10.md#xtrabackup-how) |
| `--copy-back` / `--move-back` | Chép về / chuyển về | Tùy chọn XtraBackup đưa file đã prepare vào datadir | [§2.7.3](chapter-10.md#restore-xtrabackup) |

### Backup dữ liệu và snapshot

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| SQL dump | Bản dump SQL | File chứa câu `CREATE TABLE`, `INSERT` và câu `SET` tùy chọn, chạy được để restore | [§2.6.1](chapter-10.md#logical-sql-dump) |
| Monolithic dump | Dump nguyên khối | Schema và dữ liệu của mọi bảng trong một file lớn; khó restore một bảng, chậm nạp | [§2.6.1](chapter-10.md#logical-sql-dump) |
| `--hex-blob` | (giữ nguyên) | Tùy chọn `mysqldump` ghi dữ liệu nhị phân dưới dạng hex để file đọc được | [§2.3.2](chapter-10.md#logical-backup) |
| Filesystem snapshot | Ảnh chụp filesystem | Ảnh nhất quán của filesystem tại một khoảnh khắc (LVM, ZFS, SAN, đĩa cloud) | [§2.6.2](chapter-10.md#snapshot) |
| "My snapshot is my backup" (Myth #2) | Ngộ nhận "snapshot là backup" | Snapshot copy-on-write chỉ chứa phần khác biệt; block chưa sửa bị hỏng thì mọi snapshot cùng hỏng | [§2.6.2](chapter-10.md#myth-snapshot) |
| Copy-on-write (CoW) | Chép khi ghi | Trước lần ghi đầu vào một block, bản cũ được chép sang vùng dành riêng cho snapshot | [§2.6.2](chapter-10.md#lvm-cow) |
| LVM (Logical Volume Manager) | Trình quản lý volume logic | Lớp quản lý đĩa của Linux có snapshot; phải dựng từ đầu | [§2.6.2](chapter-10.md#lvm-cow) |
| Volume group / logical volume | Nhóm volume / volume logic | Volume group chứa một hay nhiều logical volume; cần chừa chỗ trống cho snapshot | [§2.6.2](chapter-10.md#lvm-prereq) |
| Exception table | Bảng ngoại lệ | Nơi LVM ghi block nào đã được chép sang snapshot | [§2.6.2](chapter-10.md#lvm-planning) |
| Lock-free InnoDB backup | Backup InnoDB không khóa | Snapshot khi MySQL 8+ chỉ InnoDB, GTID, full ACID; restore chạy crash recovery | [§2.6.2](chapter-10.md#lvm-lockfree) |
| Doublewrite buffer | Bộ đệm ghi kép | Vùng InnoDB ghi page trước khi ghi vào chỗ thật; dùng lại cùng block nên chỉ ảnh hưởng snapshot lúc đầu | [§2.6.2](chapter-10.md#lvm-planning) |
| RAID split | Tách đĩa khỏi RAID | Tháo một đĩa khỏi mirror để có bản sao tức thời; phải resync khi gắn lại | [§2.6.2](chapter-10.md#lvm-planning) |

### Phục hồi

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `--skip-networking` | Tắt kết nối mạng | Khởi động MySQL không nhận kết nối TCP để cô lập server trong lúc phục hồi | [§2.7](chapter-10.md#recovery-steps) |
| `SQL_LOG_BIN` | Ghi binlog cho phiên | Đặt 0 để phiên nạp dump không ghi binlog (nhanh hơn, nhưng không tới replica) | [§2.7.1](chapter-10.md#restore-logical) |
| `SOURCE` | Lệnh chạy file trong client | Nạp file SQL từ trong client `mysql`; lỗi không làm dừng cả loạt câu | [§2.7.1](chapter-10.md#restore-logical) |
| `innodb_file_per_table` | Mỗi bảng một tablespace | Mỗi bảng InnoDB trong một file `.ibd`; restore riêng bảng phải "import" | [§2.7.2](chapter-10.md#file-per-table-restore) |
| Import tablespace | Nhập tablespace | Bảo InnoDB nhận một file `.ibd` chép từ nơi khác | [§2.7.2](chapter-10.md#file-per-table-restore) |
| `CHECK TABLES`, `mysqlcheck` | Kiểm tra bảng | Kiểm tra bảng sau khi khởi động instance trên backup; điều kiện để tin một backup | [§2.3.2](chapter-10.md#kiem-tra-backup) |
| `innochecksum` | Kiểm checksum file InnoDB | Kiểm tra checksum page trên file offline; sách không khuyên chỉ dùng cách này | [§2.3.2](chapter-10.md#kiem-tra-backup) |
| `SHOW TABLE STATUS` | Trạng thái bảng | Chạy ở từng database sau khi khởi động từ raw backup, rồi xem lại error log | [§2.7.4](chapter-10.md#start-after-raw) |

### Bổ sung ngoài sách (liên quan chương 10)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| `--single-transaction` | Dump trong một transaction | `mysqldump` mở `START TRANSACTION WITH CONSISTENT SNAPSHOT` để mọi bảng InnoDB thấy cùng snapshot mà không chặn ghi | [§2.6.1](chapter-10.md#single-transaction) |
| `--lock-tables` | Khóa bảng khi dump | Mặc định của `mysqldump`: `LOCK TABLES ... READ` cho từng database; nhất quán trong database nhưng chặn ghi | [§2.6.1](chapter-10.md#single-transaction) |
| `--source-data` (`--master-data`) | Ghi tọa độ binlog | Ghi vào dump tọa độ binlog khớp snapshot (khóa toàn cục rất ngắn); tên mới từ 8.0.26 | [§2.7.1](chapter-10.md#pitr-howto) |
| `--set-gtid-purged` | Ghi tập GTID vào dump | Dump chứa `SET @@GLOBAL.GTID_PURGED` và tắt binlog cho phiên nạp | [§2.7.1](chapter-10.md#restore-logical) |
| `mysqlbinlog --stop-position` / `--start-position` | Giới hạn theo vị trí | Phát lại binlog trong một khoảng vị trí, ví dụ tới ngay trước sự kiện hỏng | [§2.7.1](chapter-10.md#pitr-howto) |
| `mysqlbinlog --exclude-gtids` | Bỏ qua theo GTID | Phát lại binlog trừ các transaction có GTID chỉ định | [§2.7.1](chapter-10.md#pitr-howto) |
| `mysqlbinlog --read-from-remote-server --raw` | Kéo binlog qua mạng | Sao lưu binlog nguyên dạng từ server khác; thêm `--stop-never` để chạy liên tục | [§2.4](chapter-10.md#binlog-expire) |
| `PURGE BINARY LOGS` | Xóa binlog có kiểm soát | Cách an toàn để xóa binlog thủ công, cập nhật index | [§2.4](chapter-10.md#binlog-expire) |
| Clone plugin (`CLONE LOCAL`) | Plugin nhân bản | Tạo bản sao vật lý của dữ liệu InnoDB khi server đang chạy (8.0.17+); không chép binlog | [§2.6.2](chapter-10.md#clone-plugin) |
| Transportable tablespace | Tablespace di chuyển được | `FLUSH TABLES ... FOR EXPORT` tạo `.cfg` để import `.ibd` sang server khác (từ 5.6) | [§2.7.2](chapter-10.md#file-per-table-restore) |
| `LOCK INSTANCE FOR BACKUP` | Khóa backup của MySQL 8.0 | Chặn DDL nhưng không chặn DML trong lúc backup | [§2.6.3](chapter-10.md#xtrabackup-how) |
| `performance_schema.log_status` | Trạng thái log | Một lần đọc trả về GTID, vị trí binlog và LSN InnoDB nhất quán với nhau | [§2.6.3](chapter-10.md#xtrabackup-how) |
| `mysqlpump` | (giữ nguyên) | Công cụ dump song song theo bảng của MySQL, deprecated từ 8.0.34 | [§2.5.4](chapter-10.md#mysqldump-tool) |
| Versioned comment `/*!NNNNN ... */` | Chú thích theo phiên bản | Câu lệnh chỉ chạy trên server có phiên bản từ NNNNN | [§2.6.1](chapter-10.md#logical-sql-dump) |

<a id="chuong-11"></a>

## Chương 11 — Scaling MySQL

### Scaling và capacity

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Scaling | Mở rộng quy mô | Khả năng hệ thống hỗ trợ lưu lượng tăng dần; scale tốt hay kém đo bằng chi phí và độ đơn giản (tr. 287) | [§2.1](chapter-11.md#scaling-definition) |
| Scalability | Khả năng mở rộng | Mức độ tăng được capacity bằng cách thêm tài nguyên mà không làm chậm hệ thống; độc lập với performance (tr. 288) | [§2.1](chapter-11.md#scaling-definition) |
| Capacity | Năng lực, công suất | Lượng công việc hệ thống làm được trong một khoảng thời gian (tr. 288) | [§2.1](chapter-11.md#capacity) |
| Maximum throughput | Throughput tối đa | Con số benchmark đo được khi đẩy hệ thống hết mức; không phải capacity dùng được (tr. 288) | [§2.1](chapter-11.md#capacity) |
| Actual capacity | Capacity thực tế | Throughput đạt được trong khi vẫn giữ hiệu năng chấp nhận được (tr. 288) | [§2.1](chapter-11.md#capacity) |
| Load (quantity of data, number of users, user activity, size of related data sets) | Tải (lượng dữ liệu, số user, hoạt động của user, kích thước tập dữ liệu liên quan) | Bốn góc nhìn về tải mà hệ thống phải chịu (tr. 288–289) | [§2.1](chapter-11.md#load-dimensions) |
| Hockey stick growth | Tăng trưởng hình gậy khúc côn cầu | Tăng trưởng chậm rồi tăng vọt, lưu lượng có thể tăng nhiều bậc mỗi năm (tr. 287) | [§1](chapter-11.md#big-picture) |

### Workload

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Workload | Khối lượng công việc | Hỗn hợp mọi loại truy vấn cùng latency của chúng, không chỉ là QPS (tr. 289–290) | [§2.2.1](chapter-11.md#understanding-workload) |
| QPS (queries per second) | Số truy vấn mỗi giây | Thước đo workload thông dụng; không phản ánh chi phí từng truy vấn (tr. 289) | [§2.2.1](chapter-11.md#understanding-workload) |
| Read-bound workload | Workload giới hạn bởi đọc | Lượng `SELECT` vượt capacity của server; dấu hiệu là CPU cao, IOPS đọc cao (tr. 289–290) | [§2.2.2](chapter-11.md#read-bound) |
| Write-bound workload | Workload giới hạn bởi ghi | Lượng DML (`INSERT`, `UPDATE`, `DELETE`) vượt capacity của server (tr. 289, 291) | [§2.2.3](chapter-11.md#write-bound) |
| Write hotspot | Điểm nóng ghi | Phần dữ liệu nhận ghi tập trung, cần xem trước khi shard (tr. 298) | [§2.5](chapter-11.md#queuing) |

### Read pool

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Replica read pool | Nhóm replica phục vụ đọc | Các replica đứng sau một điểm truy cập chung để trải tải đọc; không nhất thiết gồm mọi replica (tr. 293) | [§2.4](chapter-11.md#read-pool) |
| Virtual IP (VIP) | IP ảo | Địa chỉ trung gian mà ứng dụng kết nối tới, do load balancer giữ, đứng trước các replica (tr. 293) | [§2.4](chapter-11.md#read-pool) |
| Load balancer | Bộ cân bằng tải | Thành phần phân phối kết nối tới các replica; ví dụ HAProxy, load balancer phần cứng, network load balancer của cloud (tr. 293) | [§2.4](chapter-11.md#haproxy-config) |
| HAProxy | HAProxy | Load balancer phần mềm; ví dụ trong sách dùng chế độ TCP, `mysql-check` và `leastconn` (tr. 293–294) | [§2.4](chapter-11.md#haproxy-config) |
| Service discovery | Khám phá dịch vụ | Cơ chế tự động xác định host nào được vào pool; sách khuyên Consul (2021) (tr. 294–295) | [§2.4.1](chapter-11.md#service-discovery) |
| Health check | Kiểm tra sức khỏe | Tiêu chí load balancer dùng để coi replica khỏe và sẵn sàng nhận đọc (tr. 295) | [§2.4.2](chapter-11.md#health-check) |
| Port check | Kiểm tra cổng | Health check chỉ xác nhận process MySQL sống và nhận kết nối; đủ cho nhiều trường hợp (tr. 296) | [§2.4.2](chapter-11.md#health-check) |
| HTTP check (`httpchk`) | Kiểm tra qua HTTP | Load balancer gọi một đường dẫn HTTP chạy script (ví dụ so lag với ngưỡng) và dựa vào mã trả về (tr. 296) | [§2.4.2](chapter-11.md#health-check) |
| Fallback pool | Pool dự phòng | Pool tĩnh đưa mọi node trở lại khi tất cả cùng trượt health check, ví dụ cả cluster lag (tr. 296) | [§2.4.2](chapter-11.md#health-check) |
| `read_only`, `super_read_only` | Chế độ chỉ đọc | Biến hệ thống; kiểm tra cả hai để chắc thành viên pool là replica (tr. 295). `read_only` không chặn user có `CONNECTION_ADMIN`/`SUPER`; `super_read_only` (từ 5.7.8) chặn cả họ | [§2.4.2](chapter-11.md#health-check) |
| Data staleness | Độ cũ của dữ liệu | Mức dữ liệu trên replica tụt sau source mà ứng dụng chấp nhận được; phải thỏa thuận với đội ứng dụng (tr. 296) | [§2.4.2](chapter-11.md#health-check) |

### Thuật toán cân bằng tải

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Random | Ngẫu nhiên | Chọn server ngẫu nhiên trong pool (tr. 297) | [§2.4.3](chapter-11.md#lb-algorithm) |
| Round-robin | Xoay vòng | Gửi lần lượt A, B, C, A...; không giúp khi tải cao (tr. 294, 297) | [§2.4.3](chapter-11.md#lb-algorithm) |
| Fewest connections (`leastconn`) | Ít kết nối nhất | Gửi tới server có ít kết nối hoạt động nhất; được khuyên cho MySQL nhưng có thể dội tải vào server cache lạnh (tr. 294, 297) | [§2.4.3](chapter-11.md#lb-algorithm) |
| Fastest response | Phản hồi nhanh nhất | Gửi tới server đang xử lý nhanh nhất; khó với SQL vì độ phức tạp truy vấn chênh lệch (tr. 297) | [§2.4.3](chapter-11.md#lb-algorithm) |
| Hashed | Băm | Băm IP nguồn để cùng IP luôn tới cùng server; ánh xạ đổi khi số máy đổi (tr. 297) | [§2.4.3](chapter-11.md#lb-algorithm) |
| Weighted | Có trọng số | Gán trọng số theo sức máy, ví dụ máy mạnh gấp đôi nhận gấp đôi request (tr. 297) | [§2.4.3](chapter-11.md#lb-algorithm) |
| Instant-provisioning vs queuing algorithm | Cấp ngay so với có xếp hàng | Loại thứ nhất không xếp hàng kết nối; loại thứ hai giữ concurrency tối đa N và cho yêu cầu mới chờ (tr. 298) | [§2.4.3](chapter-11.md#queuing-algorithm) |

### Hàng đợi

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Queuing | Dùng hàng đợi | Đưa phần ghi không cần tức thì vào hàng đợi, xử lý theo nhịp database chịu được (tr. 298) | [§2.5](chapter-11.md#queuing) |
| 202 Accepted | Mã 202 Đã nhận | Mã HTTP báo yêu cầu đã được nhận, sẽ xử lý sau; khác 200 là thay đổi hợp đồng API (tr. 298) | [§2.5](chapter-11.md#queuing) |
| Time in queue | Thời gian nằm trong hàng đợi | Chỉ số cho biết chiến lược hàng đợi đã hết tác dụng khi nó tăng mãi (tr. 298–299) | [§2.5](chapter-11.md#queuing) |

### Sharding

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Sharding | Phân mảnh dữ liệu | Chia dữ liệu vào các cluster nhỏ hơn để ghi song song trên nhiều source (tr. 299) | [§2.6](chapter-11.md#sharding) |
| Functional sharding / functional partitioning | Tách theo chức năng | Đưa các bảng của một chức năng kinh doanh sang cluster riêng (tr. 291–292, 299) | [§2.3](chapter-11.md#functional-sharding) |
| Data sharding (horizontal sharding) | Phân mảnh dữ liệu theo chiều ngang | Chia một tập dữ liệu thành nhiều shard trên nhiều node; cách phổ biến và thành công nhất với MySQL lớn (tr. 299) | [§2.6](chapter-11.md#functional-vs-data-sharding) |
| Shard | Mảnh | Một phần dữ liệu đã chia, lưu ở một cluster riêng (tr. 299) | [§2.6](chapter-11.md#functional-vs-data-sharding) |
| Partitioning key | Khóa phân vùng | Giá trị quyết định hàng nằm ở shard nào; trả lời "lưu ở đâu" và "tìm ở đâu" (tr. 300) | [§2.6.1](chapter-11.md#partitioning-key) |
| Unit of sharding | Đơn vị shard | Thực thể mà partitioning key đại diện, ví dụ user hay client (tr. 300) | [§2.6.1](chapter-11.md#partitioning-key) |
| Crosstalk | Nói chuyện chéo giữa node | Việc các node phải trao đổi để trả lời một yêu cầu; nguyên tắc scalability là tránh nó (tr. 300) | [§2.6.1](chapter-11.md#partitioning-key) |
| Multiple partitioning keys | Nhiều khóa phân vùng | Shard theo hơn một chiều khi ứng dụng nhìn dữ liệu từ nhiều góc; có thể phải lưu một phần dữ liệu hai lần (tr. 302) | [§2.6.2](chapter-11.md#multiple-partitioning-keys) |
| Cross-shard query | Truy vấn xuyên shard | Truy vấn phải gộp hay `JOIN` dữ liệu từ nhiều shard; phần khó nhất của data sharding (tr. 302) | [§2.6.3](chapter-11.md#cross-shard-queries) |
| Summary table | Bảng tổng hợp | Kết quả dựng sẵn bằng cách duyệt mọi shard, lưu trên mỗi shard hay ở kho riêng (tr. 303) | [§2.6.3](chapter-11.md#cross-shard-queries) |
| Global node | Node toàn cục | Nơi chứa dữ liệu không shard, được cache mạnh (tr. 303) | [§2.6.3](chapter-11.md#cross-shard-queries) |
| Random sharding | Shard ngẫu nhiên | Chia gần như ngẫu nhiên khi cần phân bố đều hay không có khóa tốt; truy vấn xuyên shard thành chuẩn mực (tr. 303) | [§2.6.3](chapter-11.md#cross-shard-queries) |
| XA transaction | Transaction XA | Transaction phân tán qua nhiều shard; dùng được nhưng hiếm vì chi phí (tr. 303) | [§2.6.3](chapter-11.md#cross-shard-consistency) |
| Clean-up process, checker script | Tiến trình dọn dẹp, script kiểm tra | Job định kỳ xóa dữ liệu còn sót và kiểm tra nhất quán giữa các shard (tr. 303) | [§2.6.3](chapter-11.md#cross-shard-consistency) |
| Blast radius | Phạm vi ảnh hưởng | Phần hệ thống bị ảnh hưởng khi một thành phần hỏng; instance nhỏ thì blast radius nhỏ (tr. 305) | [§2.6.4](chapter-11.md#vitess-philosophy) |

### Vitess

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Vitess | Vitess | Hệ thống clustering database cho MySQL: sharding, topology, failover source, thay đổi schema, connection pooling, viết lại truy vấn (tr. 303; trạng thái 2021) | [§2.6.4](chapter-11.md#vitess) |
| Vitess pod | Pod Vitess | Gói một nhóm database và các thành phần Vitess hỗ trợ chúng (tr. 304) | [§2.6.4](chapter-11.md#vitess) |
| VTGate | VTGate | Dịch vụ kiểm soát truy cập vào các instance, tương tự load balancer (tr. 304) | [§2.6.4](chapter-11.md#vitess) |
| VTTablet | VTTablet | Agent chạy trên mỗi instance do Vitess quản lý, thực thi lệnh quản trị (tr. 304) | [§2.6.4](chapter-11.md#vitess) |
| Topology (metadata store) | Kho topology, kho metadata | Kho nhất quán chứa định nghĩa cluster, instance MySQL, instance vtgate (tr. 305) | [§2.6.4](chapter-11.md#vitess) |
| vtctl, vtctld | vtctl, vtctld | Công cụ dòng lệnh và giao diện đồ họa để thay đổi vận hành trên pod (tr. 305) | [§2.6.4](chapter-11.md#vitess) |
| Semisync replication | Replication bán đồng bộ | Source chờ ít nhất một replica xác nhận trước khi báo commit; Vitess khuyên dùng để đổi latency lấy durability (tr. 305) | [§2.6.4](chapter-11.md#vitess-philosophy) |
| Externally managed vttablet | vttablet quản lý bên ngoài | Chế độ cho phép cả kết nối qua vttablet lẫn kết nối thẳng, dùng khi chuyển dần sang Vitess kiểu canary (tr. 306) | [§2.6.4](chapter-11.md#vitess-migration) |
| VReplication | VReplication | Công cụ của Vitess cho pipeline ETL và luồng thay đổi dữ liệu (tr. 311) | [§2.6.5](chapter-11.md#lb-proxy-comparison) |

### ProxySQL và tầng truy cập

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| ProxySQL | ProxySQL | Proxy viết riêng cho giao thức MySQL, GPL; nhận biết session, connection pooling, định tuyến (tr. 306–307; trạng thái 2021) | [§2.6.5](chapter-11.md#proxysql) |
| Session-aware proxy | Proxy nhận biết session | Proxy hiểu giao thức và trạng thái session nên chuyển được kết nối giữa các instance không downtime (tr. 307) | [§2.6.5](chapter-11.md#proxysql) |
| Connection pooling | Gộp kết nối | Tách kết nối phía ứng dụng khỏi kết nối tới database, tái sử dụng kết nối backend (tr. 307, 310) | [§2.6.5](chapter-11.md#connection-pooling) |
| Admin interface | Giao diện quản trị | Giao diện kiểu MySQL của ProxySQL để đổi cấu hình runtime rồi lưu ra đĩa (tr. 307) | [§2.6.5](chapter-11.md#proxysql) |
| Runtime vs disk configuration | Cấu hình runtime và trên đĩa | Hai bản cấu hình của ProxySQL; phải giữ đồng bộ để tránh bất ngờ khi khởi động lại (tr. 309) | [§2.6.5](chapter-11.md#proxysql-sharding) |
| Hostgroup | Nhóm host | Nhóm backend mà ProxySQL định tuyến tới; ví dụ `default_hostgroup` của user, `destination_hostgroup` của luật (tr. 309) | [§2.6.5](chapter-11.md#proxysql-sharding) |
| `mysql_users`, `mysql_query_rules` | Bảng user, bảng luật truy vấn | Bảng cấu hình của ProxySQL dùng để định tuyến theo user và theo schema (tr. 309) | [§2.6.5](chapter-11.md#proxysql-sharding) |
| Thundering herd | Đàn gia súc giẫm đạp | Nhiều client cùng mở kết nối hay thử lại một lúc, đè lên server đang quá tải (tr. 310, chú thích 7) | [§2.6.5](chapter-11.md#thundering-herd) |
| SPoF (single point of failure) | Điểm hỏng đơn | Thành phần mà hỏng là cả hệ thống ngừng; sách cảnh báo đừng chạy ProxySQL đơn lẻ (tr. 308) | [§2.6.5](chapter-11.md#proxysql) |

### Bổ sung ngoài sách (liên quan chương 11)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Multiplexing (ProxySQL) | Dùng chung kết nối backend | Nhiều session client lần lượt dùng một kết nối backend; bị tắt khi session giữ trạng thái như transaction mở, bảng tạm, biến user | [§2.6.5](chapter-11.md#connection-pooling) |
| MySQL partitioning (`PARTITION BY`) | Phân vùng bảng của MySQL | Chia một bảng trên **một** server; không phải sharding, không cộng năng lực ghi của nhiều máy | [§2.6](chapter-11.md#functional-vs-data-sharding) |
| Read-your-writes | Đọc được dữ liệu mình vừa ghi | Đảm bảo read pool không tự có; xử lý bằng đọc từ source ngay sau khi ghi hoặc chờ GTID (`WAIT_FOR_EXECUTED_GTID_SET()`, từ 5.7.5) | [§3](chapter-11.md#th-read-pool) |
| Scatter-gather | Rải rồi gom | Thực thi vật lý của truy vấn xuyên shard: gửi tới mọi shard, gộp ở ứng dụng; latency bằng shard chậm nhất | [§2.6.3](chapter-11.md#cross-shard-queries) |
| Heartbeat (replication) | Nhịp tim replication | Bản ghi thời gian cập nhật mỗi giây trên source để đo lag tin cậy hơn `Seconds_Behind_Source` (Chương 9, tr. 248–249) | [§2.4.2](chapter-11.md#health-check) |
| `slowstart` (HAProxy) | Khởi động chậm | Tăng dần trọng số của server vừa lên lại, tránh dội tải vào node cache lạnh | [§2.4.3](chapter-11.md#lb-algorithm) |
| Buffer pool warm-up | Làm ấm buffer pool | Nạp lại danh sách trang lúc khởi động (`innodb_buffer_pool_load_at_startup`, mặc định bật từ 5.7.7) trước khi nhận tải | [§2.4.3](chapter-11.md#lb-algorithm) |
| Backpressure | Áp lực ngược | Cơ chế làm chậm bên gửi khi bên nhận không theo kịp; liên quan worker xử lý hàng đợi | [§2.5](chapter-11.md#queuing) |

<a id="chuong-12"></a>

## Chương 12 — MySQL in the Cloud

> Mọi định nghĩa về dịch vụ cloud (Aurora, Cloud SQL, RDS Proxy) là **theo sách, thời điểm 2021**.

### Managed MySQL

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Managed MySQL | MySQL được quản lý | Dịch vụ MySQL do nhà cung cấp cloud vận hành: ít công vận hành, thường đắt hơn, ít kiểm soát; không truy cập OS, filesystem; backup và topology theo phương án của nhà cung cấp (tr. 313–314) | [§2.1](chapter-12.md#managed-mysql) |
| MySQL-compatible | tương thích MySQL | Data store có giao diện SQL như MySQL nhưng cơ chế bên trong có thể hoàn toàn khác Oracle MySQL; phải hỏi tương thích với bản lớn nào (tr. 314) | [§2.1](chapter-12.md#mysql-compatible) |
| Aurora MySQL | — | Hosted database MySQL-compatible của AWS; tách compute khỏi storage, storage nhân bản nội bộ thành sáu bản; lo snapshot backup, fast schema change, audit logging, replication trong một region (tr. 314–315) | [§2.1.1](chapter-12.md#aurora) |
| Compute/storage separation | tách compute và storage | Kiến trúc để tầng xử lý và tầng lưu trữ scale riêng, linh hoạt hơn; điểm bán hàng chính của Aurora (tr. 314) | [§2.1.1](chapter-12.md#aurora) |
| Aurora fast DDL | thay đổi schema nhanh của Aurora | Tính năng AWS coi là "lab mode" lúc sách viết; nếu vẫn vậy, dùng công cụ online schema change bên ngoài (tr. 315) | [§2.1.1](chapter-12.md#aurora) |
| Aurora cluster replication | replication trong Aurora cluster | Cơ chế độc quyền của Amazon qua tầng block storage dùng chung, không phải binlog replication; binlog vẫn bật được để replicate ra ngoài hay làm CDC (tr. 315) | [§2.1.1](chapter-12.md#aurora-replication) |
| Change data capture (CDC) | thu nhận thay đổi dữ liệu | Mẫu thiết kế xác định khi nào dữ liệu đổi và chuyển thay đổi sang miền, hệ thống khác (chú thích 2, tr. 315) | [§2.1.1](chapter-12.md#aurora-replication) |
| RDS Proxy | — | Proxy của AWS quản lý cách ứng dụng kết nối tới Aurora; sách khuyên dùng cho database mission-critical khi có nguy cơ connection storm (tr. 315) | [§2.1.1](chapter-12.md#rds-proxy) |
| Connection storm | bão kết nối | Ứng dụng thấy latency tăng nên mở thêm kết nối, làm database tốn thêm tài nguyên và cạn `max_connections` (ch. 2 tr. 31; nhắc ở tr. 315) | [§2.1.1](chapter-12.md#rds-proxy) |
| Aurora Serverless | — | Flavor bỏ compute chạy lâu dài, dùng nền tảng serverless cho tầng compute; linh hoạt chi phí khi workload không chạy liên tục; Hình 12-1 đặt cho staging, dev (tr. 315, 317) | [§2.1.1](chapter-12.md#aurora-flavors) |
| Aurora Global Database | — | Flavor đưa dữ liệu sang nhiều region mà không phải tự quản binlog replication; có đánh đổi riêng (tr. 316) | [§2.1.1](chapter-12.md#aurora-flavors) |
| Aurora Multi-Master | — | Flavor nhận ghi trên nhiều compute node cùng lúc, ưu tiên write availability trong một region; lúc viết chạy lõi 5.6, có số node tối đa, không trộn với Global Database (tr. 316) | [§2.1.1](chapter-12.md#aurora-flavors) |
| Cloud SQL | — | Managed MySQL của GCP; chạy community server nhưng tắt một số tính năng cho multitenancy: `SUPER`, nạp plugin, vài client; không SSH (tr. 317) | [§2.1.2](chapter-12.md#cloud-sql) |
| Multitenancy | đa khách thuê | Nhiều khách hàng dùng chung hạ tầng; lý do Cloud SQL tắt bớt tính năng (tr. 317) | [§2.1.2](chapter-12.md#cloud-sql) |
| Maintenance window | cửa sổ bảo trì | Khoảng thời gian dịch vụ managed nâng cấp; vẫn có downtime, phải cân với SLO của ứng dụng (tr. 318) | [§2.1.2](chapter-12.md#cloud-sql) |

### MySQL trên VM: máy và CPU

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| MySQL on VM | MySQL tự chạy trên máy ảo | Chạy MySQL trên VM giống trên bare metal: toàn quyền vận hành, replica xuyên region, delayed replica, backup tùy biến, soi OS và filesystem (tr. 318) | [§2.2](chapter-12.md#mysql-on-vm) |
| Machine type | loại máy | Cấu hình VM đặt khoảng vCPU, lượng RAM, giới hạn network và disk; đổi được bằng resize (tr. 319) | [§2.2.1](chapter-12.md#machine-types) |
| Resize | đổi cấu hình máy | Tăng hay giảm machine type theo workload, ví dụ theo mùa cao điểm; có thể cần reboot (tr. 319–320) | [§2.2.1](chapter-12.md#machine-types) |
| vCPU | CPU ảo | CPU không thuộc riêng bạn, có thể dùng chung với tenant khác trên cùng host; latency và mức dùng dao động hơn (tr. 319) | [§2.2.2](chapter-12.md#vcpu) |
| vCPU sizing formula | công thức ước lượng vCPU | (Core Count × 95% Total CPU Usage) × 2; ví dụ 40 core, đỉnh 30%, mục tiêu 50% → 24 vCPU (tr. 319) | [§2.2.2](chapter-12.md#cpu-sizing-formula) |
| Custom machine type | loại máy tùy chỉnh | Machine type tự đặt cấu hình khi không có type chuẩn phù hợp; có thể đắt hơn (chú thích 5, tr. 319) | [§2.2.2](chapter-12.md#cpu-sizing-formula) |
| Context switching | chuyển ngữ cảnh | CPU chuyển từ tác vụ này sang tác vụ khác; tăng theo mức dùng và số core, hiện ra thành latency (tr. 320) | [§2.2.2](chapter-12.md#cpu-utilization-target) |
| CPU utilization target | mức dùng CPU mục tiêu | 50% lúc bình thường, đỉnh 65–70%; duy trì từ 70% trở lên thì thêm CPU (tr. 320) | [§2.2.2](chapter-12.md#cpu-utilization-target) |
| CPU chip family | dòng chip CPU | Thế hệ chip: web traffic cao nên dùng đời mới, xử lý backend có thể dùng đời cũ để tiết kiệm (tr. 320) | [§2.2.2](chapter-12.md#cpu-utilization-target) |
| Working set | tập dữ liệu làm việc | Phần dữ liệu thực sự cần để làm việc; chọn RAM theo nó và thà thừa hơn thiếu (tr. 320; ch. 4 tr. 77) | [§2.2.2](chapter-12.md#memory-working-set) |
| Network egress | lưu lượng ra | Lưu lượng đi ra giữa zone, region; thường bị tính tiền, nhưng vẫn nên đặt replica ở zone khác (tr. 320) | [§2.2.2](chapter-12.md#network-egress) |

### MySQL trên VM: đĩa

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Disk type | loại đĩa | Quyết định khó đổi nhất khi chạy trên VM: đổi thường phải gắn đĩa mới và chép dữ liệu (tr. 320) | [§2.2.3](chapter-12.md#disk-type) |
| Locally attached disk | đĩa gắn cục bộ | Đĩa trên chính host: hiệu năng rất cao, ổn định, nhưng là dữ liệu tạm; host hỏng hay VM sang host khác có thể mất hết (tr. 321) | [§2.2.3](chapter-12.md#local-vs-network-disk) |
| Ephemeral disk | đĩa tạm | Đĩa không bảo đảm giữ dữ liệu qua sự cố host hay lần tắt bật instance; cách sách mô tả đĩa local (tr. 321) | [§2.2.3](chapter-12.md#local-vs-network-disk) |
| Network-attached disk | đĩa gắn qua mạng | Đĩa ưu tiên dư thừa và tin cậy hơn hiệu năng; có snapshot tiện lợi, nhưng có thể stall (tr. 321) | [§2.2.3](chapter-12.md#local-vs-network-disk) |
| Disk stall | đĩa bị khựng | Khoảng I/O tạm đứng của đĩa mạng; cũng có thể xảy ra khi gọi API mở rộng đĩa (tr. 321, 324) | [§2.2.3](chapter-12.md#local-vs-network-disk) |
| Software RAID | RAID phần mềm | Sách khuyên dùng với đĩa local để giảm khả năng mất dữ liệu; không chống được mất cả host (tr. 321) | [§2.2.3](chapter-12.md#local-vs-network-disk) |
| Disk snapshot | snapshot đĩa | Ảnh chụp đĩa mạng ở một thời điểm; phục hồi qua crash recovery, dùng để backup và dựng replica nhanh (tr. 321) | [§2.2.3](chapter-12.md#disk-snapshot) |
| ACID-compliant settings | thiết lập tuân thủ ACID | `innodb_flush_log_at_trx_commit=1` và `sync_binlog=1`; điều kiện để snapshot phục hồi tốt (chú thích 6, tr. 321) | [§2.2.3](chapter-12.md#disk-snapshot) |
| Battery- or flash-backed write cache | write cache có pin hoặc flash | Bộ đệm ghi của card RAID phần cứng; nhà cung cấp cloud không có (tr. 321) | [§2.2.3](chapter-12.md#write-cache) |
| Boot disk | đĩa khởi động | Đĩa chứa hệ điều hành; SSD khởi động nhanh hơn HDD 2–3 lần trong thử nghiệm của tác giả (tr. 322) | [§2.2.3](chapter-12.md#ssd-vs-hdd) |
| IOPS | số thao tác I/O mỗi giây | Một trong hai nhu cầu đĩa phải hiểu cả lịch sử lẫn xu hướng trước khi chọn đĩa (tr. 322) | [§2.2.3](chapter-12.md#iops-throughput) |
| Disk throughput | băng thông đĩa | Lượng byte đọc ghi mỗi giây; dùng để chọn đĩa và để tính tần suất auto-extend (tr. 322, 324) | [§2.2.3](chapter-12.md#iops-throughput) |
| pt-diskstats | — | Công cụ của Percona Toolkit; sách khuyên thu số liệu một ngày để đo đỉnh khi chưa có lịch sử (tr. 322; ch. 4 tr. 95) | [§2.2.3](chapter-12.md#iops-throughput) |
| Read-to-write ratio | tỷ lệ đọc trên ghi | Hiểu biết tối thiểu về workload giúp chọn đĩa cho database mới (tr. 322) | [§2.2.3](chapter-12.md#iops-throughput) |

### Vận hành MySQL trên VM

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Host reboot | khởi động lại do host | Phần cứng của host hỏng, VM dừng ngay rồi có thể khởi động lại trên host khác; gây gián đoạn nhất là trên source (tr. 322) | [§2.2.4](chapter-12.md#host-reboots) |
| Unplanned promotion | promotion ngoài kế hoạch | Failover sang replica khi source hỏng bất ngờ; khó, nên sách khuyên thường là chờ source quay lại (tr. 322; ch. 9 tr. 240) | [§2.2.4](chapter-12.md#host-reboots) |
| `read_only` | cờ chỉ đọc | Biến chặn ghi từ user không có đặc quyền; script `@reboot` có thể tắt nó trên source vừa reboot nếu hỏi được hệ thống xác nhận máy nên nhận ghi (tr. 323) | [§2.2.4](chapter-12.md#host-reboots) |
| crond `@reboot` | — | Tùy chọn của cron chạy một script lúc hệ thống khởi động (tr. 323) | [§2.2.4](chapter-12.md#host-reboots) |
| On-call notification suppression | tạm chặn thông báo trực | Chặn page "host down" tối đa năm phút để hệ thống kịp khởi động (tr. 323) | [§2.2.4](chapter-12.md#host-reboots) |
| Separate OS and MySQL data | tách đĩa hệ điều hành và dữ liệu | Để snapshot chỉ chứa dữ liệu, chuyển đĩa sang máy khác, thay OS không cần chép lại dữ liệu (tr. 323) | [§2.2.4](chapter-12.md#separate-os-data) |
| Binary log bucket | bucket chứa binary log | Nơi đẩy binlog để backup; có life-cycle control, chặn xóa sớm, máy database chỉ được ghi (tr. 323) | [§2.2.4](chapter-12.md#binlog-bucket) |
| Life-cycle control | chính sách vòng đời | Quy tắc của bucket tự xóa tệp cũ sau một thời gian, hoặc cấm xóa trước một thời gian (tr. 323) | [§2.2.4](chapter-12.md#binlog-bucket) |
| Provisioned space | dung lượng đã cấp | Với đĩa mạng, bạn trả tiền theo dung lượng cấp, không theo dung lượng dùng (tr. 324) | [§2.2.4](chapter-12.md#auto-extend) |
| Auto-extend | tự mở rộng đĩa | Code gọi API mở rộng đĩa khi vượt mốc như 90%; phải chạy dày hơn thời gian lấp đầy đĩa, có giới hạn, thử dưới tải (tr. 324) | [§2.2.4](chapter-12.md#auto-extend) |
| No free lunch | không có bữa trưa miễn phí | Thông điệp cuối chương: lựa chọn nào cũng có đánh đổi; đặt nó vào bối cảnh và độ trưởng thành của doanh nghiệp (tr. 324) | [§2.3](chapter-12.md#no-free-lunch) |

### Bổ sung ngoài sách (liên quan chương 12)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Quorum (Aurora storage) | đa số xác nhận | Theo bài báo SIGMOD 2017 mà sách dẫn: sáu bản trên ba zone, ghi cần 4/6, đọc cần 3/6 | [§2.1.1](chapter-12.md#aurora-replication) |
| Dynamic privilege | quyền động | MySQL 8.0 chia nhỏ `SUPER` thành các quyền như `SYSTEM_VARIABLES_ADMIN`, `CONNECTION_ADMIN`, `BINLOG_ADMIN` | [§2.1.2](chapter-12.md#cloud-sql) |
| Steal time | thời gian bị đánh cắp | Thời gian VM có việc nhưng hypervisor cho thứ khác chạy; cột `st` của `vmstat` (sách giải thích ở ch. 4 tr. 93) | [§2.2.2](chapter-12.md#vcpu) |
| Hyperthread | luồng phần cứng | Ở nhiều nhà cung cấp một vCPU là một luồng phần cứng, không phải một core vật lý; sai đơn vị làm công thức vCPU lệch tới hai lần | [§2.2.2](chapter-12.md#cpu-sizing-formula) |
| M/M/1 queue | hàng đợi một server | Mô hình R = S ÷ (1 − ρ): thời gian phản hồi tăng vọt khi mức dùng gần 100% | [§2.2.2](chapter-12.md#cpu-utilization-target) |
| Group commit | commit theo nhóm | Gom nhiều transaction vào một lần flush log; tăng throughput nhưng không làm một commit đơn lẻ nhanh hơn fsync | [§2.2.3](chapter-12.md#disk-type) |
| Crash-consistent snapshot | snapshot nhất quán kiểu crash | Snapshot tương đương trạng thái mất điện đột ngột; cần mọi tệp của InnoDB và binlog trên cùng volume | [§2.2.3](chapter-12.md#disk-snapshot) |
| Split brain | hai nơi cùng nhận ghi | Hai server cùng là source sau một failover; rủi ro khi script tự tắt `read_only` mà không hỏi nguồn sự thật | [§2.2.4](chapter-12.md#host-reboots) |
| Time to full (t_đầy) | thời gian tới khi đầy | (1 − ngưỡng) × kích thước đĩa ÷ throughput tối đa; chu kỳ kiểm tra cộng thời gian mở rộng phải nhỏ hơn nó | [§2.2.4](chapter-12.md#time-to-full) |

<a id="chuong-13"></a>

## Chương 13 — Compliance with MySQL

### Compliance và các khung quy định

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Compliance | Tuân thủ | Đáp ứng luật, tiêu chuẩn hay chứng nhận mà doanh nghiệp bắt buộc hoặc cần có; với kỹ sư database là thiết kế quyền, log, backup để có bằng chứng (tr. 325) | [§1](chapter-13.md#big-picture) |
| GRC (governance, risk management, and compliance) | Quản trị, quản lý rủi ro và tuân thủ | Nguyên tắc, quy trình và luật hướng dẫn cách doanh nghiệp đánh giá, ưu tiên rủi ro và tuân thủ luật về xử lý dữ liệu cá nhân, sức khỏe (tr. 326) | [§2.1](chapter-13.md#grc-controls) |
| Control | Kiểm soát | Quy trình và quy tắc công ty tự định nghĩa và thực hành để giảm xác suất một kết cục rủi ro (tr. 326) | [§2.1](chapter-13.md#grc-controls) |
| Evidence (audit evidence) | Bằng chứng kiểm toán | Log, lịch sử thay đổi, báo cáo mà kỹ sư database trình cho kiểm toán viên; nên được sinh tự động ("baked in") (tr. 328, 333) | [§2.2](chapter-13.md#s2-2) |
| SOC 2 (Service Organization Controls Type 2) | Báo cáo kiểm soát của tổ chức dịch vụ, loại 2 | Bộ kiểm soát về security, availability, processing integrity, confidentiality, privacy; với database: quản lý thay đổi, backup và restore, quản lý truy cập (tr. 326) | [§2.1.1](chapter-13.md#soc2) |
| SOX (Sarbanes–Oxley Act) | Đạo luật Sarbanes–Oxley | Luật Mỹ (2002) cho công ty niêm yết; đòi chứng minh dữ liệu ảnh hưởng doanh thu chỉ người cần mới truy cập và mọi thay đổi có log, lý do (tr. 326) | [§2.1.2](chapter-13.md#sox) |
| SOX control 404 | Kiểm soát 404 của SOX | Cách sách gọi Section 404: bằng chứng rằng báo cáo tài chính dựa trên quy trình truy cập và thay đổi dữ liệu đáng tin, có audit trail (tr. 326) | [§2.1.2](chapter-13.md#sox) |
| Audit trail | Dấu vết kiểm toán | Chuỗi bản ghi cho biết ai đổi gì, khi nào, vì sao (tr. 326) | [§2.1.2](chapter-13.md#sox) |
| PCI DSS | Tiêu chuẩn bảo mật dữ liệu ngành thẻ thanh toán | Tiêu chuẩn bảo vệ dữ liệu chủ thẻ; với database: quản lý truy cập cardholder data và quản lý dữ liệu thẻ riêng (tr. 327) | [§2.1.3](chapter-13.md#pci-dss) |
| Cardholder data | Dữ liệu chủ thẻ | Dữ liệu thẻ tín dụng mà PCI DSS bảo vệ (tr. 327) | [§2.1.3](chapter-13.md#pci-dss) |
| HIPAA | Đạo luật về trách nhiệm và chuyển đổi bảo hiểm y tế | Quy định Mỹ (1996) bảo vệ dữ liệu sức khỏe; đòi kiểm soát truy cập, mã hóa và log mọi truy cập ePHI (tr. 327) | [§2.1.4](chapter-13.md#hipaa) |
| ePHI / PHI | Thông tin sức khỏe (điện tử) được bảo vệ | Dữ liệu sức khỏe cá nhân thuộc phạm vi HIPAA; sách viết "electronic personal health information" (tr. 327, 332) | [§2.1.4](chapter-13.md#hipaa) |
| FedRAMP | Chương trình quản lý rủi ro và cấp phép liên bang | Chứng nhận cho nhà cung cấp cloud phục vụ cơ quan liên bang Mỹ; gồm configuration management, access control, đánh giá bảo mật, kiểm toán truy cập và thay đổi (tr. 327) | [§2.1.5](chapter-13.md#fedramp) |
| GDPR | Quy định chung về bảo vệ dữ liệu | Quy định EU (2016) về dữ liệu định danh người ở EU, bất kể trụ sở bên xử lý; đồng ý, giới hạn truy cập, quyền được xóa (tr. 327) | [§2.1.6](chapter-13.md#gdpr) |
| Right to be forgotten | Quyền được lãng quên | Quyền yêu cầu bên xử lý xóa dữ liệu của mình khỏi hệ thống (tr. 327) | [§2.1.6](chapter-13.md#gdpr) |
| Schrems II | Phán quyết Schrems II | Phán quyết 2020 của tòa án EU cho rằng Privacy Shield không đủ bảo vệ người ở EU khỏi giám sát của Mỹ (tr. 328) | [§2.1.7](chapter-13.md#schrems-ii) |
| Privacy Shield | Khung Privacy Shield | Cơ sở pháp lý công ty Mỹ từng dựa vào để xử lý dữ liệu người ở EU; bị Schrems II vô hiệu (tr. 328) | [§2.1.7](chapter-13.md#schrems-ii) |
| FISA | Đạo luật giám sát tình báo nước ngoài | Luật Mỹ (1978) mà phán quyết Schrems II nêu như cơ chế giám sát (tr. 328) | [§2.1.7](chapter-13.md#schrems-ii) |
| Data sovereignty | Chủ quyền dữ liệu | Yêu cầu dữ liệu nằm ở và chỉ được truy cập từ một khu vực pháp lý nhất định; mối lo mới của kiến trúc dữ liệu (tr. 325, 328) | [§2.1.7](chapter-13.md#schrems-ii) |

### Secret và tách quyền truy cập

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Secret | Bí mật (thông tin xác thực) | Mật khẩu của ứng dụng và người vận hành, API token, SSH private key, khóa certificate (tr. 329) | [§2.2.1](chapter-13.md#secrets-management) |
| Secrets management | Quản lý secret | Năng lực lưu, phân phối và xoay vòng secret an toàn, tách khỏi quản lý cấu hình (tr. 329) | [§2.2.1](chapter-13.md#secrets-management) |
| NIST | Viện Tiêu chuẩn và Công nghệ Quốc gia Mỹ | Chuẩn mã hóa của NIST là mức tối thiểu cho giải pháp secret, cần cho HIPAA, FedRAMP (tr. 329) | [§2.2.1](chapter-13.md#secrets-management) |
| Blast radius | Phạm vi ảnh hưởng | Bao nhiêu phần của hệ thống phải xử lý khi một credential bị lộ; user dùng chung làm nó lớn (tr. 330, 332) | [§2.2.1](chapter-13.md#do-not-share-users) |
| Trust but verify | Tin nhưng kiểm | Thái độ với lỗi đưa credential vào repo: quét secret trước khi merge (tr. 330) | [§2.2.1](chapter-13.md#secret-repo) |
| Space limitations | Giới hạn kích thước secret | Dịch vụ secret của cloud có thể giới hạn độ dài một secret; gây phiền khi phải lưu key, certificate (tr. 331) | [§2.2.1](chapter-13.md#choosing-secret-solution) |
| Secret rotation / credential rotation | Xoay vòng secret, xoay vòng thông tin xác thực | Thay credential theo lịch hoặc khẩn cấp mà không ảnh hưởng uptime; về bản chất là một lần deploy (tr. 331, 337) | [§2.2.3](chapter-13.md#credential-rotation) |
| Regional availability | Khả dụng theo region | Secret manager phải phân phối secret ở mọi region ứng dụng chạy; là phụ thuộc runtime mới (tr. 331–332) | [§2.2.1](chapter-13.md#choosing-secret-solution) |
| Least privilege | Đặc quyền tối thiểu | Người và code chỉ có quyền đúng với nhu cầu; nền của việc tách dữ liệu theo rủi ro (tr. 332, 336) | [§2.2.2](chapter-13.md#least-privilege) |
| Separation of roles and data | Tách vai trò và dữ liệu | Tách dữ liệu theo mức rủi ro khi lộ để áp kiểm soát phù hợp (tr. 332) | [§2.2.2](chapter-13.md#separation-roles-data) |
| Sharding for compliance reasons | Tách cluster vì compliance | Đặt dataset chịu kiểm soát khác biệt (ví dụ PHI) vào cluster riêng ngay từ đầu (tr. 332) | [§2.2.2](chapter-13.md#sharding-compliance) |
| Separate database users | User database riêng | Mỗi ứng dụng một credential, không dùng chung giữa các code base (tr. 332) | [§2.2.2](chapter-13.md#separate-users) |
| Dual password | Mật khẩu kép | Từ MySQL 8.0.14: một tài khoản có mật khẩu chính và mật khẩu phụ cùng lúc (`RETAIN CURRENT PASSWORD`, `DISCARD OLD PASSWORD`) để xoay vòng không gián đoạn (tr. 337) | [§2.2.3](chapter-13.md#dual-password) |
| Password expiration policy | Chính sách hết hạn mật khẩu | Cơ chế buộc đổi mật khẩu sau một thời hạn; sách nói kết hợp với dual password giúp xoay vòng dễ hơn (tr. 337) | [§2.2.3](chapter-13.md#dual-password) |

### Theo dõi thay đổi và log truy cập

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Tracking changes | Theo dõi thay đổi | Kiểm soát về cách thay đổi trên dữ liệu và hệ thống được review, test, ghi lại (tr. 333) | [§2.2.3](chapter-13.md#tracking-changes) |
| Structured logs | Log có cấu trúc | Log dạng trường rõ ràng, gửi về một nơi từ mọi tác vụ (audit, schema, backup, config) để kiểm toán dễ (tr. 333, Hình 13-1) | [§2.2.3](chapter-13.md#tracking-changes) |
| Data access logging | Ghi log truy cập dữ liệu | Ghi lại thay đổi hoặc mọi truy cập tới dataset nhạy cảm, ở chặng cuối trước khi dữ liệu đổi (tr. 334) | [§2.2.3](chapter-13.md#data-access-logging) |
| Audit log plug-in (Percona) | Plug-in audit log của Percona | Plug-in của Percona Server sinh sự kiện audit; không cài, không bật mặc định; cài bằng `INSTALL PLUGIN` (tr. 334) | [§2.2.3](chapter-13.md#percona-audit) |
| MySQL Enterprise Audit | Audit của MySQL Enterprise | Plug-in audit của Oracle; thuộc Enterprise Edition, không có trong Community (sách tr. 334 viết khác; xem ghi chú) | [§2.2.3](chapter-13.md#data-access-logging) |
| `audit_log_policy` | Chính sách ghi audit | Biến của plug-in Percona cho biết ghi truy vấn, đăng nhập hay cả hai; sách dùng để kiểm tra plug-in đang ghi truy vấn (tr. 335) | [§2.2.3](chapter-13.md#percona-audit) |
| rsyslog | rsyslog | Công cụ quản lý log trên Unix; dùng để chuyển sự kiện audit tới nền tảng log của tổ chức (tr. 335) | [§2.2.3](chapter-13.md#audit-ingest) |
| Audit trigger | Trigger để audit | Dùng trigger ghi thay đổi vào bảng audit; sách không khuyên vì tốn chi phí ghi, giấu logic, né quy trình deploy, không thấy câu đọc (tr. 334) | [§2.2.3](chapter-13.md#triggers-audit) |
| Chaos testing (log pipeline) | Thử nghiệm hỗn loạn cho pipeline log | Cố ý làm chậm, hỏng đường chuyển log để thấy ảnh hưởng tới database (tr. 336) | [§2.2.3](chapter-13.md#audit-ingest) |
| Version control for schema changes | Quản lý phiên bản cho thay đổi schema | Dùng repo để theo dõi và chạy thay đổi schema; cho sẵn ai yêu cầu, ai duyệt, chạy ra sao; nên một repo cho mỗi cluster (tr. 336) | [§2.2.3](chapter-13.md#schema-version-control) |
| Configuration management (cho database user) | Quản lý cấu hình | Quản lý user và quyền trong repo với PR và peer review làm bằng chứng (tr. 337) | [§2.2.3](chapter-13.md#user-management) |
| `performance_schema.users` | Bảng users của Performance Schema | Đếm kết nối hiện tại và tổng theo tên user từ khi server khởi động; dùng để tìm user không còn dùng (tr. 338) | [§2.2.3](chapter-13.md#retire-users) |
| Retire unused users | Gỡ user không dùng | Định kỳ đối chiếu user active với cấu hình ứng dụng và drop user thừa; sách gợi ý ngưỡng sáu tháng (tr. 337–338) | [§2.2.3](chapter-13.md#retire-users) |

### Backup như một kiểm soát

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Backup test | Kiểm thử backup | Khôi phục thử backup một cách tự động; SOC 2 thường có kiểm soát cho cả tạo lẫn test backup (tr. 338–339) | [§2.2.4](chapter-13.md#backup-restore) |
| Bread crumbs | Dấu vết hoàn tất | Dấu hiệu (ví dụ tệp chứa timestamp) mà job backup để lại khi xong, để monitoring kiểm tra nhanh (tr. 339) | [§2.2.4](chapter-13.md#backup-automation) |
| Track record | Hồ sơ liên tục | Lịch sử các lần backup thành công và các lần thất bại đã thành work item, thứ SOC 2 đòi (tr. 339) | [§2.2.4](chapter-13.md#backup-automation) |
| Centralized logging | Log tập trung | Log backup và test backup gửi về một nơi, vì host có thể bị thay trước kỳ kiểm toán (tr. 340) | [§2.2.4](chapter-13.md#backup-central-logs) |
| Disaster-recovery planning | Lập kế hoạch khôi phục sau thảm họa | Bằng chứng test backup, theo dõi lỗi và biết khôi phục mất bao lâu (tr. 340) | [§2.2.4](chapter-13.md#dr-planning) |
| MTTR (mean time to recovery) | Thời gian khôi phục trung bình | Mốc so sánh với thời gian test restore đo được; vượt thì chia nhỏ dataset hoặc xem lại SLA (tr. 340) | [§2.2.4](chapter-13.md#dr-planning) |
| Backup bucket | Bucket chứa backup | Nơi lưu backup trên cloud; quyền mặc định rộng là nguồn lộ dữ liệu hay gặp (tr. 340) | [§2.2.4](chapter-13.md#backup-access) |

### Bổ sung ngoài sách (liên quan chương 13)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| Role (MySQL 8.0) | Vai trò | Tập quyền có tên gán được cho nhiều user; cách gọn để cài least privilege | [§2.2.2](chapter-13.md#mysql-roles) |
| Default role / `activate_all_roles_on_login` | Role mặc định | Role tự kích hoạt khi đăng nhập; mặc định `activate_all_roles_on_login = OFF`, thiếu default role thì `CURRENT_ROLE()` là `NONE` (đã đo) | [§2.2.2](chapter-13.md#mysql-roles) |
| `ACCOUNT LOCK` | Khóa tài khoản | Chặn đăng nhập (`ERROR 3118`) nhưng đảo ngược được; bước đệm trước `DROP USER` | [§2.2.3](chapter-13.md#retire-users) |
| `FAILED_LOGIN_ATTEMPTS` / `PASSWORD_LOCK_TIME` | Khóa tạm do đăng nhập sai | Từ 8.0.19: sau N lần sai, chặn cả mật khẩu đúng (`ERROR 3955`) trong thời gian khóa | [§2.2.3](chapter-13.md#credential-rotation) |
| Password rewriting (log) | Viết lại mật khẩu trong log | Server thay mật khẩu trong `CREATE USER`, `ALTER USER`, `SET PASSWORD` bằng `<secret>` ở general log; binlog ghi hash | [§2.2.1](chapter-13.md#choosing-secret-solution) |
| `performance_schema.accounts` | Bảng accounts của Performance Schema | Như `users` nhưng theo cặp user và host; cho biết host nào còn dùng một user | [§2.2.3](chapter-13.md#retire-users) |
| `DEFINER` vs `USER()` trong trigger | Người định nghĩa và người gọi | Trigger chạy với quyền definer; `CURRENT_USER()` trả definer, `USER()` trả người gọi (đã đo) | [§2.2.3](chapter-13.md#triggers-audit) |

<a id="phu-luc-a"></a>

## Phụ lục A — Upgrading MySQL

### Quyết định và vòng đời nâng cấp

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| **Regression** | Hồi quy, suy giảm sau nâng cấp | Thứ từng đúng hoặc từng nhanh ở bản cũ nay sai hoặc chậm ở bản mới; rủi ro chính khi lên bản quá mới (tr. 343). | [§2.1](appendix-a.md#on-dinh-va-tinh-nang) |
| **Dot-20 rule** | Quy tắc "chấm 20" | Heuristic của Stewart Smith: phần mềm chỉ thật sự trưởng thành từ bản .20; không phải quy tắc cứng, nhấn mạnh đánh đổi giữa mới và ổn định (tr. 343, chú thích 1). | [§2.1](appendix-a.md#dot-20-rule) |
| **Release notes** | Ghi chú phát hành | Liệt kê tính năng mới, thay đổi, tính năng deprecated và bug đã sửa của một phiên bản; đọc từ bản đang chạy trở đi để chẩn đoán hành vi lạ (tr. 344). | [§2.1](appendix-a.md#release-notes) |
| **Upgrade notes** | Hướng dẫn nâng cấp | Phần tài liệu chính thức mô tả cách nâng cấp và những điều phải biết trước khi tiếp tục (tr. 344). | [§2.2](appendix-a.md#vong-doi-nang-cap) |
| **Major, minor, point release** | Phiên bản chính, phụ, bản vá | Ví dụ 5.7 → 8.0, 8.0 → 8.1, 8.0.21 → 8.0.22; MySQL không theo chặt quy ước này, point release có thể mang tính năng mới (tr. 344). | [§2.1](appendix-a.md#ly-do-nang-cap) |
| **End-of-life (EOL)** | Hết hỗ trợ | Mốc Oracle ngừng hỗ trợ một phiên bản; nên ở bản còn hỗ trợ để ít nhất còn bản vá bảo mật (tr. 344). | [§2.1](appendix-a.md#eol) |
| **Deprecated** | Đánh dấu sẽ bỏ | Tính năng hay biến vẫn chạy nhưng sẽ bị bỏ ở bản sau; release notes liệt kê (tr. 344), phiên bản mới hay đổi tên hoặc bỏ biến (tr. 345). | [§2.3](appendix-a.md#config-review) |
| **Upgrade life cycle** | Vòng đời nâng cấp | Đọc release notes, đọc upgrade notes, thử nghiệm, rồi mới nâng cấp server; kèm kế hoạch khi có sự cố (tr. 344). | [§2.2](appendix-a.md#vong-doi-nang-cap) |
| **Downgrade** | Hạ cấp | Major, minor: chỉ bằng restore backup chụp trước nâng cấp; từ 8.0, point release cũng không hạ cấp được nếu không export rồi import (tr. 344–345). | [§2.2](appendix-a.md#downgrade) |

### Thử nghiệm nâng cấp

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| **Development environment testing** | Thử trên môi trường dev | Bắt lỗi cú pháp hiển nhiên; thường thiếu dữ liệu cỡ production nên không bắt được regression hiệu năng (tr. 345). | [§2.3.1](appendix-a.md#dev-env-testing) |
| **Production mirror** | Bản sao production | Bản sao dữ liệu production đã ngừng replication và nâng cấp, nhận bản sao traffic SQL qua `tcpdump` và `pt-query-digest` (tr. 345). | [§2.3.2](appendix-a.md#production-mirror) |
| **Depool** | Rút khỏi pool | Đưa một replica ra khỏi nhóm nhận traffic; cho phép nâng cấp một replica trước để thử read traffic thật (tr. 346). | [§2.3.3](appendix-a.md#replica-testing) |
| **pt-upgrade** | | Công cụ Percona Toolkit chạy cùng tập truy vấn trên hai đích, báo khác biệt số hàng, dữ liệu hàng, lỗi; đầu vào là slow log, general log hoặc binary log (tr. 346). | [§2.3.4](appendix-a.md#pt-upgrade) |

### Nâng cấp ở quy mô lớn

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| **In-place upgrade** | Nâng cấp tại chỗ | Dừng MySQL, thay binary, khởi động lại trên datadir cũ, chạy `mysql_upgrade` với bản cũ hơn 8.0 (tr. 346). | [§2.4](appendix-a.md#in-place-upgrade) |
| **`mysql_upgrade`** | | Script nâng cấp bảng hệ thống sau khi đổi binary; từ 8.0 việc này chuyển vào lúc server khởi động (tr. 347, chú thích 2). | [§2.4](appendix-a.md#server-upgrade-startup) |
| **Upgrade playbook, runbook** | Kịch bản nâng cấp tự động | Quy trình 11 bước có thể viết thành playbook Ansible, chĩa vào server bất kỳ mà chỉ nâng node chưa nâng cấp và không nhận traffic (tr. 346–348). | [§2.4](appendix-a.md#playbook-nang-cap) |
| **Verify target** | Kiểm tra đích | Bước đầu của playbook: chặn nâng cấp nhầm server đang nhận traffic hoặc ghi được (dùng `read_only`), và bỏ qua server đã nâng cấp (tr. 346). | [§2.4](appendix-a.md#verify-target) |
| **`read_only`, `super_read_only`** | Chế độ chỉ đọc | Chặn ghi từ ứng dụng; `super_read_only` chặn cả tài khoản quản trị, phải tạm tắt khi chạy `mysql_upgrade` (tr. 346–347). | [§2.4](appendix-a.md#verify-target) |
| **Downtime, alert suppression** | Đặt downtime, tắt cảnh báo | Tạm tắt cảnh báo trong lúc MySQL khởi động lại ở bản mới, bỏ sau khi xong để quan sát node lỗi (tr. 347). | [§2.4](appendix-a.md#playbook-nang-cap) |

### Thay đổi schema tức thì

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| **Instant ADD COLUMN** | Thêm cột tức thì | Tính năng InnoDB của 8.0: thêm cột mà không dựng lại bảng, tiết kiệm nhiều thời gian cho đội hay chạy `ALTER TABLE ... ADD COLUMN` (tr. 348). | [§2.5](appendix-a.md#instant-add-column) |
| **Table rebuild** | Dựng lại bảng | Tạo bảng mới và chuyển mọi hàng sang; thời gian và I/O tỉ lệ với kích thước bảng (tr. 348). | [§2.5](appendix-a.md#instant-add-column) |

### Bổ sung ngoài sách (liên quan phụ lục A)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| **Innovation release, LTS** | Bản đổi mới, bản hỗ trợ dài hạn | Mô hình phát hành từ 2023: Innovation (8.1, 8.2, 8.3, 9.x) có tính năng mới; LTS (8.4) chủ yếu sửa lỗi; 8.0 từ 8.0.34 chỉ sửa lỗi. | [§2.1](appendix-a.md#release-model) |
| **`ALGORITHM=INSTANT`, `INPLACE`, `COPY`** | Thuật toán ALTER | INSTANT chỉ sửa metadata; INPLACE làm trong InnoDB, có thể rebuild; COPY chép từng hàng sang bảng tạm. Không ghi thì MySQL chọn cách rẻ nhất được phép. | [§2.5](appendix-a.md#instant-add-column) |
| **Row version (`TOTAL_ROW_VERSIONS`)** | Phiên bản hàng | Từ 8.0.29, mỗi lần thêm hay xóa cột tức thì tạo một row version; tối đa 64, rebuild đưa về 0. | [§2.5](appendix-a.md#instant-add-column) |
| **Điều kiện INSTANT** | | Không áp dụng cho đổi kiểu cột, thêm index, bảng có `FULLTEXT`, bảng `COMPRESSED`, bảng tạm (đo trên 8.0.42). | [§2.5](appendix-a.md#instant-conditions) |
| **Metadata lock khi chạy DDL** | Khóa metadata | INSTANT vẫn cần exclusive metadata lock; có thể chờ transaction dài và chặn các `SELECT` mới. | [§2.5](appendix-a.md#instant-conditions) |
| **`--upgrade` (AUTO, NONE, MINIMAL, FORCE)** | Chế độ nâng cấp của server | Tùy chọn `mysqld` từ 8.0.16 điều khiển việc tự nâng cấp lúc khởi động; NONE chặn nâng cấp ngoài ý muốn. | [§2.4](appendix-a.md#server-upgrade-startup) |
| **`--validate-config`** | Kiểm tra cấu hình | Tùy chọn `mysqld` từ 8.0.16 kiểm tra tùy chọn mà không khởi động; bắt tùy chọn đã bị bỏ, không báo tùy chọn deprecated. | [§2.3](appendix-a.md#config-review) |
| **Idempotent** | Lũy đẳng | Chạy nhiều lần cho cùng kết quả như một lần; playbook nâng cấp bỏ qua node đã nâng cấp. | [§2.4](appendix-a.md#verify-target) |
| **Point of no return** | Điểm không quay lại | Lúc node bản mới nhận ghi; từ đó node bản cũ không replicate được từ nó, chỉ còn restore backup hoặc export rồi import. | [§2.2](appendix-a.md#point-of-no-return) |
| **Canary release** | Thả bản thử cho một phần traffic | Đưa bản mới cho một phần nhỏ traffic trước; nâng cấp một replica rồi repool là một dạng canary. | [§2.3.3](appendix-a.md#replica-testing) |
| **Thay đổi logical và physical** | | Khung phân loại: thay đổi ngữ nghĩa, cú pháp (bắt được với dữ liệu nhỏ) và thay đổi plan (cần dữ liệu cỡ production). | [§2.3](appendix-a.md#logical-vs-physical) |

<a id="phu-luc-b"></a>

## Phụ lục B — MySQL on Kubernetes

### Kubernetes và provisioning

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| **FUD** (fear, uncertainty, doubt) | Nỗi sợ, mơ hồ, hoài nghi | Những nhận định cảm tính quanh việc chạy MySQL trên Kubernetes mà phụ lục muốn gỡ bỏ (tr. 349). | [§1](appendix-b.md#big-picture) |
| **Provisioning** | Cấp phát và dựng tài nguyên | Tạo và cấu hình VM, server, lưu trữ cho một workload; trước Kubernetes thường bằng hệ thống tự xây hoặc ghép nhiều dự án mã nguồn mở (tr. 349). | [§2.1](appendix-b.md#provisioning-stack) |
| **Provisioning stack** | Stack cấp phát | Bộ công cụ quản lý vòng đời tài nguyên; Kubernetes hấp dẫn như một stack chung cho cả tính toán lẫn lưu trữ (tr. 349). | [§2.1](appendix-b.md#provisioning-stack) |
| **Stateless, stateful workload** | Workload không trạng thái, có trạng thái | Stateless: các bản chạy thay thế nhau được; stateful như MySQL giữ dữ liệu, từng bị coi là "không chạy trong container được" (tr. 349). | [§2.1](appendix-b.md#stateless-stateful) |
| **Elasticity** | Tính co giãn | Thêm bớt tài nguyên theo tải; là lợi ích chính của Kubernetes với workload stateless (tr. 349). | [§2.1.1](appendix-b.md#scope-goal) |

### Control plane và vận hành

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| **Control plane** (cho database) | Lớp điều khiển | Hệ thống quản lý các instance database: có thể làm cả provisioning, failover, quản lý kết nối, hoặc chỉ provisioning (tr. 350). | [§2.1.2](appendix-b.md#control-plane) |
| **MySQL operator** | | Phần mềm quản lý MySQL trên Kubernetes; chọn cái nào là hệ quả của phạm vi đã quyết định (tr. 350). | [§2.1.2](appendix-b.md#control-plane) |
| **Vitess** | | Control plane đã được kiểm chứng trong hệ sinh thái Kubernetes mà tác giả khuyên học (tr. 351); chi tiết ở chương 11 (tr. 303–311). | [§2.1.2](appendix-b.md#control-plane) |
| **Dedicated nodes** | Node chuyên dụng | Node chỉ chạy database, không chia với workload stateless co giãn (tr. 350). | [§2.1.3](appendix-b.md#tam-cau-hoi) |
| **Failure modes** | Chế độ hỏng | Các cách hệ thống có thể hỏng; phải trả lời được "dựng lại thế nào, có mất dữ liệu không" (tr. 351). | [§2.1.3](appendix-b.md#failure-modes) |
| **Crawl before you run** | Bò trước khi chạy | Chứng minh với workload stateless trước, rồi database nhỏ (vài GB) và ít quan trọng (tr. 351). | [§2.1.3](appendix-b.md#crawl-before-run) |
| **SLO cho đội tính năng** | | Kết quả của việc thống nhất với đội Kubernetes; nói rõ giải pháp lo gì và đội tự lo gì (tr. 351). | [§2.1.3](appendix-b.md#slo-hop-tac) |
| **Risk and reward** | Rủi ro và phần thưởng | Lăng kính để quyết định chạy MySQL trên Kubernetes, cân với nhu cầu ổn định của tầng dữ liệu (tr. 351). | [§2.2](appendix-b.md#rui-ro-loi-ich) |

### Bổ sung ngoài sách (liên quan phụ lục B)

| Thuật ngữ | Tiếng Việt | Định nghĩa ngắn | Xem |
| --- | --- | --- | --- |
| **Container** | | Tiến trình chạy cô lập kèm hệ tệp riêng; hệ tệp mất khi container bị xóa, trừ phần gắn từ ngoài vào. | [§1](appendix-b.md#big-picture) |
| **Pod, node** | | Pod là đơn vị triển khai nhỏ nhất (một hay vài container); node là máy chạy pod. | [§1](appendix-b.md#big-picture) |
| **Deployment, StatefulSet** | | Deployment cho workload stateless; StatefulSet cho workload stateful: danh tính ổn định, volume riêng, cập nhật theo thứ tự. | [§1](appendix-b.md#big-picture) |
| **PersistentVolume, PersistentVolumeClaim** | Volume bền | Lưu trữ có vòng đời tách khỏi pod; pod chết, dữ liệu còn. | [§2.1.3](appendix-b.md#tam-cau-hoi) |
| **Operator (Kubernetes)** | | Phần mềm mở rộng Kubernetes bằng tài nguyên riêng và liên tục đưa trạng thái thật về trạng thái mong muốn. | [§1](appendix-b.md#big-picture) |
| **Drain node, PodDisruptionBudget** | Rút node, giới hạn gián đoạn | Rút node đuổi các pod đi (thường khi nâng cấp cluster); PodDisruptionBudget giới hạn số pod bị đuổi cùng lúc. | [§3.2](appendix-b.md#tinh-huong-drain) |
