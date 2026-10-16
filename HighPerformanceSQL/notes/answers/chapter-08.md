# Đáp án — Chương 8: Tối ưu hiệu năng truy vấn

Quay lại ghi chú: [../chapter-08.md](../chapter-08.md). "tr. N" là số trang in (trang PDF = N + 22). Số liệu đánh dấu **Kết quả đã đo** lấy từ [bài thực hành](../experiments/chapter-08.md) trên MySQL 8.0.42 với dữ liệu tổng hợp.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Ba cách tác động lên một nhiệm vụ con

*Ba cách tác động lên một nhiệm vụ con là gì? Với mỗi cách, cho một kỹ thuật cụ thể trong MySQL.*

Sách nói muốn tối ưu truy vấn thì tối ưu các nhiệm vụ con của nó: **loại bỏ** chúng, cho chúng **xảy ra ít lần hơn**, hoặc làm chúng **nhanh hơn** (tr. 191); summary nhắc lại thành "thôi làm, làm ít lần hơn, làm nhanh hơn" (tr. 226).

- **Loại bỏ:** thêm `LIMIT` để server không sinh và gửi hàng thừa (tr. 193); dùng `UNION ALL` để bỏ bước loại trùng trong bảng tạm (tr. 226); bỏ `SELECT *` để không phải quay về clustered index cho từng hàng (tr. 193).
- **Ít lần hơn:** một truy vấn 10 hàng thay cho 10 truy vấn một hàng (tr. 199); thứ tự join bắt đầu từ bảng ít hàng hơn để giảm số lần dò bảng sau (tr. 215–216); cache dữ liệu bị lấy lặp (tr. 193–194).
- **Nhanh hơn:** thêm index để đổi access type từ `ALL` sang `ref` (tr. 195–196); covering index (tr. 198); hash join thay vòng lặp quét lại bảng trong (tr. 212).

