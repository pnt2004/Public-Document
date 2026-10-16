# Đáp án — Chương 3: Performance Schema

Về ghi chú chương: [../chapter-03.md](../chapter-03.md) · Bài thực hành: [../experiments/chapter-03.md](../experiments/chapter-03.md)

> Hãy tự trả lời trước khi đọc. "tr. N" là số trang in của sách (trang PDF = N + 22). Phần không có trong sách được ghi **Bổ sung ngoài sách**; số liệu đo được ghi **Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng).

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Instrument khác consumer thế nào; instrument bật mà consumer tắt

*Instrument và consumer khác nhau thế nào? Nếu instrument của một loại event đang bật nhưng consumer tương ứng tắt, dữ liệu đi đâu?*

**Instrument** là một đoạn mã của MySQL được "gắn cảm biến" để thu thông tin, ví dụ `wait/lock/metadata/sql/mdl` cho metadata lock (tr. 41). **Consumer** là bảng lưu kết quả của các đoạn mã đã được instrument, ví dụ bảng chứa số lần chạy, số lần không dùng index, thời gian của câu lệnh (tr. 41–42). Khi một lệnh được instrument chạy, Performance Schema bọc lời gọi bằng hai macro rồi ghi vào consumer tương ứng (tr. 42).

Nếu instrument bật mà consumer tắt, event vẫn được sinh ra (tốn CPU cho các macro) nhưng **không được lưu** vào bảng đó. Bảng `_history` hay `_history_long` của loại event ấy sẽ trống. Một số bảng khác có thể vẫn nhận dữ liệu nếu consumer của chúng bật; ví dụ trên 8.0.42 mặc định, `events_statements_history_long` tắt nhưng `statements_digest` bật, nên digest vẫn cộng dồn còn bảng lịch sử dài thì trống (**Kết quả đã đo**, thí nghiệm 1 và 2).

**Bổ sung ngoài sách.** Muốn tiết kiệm CPU thì tắt instrument; tắt consumer chủ yếu tiết kiệm chỗ ghi, không tiết kiệm việc đo.

