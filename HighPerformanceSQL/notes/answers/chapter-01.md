# Đáp án — Chương 1: Kiến trúc MySQL

Về ghi chú chương: [../chapter-01.md](../chapter-01.md) · Bài thực hành: [../experiments/chapter-01.md](../experiments/chapter-01.md)

> Trả lời câu hỏi trước rồi mới đọc đáp án. "tr. N" là số trang in của sách (*High Performance MySQL*, 4th ed.). Số liệu ghi **Kết quả đã đo** lấy từ thí nghiệm trên MySQL 8.0.42; phần ghi **Bổ sung ngoài sách** không có trong sách.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Tầng nào parse và chọn index, tầng nào lưu dữ liệu?

*Trong ba tầng của sách, tầng nào parse SQL và chọn index, tầng nào lưu dữ liệu? Vì sao storage engine không cần hiểu SQL, và chú thích 1 nêu ngoại lệ nào?*

- **Tầng thứ hai** (tầng server, "bộ não") parse, phân tích, tối ưu truy vấn, tức là cả việc chọn index; nó cũng chứa hàm dựng sẵn, stored procedure, trigger, view (tr. 1).
- **Tầng thứ ba**, các storage engine, lưu và lấy dữ liệu (tr. 1).
- Engine không cần hiểu SQL vì server chỉ gọi nó qua **storage engine API**, gồm vài chục hàm mức thấp kiểu "bắt đầu transaction", "lấy hàng có primary key này". Engine chỉ trả lời yêu cầu, không parse SQL, không nói chuyện với nhau (tr. 2).
- Ngoại lệ (chú thích 1, tr. 2): **InnoDB parse định nghĩa foreign key**, vì server chưa tự cài đặt foreign key.