[về mục 2.1](../chapter-08.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Thời gian của một truy vấn trả 50.000 hàng

*Một truy vấn trả 50.000 hàng cho ứng dụng chạy trên máy khác. Ngoài việc đọc dữ liệu trong storage engine, thời gian còn có thể bị tiêu ở đâu? Dự đoán điều gì thay đổi nếu ứng dụng chạy cùng máy với MySQL.*

Theo vòng đời truy vấn (tr. 192), thời gian còn nằm ở:

- **Mạng:** 50.000 hàng đi thành 50.000 packet của giao thức (tr. 218), qua TCP; độ trễ và băng thông đều tính.
- **CPU của server:** dựng từng hàng, đánh giá biểu thức, có thể sort hay gộp nhóm.
- **Chờ khóa** và các mutex nội bộ (tr. 192).
- **Phía client:** thư viện mặc định đệm **toàn bộ** result set vào bộ nhớ trước khi ứng dụng thấy hàng đầu (tr. 202–203), tốn bộ nhớ và CPU ở máy ứng dụng (tr. 192). Trong lúc đó server giữ tài nguyên của truy vấn tới khi gửi xong (tr. 202).

Nếu chạy cùng máy (**Dự đoán**): phần độ trễ mạng giảm mạnh, nhưng chi phí chuyển dữ liệu **không** về 0, như chú thích 1 (tr. 192) nói. Còn có thể xuất hiện chi phí mới: ứng dụng và MySQL tranh nhau CPU và bộ nhớ trên cùng máy (**Bổ sung ngoài sách**). Cách giảm thật sự vẫn là trả ít hàng hơn.

[về mục 2.1](../chapter-08.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Vì sao lời gọi storage engine tốn nhất

*Vì sao sách nhấn mạnh lời gọi storage engine để lấy hàng là nơi tiêu nhiều thời gian nhất? Khi nào phần đó chủ yếu tốn CPU, khi nào chủ yếu tốn I/O?*

Thực thi gồm **rất nhiều** lời gọi tới storage engine để lấy hàng (tr. 192): số lời gọi tăng theo số hàng được đọc, nên đây là phần lớn nhất với đa số truy vấn. Mỗi lời gọi tốn thao tác bộ nhớ, CPU, và đặc biệt là **I/O nếu dữ liệu không nằm trong bộ nhớ**; tùy engine còn có context switch và system call (tr. 192).

- **Chủ yếu CPU** khi working set nằm trong buffer pool: mỗi lần "đọc page" chỉ là truy cập bộ nhớ. Trong TN1, `SELECT *` tạo 200.447 lượt đọc page, toàn bộ từ bộ nhớ (**Kết quả đã đo**).
- **Chủ yếu I/O** khi page phải đọc từ đĩa, nhất là đọc **ngẫu nhiên** như quay về clustered index cho từng hàng của một index phụ. Khi đó mỗi lượt đọc page có thể thành một I/O (đó cũng là đơn vị chi phí gốc của optimizer, tr. 205).

[về mục 2.1](../chapter-08.md#s2-1)

<a id="q2-2-1"></a>

### Câu 2.2.1 — Lấy 100 hàng, đọc 10 rồi đóng

*Ứng dụng chạy SELECT lấy 100 bài mới nhất, đọc 10 hàng đầu rồi đóng result set. Developer tin MySQL chỉ làm việc cho 10 hàng. Lập luận này sai ở đâu?*

MySQL **không** trả kết quả "theo yêu cầu": nó **tính trọn** result set và client library **nhận hết** rồi bỏ phần lớn (tr. 193). Giao thức half-duplex không có cách bảo server dừng giữa chừng: server **đẩy** hàng, client chỉ nhận (tr. 202). Nếu thư viện để mặc định (buffered), cả 100 hàng đã nằm trong bộ nhớ ứng dụng trước khi vòng lặp đọc hàng đầu tiên (tr. 203). Nếu unbuffered, đóng sớm buộc client đọc nốt hoặc ngắt kết nối thô bạo (tr. 202). Cách sửa của sách: thêm `LIMIT 10` (tr. 193). Thêm nữa, nếu ORDER BY không có index hỗ trợ, server có thể phải sắp **cả bảng** để tìm 100 bài mới nhất, nên phần thừa còn lớn hơn 90 hàng.

[về mục 2.2](../chapter-08.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Vì sao `SELECT *` chậm khi mạng không phải nút thắt

*Vì sao `SELECT *` có thể làm truy vấn chậm hơn nhiều ngay cả khi mạng không phải nút thắt? Dùng số liệu TN1 để giải thích.*

Sách nói lấy mọi cột có thể **ngăn covering index** và thêm I/O, bộ nhớ, CPU cho server (tr. 193). Cơ chế (**Bổ sung ngoài sách**): index phụ của InnoDB chỉ chứa cột được index và primary key. Truy vấn chỉ cần `rental_id, customer_id` đọc lá của `idx_customer` theo thứ tự: **106** lượt đọc page cho 50.086 hàng. `SELECT *` cần thêm các cột khác nên **với từng hàng** phải đi từ gốc xuống lá clustered index: **200.447** lượt đọc page, khoảng 4 lượt mỗi hàng (**Kết quả đã đo**). Khi dữ liệu không nằm trong bộ nhớ, mỗi lần tra đó có thể là một random I/O. Byte gửi đi cũng gấp khoảng 10 lần (6,9 MB so với 0,7 MB), nhưng đó là phần nhỏ hơn. Ngoài ra `SELECT *` có rủi ro khi ai đó đổi danh sách cột của bảng (tr. 193).

[về mục 2.2](../chapter-08.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Tỉ lệ đọc so với trả cao có chắc là tệ?

*Hai truy vấn cùng trả 200 hàng: A đọc 200 hàng, B đọc 5.400 hàng. Có chắc B "tệ" và sửa được bằng index không?*

Không chắc. Sách cho đúng ví dụ này: `SELECT actor_id, COUNT(*) ... GROUP BY actor_id` trả 200 hàng nhưng đọc toàn bộ index (tr. 197). Index **không thể** giảm số hàng đọc vì **không có WHERE** để loại hàng; truy vấn về bản chất phải xem mọi hàng để đếm (tr. 198). Sách cũng nói MySQL chỉ báo tổng số hàng đã truy cập, không báo bao nhiêu hàng thực sự góp vào kết quả; muốn biết phải **suy luận về truy vấn** (tr. 198). Nếu B đọc nhiều vì access type kém (ví dụ `ALL` với WHERE chọn lọc) thì index là cách sửa (tr. 195–196). Nếu như ví dụ GROUP BY, các lựa chọn còn lại là covering index (đã dùng, `Using index`), **summary table** cập nhật dần (tr. 198), cache kết quả, hoặc chấp nhận con số gần đúng.

[về mục 2.2](../chapter-08.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Ba cách áp WHERE

*Phân biệt ba cách MySQL áp WHERE và dấu hiệu `EXPLAIN` của chúng. Vì sao thứ tự "tốt tới tệ" lại như vậy? ICP nằm ở đâu?*

Theo tr. 196–197:

1. Áp điều kiện **trong thao tác tra index**, loại hàng ngay ở **storage engine** (access type `ref`, `range`...): chỉ đọc mục khớp.
2. Dùng **covering index** (`Using index`): lọc ở tầng server trên từng mục index, nhưng **không đọc hàng** từ bảng.
3. **Đọc hàng** rồi mới lọc ở server (`Using where`): đọc đầy đủ mọi hàng mà access path đưa lên rồi vứt hàng không khớp.

Thứ tự phản ánh **lượng dữ liệu phải đọc**: (1) đọc ít nhất; (2) đọc nhiều mục hơn nhưng mỗi mục nhỏ và không phải tra bảng; (3) đọc cả hàng đầy đủ cho mọi ứng viên. Ví dụ `film_actor` không index: đọc 5.400 hàng để trả 7 (tr. 196; TN1 đo được 5.400 so với 7). **ICP** (`Using index condition`, từ MySQL 5.6, **Bổ sung ngoài sách**) nằm giữa (1) và (3): engine kiểm tra phần điều kiện chỉ dùng cột của index trên mục index **trước khi** quay về bảng lấy hàng.

[về mục 2.2](../chapter-08.md#s2-2)

<a id="q2-2-5"></a>

### Câu 2.2.5 — 5 ms lúc 3 giờ sáng, 800 ms lúc cao điểm

*Một truy vấn chạy 5 ms lúc 3 giờ sáng và 800 ms lúc cao điểm, plan không đổi. Dùng khái niệm service time và queue time để giải thích, và nêu cách tìm nguyên nhân gốc.*

Response time = **service time** + **queue time** (tr. 194). Plan không đổi nên service time gần như không đổi; phần tăng nằm ở **queue time**: chờ I/O (truy vấn khác đẩy page ra khỏi buffer pool), chờ row lock do các giao dịch ghi, tranh CPU khi concurrency cao, phần cứng bận (tr. 194). Vì vậy sách nói response time không ổn định khi tải thay đổi, và có thể là **triệu chứng** chứ không phải nguyên nhân (tr. 194). Cách tìm (**Bổ sung ngoài sách**): đo riêng từng loại chờ, ví dụ `Lock_time` trong slow log, `sys.innodb_lock_waits`, wait và stage events trong Performance Schema ([Chương 3](../chapter-03.md)), tỉ lệ đọc đĩa của buffer pool, mức dùng CPU lúc cao điểm. So sánh với một ước lượng QUBE (tr. 195) cũng giúp biết service time "hợp lý" là bao nhiêu.

[về mục 2.2](../chapter-08.md#s2-2)

<a id="q2-3-1"></a>

### Câu 2.3.1 — Một truy vấn phức tạp hay nhiều truy vấn đơn giản

*Khi nào tách một truy vấn phức tạp thành nhiều truy vấn đơn giản là có lợi, và khi nào dùng nhiều truy vấn lại là sai lầm?*

Sách cân hai chi phí (tr. 198–199). Một bên: MySQL kết nối và trả lời truy vấn nhỏ rất nhanh, hơn 100.000 truy vấn đơn giản mỗi giây, hơn 2.000 QPS từ một client trên mạng gigabit. Bên kia: phản hồi qua kết nối vẫn chậm so với tốc độ duyệt hàng bên trong, hàng triệu hàng mỗi giây với dữ liệu trong bộ nhớ.

- **Tách có lợi** khi câu phức tạp làm optimizer chọn plan tệ, đọc lặp dữ liệu, hay ngăn cache; vài câu đơn giản dùng index tốt rẻ hơn dù thêm vòng hỏi đáp (tr. 199). Join decomposition là một dạng như vậy (tr. 200).
- **Sai lầm** khi tách vụn: 10 truy vấn một hàng thay cho một truy vấn 10 hàng, hay lấy từng cột (tr. 199). Phép tính (**Bổ sung ngoài sách**): khoảng 0,5 ms mỗi vòng ở 2.000 QPS, nên 1.000 vòng tốn khoảng 0,5 giây, trong khi một truy vấn trả 1.000 hàng tốn vài mili giây.

Quy tắc của sách: khi mọi thứ như nhau, dùng ít truy vấn; còn lại thì cân chi phí và chọn cách ít việc hơn (tr. 199).

[về mục 2.3](../chapter-08.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Vì sao một DELETE khóa 303.660 hàng

*Trong TN5, một câu DELETE xóa 131.039 hàng nhưng khóa 303.660 hàng và chặn UPDATE vào một hàng không liên quan. Giải thích cơ chế. Chia lô thay đổi điều gì?*

**Kết quả đã đo** kèm cơ chế (**Bổ sung ngoài sách**, theo tài liệu InnoDB):

- Với 44% số hàng khớp, optimizer chọn **quét toàn bảng** (`type: ALL`) thay vì đi qua `idx_created`.
- DELETE là locking read. Ở REPEATABLE READ, InnoDB khóa **mọi hàng nó quét qua**, kể cả hàng không thỏa WHERE, và giữ tới commit. Nên gần như cả bảng bị khóa; UPDATE vào hàng id 250.000 (tin nhắn mới) phải chờ rồi lỗi 1205.
- Chia lô `LIMIT 10000`: optimizer chuyển sang `range` trên `idx_created` (chỉ cần 10.000 hàng đầu), nên mỗi transaction chỉ khóa khoảng 10.000 mục index cộng 10.000 hàng (20.009), trong khoảng 0,1 giây, rồi commit và nhả khóa. UPDATE của ứng dụng xen vào giữa các lô, xong sau 2,9 ms.

Đây đúng là điều sách cảnh báo: một truy vấn lớn có thể khóa nhiều hàng lâu và chặn truy vấn nhỏ; transaction nhỏ hơn cũng làm transaction log và replication lag nhẹ hơn (tr. 199–200).

[về mục 2.3](../chapter-08.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Chia lô "cải thiện hiệu năng" theo nghĩa nào

*Chia lô làm tổng thời gian purge dài hơn trong phép đo. Vậy sách nói "cải thiện hiệu năng" theo nghĩa nào? Nêu hai cái giá phải trả khi chia lô.*

Sách nói chia DELETE thành truy vấn cỡ vừa có thể cải thiện hiệu năng đáng kể và **giảm replication lag**; 10.000 hàng mỗi lần đủ lớn để hiệu quả, đủ ngắn để **giảm tác động lên server**; nghỉ giữa các lô để **rải tải** và giảm thời gian giữ khóa (tr. 199–200). Tức là hiệu năng của **cả hệ thống**: các truy vấn khác không bị chặn, transaction nhỏ, replica theo kịp. Hiệu năng của riêng job purge không phải mục tiêu: đo được tổng 1,65 giây so với 1,0 giây (trung vị 3 lần).

Cái giá (**Bổ sung ngoài sách**):

- **Tổng thời gian và số câu lệnh tăng**, cộng thời gian nghỉ.
- **Mất tính nguyên tử**: người khác thấy trạng thái đã xóa một phần; job phải chạy lại được an toàn khi bị ngắt.
- Cần **index** cho điều kiện chia lô, nếu không mỗi lô lại quét bảng. `DELETE ... LIMIT` không ORDER BY chọn hàng không xác định, không an toàn với replication dạng statement.

[về mục 2.3](../chapter-08.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — Join decomposition: lợi và hại

*Nêu ba lý do join decomposition có thể nhanh hơn một câu join, và một tình huống nó phản tác dụng.*

Lý do theo tr. 200 (chọn ba trong năm):

- **Cache hiệu quả hơn:** đối tượng đã có trong cache thì bỏ được truy vấn hoặc rút ngắn danh sách `IN()`.
- **Truy vấn hiệu quả hơn:** danh sách `IN()` cho phép MySQL sắp xếp ID rồi lấy hàng tối ưu hơn join.
- **Giảm truy cập lặp:** join trong truy vấn giống phi chuẩn hóa, có thể đọc lặp cùng dữ liệu; join trong ứng dụng lấy mỗi hàng một lần, giảm cả mạng và bộ nhớ.
- Ngoài ra: giảm tranh chấp khóa; dễ đặt bảng trên các server khác nhau.

Phản tác dụng (**Bổ sung ngoài sách**): khi cache gần như không trúng và các bảng cùng một server, ba vòng hỏi đáp thay cho một mà không bớt được việc gì; khi cần kết quả nhất quán nhưng ba câu chạy ngoài transaction và thấy ba trạng thái dữ liệu khác nhau; khi danh sách ID quá lớn khiến `IN()` tốn kém (tr. 220).

[về mục 2.3](../chapter-08.md#s2-3)

<a id="q2-4-1"></a>

### Câu 2.4.1 — Không dừng được server giữa chừng

*Vì sao client MySQL không thể bảo server ngừng gửi giữa chừng? Hệ quả với lock và tài nguyên server khi ứng dụng dùng unbuffered result rồi xử lý chậm từng hàng?*

Giao thức là **half-duplex**: tại một thời điểm chỉ một bên gửi, không có flow control, không cắt ngắn được message; server **đẩy** hàng khi tạo ra, client chỉ nhận, như "uống từ vòi cứu hỏa" (tr. 202). Muốn bỏ phần còn lại, client phải nhận hết rồi vứt, hoặc ngắt kết nối thô bạo (tr. 202).

Với unbuffered result, server **chưa giải phóng lock và tài nguyên** của truy vấn tới khi mọi hàng được lấy xong (tr. 202–203). Ứng dụng xử lý chậm từng hàng sẽ kéo dài thời gian đó, chặn truy vấn khác cần các lock ấy. **Bổ sung ngoài sách:** với InnoDB, SELECT thường không giữ row lock, nhưng câu lệnh đang mở vẫn giữ read view, làm purge của InnoDB chậm lại; còn SELECT có khóa (`FOR UPDATE`) thì giữ row lock suốt thời gian đó. Cách giảm: dùng buffered cho kết quả cỡ thường, `LIMIT`, hoặc `SQL_BUFFER_RESULT` để server đưa kết quả vào bảng tạm và nhả lock sớm (chú thích 5 và 13, tr. 203, 218).

[về mục 2.4](../chapter-08.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — Vì sao optimizer chọn sai plan

*Nêu bốn lý do optimizer có thể chọn plan không tốt nhất. Với hai lý do, mô tả dữ liệu hoặc workload cụ thể khiến nó xảy ra.*

Bảy lý do của sách (tr. 206): statistics không chính xác; chỉ số chi phí không bằng chi phí thật; "tối ưu" của MySQL là chi phí thấp chứ không phải thời gian ngắn; không xét truy vấn đồng thời; có lúc theo luật chứ không theo chi phí (ví dụ `MATCH()` luôn dùng FULLTEXT index); không tính chi phí stored function hay UDF; không xét hết mọi plan.

Ví dụ cụ thể:

- **Statistics:** InnoDB lấy mẫu page để ước lượng. Trong TN2, `rental` có 1.000.000 hàng nhưng statistics ghi 992.250; ước lượng 7,41 hàng mỗi lần tra `film_actor` theo `film_id`, thực tế 5,4 (**Kết quả đã đo**). Với dữ liệu lệch mạnh, ví dụ một giá trị chiếm 90% số hàng, ước lượng trung bình có thể khiến optimizer chọn index cho đúng giá trị đó, dẫn tới hàng trăm nghìn lần quay về bảng (**Ví dụ minh họa do AI xây dựng**).
- **Chi phí stored function:** `WHERE fn_check(col) = 1` với hàm đắt (mỗi lần gọi tự chạy một truy vấn khác) bị optimizer coi như phép so sánh rẻ, nên nó không có lý do ưu tiên plan gọi hàm ít lần hơn (**Ví dụ minh họa do AI xây dựng**).

[về mục 2.4](../chapter-08.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Vì sao bắt đầu từ `actor`

*Trong ví dụ join `actor`, `film_actor`, `film`, vì sao optimizer bắt đầu từ `actor`? Số lần tra bảng thứ ba có giảm theo không?*

Sách giải thích: bắt đầu từ `actor` (200 hàng) chỉ cần khoảng 200 lần dò vào bảng sau, còn từ `film` (1.000 hàng) cần khoảng 1.000; thứ tự đảo ít quay lui và đọc lại hơn (tr. 215–216). Chú thích 12: optimizer thật ra tối thiểu số lần **đọc page**, không phải số hàng.

**Kết quả đã đo** (TN2): chi phí 2.501,5 so với 3.688,3; `loops` vào `film_actor` là 200 so với 1.000; `Handler_read_key` 5.601 so với 6.401; `Handler_read_rnd_next` 201 so với 1.001. Nhưng số lần tra **bảng thứ ba** bằng nhau (5.400 ở cả hai), vì đó là số cặp phim–diễn viên, không phụ thuộc thứ tự. Phần tiết kiệm nằm ở bảng đầu (1 page của `actor` so với 15 page của `film`) và ở số lần dò bảng giữa.

[về mục 2.4](../chapter-08.md#s2-4)

<a id="q2-4-4"></a>

### Câu 2.4.4 — Hash join và index nested-loop

*Từ 8.0.20, khi nào MySQL dùng hash join và được lợi gì so với nested-loop không index? Trường hợp nào index nested-loop vẫn là lựa chọn tốt hơn?*

Sách: từ 8.0.20 block nested-loop không còn được dùng, hash join thay thế, nhanh bằng hoặc hơn, nhất là khi một tập dữ liệu vừa bộ nhớ (tr. 212). **Bổ sung ngoài sách:** hash join có từ 8.0.18 cho inner equi-join không có index dùng được; từ 8.0.20 áp cả cho outer join, semijoin, antijoin, join không bằng, ở mọi chỗ trước đây dùng BNL.

**Lợi:** mỗi đầu vào chỉ đọc **một lần**; pha build dựng bảng băm trên đầu vào nhỏ, pha probe tra băm cho từng hàng của đầu vào kia. TN2 đo được 600 hàng đọc, 0,78 ms, so với 120.000 hàng và 102 ms khi ép nested-loop bằng `NO_BNL` (**Kết quả đã đo**).

**Index nested-loop tốt hơn** khi bảng ngoài, sau khi lọc, chỉ còn **ít hàng** và bảng trong có index trên cột join: vài lần tra index rẻ hơn nhiều so với đọc trọn một bảng để dựng bảng băm. Ví dụ `film.film_id = 1` join `film_actor`: 1 hàng ngoài, 7 lần đọc qua index. Hash join cũng cần bộ nhớ (`join_buffer_size`) và tràn đĩa khi đầu vào build lớn.

[về mục 2.4](../chapter-08.md#s2-4)

<a id="q2-4-5"></a>

### Câu 2.4.5 — ORDER BY trên bảng thứ hai của join

*ORDER BY trên cột của bảng thứ hai trong join cho `Using temporary; Using filesort`. Giải thích vì sao, và vì sao theo sách LIMIT không cứu được bao nhiêu. MySQL 8.0 có cơ chế gì làm nhẹ bớt khi LIMIT nhỏ?*

Nếu ORDER BY chỉ dùng cột của bảng **đầu tiên** trong thứ tự join, MySQL filesort bảng đó trước rồi join theo thứ tự đã sắp (`Using filesort`). Còn lại, thứ tự cuối chỉ biết được **sau khi join**, nên MySQL đổ kết quả join vào **bảng tạm** rồi filesort bảng tạm (tr. 217). `LIMIT` được áp **sau** filesort, nên bảng tạm và sort vẫn chứa toàn bộ kết quả join, có thể rất lớn (tr. 217).

**Bổ sung ngoài sách:** khi N của LIMIT đủ nhỏ để vừa sort buffer, filesort dùng **priority queue**, chỉ giữ N hàng tốt nhất trong bộ nhớ. TN3 đo được với `LIMIT 20` trên 1 triệu hàng: bộ nhớ đỉnh 8.925 byte, không tràn đĩa. Nhưng mọi hàng **vẫn phải được đọc** (và với join, vẫn phải ghi vào bảng tạm). Cách sửa gốc theo sách: để ORDER BY chỉ dùng cột của **một** bảng có index hỗ trợ thứ tự (tr. 223).

[về mục 2.4](../chapter-08.md#s2-4)

<a id="q2-5-1"></a>

### Câu 2.5.1 — Còn cần đẩy LIMIT vào nhánh UNION không

*Trên 8.0.42, câu UNION ALL + `LIMIT 20` đầu tiên của sách không tạo bảng tạm và chỉ đọc 20 hàng. Vậy lời khuyên đẩy `LIMIT` vào từng nhánh còn cần không? Khi nào?*

Vẫn cần, với câu **đúng ngữ nghĩa**. Câu đầu của sách không có ORDER BY chung nên thứ tự là không xác định; ORDER BY trong nhánh không kèm LIMIT bị bỏ qua (**Bổ sung ngoài sách**), nên 8.0.42 trả thẳng và dừng sau 20 hàng bất kỳ. Sách cũng nói phải thêm ORDER BY chung trước LIMIT cuối (tr. 219). Khi viết đúng như vậy, MySQL phải materialize cả 799 hàng rồi sắp; đẩy `ORDER BY ... LIMIT 20` vào từng nhánh giảm bảng tạm xuống 40 hàng (**Kết quả đã đo**: 799 so với 40 lần ghi, khớp con số "40 rows" của sách). Nên áp lời khuyên khi có ORDER BY chung kèm LIMIT, và khi dùng UNION (không ALL) vì loại trùng luôn cần bảng tạm. Các bản 8.0 gần đây tự đẩy được **WHERE** vào derived table chứa UNION, nhưng không đẩy LIMIT và ORDER BY.

[về mục 2.5](../chapter-08.md#s2-5)

<a id="q2-5-2"></a>

### Câu 2.5.2 — Equality propagation và IN() rất lớn

*Vì sao equality propagation thường có lợi, nhưng có thể làm chậm khi đi cùng một danh sách `IN()` rất lớn? Bạn sẽ thiết kế lại truy vấn thế nào?*

Có lợi vì điều kiện trên một cột được áp cho cột bằng nó ở bảng khác, cho optimizer thêm lựa chọn: dùng index ở bảng kia, lọc sớm hơn (tr. 210). TN2 đo được: viết `film.film_id > 990`, MySQL dùng range trên `film_actor`. Với `IN()` khổng lồ, optimizer **chép** cả danh sách sang mọi cột liên quan; danh sách rất lớn làm tối ưu và thực thi chậm hơn, và lúc viết sách không có cách lách sẵn (tr. 220). **Bổ sung ngoài sách:** phân tích range cho mỗi bản sao tốn thời gian và bộ nhớ (`eq_range_index_dive_limit`, `range_optimizer_max_mem_size`). Thiết kế lại: nạp ID vào bảng tạm có primary key rồi join; hoặc chia danh sách thành nhiều lô vừa phải; hoặc xem lại join decomposition để không phải chuyển danh sách lớn qua lại (tr. 200).

[về mục 2.5](../chapter-08.md#s2-5)

<a id="q2-5-3"></a>

### Câu 2.5.3 — Không có thực thi song song

*"MySQL không chạy song song một truy vấn" ảnh hưởng thế nào tới cách tối ưu một truy vấn báo cáo lớn? Ngoại lệ hẹp nào có trong 8.0?*

Một truy vấn chỉ dùng một CPU (tr. 220), nên thời gian của nó bị chặn bởi tốc độ một luồng; thêm lõi không giúp truy vấn đó. Hướng tối ưu là **đọc ít hơn**: index và covering index, summary table, tính trước (tr. 198); hoặc ứng dụng tự chia việc theo khoảng khóa và chạy trên nhiều kết nối hay nhiều replica (**Bổ sung ngoài sách**). Ngoại lệ (**Bổ sung ngoài sách**): từ 8.0.14, InnoDB đọc clustered index song song cho `SELECT COUNT(*)` không WHERE và `CHECK TABLE`, theo `innodb_parallel_read_threads` (mặc định 4); `FORMAT=TREE` hiện `Count rows in ...` (TN4). Đây không phải parallel query tổng quát.

[về mục 2.5](../chapter-08.md#s2-5)

<a id="q2-5-4"></a>

### Câu 2.5.4 — Lỗi 1093 và derived table

*Vì sao derived table có GROUP BY lách được lỗi 1093, còn ép `MERGE` thì lỗi quay lại?*

MySQL không cho đọc bảng đích của UPDATE trong cùng câu lệnh (tr. 220). Derived table có GROUP BY không gộp được vào truy vấn ngoài nên được **materialize** thành bảng tạm: thực chất là hai truy vấn, SELECT trong subquery mở và đóng bảng xong trước khi UPDATE mở bảng (tr. 220–221). Ép `MERGE` thì derived table thành một phần của truy vấn ngoài, tức lại đọc trực tiếp bảng đích, nên lỗi 1093 quay lại (**Kết quả đã đo**). TN4 còn cho thấy derived table **không** có GROUP BY cũng chạy được vì 8.0.42 tự materialize nó. Vậy điều kiện thật là **materialization**, không phải cú pháp derived table. **Bổ sung ngoài sách:** giới hạn này tránh việc phần đọc thấy những hàng vừa bị chính câu đó sửa.

[về mục 2.5](../chapter-08.md#s2-5)

<a id="q2-6-1"></a>

### Câu 2.6.1 — COUNT(\*) và COUNT(col)

*`COUNT(*)` và `COUNT(col)` khác nhau thế nào về ngữ nghĩa và về plan? Khi nào hai câu cho kết quả khác nhau? Vì sao `ROWS_EXAMINED = 0` của `COUNT(*)` trong Performance Schema không có nghĩa là rẻ?*

`COUNT(col)` đếm **giá trị** khác NULL; `COUNT(*)` đếm **hàng**, không bung `*` thành cột (tr. 221). Khi MySQL biết biểu thức không bao giờ NULL, nó đếm hàng (tr. 221). TN4 (**Kết quả đã đo**): `COUNT(*)`, `COUNT(rental_id)`, `COUNT(customer_id)` (NOT NULL) đều thành `Count rows in rental`; `COUNT(return_date)` (nullable, không index) cho **900.000** thay vì 1.000.000 và phải quét bảng để xem từng giá trị. Hai câu khác kết quả khi cột có NULL. Muốn đếm hàng thì dùng `COUNT(*)` (tr. 221). `ROWS_EXAMINED = 0` vì việc đếm nằm trong InnoDB, không đi qua đường đếm hàng của tầng server; thực tế nó vẫn duyệt khoảng một triệu bản ghi, mất khoảng 260 ms (**Bổ sung ngoài sách**).

[về mục 2.6](../chapter-08.md#s2-6)

<a id="q2-6-2"></a>

### Câu 2.6.2 — Đếm nhiều giá trị trong một truy vấn

*Vì sao `COUNT(color = 'blue' OR color = 'red')` không đếm riêng từng màu, và trong TN4 biểu thức tương tự còn trả 1.000.000? Viết lại cho đúng theo hai cách.*

Một biểu thức chỉ cho **một** con số, nên không tách được hai màu (tr. 222). Hơn nữa, phép so sánh trả **0** (không phải NULL) khi sai, và `COUNT()` đếm mọi giá trị khác NULL, nên mọi hàng có `color` khác NULL đều được đếm. TN4 đo được `COUNT(customer_id = 1 OR customer_id = 2)` = 1.000.000. Hai cách đúng của sách (tr. 222): `SUM(IF(color = 'blue', 1, 0)) AS blue, SUM(IF(color = 'red', 1, 0)) AS red`; hoặc `COUNT(color = 'blue' OR NULL) AS blue, COUNT(color = 'red' OR NULL) AS red`, vì `FALSE OR NULL` là NULL nên hàng không khớp không được đếm. Chú thích 14 thêm dạng `SUM(color = 'blue')`. TN4 cho cả hai cách cùng 1.669 hàng mỗi giá trị.

[về mục 2.6](../chapter-08.md#s2-6)

<a id="q2-6-3"></a>

### Câu 2.6.3 — OFFSET, deferred join, keyset

*So sánh OFFSET lớn, deferred join và keyset theo số hàng phải đọc. Vì sao keyset viết bằng row constructor lại chậm trên 8.0.42?*

- **OFFSET:** sinh OFFSET + N hàng rồi vứt OFFSET hàng (tr. 224). TN3: 500.020 hàng cho 20 hàng (323,8 ms); khi thứ tự không trùng primary key và lấy mọi cột, plan thành quét bảng + filesort, 1.300.020 hàng (1.631 ms).
- **Deferred join:** vẫn duyệt OFFSET + N mục, nhưng trên **covering index** nhỏ, rồi chỉ đọc N hàng đầy đủ (tr. 224). TN3: 300.020 mục index và 20 hàng, 83,3 ms.
- **Keyset:** bắt đầu từ khóa của hàng cuối, chỉ đọc N hàng, **bất kể độ sâu** (tr. 224–225). TN3: 20 hàng, 0,7 ms (PK) và 1,8 ms (index phụ).

Row constructor `(customer_id, rental_id) > (180, 300000)`: trên 8.0.42 optimizer không dựng range từ điều kiện này, mà quét `idx_customer` từ đầu rồi lọc, đọc 299.360 hàng (1.076 ms). Viết dạng mở rộng `customer_id > 180 OR (customer_id = 180 AND rental_id > 300000)` thì có range và chỉ đọc 20 hàng (**Kết quả đã đo**, **Bổ sung ngoài sách**).

[về mục 2.6](../chapter-08.md#s2-6)

<a id="q2-6-4"></a>

### Câu 2.6.4 — SQL_CALC_FOUND_ROWS

*`SQL_CALC_FOUND_ROWS` tốn ở đâu? Trong TN3, nó còn làm plan thay đổi thế nào? Nêu ba cách thay thế của sách.*

Server không đếm được hàng nó không tìm; hint chỉ bảo server **sinh nốt và vứt** phần còn lại của kết quả thay vì dừng khi đủ LIMIT (tr. 225). TN3 (**Kết quả đã đo**): không hint, truy vấn đi range trên `idx_customer` đúng thứ tự và dừng sau 21 hàng (1,5 ms). Có hint, optimizer đổi sang quét toàn bảng + sắp 168.612 hàng, 1.168.612 rows examined, 801,9 ms; kèm cảnh báo deprecated 1287 (từ 8.0.17). Thay thế (tr. 225): link "next" với `LIMIT 21`; lấy và cache khoảng 1.000 hàng rồi phục vụ các trang từ cache; ước lượng bằng `rows` của `EXPLAIN`; hoặc `COUNT(*)` riêng trên covering index (57,5 ms trong TN3).

[về mục 2.6](../chapter-08.md#s2-6)

<a id="q2-6-5"></a>

### Câu 2.6.5 — UNION và UNION ALL

*Chi phí ẩn của UNION (không ALL) nằm ở đâu? Nếu hai nhánh chắc chắn không trùng, chọn gì và vì sao? Lời sách "ALL không bỏ được bảng tạm" còn đúng trên 8.0.42 không?*

Không có ALL, MySQL thêm tùy chọn distinct vào bảng tạm và dùng **cả hàng** để xét trùng (tr. 226): mọi hàng phải ghi vào bảng tạm có ràng buộc duy nhất; bảng lớn thì tràn ra đĩa. TN4 (**Kết quả đã đo**): 200.000 hàng, UNION 1.059 ms với một bảng tạm trên đĩa và 400.000 rows examined; UNION ALL 298 ms, không bảng tạm. Hai nhánh không trùng thì chọn **UNION ALL**: kết quả y hệt, bỏ cả bước loại trùng (tr. 226). Câu "ALL không bỏ được bảng tạm" (tr. 226) **không còn đúng** trên 8.0.42 khi không có ORDER BY chung: plan là `Append` và `Stream results`, `Created_tmp_tables` = 0. Có ORDER BY chung thì UNION ALL vẫn materialize.

[về mục 2.6](../chapter-08.md#s2-6)

<a id="q2-7-1"></a>

### Câu 2.7.1 — Ba hướng tối ưu và nhiệm vụ con

*Với mỗi hướng "thôi làm, làm ít lần hơn, làm nhanh hơn", chọn một kỹ thuật trong chương và giải thích nó tác động vào nhiệm vụ con nào.*

Ví dụ một cách trả lời (tr. 226 cho ba hướng):

- **Thôi làm:** bỏ `SQL_CALC_FOUND_ROWS` (tr. 225). Nhiệm vụ con bị bỏ: sinh và sắp các hàng sau trang hiện tại, cụ thể là 1,17 triệu hàng đọc và sort 168.612 hàng trong TN3.
- **Làm ít lần hơn:** keyset thay OFFSET (tr. 224–225). Nhiệm vụ con "gọi storage engine lấy hàng" giảm từ OFFSET + N lần xuống N lần.
- **Làm nhanh hơn:** covering index (tr. 198). Mỗi hàng vẫn được đọc, nhưng từ lá index phụ thay vì một lần tra clustered index; TN1 đo 106 so với 200.447 lượt đọc page.

[về mục 2.7](../chapter-08.md#s2-7)

<a id="q2-7-2"></a>

### Câu 2.7.2 — Truy cập một bảng dựa trên bảng khác

*Summary nói chiều mới của chương 8 là truy cập một bảng dựa trên dữ liệu tìm được ở bảng khác. Liên hệ ý này với join optimizer và lời khuyên "chỉ cần index ở bảng thứ hai".*

Trong nested-loop join, MySQL đọc một hàng ở bảng trước rồi dùng **giá trị** của nó để tìm hàng khớp ở bảng sau (tr. 212); summary gọi đó là chiều mới so với chương 7 (tr. 226). Vì vậy:

- Join optimizer chọn bảng nào **dẫn** (đọc trước) để số lần dò bảng sau ít nhất (tr. 215–216).
- Index cần nằm trên cột join của bảng **được dò**, tức bảng thứ hai trong thứ tự join; index trên cột join của bảng dẫn không giúp gì cho việc join (tr. 223).
- Hai việc phụ thuộc nhau: thêm hay bỏ một index có thể đổi thứ tự join tối ưu, và không có index ở bảng được dò thì 8.0.20+ chuyển sang hash join (tr. 212).

[về mục 2.7](../chapter-08.md#s2-7)

<a id="q2-7-3"></a>

### Câu 2.7.3 — Lời khuyên phụ thuộc phiên bản

*Vì sao lời khuyên ở chương này "phụ thuộc phiên bản"? Nêu ba chỗ MySQL 8.0.42 đã khác lời sách, và rút ra nguyên tắc làm việc.*

Optimizer và execution engine thay đổi qua từng bản; tác giả nói rõ phần lớn lời khuyên phụ thuộc phiên bản và server có thể tự làm những tối ưu này trong tương lai (tr. 221). Ba chỗ khác (**Kết quả đã đo** trên 8.0.42):

1. **UNION ALL** không có ORDER BY chung không dùng bảng tạm (sách: luôn dùng, tr. 226).
2. **Cost model** phân biệt page nóng và lạnh: 101.203 so với 107.138 cho cùng một quét bảng (sách: không tính cache, tr. 206).
3. **`EXPLAIN EXTENDED`** báo lỗi cú pháp; `SQL_NO_CACHE` và `SQL_CALC_FOUND_ROWS` sinh cảnh báo deprecated (sách dùng cả ba, tr. 205, 212, 225).

Có thể kể thêm: biến tăng với "range checked for each record" là `Select_range_check` (tr. 207); filesort dùng priority queue cho LIMIT nhỏ.

Nguyên tắc: coi lời khuyên trong sách là **giả thuyết**, rồi kiểm chứng trên đúng phiên bản đang chạy bằng `EXPLAIN FORMAT=TREE`, `EXPLAIN ANALYZE`, `Handler_*` và Performance Schema. Đo lại sau mỗi lần nâng cấp; phụ lục A của sách bàn việc thử nghiệm nâng cấp.

[về mục 2.7](../chapter-08.md#s2-7)