[về mục 2.1](../chapter-03.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Vì sao instrument statement rẻ còn instrument lock đắt

*Vì sao bật instrument cho statement gần như không làm tăng CPU, còn bật instrument cho lock (wait) có thể làm CPU tăng đáng kể? Dựa vào cơ chế hai macro để giải thích.*

Mỗi lời gọi được instrument thêm **hai lời gọi macro** (bắt đầu và kết thúc event) để ghi dữ liệu (tr. 42, 45). Chi phí vì thế tỷ lệ với **số lần đoạn mã đó chạy**:

- Một câu `SELECT` chỉ cần **một** lần xác định nó là `statement/sql/select`, nên instrument statement thêm hai macro cho mỗi truy vấn, không đáng kể (tr. 45).
- Quét một bảng InnoDB một triệu hàng cần đặt và nhả **một triệu row lock**. Instrument lock thêm hai macro cho mỗi lần đó, tức hàng triệu lời gọi thêm cho một câu. CPU có thể tăng đáng kể (tr. 45).

Sách nói memory và metadata lock instrumentation cũng rẻ như statement (tr. 45). Đây là lý do mặc định của 8.0.42 bật toàn bộ `memory/*`, `statement/*` (trừ phần lớn `statement/sp/*`), MDL, nhưng tắt toàn bộ 312 instrument `wait/synch/*` (**Kết quả đã đo**, thí nghiệm 1). Chi phí CPU thực tế không được đo trong bài thực hành.

[về mục 2.1](../chapter-03.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Digest cho 1 triệu câu khác hằng số

*Ứng dụng chạy 1 triệu lần `SELECT ... WHERE user_id = ?` với giá trị khác nhau. Có bao nhiêu dòng trong bảng digest, và vì sao điều này quan trọng với bộ nhớ của Performance Schema? Khi nào hai câu khác nhau lại bị gộp chung một digest?*

**Một dòng** (trong cùng schema). Digest chuẩn hóa câu bằng cách thay hằng số bằng `?`, nên các biến thể chỉ khác giá trị `user_id` có chung digest `SELECT user, birthdate FROM users WHERE user_id = ?` (tr. 44–45). Nhờ vậy Performance Schema theo dõi latency và các số đo khác **cho từng mẫu** mà không phải lưu từng biến thể (tr. 45). Bộ nhớ cho bảng digest có kích thước cố định (đo được 40,28 MB cho 10.000 dòng mặc định, thí nghiệm 4), nên việc gộp là điều kiện để nó không phình theo lưu lượng. **Kết quả đã đo** (thí nghiệm 2): 200 câu tra theo khóa chính khác hằng số nằm chung một dòng với `COUNT_STAR = 200`.

Hai câu khác nhau có thể bị gộp khi (**Bổ sung ngoài sách**):

- chúng giống nhau trong 1.024 byte đầu sau chuẩn hóa, vì digest bị cắt ở `max_digest_length` (tr. 53 nêu giới hạn 1.024 byte và "phần dài hơn bị cắt");
- bảng digest đã đầy (`performance_schema_digests_size`, mặc định 10.000 dòng): digest mới bị dồn vào một dòng chung, và biến trạng thái `Performance_schema_digest_lost` tăng.

Ngược lại, hai câu cùng ý nghĩa nhưng khác cấu trúc (khác tên cột, khác thứ tự điều kiện, khác số phần tử trong danh sách) có thể tạo **hai** digest.

[về mục 2.1](../chapter-03.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — Bật memory instrumentation muộn

*Một DBA khởi động server với mọi instrument tắt, hai ngày sau bật memory instrumentation và thấy InnoDB "chỉ dùng vài MB". Điều gì sai trong cách đọc này?*

Đây đúng là giới hạn thứ hai của sách: Performance Schema **chỉ thu dữ liệu sau khi** instrument và consumer được bật. Buffer pool của InnoDB được cấp phát lúc khởi động, **trước** khi bật memory instrumentation, nên không được tính; DBA không thể biết chính xác bộ nhớ của các buffer toàn cục như vậy (tr. 46). Con số "vài MB" chỉ là các cấp phát **mới** xảy ra sau khi bật.

Bài học (**Bổ sung ngoài sách**, gắn với lời khuyên ở tr. 73): với nhóm instrument rẻ như memory, giữ bật từ lúc khởi động (mặc định của 8.0 đã làm vậy, tr. 53). Trên 8.0.42 mặc định, `memory/innodb/buf_buf_pool` hiện đúng 130,8 MB cho buffer pool 128 MB (**Kết quả đã đo**, thí nghiệm 4). Mẹo "tắt hết rồi bật trước sự cố" (tr. 48) có cái giá chính là điểm mù này.

[về mục 2.1](../chapter-03.md#s2-1)

<a id="q2-1-5"></a>

### Câu 2.1.5 — Vì sao `KILL 61` sai khi 61 là THREAD_ID

*Bạn tìm thấy `OWNER_THREAD_ID = 61` đang giữ metadata lock. Vì sao `KILL 61` có thể thất bại hoặc tệ hơn là giết nhầm kết nối, và cách đúng là gì?*

`OWNER_THREAD_ID` là **THREAD_ID**, định danh nội bộ của Performance Schema. `KILL` nhận **PROCESSLIST_ID**, tức mã kết nối thấy trong `SHOW PROCESSLIST`. Hai số này khác nhau (cảnh báo tr. 47). Performance Schema dùng `THREAD_ID` ở mọi bảng, còn `PROCESSLIST_ID` chỉ có trong bảng `threads`; muốn kill kết nối giữ lock phải tra bảng `threads` (tr. 47).

- **Thất bại:** nếu không có kết nối nào có `PROCESSLIST_ID` 61, MySQL báo lỗi. **Kết quả đã đo** (thí nghiệm 3): `KILL 61` báo `ERROR 1094 … Unknown thread id: 61`.
- **Giết nhầm:** nếu tình cờ có một kết nối khác mang `PROCESSLIST_ID` 61, `KILL 61` sẽ cắt kết nối vô can đó (**Dự đoán**, không thử).

Cách đúng: `SELECT PROCESSLIST_ID FROM performance_schema.threads WHERE THREAD_ID = 61;` rồi `KILL <số đó>`. Trong thí nghiệm 3, số đó là 17 và `KILL 17` giải phóng lock ngay.

[về mục 2.1](../chapter-03.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Cái gì sống sót qua restart

*Bạn bật `statement/sp/%` bằng `UPDATE setup_instruments`, và chèn luật vào `setup_actors`. Sau một lần restart, cái nào còn, cái nào mất? Muốn giữ cả hai thì làm thế nào?*

**Cả hai đều mất.** Thay đổi bằng `UPDATE` (hay bằng thủ tục `sys.ps_setup_enable_instrument`) không được lưu qua restart (tr. 49). Bảng `setup_actors` cũng không được lưu (tr. 52). **Kết quả đã đo** (thí nghiệm 1): các instrument `stage/%` và `statement/sp/%` đã bật bằng `UPDATE` quay về `NO` sau restart.

Cách giữ:

- **Instrument:** dùng tùy chọn khởi động `performance-schema-instrument='statement/sp/%=ON'` trong file cấu hình (tr. 49–50). Tương tự `performance-schema-consumer-...` cho consumer (tr. 50).
- **`setup_actors`, `setup_objects`** (và `setup_threads`): không có tùy chọn file cấu hình; ghi các câu `INSERT`/`UPDATE` vào một file SQL và nạp lúc khởi động bằng `init_file` (tr. 51–52).

[về mục 2.2](../chapter-03.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Hai tùy chọn `performance-schema-instrument` mâu thuẫn

*File cấu hình có hai dòng theo thứ tự: `performance-schema-instrument='statement/sql/select=ON'` rồi `performance-schema-instrument='statement/sql/%=OFF'`. Dự đoán trạng thái của `statement/sql/select` và `statement/sql/insert` sau khi khởi động. Nếu đảo thứ tự hai dòng thì sao?*

Theo cảnh báo của sách, khi có nhiều tùy chọn, **chuỗi instrument dài hơn (cụ thể hơn) được ưu tiên bất kể thứ tự** (tr. 50). Vì vậy:

- `statement/sql/select`: **bật** (luật cụ thể `statement/sql/select` thắng luật wildcard `statement/sql/%`).
- `statement/sql/insert`: **tắt** (chỉ khớp luật wildcard).
- Đảo thứ tự: **kết quả như cũ**.

**Kết quả đã đo** (thí nghiệm 1, đúng thứ tự như câu hỏi): `statement/sql/select` = `YES`, `statement/sql/insert` và `update` = `NO`, chỉ 1 trong 159 instrument `statement/sql/*` bật. Người viết không chạy lại với thứ tự đảo; kết luận cho trường hợp đó dựa trên quy tắc của sách.

[về mục 2.2](../chapter-03.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Tắt `global_instrumentation` hay `events_statements_current`

*`events_statements_history` đang `YES` nhưng ai đó tắt `global_instrumentation`. Bảng lịch sử có nhận event mới không? Còn nếu chỉ tắt `events_statements_current`?*

- Tắt `global_instrumentation`: **không**. Theo Bảng 3-1, khi consumer này tắt thì không tham số riêng nào được kiểm tra, không giữ dữ liệu toàn cục hay theo thread, không thu event nào (tr. 50). `thread_instrumentation` cũng chỉ được xét khi `global_instrumentation` bật.
- Chỉ tắt `events_statements_current`: sách không nói trực tiếp. **Bổ sung ngoài sách** (MySQL 8.0 Reference Manual, "Pre-Filtering by Consumer"): `events_statements_history` và `_history_long` phụ thuộc `events_statements_current`, nên tắt `_current` cũng làm hai bảng lịch sử ngừng nhận event mới. `statements_digest` chỉ phụ thuộc `global_instrumentation`, nên digest vẫn được cộng dồn.

[về mục 2.2](../chapter-03.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Luật cụ thể trong `setup_objects`

*`setup_objects` có hai dòng: (`TRIGGER`, `test`, `%`, `NO`) và (`TRIGGER`, `test`, `my_trigger`, `YES`). Trigger `test.my_trigger` và `test.audit_trg` có được theo dõi không? Vì sao thứ tự chèn hai dòng không quan trọng?*

- `test.my_trigger`: **có**, nhờ luật cụ thể (`OBJECT_NAME = my_trigger`, `ENABLED = YES`).
- `test.audit_trg`: **không**, chỉ khớp luật `test.%` với `ENABLED = NO`.

Thứ tự không quan trọng vì khi quyết định, Performance Schema **tìm luật cụ thể nhất trước** rồi mới lùi về luật kém cụ thể hơn (tr. 51). Đây đúng là ví dụ của sách với `test.my_trigger` và `test.some_other_trigger`. Lưu ý thêm: bảng này không có tùy chọn file cấu hình, muốn giữ qua restart phải dùng `init_file` (tr. 51).

[về mục 2.2](../chapter-03.md#s2-2)

<a id="q2-2-5"></a>

### Câu 2.2.5 — Truy vấn stage trả tập rỗng trên 8.0.42 mặc định

*Trên MySQL 8.0.42 mặc định, bạn chạy truy vấn mẫu của sách để tìm stage tốn hơn một giây và nhận tập rỗng. Liệt kê những gì cần bật, và giải thích vì sao MySQL không bật sẵn chúng.*

Truy vấn ở tr. 61 join `events_stages_history_long` với `events_statements_history_long`. Trên 8.0.42 mặc định (**Kết quả đã đo**, thí nghiệm 1):

1. Consumer `events_stages_current`, `events_stages_history`, `events_stages_history_long` đều **tắt**; cần bật (ít nhất `_current` và `_history_long`, vì `_history_long` phụ thuộc `_current`).
2. Consumer `events_statements_history_long` **tắt**; cần bật để có vế bên kia của join.
3. Chỉ 16/132 instrument `stage/*` bật (các stage báo tiến độ); cần bật `stage/%` như sách nói (tr. 61).

Ngoài ra, cần sửa truy vấn: stage "Sending data" không còn trên 8.0.42 (thay bằng `executing`); ngưỡng `1*10000000000` là 10 ms chứ không phải 1 s vì đơn vị picosecond; nên join thêm `THREAD_ID` (**Bổ sung ngoài sách**, kiểm chứng ở thí nghiệm 2).

Vì sao không bật sẵn: stage xảy ra **nhiều lần mỗi câu lệnh** (thí nghiệm 2: khoảng 18 stage cho một câu `SELECT`), nên tốn CPU hơn statement; và các bảng `_history_long` chiếm bộ nhớ (14,42 MB riêng `events_statements_history_long`, thí nghiệm 4). Mặc định của MySQL ưu tiên chi phí nền thấp; bật thêm khi cần là đúng tinh thần của sách (tr. 48, 73).

[về mục 2.2](../chapter-03.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — `UPDATE` quét toàn bảng nhưng không có trong `statements_with_full_table_scans`

*Một câu `UPDATE … WHERE amount = ?` trên bảng 300.000 hàng không có index ở `amount` mất 500 ms mỗi lần, nhưng không xuất hiện trong `sys.statements_with_full_table_scans`. Vì sao? Điều kiện nào trong Bảng 3-4 hay điều kiện dài của sách bắt được nó?*

**Kết quả đã đo** (thí nghiệm 2): câu `UPDATE orders SET note = CONCAT(note, ?) WHERE amount = ?` chạy 3 lần, trung bình 505,5 ms, đọc 900.060 hàng, nhưng `SUM_NO_INDEX_USED` = 0, `SUM_SELECT_SCAN` = 0. View `sys.statements_with_full_table_scans` dựa trên các cờ "không dùng index" (sách mô tả nó là danh sách câu đã quét toàn bảng, tr. 57), nên bỏ sót câu này. Các cột `SELECT_*` và `NO_INDEX_USED` trong Bảng 3-4 được mô tả theo ngữ cảnh truy vấn đọc và join (tr. 55–56); trên 8.0.42, đường thực thi `UPDATE` một bảng không bật chúng (**quan sát**; nguyên nhân trong mã nguồn chưa kiểm chứng).

Điều bắt được nó là điều kiện **`ROWS_EXAMINED > ROWS_AFFECTED`** trong điều kiện dài ở tr. 56: 900.060 so với 0. (`ROWS_AFFECTED` = 0 vì giá trị gán không đổi, InnoDB không coi là hàng bị thay đổi.) Cách thực hành: ngoài các view sys, luôn xếp digest theo tổng thời gian và so `SUM_ROWS_EXAMINED` với `SUM_ROWS_SENT + SUM_ROWS_AFFECTED`. Sau đó dùng `EXPLAIN` cho câu `UPDATE` để thấy access path là quét toàn bảng (chương 7, 8).

[về mục 2.3](../chapter-03.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — `TIMER_WAIT > 5000000000` và xếp latency theo `LOCK_TIME`

*Đồng nghiệp lọc `events_statements_history_long` bằng `TIMER_WAIT > 5000000000` để tìm "câu chậm hơn 5 giây" và xếp hạng latency theo tổng `LOCK_TIME`. Chỉ ra hai lỗi, sửa lại truy vấn, và dự đoán điều gì xảy ra với kết quả.*

**Lỗi 1: đơn vị.** `TIMER_WAIT` tính bằng picosecond (**Bổ sung ngoài sách**, theo MySQL Reference Manual). 5.000.000.000 ps = 5 ms. Con số trong sách (tr. 56) vì vậy lọc câu dài hơn 5 ms. **Kết quả đã đo** (thí nghiệm 2): `SELECT SLEEP(1)` có `TIMER_WAIT` ≈ 1,0097 × 10¹²; điều kiện `> 5000000000` khớp 42 câu, `> 5000000000000` khớp 0 câu. Sửa: `WHERE TIMER_WAIT > 5 * 1e12` (hoặc viết `5000000000000`).

**Lỗi 2: `LOCK_TIME` không phải latency.** `LOCK_TIME` là thời gian chờ **khóa bảng**; latency là `TIMER_WAIT`. Sách dùng `SUM(LOCK_TIME/1000000)` và gọi là `latency_ms` (tr. 62); ngoài nhầm đại lượng, phép chia 10⁶ cho ra micro giây chứ không phải mili giây. **Kết quả đã đo** (thí nghiệm 2): 436 câu `SELECT` có tổng `TIMER_WAIT` 5.951 ms nhưng tổng `LOCK_TIME` 0,784 ms; thứ hạng còn đảo: theo `LOCK_TIME` thì `insert` (0,052 ms) trên `update` (0,042 ms), theo `TIMER_WAIT` thì `update` (1.564,7 ms) gấp gần 40 lần `insert` (39,9 ms). Sửa:

```sql
SELECT EVENT_NAME, COUNT(*) AS n,
       ROUND(SUM(TIMER_WAIT) / 1e9, 1) AS total_ms,
       ROUND(AVG(TIMER_WAIT) / 1e9, 2) AS avg_ms
FROM performance_schema.events_statements_history_long
GROUP BY EVENT_NAME ORDER BY total_ms DESC;
```

**Dự đoán khi sửa:** danh sách "câu chậm hơn 5 s" co lại rất nhiều (thường còn 0 hoặc vài câu thật sự chậm); thứ hạng latency theo loại câu đổi hẳn, vì câu có `LOCK_TIME` lớn chưa chắc là câu chạy lâu. Nhớ thêm: `_history_long` mặc định tắt trên 8.0.42 và chỉ giữ 10.000 câu gần nhất; với khoảng thời gian dài nên dùng bảng summary.

[về mục 2.3](../chapter-03.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — ALTER TABLE làm mọi SELECT treo

*Sau khi chạy `ALTER TABLE orders ADD COLUMN …`, mọi `SELECT` vào `orders` đều treo, dù `SHOW PROCESSLIST` không có câu nào đang chạy trên `orders` ngoài ALTER. Giải thích cơ chế và trình bày các bước tìm, xử lý bằng Performance Schema.*

**Cơ chế.** Mọi câu SQL đặt **shared metadata lock** và giữ nó **tới hết transaction**; shared MDL chặn câu đổi định nghĩa như `ALTER TABLE` (tr. 63). Một transaction đã đọc `orders` rồi để mở (không còn chạy câu nào trên `orders`) vẫn giữ `SHARED_READ`, nên **biến mất khỏi process list** về mặt "đang làm gì với `orders`" (tr. 63). ALTER xin `EXCLUSIVE` và nằm ở trạng thái `PENDING`. Mọi `SELECT` mới xin `SHARED_READ` lại **xếp hàng sau** yêu cầu `EXCLUSIVE` đang chờ (**Kết quả đã đo**, thí nghiệm 3; lý do ưu tiên là **Bổ sung ngoài sách**), nên cả bảng như bị khóa.

**Các bước.**

1. `metadata_locks` join `threads` theo `OWNER_THREAD_ID = THREAD_ID`, lọc `OBJECT_NAME = 'orders'`: thấy ALTER `EXCLUSIVE` `PENDING`, các `SELECT` `SHARED_READ` `PENDING`, và một thread `SHARED_READ` `GRANTED` (tr. 63–64). `sys.schema_table_lock_waits` cho kết quả tương tự kèm câu `KILL` gợi ý, nhưng có thể in cả dòng ALTER "chặn chính nó" (thí nghiệm 3).
2. Lấy `PROCESSLIST_ID` của thread đang giữ từ bảng `threads` (tr. 47).
3. Xem `events_statements_history` của thread đó để biết transaction đã làm gì, và `events_transactions_current` để biết nó mở bao lâu (thí nghiệm 3 thấy câu `SELECT` đã lấy lock dù câu hiện tại là `SLEEP`).
4. Quyết định (tr. 63: "kill hay chờ"): kill kẻ giữ lock nếu đó là phiên rảnh quên commit (transaction của nó sẽ rollback); kill ALTER nếu kẻ giữ là việc quan trọng đang chạy dài; chờ hầu như không nên.
5. Phòng ngừa (**Bổ sung ngoài sách**): đặt `lock_wait_timeout` nhỏ cho session chạy DDL, giữ transaction ngắn, theo dõi transaction mở lâu.

[về mục 2.3](../chapter-03.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — `memory_global_total` 9 GiB nhưng hệ điều hành báo 14 GiB

*`sys.memory_global_total` báo 9 GiB nhưng hệ điều hành báo `mysqld` dùng 14 GiB. Nêu các nguyên nhân có thể của chênh lệch, và cách tìm một kết nối đang giữ nhiều bộ nhớ bất thường.*

`memory_global_total` chỉ cộng bộ nhớ **được instrument** (tr. 65). Chênh lệch có thể do:

- **Thành phần không hỗ trợ memory instrumentation**, ví dụ một storage engine; sách nêu đúng tình huống này là giới hạn đầu tiên (tr. 46).
- **Cấp phát trước khi bật instrument** (giới hạn thứ hai, tr. 46), nếu server từng khởi động với memory instrumentation tắt.
- **Bổ sung ngoài sách:** bộ nhớ của thư viện, stack của thread, phần allocator giữ lại sau khi MySQL đã trả (phân mảnh), bộ nhớ ánh xạ file. RSS và bộ nhớ được instrument là hai đại lượng khác nhau.

Tìm kết nối bất thường:

1. `sys.memory_by_thread_by_current_bytes`, đã sắp giảm dần theo bộ nhớ hiện dùng; cột `user` cho biết tài khoản hay thread hệ thống (tr. 65–66). Ví dụ của sách là một kết nối giữ 36 GiB (tr. 66).
2. Phân rã thread đó bằng `memory_summary_by_thread_by_event_name` để biết loại cấp phát. **Kết quả đã đo** (thí nghiệm 4): một session đặt biến 50 MB hiện ở đầu view sys với 50,10 MiB, phân rã ra `memory/sql/user_var_entry::value` 50 MB; `user_variables_by_thread` cho thấy biến `big` dài 52.428.800 byte.
3. Lấy `PROCESSLIST_ID` từ `threads` để xem kết nối đang chạy gì hoặc kill nếu cần.

Đừng quên kiểm tra chính Performance Schema: trên instance mặc định đo được nó chiếm khoảng 230 MB, nhiều hơn cả InnoDB (thí nghiệm 4).

[về mục 2.3](../chapter-03.md#s2-3)

<a id="q2-3-5"></a>

### Câu 2.3.5 — Lỗi bị handler nuốt và kết nối dùng `SERIALIZABLE`

*Ứng dụng có stored procedure dùng `CONTINUE HANDLER`, và có vẻ một số kết nối đặt mức isolation khác mặc định. Viết hướng tìm: (a) lỗi nào bị handler nuốt thường xuyên; (b) kết nối nào đang dùng `SERIALIZABLE`. Chú ý điều gì khác với ví dụ trong sách khi chạy trên 8.0?*

**(a) Lỗi bị handler nuốt.** Dùng bảng gộp lỗi, xét **cả hai** cột `SUM_ERROR_RAISED` và `SUM_ERROR_HANDLED` (tr. 70):

```sql
SELECT ERROR_NUMBER, ERROR_NAME, SUM_ERROR_RAISED, SUM_ERROR_HANDLED, LAST_SEEN
FROM performance_schema.events_errors_summary_global_by_error
WHERE SUM_ERROR_HANDLED > 0
ORDER BY SUM_ERROR_HANDLED DESC;
```

**Kết quả đã đo** (thí nghiệm 5): lỗi 1048 bị `CONTINUE HANDLER` bắt có `SUM_ERROR_RAISED` = 0 và `SUM_ERROR_HANDLED` = 2. Nếu lọc như ví dụ của sách (`SUM_ERROR_RAISED > 10`, tr. 70) sẽ không thấy nó. Bảng gộp còn đếm cả cảnh báo và note, nên cần lọc theo mã lỗi quan tâm. Muốn biết handler đã chạy những gì, bật `statement/sp/%` và đọc `events_statements_history`: lời gọi có handler để lại `hpush_jump`, các câu trong handler, `hreturn`, `hpop` (tr. 59–60; thí nghiệm 5 tái hiện đủ 7 event).

**(b) Kết nối dùng `SERIALIZABLE`.** Dùng `variables_by_thread` join `threads` (tr. 67):

```sql
SELECT t.PROCESSLIST_ID, t.PROCESSLIST_USER, t.PROCESSLIST_HOST, v.VARIABLE_VALUE
FROM performance_schema.variables_by_thread v
JOIN performance_schema.threads t USING (THREAD_ID)
WHERE v.VARIABLE_NAME = 'transaction_isolation' AND v.VARIABLE_VALUE = 'SERIALIZABLE';
```

**Khác với sách trên 8.0:** ví dụ ở tr. 67 dùng biến `tx_isolation`; biến này đã bị bỏ từ 8.0.3 (**Bổ sung ngoài sách**). **Kết quả đã đo**: trên 8.0.42, `tx_isolation` trả 0 dòng, phải dùng `transaction_isolation`. Sách cũng nhắc rằng `SERIALIZABLE` có thể khiến transaction lấy nhiều lock hơn mức mặc định (tr. 67). Ngoài ra, cảnh báo về `show_compatibility_56` chỉ áp dụng cho 5.7 (tr. 67).

[về mục 2.3](../chapter-03.md#s2-3)

---

<a id="q2-4-1"></a>

### Câu 2.4.1 — Có nên tắt Performance Schema để tiết kiệm?

*Một đồng nghiệp đề xuất đặt `performance_schema = OFF` trên mọi server production để "tiết kiệm tài nguyên". Phản biện bằng lập luận của sách và số liệu đã đo; có trường hợp nào đề xuất này hợp lý không?*

**Lập luận của sách.** Lời khuyên tắt Performance Schema xuất phát từ các bản MySQL cũ có cách cài đặt chưa tối ưu (tr. 73). Ngày nay mặc định chỉ bật nhóm instrument rẻ (tr. 53), và chi phí CPU tỷ lệ với tần suất gọi, không đáng kể với statement, memory, MDL (tr. 45). Tác giả khuyên **giữ bật**, bật động phần cần thiết, dùng sys làm lối tắt (tr. 73). Hơn nữa `performance_schema` là biến chỉ đọc (tr. 48): tắt rồi thì lúc có sự cố phải **restart** mới bật lại, và mất luôn trạng thái đang cần điều tra.

**Số liệu đã đo.** Chi phí lớn nhất thấy được là **bộ nhớ**: khoảng 230 MB trên instance mặc định (thí nghiệm 4), phần lớn là các bảng digest, lịch sử, gộp lỗi có kích thước cố định. Đổi lại là mọi công cụ ở chương này: tìm câu tốn nhất, metadata lock, bộ nhớ theo thread, lỗi theo account. CPU overhead không được đo.

**Khi nào có thể hợp lý** (**Bổ sung ngoài sách**): máy rất nhỏ (vài trăm MB RAM) mà 230 MB là đáng kể; hoặc đã đo được chi phí không chấp nhận được trên workload cụ thể. Ngay cả khi đó, cách tốt hơn thường là **giảm kích thước** consumer bằng các biến khởi động (tr. 52) thay vì tắt hẳn.

[về mục 2.4](../chapter-03.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — Bộ nhớ không giảm sau khi tắt consumer: có phải rò rỉ?

*Sau khi bật một loạt consumer để điều tra rồi tắt đi, bộ nhớ `mysqld` không giảm. Đây có phải rò rỉ bộ nhớ? Làm sao kiểm chứng bằng chính Performance Schema?*

**Không phải rò rỉ** theo nghĩa thông thường. Sách nói bộ nhớ của Performance Schema, một khi đã cấp, **không được trả lại** kể cả khi tắt instrument và `TRUNCATE` bảng (tr. 45), chỉ trả khi restart (tr. 46); MySQL "không rò rỉ", chỉ giữ dữ liệu consumer trong bộ nhớ (tr. 73). Rò rỉ là tăng không giới hạn; ở đây phần đã giữ được **dùng lại**.

**Kiểm chứng:**

1. Đọc tổng `memory/performance_schema/%` trong `memory_summary_global_by_event_name`, hoặc dòng cuối `performance_schema.memory` của `SHOW ENGINE PERFORMANCE_SCHEMA STATUS` (tr. 71–73).
2. Chụp từng instrument `memory/performance_schema/%` trước và sau sự kiện để biết container nào tăng.
3. Lặp lại tác nhân: nếu lần sau không tăng thêm (hoặc tăng ít hơn nhiều), bộ nhớ đang được dùng lại.

**Kết quả đã đo** (thí nghiệm 4): bật waits rồi tắt và `TRUNCATE` thì tổng vẫn 229,59 MB; tạo 1.500 bảng làm tăng lên 232,36 MB và xóa bảng không làm giảm; lượt 2 tạo **3.000** bảng chỉ tăng thêm 0,58 MB (ở `mutex_instances`, `rwlock_instances`). Nếu tổng `memory/performance_schema/%` ổn định mà RSS vẫn tăng, hãy tìm chỗ khác (câu 2.3.4).

[về mục 2.4](../chapter-03.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Quy trình "đo, đổi, đo lại" cho một index mới

*Thiết kế một quy trình "đo, đổi, đo lại" (tr. 41) cho việc thêm một index, chỉ dùng Performance Schema và sys. Nêu ít nhất hai cái bẫy về số liệu cộng dồn mà quy trình phải tránh.*

**Quy trình** (**Ví dụ minh họa do AI xây dựng**, dựa trên tr. 41, 44–45, 55–58, 73):

1. **Giả thuyết:** "thêm index trên `orders(amount)` thì digest `SELECT COUNT(*) FROM orders WHERE amount BETWEEN ? AND ?` giảm `SUM_ROWS_EXAMINED` mỗi lần từ 300.000 xuống cỡ số hàng khớp, và `SUM_NO_INDEX_USED` về 0."
2. **Đo trước:** chụp `events_statements_summary_by_digest` hai lần cách nhau một cửa sổ cố định (ví dụ 10 phút giờ cao điểm), lấy hiệu cho digest mục tiêu: số lần chạy, tổng và trung bình `TIMER_WAIT`, rows examined trên rows sent, `NO_INDEX_USED`.
3. **Đổi:** thêm index (lưu ý DDL lấy metadata lock, chọn lúc ít tải; xem mục 2.3.3).
4. **Đo lại:** cùng độ dài cửa sổ, cùng giờ trong ngày; so các chỉ số. Kiểm tra cả các digest khác có bị ảnh hưởng (ví dụ `INSERT`, `UPDATE` chậm hơn vì phải bảo trì index mới).
5. **Xác nhận index được dùng** bằng `table_io_waits_summary_by_index_usage` hay `sys.schema_unused_indexes` sau một cửa sổ đủ dài (chương 7, tr. 185).

**Bẫy về số liệu cộng dồn:**

- **Trung bình cộng dồn che kết quả.** Cảnh báo ở tr. 58: 10 lần chạy có full join, thêm index, chạy thêm 1 lần thì trung bình chỉ giảm từ 1 xuống 0,9. Phải so **hiệu** trong cửa sổ, hoặc `TRUNCATE` bảng summary trước khi đo.
- **Bộ đếm bị đặt lại ngoài ý muốn.** Restart xóa mọi summary; **Kết quả đã đo** (thí nghiệm 5): chính DDL trên bảng cũng đặt lại bộ đếm dùng index của mọi index trên bảng đó, nên ngay sau `CREATE INDEX` mọi index đều trông như "chưa dùng".
- **Cửa sổ quá ngắn hay không tương đương.** `_history_long` chỉ giữ 10.000 câu gần nhất (tr. 44, 53); so giờ cao điểm với giờ thấp điểm cho kết luận sai.
- **Digest bị gộp hay mất.** Câu dài hơn 1.024 byte bị cắt (tr. 53); bảng digest đầy thì digest mới bị dồn chung (**Bổ sung ngoài sách**).

[về mục 2.4](../chapter-03.md#s2-4)