[về mục 2.1](../chapter-01.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Một truy vấn dùng bao nhiêu core? Thread cache giải quyết gì?

*Một truy vấn phân tích nặng chạy trên một connection. Theo mô hình thread ở tr. 2, nó dùng được bao nhiêu core cùng lúc? Thread cache giải quyết chi phí gì và không giải quyết điều gì?*

- Mỗi connection có **một thread**; các truy vấn của connection chạy trong thread đó, và thread nằm trên **một core** tại một thời điểm (tr. 2). Vậy nói chung truy vấn chỉ dùng **một core**. (**Bổ sung ngoài sách:** MySQL 8.0 có vài ngoại lệ hẹp như đọc song song clustered index cho `CHECK TABLE` hay `SELECT COUNT(*)` không có `WHERE`; không phải cơ chế chung cho mọi truy vấn.)
- Thread cache giữ thread sẵn sàng để **không phải tạo và hủy thread** cho mỗi connection mới (tr. 2; chương 5, tr. 111 hướng dẫn theo dõi `Threads_created`).
- Thread cache **không giới hạn số thread chạy đồng thời**. Khi hàng trăm connection cùng chạy truy vấn, chúng vẫn tranh CPU và mutex. Giới hạn concurrency bên trong InnoDB là chuyện của `innodb_thread_concurrency` (chương 5, tr. 119), còn giảm số connection thật là chuyện của connection pooling ở tầng truy cập (chương 11, tr. 307).
- **Kết quả đã đo:** instance 8.0.42 mặc định có `thread_handling = one-thread-per-connection`, `thread_cache_size = 9`.

[về mục 2.1](../chapter-01.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Optimizer không quan tâm engine, sao đổi engine lại đổi plan?

*Optimizer nằm ở tầng server và "không quan tâm" engine. Vì sao đổi engine của một bảng vẫn có thể đổi execution plan?*

Vì optimizer **hỏi engine** về khả năng của nó, **chi phí** của một số thao tác và **thống kê** dữ liệu của bảng (tr. 3). Engine khác nhau thì:

- có loại index khác nhau (sách nêu ví dụ: có engine hỗ trợ loại index giúp ích cho một số truy vấn, tr. 3);
- cách tổ chức dữ liệu khác nhau nên chi phí đọc khác nhau (**Bổ sung ngoài sách:** ví dụ InnoDB có clustered index, tra theo primary key rẻ, còn secondary index phải tra thêm một bước);
- thống kê khác nhau.

Plan là kết quả của mô hình chi phí trên các con số đó, nên đầu vào đổi thì plan có thể đổi. **Kết quả đã đo** cho thấy ngay cả với cùng engine, chỉ đổi danh sách cột chọn cũng đổi plan (quét `idx_k` covering so với range trên `PRIMARY`) và đổi luôn phạm vi khóa ([TN2 phần B, C](../experiments/chapter-01.md#tn2)).

[về mục 2.1](../chapter-01.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — Query cache bị gỡ nhưng cache kết quả vẫn tốt?

*Query cache bị gỡ trong 8.0, nhưng sách vẫn nói cache kết quả là thực hành tốt. Giải thích sự khác nhau giữa hai loại cache này.*

- Query cache nằm **bên trong server**, dùng chung cho mọi connection. Khi concurrency tăng nó thành **nút thắt khét tiếng**; deprecated từ 5.7.20 và bị gỡ trong 8.0 (tr. 3).
- **Bổ sung ngoài sách** về lý do: cấu trúc chung được bảo vệ bằng khóa toàn cục, nên mọi truy vấn phải xếp hàng qua nó; mỗi lần ghi vào bảng làm mất hiệu lực mọi kết quả liên quan tới bảng đó, nên workload ghi nhiều làm cache gần như vô dụng mà vẫn tốn công.
- Cache **bên ngoài** (memcached, Redis, tr. 3) scale độc lập với database, không chặn đường đi của truy vấn trong server, và để ứng dụng tự chọn TTL, cách làm mất hiệu lực, dữ liệu nào đáng cache. Đổi lại ứng dụng phải chấp nhận dữ liệu có thể cũ.

[về mục 2.1](../chapter-01.md#s2-1)

<a id="q2-1-5"></a>

### Câu 2.1.5 — Thread cache, thread pool, connection pooling; tham chiếu "Chapter 5"

*Phân biệt thread cache, thread pool trong server và connection pooling ở tầng truy cập. Tham chiếu "Chapter 5" ở chú thích 2 có đúng không?*

| Cơ chế | Làm gì | Ở đâu |
| --- | --- | --- |
| Thread cache | Tái dùng thread khi connection đóng mở; vẫn **một thread cho mỗi connection** | Mọi bản MySQL (tr. 2; chương 5, tr. 111) |
| Thread pool trong server | Một nhóm thread **cố định** phục vụ nhiều connection, hạn chế số câu chạy cùng lúc | API plug-in từ 5.5, ít dùng (chú thích 2, tr. 2). **Bổ sung ngoài sách:** plug-in của bản Enterprise; Percona Server, MariaDB có sẵn |
| Connection pooling ở tầng truy cập | Proxy hay thư viện giữ ít connection thật tới database và chia cho nhiều client | ProxySQL, Vitess (chương 11, tr. 303, 306–307) |

**Kết quả kiểm chứng.** Chú thích 2 nói pooling ở tầng truy cập "bàn ở Chapter 5". Chương 5 **không có** cụm "thread pool" hay "access layer"; nó chỉ có *The Thread Cache* (tr. 111) và *Configuring MySQL Concurrency* (tr. 119–120). Pooling ở tầng truy cập nằm ở **chương 11** (tr. 303, 306–307). Vậy tham chiếu **không chính xác** (chi tiết ở [mục 2.1.1](../chapter-01.md#thread-pooling-check)).

[về mục 2.1](../chapter-01.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Vì sao người đọc cũng cần khóa, và InnoDB tránh điều đó ra sao?

*Vì sao trong mô hình read/write lock của sách, người chỉ đọc cũng cần khóa? InnoDB tránh việc người đọc chờ người ghi cho `SELECT` thường bằng cơ chế nào?*

- Nếu có người **xóa ô A25** trong khi người khác đang đọc bảng tính, người đọc có thể nhận một góc nhìn **hỏng hoặc không nhất quán** (tr. 4). Vì vậy trong mô hình khóa thuần, người đọc xin read lock; read lock không chặn nhau nhưng bị write lock chặn (tr. 4).
- InnoDB tránh điều này cho `SELECT` thường bằng **MVCC**: đọc phiên bản phù hợp từ undo log, không xin khóa (tr. 13–14).
- **Kết quả đã đo** ([TN1 phần E](../experiments/chapter-01.md#tn1)): khi một session giữ X lock trên hàng `id=2`, `SELECT` thường ở RC và RR trả 200 trong 0,00 s; `SELECT ... FOR SHARE` (locking read, vẫn theo mô hình read/write lock) chờ 1,99 s.

[về mục 2.2](../chapter-01.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Khi nào table lock hợp lý, khi nào gây hại?

*Table lock có overhead thấp nhất, row lock cho concurrency cao nhất. Nêu một workload mà table lock hợp lý và một workload mà nó gây hại, giải thích bằng lập luận overhead và an toàn của tr. 4–5.*

Lập luận của sách: khóa nhỏ thì nhiều thay đổi chạy song song, nhưng mỗi thao tác khóa có overhead; chiến lược khóa là thỏa hiệp giữa overhead và an toàn (tr. 4–5).

- **Hợp lý** (**Ví dụ minh họa do AI xây dựng**): một bảng được nạp hàng loạt bởi đúng một tiến trình vào ban đêm, ít người đọc cùng lúc. Chỉ có một người ghi, nên row lock không mang lại concurrency gì mà vẫn tốn công theo dõi hàng triệu khóa; một table lock vừa rẻ vừa đủ an toàn.
- **Gây hại**: bảng đơn hàng OLTP với hàng trăm request mỗi giây, mỗi request sửa một đơn khác nhau. Table lock bắt mọi lần ghi xếp hàng tuần tự và chặn cả người đọc khi có người ghi (tr. 5); hàng đợi ghi lại được ưu tiên hoàn toàn hơn hàng đợi đọc (tr. 5), nên người đọc có thể chờ lâu. Row lock cho các request này chạy song song (đã đo: sửa hàng `id=30` không bị chặn khi hàng `id=20` đang bị khóa, [TN2 phần A](../experiments/chapter-01.md#tn2)).

[về mục 2.2](../chapter-01.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — `LOCK TABLES` có hiện trong `data_locks` không?

*Dự đoán: session A chạy `LOCK TABLES t2 READ`, session B chạy `UPDATE t2 ... WHERE id = 10`. `performance_schema.data_locks` thấy gì? B chờ ở tầng nào, xem ở đâu?*

- `LOCK TABLES` được cài **trong server**, không trong engine (tr. 12), nên InnoDB không biết và `data_locks` **không có dòng nào** cho nó.
- B chờ ở **tầng server**, trên metadata lock của bảng.
- **Kết quả đã đo** ([TN2 phần G](../experiments/chapter-01.md#tn2)): `data_locks` 0 dòng; `performance_schema.metadata_locks` có `SHARED_READ_ONLY` GRANTED (A) và `SHARED_WRITE` PENDING (B); processlist của B ở trạng thái "Waiting for table metadata lock"; B chờ 2,01 s, tới khi A `UNLOCK TABLES`. Một `SELECT` từ session thứ ba chạy ngay vì read lock dùng chung.

[về mục 2.2](../chapter-01.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Row lock khóa cái gì, vì sao phụ thuộc access path?

*"Row lock" của InnoDB về mặt vật lý khóa cái gì? Vì sao số bản ghi bị khóa phụ thuộc access path chứ không chỉ phụ thuộc điều kiện `WHERE`?*

- InnoDB khóa **bản ghi trong index**. Sách nói khi cập nhật một hàng, transaction khóa hàng đó "trong primary key index và mọi unique index chứa nó" (tr. 10). Ở REPEATABLE READ nó còn khóa **khoảng trống** giữa các bản ghi (next-key locking, tr. 16).
- Engine khóa những bản ghi nó **duyệt qua** khi tìm hàng; server chỉ lọc phần điều kiện còn lại sau khi engine trả hàng (cơ chế "chuyền hàng qua lại qua API", tr. 18). Access path quyết định engine duyệt bản ghi nào.
- **Kết quả đã đo:**
  - `UPDATE ... WHERE c = 999` không có index trên `c`, không hàng nào khớp: ở RR vẫn khóa cả 4 hàng và supremum ([TN2 phần D](../experiments/chapter-01.md#tn2)).
  - Một `FOR UPDATE` trên khoảng `id` mà optimizer chọn quét toàn bộ `idx_k`: khóa mọi bản ghi; khi plan là range trên `PRIMARY`: chỉ khóa khoảng 10–40 ([TN2 phần B, C](../experiments/chapter-01.md#tn2)).
  - Job `UPDATE` 1.333 đơn trên bảng 100.000 hàng: 100.261 bản ghi bị khóa khi không có index, 4.005 khi có index `(status, updated_at)` ([tình huống 2](../chapter-01.md#th2)).

[về mục 2.2](../chapter-01.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — Chữ C trong ví dụ ngân hàng và lost update

*Trong ví dụ chuyển $200 (tr. 6–7), chữ C của ACID bảo vệ điều gì? Vì sao chỉ bọc ba câu lệnh trong transaction ở REPEATABLE READ chưa đủ để chặn hai lần rút tiền chạy song song?*

- Theo sách, **consistency** bảo đảm database luôn đi từ trạng thái nhất quán sang trạng thái nhất quán; trong ví dụ, crash giữa dòng 3 (trừ checking) và dòng 4 (cộng savings) không làm **$200 biến mất**, vì transaction chưa commit thì không thay đổi nào của nó hiện ra (tr. 7).
- Chưa đủ chặn rút hai lần (**Bổ sung ngoài sách**, có đo): câu `SELECT balance` ở dòng 2 là **consistent read** đọc từ snapshot và **không khóa**. Hai transaction cùng đọc 1000, cùng quyết định "đủ tiền", cùng ghi `1000 − 200`. Kết quả **800** thay vì 600: một lần rút bị mất (lost update). Sách cũng gợi ý điều này khi hỏi "nếu một tiến trình khác chen vào giữa dòng 3 và 4 thì sao" (tr. 6–7).
- Cách đúng (đã đo, đều cho 600): `SELECT ... FOR UPDATE` để đọc bản mới nhất và khóa hàng; hoặc `UPDATE ... SET balance = balance - 200 WHERE ... AND balance >= 200` rồi kiểm tra số hàng bị đổi ([TN2 phần H](../experiments/chapter-01.md#tn2)).

[về mục 2.3](../chapter-01.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — InnoDB so với Bảng 1-1

*Với InnoDB, mỗi isolation level cho kết quả gì về dirty read, nonrepeatable read và phantom (với consistent read và với locking read)? Chỗ nào khác Bảng 1-1?*

**Kết quả đã đo** ([TN1](../experiments/chapter-01.md#tn1), [TN2](../experiments/chapter-01.md#tn2)):

| Level | Dirty read | Nonrepeatable read | Phantom, consistent read | Phantom, locking read |
| --- | --- | --- | --- | --- |
| READ UNCOMMITTED | **Có** (đọc 999 chưa commit) | (không đo) | (không đo) | (không đo) |
| READ COMMITTED | Không | **Có** (100 → 101) | **Có** (2 → 3) | không có gap lock nên không chặn chèn (TN2 phần C) |
| REPEATABLE READ | Không | Không (100 → 100) | **Không** (2 → 2; 4 → 4) | Locking read thấy hàng mới (4 → 5), nhưng sau khi đã khóa thì next-key lock chặn chèn vào khoảng |
| SERIALIZABLE | (không đo) | (không đo) | (không đo) | `SELECT` thường thành locking read khi autocommit tắt |

Khác Bảng 1-1 (tr. 9): bảng ghi REPEATABLE READ **có** phantom, vì đó là chuẩn ANSI. InnoDB chặn phantom cho consistent read bằng MVCC (tr. 9) và cho locking read sau khi khóa bằng next-key locking (tr. 16). Tuy vậy trong cùng một transaction, consistent read và locking read có thể thấy số hàng khác nhau (4 so với 5), điều Bảng 1-1 không mô tả.

[về mục 2.3](../chapter-01.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Vì sao StockPrice deadlock, và làm gì?

*Trong ví dụ StockPrice, vì sao hai transaction deadlock? Nêu hai thay đổi phía ứng dụng để giảm deadlock, và một việc ứng dụng vẫn bắt buộc phải làm.*

- Hai transaction khóa **cùng hai hàng theo thứ tự ngược nhau**: T1 khóa (4, 2020-05-01) rồi xin (3, 2020-05-02); T2 khóa (3, 2020-05-02) rồi xin (4, 2020-05-01). Mỗi bên giữ cái bên kia cần: vòng phụ thuộc (tr. 9–10).
- Thay đổi giảm deadlock (**Bổ sung ngoài sách**):
  1. Luôn khóa theo **cùng một thứ tự**, ví dụ sắp các hàng cần sửa theo primary key trước khi cập nhật.
  2. Giữ transaction **ngắn** và khóa **ít** bản ghi: không làm việc chậm khi đang giữ khóa, có index phù hợp để không khóa thừa.
- Việc bắt buộc: **xử lý lỗi deadlock và chạy lại cả transaction** (tr. 10). Deadlock là chuyện bình thường trong hệ có transaction; có loại không tránh được vì xung đột dữ liệu thật (tr. 10).
- **Kết quả đã đo** ([TN3](../experiments/chapter-01.md#tn3)): bên khép vòng nhận `ERROR 1213` ngay, bên kia chạy tiếp sau 0,52 s chờ. Khi tắt phát hiện deadlock, cả hai chờ tới timeout và, nếu ứng dụng bỏ qua lỗi rồi commit, mỗi bên ghi nửa transaction.

[về mục 2.3](../chapter-01.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — WAL ghi hai lần mà vẫn nhanh?

*Vì sao write-ahead logging ghi dữ liệu xuống đĩa hai lần mà vẫn nhanh hơn ghi thẳng vào bảng? Crash sau khi ghi log nhưng trước khi cập nhật trang dữ liệu thì sao?*

- Lần ghi thứ nhất là **nối vào log**: I/O **tuần tự** trong một vùng nhỏ, tương đối nhanh; đổi dữ liệu trong bộ nhớ thì rất nhanh. Lần ghi thứ hai (cập nhật bảng trên đĩa) diễn ra **sau**, không nằm trên đường đi của transaction (tr. 10). Ghi thẳng vào bảng thì mỗi thay đổi là I/O **ngẫu nhiên** ở nhiều nơi.
- **Bổ sung ngoài sách:** nhiều commit có thể gộp vào một lần đẩy log (group commit), và một page bị sửa nhiều lần chỉ cần ghi xuống file dữ liệu một lần.
- Crash sau khi log đã ghi nhưng page chưa cập nhật: khi khởi động lại, engine dùng log để **khôi phục** thay đổi; cách khôi phục tùy engine (tr. 11). Vì vậy commit chỉ cần log bền, không cần page bền.

[về mục 2.3](../chapter-01.md#s2-3)

<a id="q2-3-5"></a>

### Câu 2.3.5 — DDL giữa transaction, và trộn MyISAM

*Dự đoán: `START TRANSACTION; INSERT INTO t_inno ...; CREATE TABLE x (...); ROLLBACK;` thì hàng vừa chèn còn không? Nếu thay `CREATE TABLE` bằng một `INSERT` vào bảng MyISAM rồi `ROLLBACK` thì sao?*

- Hàng **còn**. DDL như `CREATE TABLE`, `ALTER TABLE` (và `LOCK TABLES`) khiến MySQL **commit transaction đang mở trước khi chạy** (tr. 11). `ROLLBACK` sau đó không còn gì để hoàn tác. **Kết quả đã đo:** `COUNT(*) = 1` ([TN4](../experiments/chapter-01.md#tn4)).
- Với MyISAM: phần InnoDB được rollback, phần MyISAM **không** hoàn tác được (tr. 12). **Kết quả đã đo:** `Warning 1196 Some non-transactional changed tables couldn't be rolled back`; InnoDB còn 0 hàng, MyISAM còn 1 hàng. Sách lưu ý phần lớn trường hợp không có cảnh báo nào, nên đừng trông vào nó (tr. 12).

[về mục 2.3](../chapter-01.md#s2-3)

---

<a id="q2-4-1"></a>

### Câu 2.4.1 — Các bước trong Hình 1-2

*Theo Hình 1-2, khi transaction B đọc một hàng mà A đã sửa nhưng chưa commit, InnoDB làm những bước gì, và vòng lặp kết thúc khi nào?*

Phía A (khi ghi, tr. 13–14): A ghi hàng; một undo record mang transaction ID của A được ghi vào undo log; rollback pointer trỏ tới undo record đó; một bản ghi redo được ghi vào redo log.

Phía B (khi đọc, tr. 14):

1. B đọc bản ghi trong clustered index.
2. InnoDB so **transaction ID của bản ghi** với **read view** của B.
3. Nếu bản hiện tại không được phép thấy (A chưa commit), InnoDB **lần theo undo record và áp dụng** để dựng lại bản trước.
4. Lặp lại tới khi gặp một transaction ID mà read view của B được phép thấy.

Vòng lặp kết thúc khi tìm được phiên bản nhìn thấy được, hoặc khi gặp undo record **xóa hẳn** hàng, nghĩa là với read view của B hàng **không tồn tại** (tr. 14).

[về mục 2.4](../chapter-01.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — trx_id và thời điểm tạo read view

*Sách nói transaction ID được gán lần đầu transaction đọc dữ liệu. Thí nghiệm cho thấy gì về `trx_id` trước và sau câu ghi đầu tiên, và read view ở REPEATABLE READ được tạo lúc nào?*

**Kết quả đã đo** ([TN1 phần C, D](../experiments/chapter-01.md#tn1)):

- Sau `START TRANSACTION`: chưa có dòng nào trong `innodb_trx`.
- Sau `SELECT` đầu tiên: có dòng với `trx_id = 284026416991296`, một số rất lớn không lấy từ bộ đếm.
- Sau `UPDATE` đầu tiên: `trx_id = 1455`, một ID thật.
- Read view ở RR được tạo ở **câu consistent read đầu tiên**: mở transaction lúc t=0, session khác commit lúc t=1, đọc lúc t=2 thì thấy giá trị **mới**. `START TRANSACTION WITH CONSISTENT SNAPSHOT` tạo read view ngay và thấy giá trị **cũ**.

Như vậy câu của sách (tr. 13) đúng ở chỗ InnoDB "nhận" transaction khi nó chạm dữ liệu lần đầu, và read view hình thành ở lần đọc đầu; nhưng **ID thật** chỉ được cấp khi transaction ghi (**Bổ sung ngoài sách**, theo manual: transaction chỉ đọc và không khóa không được cấp ID).

[về mục 2.4](../chapter-01.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Vì sao MVCC không dùng với RU và SERIALIZABLE?

*Vì sao MVCC không dùng với READ UNCOMMITTED và SERIALIZABLE?*

- **READ UNCOMMITTED** không đọc phiên bản phù hợp với transaction của mình mà luôn đọc **bản mới nhất**, kể cả chưa commit (tr. 15). Không cần read view hay undo để dựng bản cũ. (Đã đo: đọc được 999 chưa commit.)
- **SERIALIZABLE** đọc bằng cách **khóa mọi hàng** nó trả về (tr. 15), tức là đọc bản mới nhất có khóa, không đọc snapshot. (Đã đo: `S,REC_NOT_GAP` trên hàng được đọc, người ghi chờ 2,00 s.)
- MVCC chỉ dùng với REPEATABLE READ và READ COMMITTED (tr. 15).

[về mục 2.4](../chapter-01.md#s2-4)

<a id="q2-4-4"></a>

### Câu 2.4.4 — Báo cáo giữ snapshot hai giờ

*Một báo cáo giữ transaction REPEATABLE READ mở hai giờ trên server OLTP bận. Dự đoán ảnh hưởng tới chính báo cáo và tới server, và chỉ số nào xác nhận.*

- **Tới server:** undo của mọi thay đổi sau thời điểm snapshot **không được purge**, vì read view của báo cáo còn có thể cần (tr. 13–14 nói về undo; "housekeeping" là cái giá được nêu ở tr. 14). Undo phình ra; theo tr. 14, kích thước undo và redo ảnh hưởng hiệu năng ở concurrency cao.
- **Tới chính báo cáo:** mỗi hàng bị sửa nhiều lần thì báo cáo phải đi ngược nhiều undo record, nên càng về sau càng chậm ("làm thêm việc khi xét hàng", tr. 14).
- **Chỉ số:** `History list length` trong `SHOW ENGINE INNODB STATUS`, hay `trx_rseg_history_len` trong `INNODB_METRICS`; tuổi transaction cũ nhất (`trx_started` trong `innodb_trx`).
- **Kết quả đã đo** ([TN1 phần F](../experiments/chapter-01.md#tn1), 50.000 hàng, trong vài giây chứ không phải hai giờ): cùng câu `SUM(v)` mất 17 ms qua snapshot mới nhưng **390 ms** qua snapshot cũ sau 5 lần sửa cả bảng; history list length 10 → **1.007**. Hai giờ trên server bận thì **Dự đoán** mức tăng lớn hơn nhiều.

[về mục 2.4](../chapter-01.md#s2-4)

<a id="q2-4-5"></a>

### Câu 2.4.5 — "MVCC nghĩa là không cần khóa"?

*"MVCC nghĩa là InnoDB không cần khóa": đúng hay sai? Dẫn chứng từ sách và từ thí nghiệm.*

**Sai.**

- Sách: engine dùng row-level locking **kết hợp** với MVCC (tr. 13); MVCC cho phép đọc không khóa "trong khi chỉ khóa những hàng cần khi ghi" (tr. 13); InnoDB dùng two-phase locking (tr. 12) và next-key locking (tr. 16).
- Thí nghiệm: `UPDATE` đặt X lock và người ghi khác phải chờ 2,01 s ([TN2 phần A](../experiments/chapter-01.md#tn2)); `FOR SHARE` chờ 1,99 s ([TN1 phần E](../experiments/chapter-01.md#tn1)); SERIALIZABLE khóa cả `SELECT` thường; deadlock vẫn xảy ra ([TN3](../experiments/chapter-01.md#tn3)).
- Điều đúng: MVCC làm cho **consistent read** (`SELECT` thường ở RC và RR) không cần khóa (tr. 14).

[về mục 2.4](../chapter-01.md#s2-4)

---

<a id="q2-5-1"></a>

### Câu 2.5.1 — Một node nhận ghi

*Vì sao MySQL được thiết kế cho một node nhận ghi tại một thời điểm, và đánh đổi của lựa chọn đó là gì?*

- Một node nhận ghi giúp **quản lý nhất quán** dễ hơn: mọi lần ghi có một thứ tự duy nhất tại một nơi (tr. 15).
- Đánh đổi: khi cần dữ liệu được ghi ở **nhiều server hay nhiều vị trí**, phải có cơ chế chép từ node nhận ghi sang node khác, tức replication (tr. 15). **Bổ sung ngoài sách:** khả năng ghi bị giới hạn bởi một node (scale ghi cần sharding, chương 11), và replica bất đồng bộ có thể trễ so với source.

[về mục 2.5](../chapter-01.md#s2-5)

<a id="q2-5-2"></a>

### Câu 2.5.2 — Binary log và redo log

*Binary log trong Hình 1-3 khác redo log ở mục 2.3.3 như thế nào về tầng và mục đích?*

**Bổ sung ngoài sách** (sách chỉ ghi nhãn "Binary log" trên Hình 1-3 và nói transaction log dùng để phục hồi sau crash, tr. 10–11, 15):

| | Binary log | Redo log (transaction log của InnoDB) |
| --- | --- | --- |
| Tầng | Server, độc lập engine | Storage engine InnoDB |
| Nội dung | Thay đổi ở mức logic: câu lệnh hoặc hàng (`binlog_format = ROW` mặc định, đã đọc trên 8.0.42) | Thay đổi vật lý trên page |
| Mục đích | Replication, phục hồi tới một thời điểm | Crash recovery, durability của transaction |
| Trên đĩa (instance thí nghiệm) | `binlog.000001`, `binlog.index` | thư mục `#innodb_redo` |

[về mục 2.5](../chapter-01.md#s2-5)

<a id="q2-5-3"></a>

### Câu 2.5.3 — Ba replica và backup

*Sách khuyên ít nhất ba replica ở các vùng khác nhau. Mục tiêu đó là gì, và replica có thay được backup không?*

- Mục tiêu là **disaster recovery**: dữ liệu production có bản sao ở nhiều vị trí (region trên cloud) để mất một nơi vẫn còn nơi khác (tr. 15).
- Replica **không thay được backup**. Chương 10 nói rõ replica không phải backup (Summary, tr. 286; xem [roadmap chương 10](../learning-roadmap.md#ch10)). Lý do (**Bổ sung ngoài sách**): replication chép cả lỗi, ví dụ một `DROP TABLE` hay `DELETE` nhầm được áp dụng trên mọi replica gần như ngay lập tức; backup cho phép quay về một thời điểm trước lỗi.

[về mục 2.5](../chapter-01.md#s2-5)

---

<a id="q2-6-1"></a>

### Câu 2.6.1 — Metadata nằm ở đâu, file trên đĩa ra sao?

*Trong MySQL 8.0, metadata của một bảng InnoDB nằm ở đâu? Sách viết "`.ibd` và `.frm` được thay bằng `.sdi`"; thí nghiệm cho thấy thư mục dữ liệu thực tế ra sao?*

- Theo sách, 8.0 chuyển metadata vào **data dictionary** lưu trong bảng InnoDB, và có dictionary object cache trong bộ nhớ (tr. 16–17).
- **Kết quả đã đo** ([TN4](../experiments/chapter-01.md#tn4)):
  - Bảng InnoDB chỉ có file `.ibd`; **không có `.frm`**, **không có `.sdi` riêng**. `ibd2sdi` đọc được định nghĩa bảng (SDI) **bên trong** `.ibd`.
  - Bảng MyISAM có `.MYD`, `.MYI` và một file `.sdi`.
  - Dictionary gốc ở `mysql.ibd`; truy vấn thẳng `mysql.tables` bị từ chối (`ERROR 3554`).
- Vậy câu của sách cần đọc là: `.frm` bị bỏ; `.ibd` vẫn còn và mang thêm SDI; file `.sdi` riêng chỉ dành cho bảng không phải InnoDB.

[về mục 2.6](../chapter-01.md#s2-6)

<a id="q2-6-2"></a>

### Câu 2.6.2 — Lợi ích của dictionary object cache

*Dictionary object cache giúp gì cho hiệu năng, và khi nào lợi ích lớn nhất?*

- Đó là cache **LRU trong bộ nhớ** cho định nghĩa partition, bảng, stored program, charset, collation (tr. 16). Server lấy metadata từ bộ nhớ thay vì đọc lại, nên **giảm I/O** (tr. 16).
- Lợi ích lớn nhất khi **một tập con** bảng chiếm phần lớn hoạt động: chúng nằm thường trực trong cache (tr. 16). **Bổ sung ngoài sách:** nếu số bảng hoạt động vượt kích thước cache (ví dụ `table_definition_cache = 2000` trên instance thí nghiệm), LRU sẽ liên tục đẩy ra và nạp lại.

[về mục 2.6](../chapter-01.md#s2-6)

<a id="q2-6-3"></a>

### Câu 2.6.3 — Vì sao backup phải đổi

*Vì sao chuyển metadata vào data dictionary buộc quy trình backup phải đổi?*

Quy trình backup cũ đọc **file metadata** của bảng (ví dụ `.frm`) để lấy định nghĩa bảng. Trong 8.0 các file đó không còn; định nghĩa nằm trong dictionary (bảng InnoDB), nên công cụ backup phải **truy vấn data dictionary** để lấy định nghĩa (tr. 18). Đây là ví dụ sách dùng cho câu "thay đổi lớn trong vận hành" (tr. 18). (Đã đo: không có `.frm` nào trong datadir 8.0.42.)

[về mục 2.6](../chapter-01.md#s2-6)

---

<a id="q2-7-1"></a>

### Câu 2.7.1 — Next-key locking chặn phantom thế nào?

*Next-key locking chặn phantom thế nào? Trong thí nghiệm khóa khoảng `id BETWEEN 15 AND 35` ở REPEATABLE READ, vì sao chèn `id = 25` phải chờ còn `id = 5` và `id = 50` thì không?*

- Thay vì chỉ khóa các hàng truy vấn chạm tới, InnoDB **khóa cả khoảng trống trong index**, nên không ai chèn được hàng mới vào khoảng đã bị khóa (tr. 16).
- **Kết quả đã đo** ([TN2 phần C](../experiments/chapter-01.md#tn2)), bảng có `id` 10, 20, 30, 40, plan range trên `PRIMARY`:
  - Khóa: `X` trên 20 (bản ghi 20 cộng khoảng 10–20), `X` trên 30 (bản ghi 30 cộng khoảng 20–30), `X,GAP` trên 40 (khoảng 30–40).
  - 25 nằm trong khoảng 20–30 đã bị khóa, nên phải chờ 2,00 s (khóa chờ `X,GAP,INSERT_INTENTION`).
  - 5 nằm trước 10, 50 nằm sau 40: ngoài các khoảng bị khóa, chèn ngay.
- Lưu ý: nếu plan là quét toàn bộ một index (TN2 phần B), mọi khoảng đều bị khóa và chèn 50 cũng phải chờ.

[về mục 2.7](../chapter-01.md#s2-7)

<a id="q2-7-2"></a>

### Câu 2.7.2 — Primary key lớn làm secondary index lớn

*Vì sao primary key lớn làm mọi secondary index lớn theo? Output `data_locks` nào là bằng chứng cho cấu trúc đó?*

- Bảng InnoDB được xây trên clustered index; **secondary index chứa các cột primary key** để từ mục index tìm về hàng trong clustered index (tr. 16–17). Mỗi mục của mỗi secondary index mang theo một bản sao primary key, nên primary key càng rộng thì mọi index khác càng phình. Sách khuyên giữ primary key nhỏ khi bảng có nhiều index (tr. 17).
- **Bằng chứng đã đo** ([TN2 phần B](../experiments/chapter-01.md#tn2)): khóa trên `idx_k(k)` có `LOCK_DATA` dạng "10, 10", "20, 20": giá trị `k` **kèm primary key `id`**. Optimizer cũng dùng `idx_k` làm covering index cho `SELECT id ...` chính vì `id` có sẵn trong nó.

[về mục 2.7](../chapter-01.md#s2-7)

<a id="q2-7-3"></a>

### Câu 2.7.3 — Kiểm chứng tham chiếu "Chapter 4"

*Tr. 17 nói read-ahead, adaptive hash index và insert buffer "được bàn ở Chapter 4". Kết quả kiểm chứng là gì, và muốn đọc về chúng thì tìm ở đâu trong sách?*

Câu này có trong bản in (ảnh trang PDF 39), nhưng **không chính xác**:

| Tối ưu | Nơi thực sự được nhắc |
| --- | --- |
| Predictive read-ahead | Chương 4 chỉ nhắc lướt rằng InnoDB tự làm read-ahead, nên read-ahead của RAID (tr. 84) hay filesystem (tr. 88) có thể thừa |
| Adaptive hash index | Chương 5: nằm trong buffer pool (tr. 110). Chương 7: định nghĩa, tự động, chỉ tắt được (tr. 159); giảm chi phí tra cứu của clustered index (tr. 173) |
| Insert buffer (change buffer) | Chương 10: tablespace hệ thống dùng cho insert buffer (tr. 265). Chương 5: change buffer nằm trong buffer pool (tr. 110) và tablespace (tr. 116) |

Không chương nào giải thích cơ chế của ba tối ưu này. **Bổ sung ngoài sách:** trên 8.0.42 mặc định `innodb_adaptive_hash_index = ON`, `innodb_change_buffering = all` (đã đọc); muốn hiểu sâu hãy đọc manual InnoDB.

[về mục 2.7](../chapter-01.md#s2-7)

<a id="q2-7-4"></a>

### Câu 2.7.4 — Atomic DDL có phải transactional DDL?

*Atomic DDL có nghĩa là DDL nằm trong transaction và `ROLLBACK` được không? Giải thích bằng hai kết quả thí nghiệm.*

**Không.**

- Atomic DDL nghĩa là **một câu DDL** hoặc hoàn tất trọn vẹn, hoặc rollback trọn vẹn, nhờ undo và redo log riêng cho DDL (tr. 18). **Đã đo:** `DROP TABLE d1, khong_ton_tai` báo `ERROR 1051` và `d1` vẫn còn ([TN4](../experiments/chapter-01.md#tn4)).
- DDL vẫn **commit ngầm** transaction đang mở (tr. 11) và bản thân nó không được `ROLLBACK` bởi người dùng. **Đã đo:** `START TRANSACTION; INSERT ...; CREATE TABLE ...; ROLLBACK;` thì hàng đã chèn vẫn còn.

[về mục 2.7](../chapter-01.md#s2-7)

<a id="q2-7-5"></a>

### Câu 2.7.5 — JSON và multi-valued index

*Kiểu JSON của 5.7 cải thiện gì so với lưu JSON trong BLOB, và multi-valued index giúp loại truy vấn nào?*

- JSON (5.7) có **tự kiểm tra tính hợp lệ**, cách lưu **tối ưu cho đọc nhanh**, và **hàm SQL** để thao tác trên tài liệu; BLOB thì chỉ là chuỗi byte, không kiểm tra, không có đường truy cập riêng (tr. 17). **Đã đo:** chèn JSON sai cú pháp bị từ chối với `ERROR 3140`.
- Multi-valued index định nghĩa trên **mảng JSON**: mỗi phần tử mảng thành một mục index. Nó tăng tốc truy vấn kiểu "tài liệu nào có mảng chứa giá trị X" khi index khớp với mẫu truy cập (tr. 17). **Ví dụ minh họa do AI xây dựng, đã chạy trên 8.0.42:** index `CAST(profile->'$.zipcodes' AS UNSIGNED ARRAY)` được dùng (`type = ref`) cho `WHERE 94507 MEMBER OF (profile->'$.zipcodes')`.
- Phiên bản: sách ghi 8.0.7; **Bổ sung ngoài sách:** tài liệu MySQL ghi 8.0.17. Chi tiết hiệu năng của JSON ở chương 6, "JSON Data", tr. 139.

[về mục 2.7](../chapter-01.md#s2-7)

---

<a id="q2-8-1"></a>

### Câu 2.8.1 — "Chuyền hàng qua lại qua API"

*Diễn giải câu "MySQL thực thi truy vấn bằng cách chuyền hàng qua lại qua storage engine API". Nó giải thích hiện tượng nào trong các thí nghiệm khóa của chương?*

- Diễn giải: server lập plan, rồi gọi engine theo kiểu "đưa hàng đầu tiên", "đưa hàng kế tiếp" theo access path đã chọn; engine trả **từng hàng**, server lọc phần điều kiện còn lại, tính biểu thức và trả kết quả (tr. 2, 18; chi tiết executor ở chương 8).
- Hiện tượng được giải thích: engine **khóa những bản ghi nó đưa lên**, không biết hàng đó có khớp `WHERE` hay không cho tới khi server lọc. Vì vậy:
  - `UPDATE ... WHERE c = 999` không có index khóa cả bảng ở RR, dù không hàng nào khớp ([TN2 phần D](../experiments/chapter-01.md#tn2));
  - job hết hạn đơn khóa 100.261 bản ghi để sửa 1.333 hàng ([tình huống 2](../chapter-01.md#th2));
  - ở READ COMMITTED, InnoDB nhả khóa của hàng không khớp sau khi điều kiện được đánh giá, nên người khác không bị chặn (**Bổ sung ngoài sách**, đã đo).

[về mục 2.8](../chapter-01.md#s2-8)

<a id="q2-8-2"></a>

### Câu 2.8.2 — Vì sao các chương sau chỉ nói InnoDB

*Vì sao các chương sau gần như chỉ nói về InnoDB? Nêu ba thay đổi cho thấy MySQL "dồn" vào InnoDB.*

- InnoDB là engine mặc định và **phủ gần như mọi use case**, nên các chương sau tập trung vào nó khi bàn tính năng, hiệu năng, giới hạn (tr. 18).
- Ba thay đổi (tr. 18; thêm tr. 16–17):
  1. Chuyển metadata bảng, **xác thực và phân quyền** từ MyISAM sang InnoDB (data dictionary).
  2. **Atomic DDL** dựa trên undo, redo log của InnoDB.
  3. **Online DDL** mạnh hơn, chịu crash tốt hơn, vận hành tốt hơn cho triển khai chú trọng bảo mật.

[về mục 2.8](../chapter-01.md#s2-8)

<a id="q2-8-3"></a>

### Câu 2.8.3 — Những chỗ có thể phải chờ khóa

*Liệt kê những chỗ một câu lệnh có thể phải chờ khóa trong MySQL 8.0, chia theo tầng server và tầng engine, kèm công cụ quan sát tương ứng.*

| Tầng | Loại khóa | Ví dụ | Quan sát bằng |
| --- | --- | --- | --- |
| Server | Metadata lock (chú thích 4, tr. 6) | DDL, `LOCK TABLES` (tr. 12) | `performance_schema.metadata_locks`; processlist "Waiting for table metadata lock" |
| Server | Khóa mức ứng dụng (chú thích 4, tr. 6) | `GET_LOCK()` (**Bổ sung ngoài sách**) | `performance_schema.metadata_locks` (**Bổ sung ngoài sách**, chưa đo) |
| Engine (InnoDB) | Row lock, gap lock, next-key lock (tr. 5–6, 16) | `UPDATE`, `DELETE`, `FOR SHARE`, `FOR UPDATE`, chèn vào khoảng bị khóa | `performance_schema.data_locks`, `data_lock_waits`, `sys.innodb_lock_waits`, `information_schema.innodb_trx` |
| Engine (InnoDB) | Deadlock (tr. 9–10) | Khóa theo thứ tự ngược nhau | `SHOW ENGINE INNODB STATUS` (`LATEST DETECTED DEADLOCK`), error log với `innodb_print_all_deadlocks`, `INNODB_METRICS.lock_deadlocks` |

Tầng server và tầng engine có công cụ khác nhau: `data_locks` không thấy `LOCK TABLES` (đã đo, [TN2 phần G](../experiments/chapter-01.md#tn2)).

[về mục 2.8](../chapter-01.md#s2-8)
