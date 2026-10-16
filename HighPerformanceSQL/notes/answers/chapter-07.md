# Đáp án — Chương 7: Indexing for High Performance

Quay lại ghi chú: [../chapter-07.md](../chapter-07.md). Hãy tự trả lời trước khi đọc. "tr. N" là số trang in; nhãn **Bổ sung ngoài sách**, **Ví dụ minh họa do AI xây dựng**, **Kết quả đã đo** dùng như trong ghi chú chương.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Index `(last_name, first_name, dob)` và truy vấn `WHERE first_name = 'Bill'`

*Với index `(last_name, first_name, dob)`, truy vấn `WHERE first_name = 'Bill'` được thực thi thế nào trên MySQL 5.7 và trên 8.0.42? Cần những điều kiện gì để MySQL 8.0 dùng được index?*

- Theo sách, B-tree chỉ dùng được khi tra cứu bắt đầu từ **cột trái nhất**; index này không giúp tìm mọi người tên Bill (tr. 160). Trên 5.7 (và trên 8.0 nếu truy vấn cần cột ngoài index), optimizer quét toàn bảng hoặc quét toàn index nếu truy vấn chỉ cần cột trong index.
- **Bổ sung ngoài sách:** MySQL 8.0.13 thêm **skip scan**. Optimizer coi `last_name` như một chuỗi nhóm, nhảy qua từng giá trị phân biệt của `last_name` và tra `first_name = 'Bill'` trong mỗi nhóm. Điều kiện chính: một bảng, truy vấn chỉ dùng cột của index (covering), không có `GROUP BY` hay `DISTINCT`, và cột đầu nên có ít giá trị phân biệt để số lần nhảy nhỏ.
- **Kết quả đã đo:** truy vấn chỉ lấy ba cột của index cho `Using index for skip scan`, 4.001 lần tra, 1.680 mục đọc, khoảng 30 ms. Truy vấn cần thêm cột `bio` thì quét bảng 500.000 hàng, khoảng 275 ms.
- Ý chính: skip scan là bản vá cho chú thích 3 (tr. 159), không phải lý do để bỏ qua thứ tự cột; nếu `first_name` là điều kiện thường gặp, vẫn nên có index bắt đầu bằng `first_name`.

[về mục 2.1](../chapter-07.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — `key_len` đủ ba cột có chứng tỏ `dob` thu hẹp phạm vi quét?

*Truy vấn `WHERE last_name='Smith' AND first_name LIKE 'J%' AND dob='1976-12-23'` có `EXPLAIN` báo `key_len` đủ ba cột. Điều đó có chứng tỏ `dob` thu hẹp phạm vi quét không? Kiểm tra bằng cách nào?*

- Không. Sách nói engine không tối ưu được các cột bên phải điều kiện khoảng đầu tiên; truy cập index chỉ dùng hai cột đầu, server dùng cột còn lại "cho mục đích khác" (tr. 160–161).
- **Kết quả đã đo:** `EXPLAIN` trên 8.0.42 báo `key_len` 407 (đủ ba cột). Nhưng:
  - optimizer trace ghi khoảng quét là `last_name = 'Smith' AND 'J' <= first_name <= 'J…'`, không có `dob`;
  - tắt `index_condition_pushdown` thì `Handler_read_next` = 23, đúng bằng số mục họ Smith có tên bắt đầu bằng J; nếu `dob` thu hẹp khoảng thì chỉ cần đọc 1 mục.
- "Mục đích khác" ở đây là **index condition pushdown**: `dob` được so ngay trên mục index để tránh đọc hàng, nhưng các mục vẫn phải duyệt. Khi bật ICP, `Handler_read_next` chỉ còn 1 vì nó không đếm mục bị loại trong engine; đây là bẫy khi đọc bộ đếm.
- Cách kiểm tra: optimizer trace (`range_scan_alternatives`), so `Handler_read_next` khi tắt và bật ICP, hoặc so với phiên bản viết lại bằng `first_name IN (...)` (20 lần tra, 1 mục đọc).

[về mục 2.1](../chapter-07.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — B-tree giúp `WHERE` lẫn `ORDER BY`; full-text thì không thay B-tree được

*Vì sao một B-tree giúp được cả `WHERE` lẫn `ORDER BY`, còn full-text index không thay thế được B-tree trên cùng cột?*

- B-tree lưu giá trị **đã sắp thứ tự**. Cùng một tính chất cho hai công dụng: tìm vị trí (tra bằng, khoảng, tiền tố) và đọc ra theo thứ tự. Sách nói nếu B-tree giúp tìm hàng theo một cách thì nó giúp sắp theo cùng tiêu chí (tr. 160), và B-tree khai thác được cho `ORDER BY`, `GROUP BY` (tr. 161).
- Full-text index tìm **từ khóa** trong văn bản, có xử lý stop word, stemming, số nhiều, Boolean; nó giống search engine hơn là so khớp `WHERE` (tr. 161). Nó phục vụ `MATCH AGAINST`, không phục vụ điều kiện `WHERE` thông thường hay sắp xếp theo giá trị cột. Vì vậy có full-text index không làm mất giá trị của B-tree trên cùng cột (tr. 161), và hai index khác loại không bị coi là trùng (chú thích 13, tr. 183).

[về mục 2.1](../chapter-07.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — Ba lợi ích và ba sao; chấm sao cho một truy vấn có khoảng và `ORDER BY`

*Ba lợi ích của index (tr. 161) tương ứng với ba sao của Lahdenmaki và Leach thế nào? Với truy vấn `WHERE status = ? AND created_at > ? ORDER BY created_at LIMIT 50`, hãy đề xuất một index và chấm sao.*

- Tương ứng (tr. 161–162, 189–190):
  - "Giảm dữ liệu phải xem xét" và "biến random I/O thành sequential I/O" ↔ **sao 1**: hàng liên quan nằm cạnh nhau, đọc một lát cắt liền.
  - "Tránh sắp xếp và bảng tạm" ↔ **sao 2**: hàng đã đúng thứ tự.
  - Index lưu bản sao giá trị nên trả lời được chỉ từ index ↔ **sao 3**: đủ cột.
- **Ví dụ minh họa do AI xây dựng:** index `(status, created_at)`.
  - Sao 1: `status` so bằng nên các hàng khớp `status` và `created_at > ?` là một khoảng liền.
  - Sao 2: trong một giá trị `status`, mục đã sắp theo `created_at`, đúng `ORDER BY`; ở đây cột khoảng và cột sắp xếp trùng nhau nên không có xung đột.
  - Sao 3: chỉ có nếu truy vấn chỉ cần các cột trong index (cộng PK). Nếu `SELECT *`, index có 2 sao; nhưng với `LIMIT 50`, chỉ khoảng 50 lần tra hàng, chi phí nhỏ.
- Nếu đảo thành `(created_at, status)` thì `status` nằm sau điều kiện khoảng, chỉ còn được lọc trong index (ICP), lát cắt rộng hơn.

[về mục 2.1](../chapter-07.md#s2-1)

<a id="q2-1-5"></a>

### Câu 2.1.5 — Adaptive hash index và kết quả "40% lượt tìm qua hash, thời gian không đổi"

*Adaptive hash index khác một hash index do người dùng tạo ở điểm nào? Vì sao thí nghiệm thấy 40% lượt tìm đi qua AHI mà thời gian gần như không đổi?*

- Khác biệt (tr. 159): AHI do InnoDB **tự dựng trong bộ nhớ**, đặt chồng lên B-tree sẵn có, chỉ cho các giá trị được truy cập rất thường xuyên. Người dùng không chọn cột hay khóa; chỉ bật hoặc tắt được (trên 8.0 còn chỉnh được số phân vùng, **Bổ sung ngoài sách**). Dữ liệu vẫn nằm trong B-tree; AHI chỉ là lối tắt, mất đi khi tắt hay khởi động lại. Hash index do người dùng tạo là cấu trúc lưu trữ thật, thuộc về một bảng.
- **Kết quả đã đo:** AHI phục vụ 40.000 trên 100.000 lượt tìm trong B-tree, nhưng thời gian trung vị 2,05 s (bật) so với 2,06 s (tắt).
- Giải thích (**Bổ sung ngoài sách**): mỗi vòng lặp chạy hai câu SQL trong thủ tục lưu trữ; phần lớn thời gian nằm ở phân tích, tối ưu và thực thi câu lệnh, còn đi xuống một cây 3 tầng đã nằm trong buffer pool chỉ tốn vài micro giây. AHI chỉ cắt được phần nhỏ đó. Nó có lợi rõ hơn khi tra bằng cực nhiều trên cây sâu với khóa dài, và có thể có hại khi tải đồng thời cao gây tranh chấp latch; đó là lý do MySQL 8.4 tắt nó mặc định (theo ghi chú phát hành 8.4).

[về mục 2.1](../chapter-07.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Vì sao trường hợp xấu nhất quan trọng với prefix index

*Trong thí nghiệm `city_demo`, selectivity trung bình của tiền tố 4 ký tự bằng 70% của cả cột, nhưng nhóm "San " có số hàng gấp 20 lần một tên đầy đủ. Vì sao trường hợp xấu nhất quan trọng với prefix index, và bạn phát hiện nó bằng truy vấn nào?*

- Selectivity là **trung bình** trên mọi giá trị. Nhưng mỗi truy vấn cụ thể tra **một** giá trị. Truy vấn tra "San Felipe" bằng `city(4)` phải đọc mọi mục bắt đầu bằng "San " (khoảng 10.000), rồi tra từng hàng để so tên đầy đủ (prefix index luôn cần `Using where`). Chi phí của truy vấn đó do kích thước nhóm tiền tố quyết định, không do trung bình (tr. 165).
- Dữ liệu thật càng lệch thì càng tệ: tên thành phố thật có rất nhiều "San" và "New" (tr. 165).
- Phát hiện: ngoài `COUNT(DISTINCT LEFT(col, n))/COUNT(*)`, chạy `SELECT COUNT(*) AS c, LEFT(col, n) AS pref FROM t GROUP BY pref ORDER BY c DESC LIMIT 10` cho từng `n`, và so số hàng của tiền tố phổ biến nhất với số hàng của giá trị đầy đủ phổ biến nhất (tr. 163–165).
- **Kết quả đã đo:** phải tới 9 ký tự thì cả trung bình lẫn trường hợp xấu nhất mới tương đương cả cột; với 8 ký tự, trung bình đã bằng nhưng nhóm xấu nhất vẫn gấp đôi.

[về mục 2.2](../chapter-07.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Khi nào thứ tự cột quan trọng

*Với truy vấn chỉ gồm `WHERE staff_id = 2 AND customer_id = 584`, thứ tự `(staff_id, customer_id)` hay `(customer_id, staff_id)` có khác nhau không? Khi nào thứ tự bắt đầu quan trọng, và khi đó nguyên tắc nào thắng quy tắc "chọn lọc nhất đặt đầu"?*

- Với truy vấn này: **gần như không khác**. Cả hai index cho cùng một lát cắt (các mục có đúng cặp giá trị); **Kết quả đã đo:** cả hai đều `ref` trên hai cột, đọc 15 mục.
- Thứ tự bắt đầu quan trọng khi:
  1. có truy vấn chỉ dùng **một phần** index: `WHERE customer_id = 584` chỉ dùng được `(customer_id, ...)`; đây là trường hợp sách nói quy tắc chọn lọc có ích (tr. 167–168);
  2. có **điều kiện khoảng**: các cột sau cột khoảng không thu hẹp phạm vi quét (tr. 160). **Kết quả đã đo:** `staff_id = 2 AND customer_id BETWEEN 500 AND 600` đọc 1.664 mục với `(staff_id, customer_id)` nhưng 3.634 mục với `(customer_id, staff_id)`, dù `staff_id` gần như không chọn lọc;
  3. có `ORDER BY`, `GROUP BY` hay `DISTINCT` cần thứ tự (tr. 167).
- Nguyên tắc thắng: **cột so bằng trước, cột sắp xếp tiếp theo, cột khoảng sau cùng**, rồi mới xét selectivity. Sách tóm tắt: sắp xếp, nhóm và điều kiện khoảng tạo khác biệt lớn hơn nhiều (tr. 170); và không nên chọn theo quy tắc "chọn lọc nhất đặt đầu" một cách máy móc (tr. 190).
- Đừng quên giá trị đặc biệt: khách "guest" có 39.948 hàng thì thứ tự nào cũng phải đọc chừng ấy (tr. 169–170).

[về mục 2.2](../chapter-07.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Vì sao secondary index lưu PK

*Vì sao secondary index của InnoDB lưu giá trị PK thay vì vị trí vật lý của hàng? Nêu một hệ quả tốt và hai hệ quả xấu về hiệu năng.*

- Lý do (tr. 174): dùng PK làm "con trỏ" thì khi hàng **di chuyển** (page split, cập nhật làm hàng không vừa chỗ cũ, `OPTIMIZE`), InnoDB không phải sửa con trỏ trong mọi secondary index.
- Hệ quả tốt:
  - giảm công bảo trì secondary index khi hàng dịch chuyển (tr. 174);
  - secondary index có sẵn PK như "cột phụ", nên cover được truy vấn lấy PK (tr. 179) và dùng được cho `WHERE A = ? ORDER BY ID` (tr. 185).
- Hệ quả xấu:
  - **hai lần tra** cho mỗi hàng: tìm trong secondary rồi đi lại từ root của clustered index (tr. 172–173);
  - secondary index **to hơn**, nhất là khi PK dài (tr. 172, 174); PK `CHAR(36)` bị lặp lại trong mọi secondary index.
- **Kết quả đã đo:** Q2 đọc khoảng 32.000–43.000 page buffer pool khi phải tra hàng, so với khoảng 207 page khi index cover.

[về mục 2.2](../chapter-07.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Chèn PK ngẫu nhiên so với tuần tự

*Dự đoán và giải thích: chèn 300.000 hàng với PK `INT` theo thứ tự ngẫu nhiên so với `AUTO_INCREMENT`, khi buffer pool lớn hơn bảng và khi nhỏ hơn bảng. Vì sao `UUID()` của MySQL trong thí nghiệm lại không gây phân mảnh?*

- Khi vừa bộ nhớ: thứ tự truy cập ít quan trọng (tr. 172). Chèn ngẫu nhiên vẫn gây page split nên page thưa hơn và tốn CPU dời dữ liệu (tr. 177), nhưng không có I/O. **Kết quả đã đo:** 6,0 s so với 4,6 s; 4.095 so với 2.440 trang lá.
- Khi bảng lớn hơn bộ nhớ: page đích của mỗi lần chèn ngẫu nhiên thường đã bị đẩy khỏi buffer pool, phải đọc lại từ đĩa, rồi page bẩn lại phải ghi ra (tr. 177). **Kết quả đã đo** (buffer pool 16 MB): 63,7 s so với 4,4 s, khoảng 193.000 lần đọc page so với khoảng 1.200. Tương ứng hàng 3 triệu của Bảng 7-1 (tr. 175).
- `UUID()` của MySQL sinh UUID **phiên bản 1** dựa trên thời gian (**Bổ sung ngoài sách**). Phần đầu chuỗi là phần thấp của bộ đếm thời gian, tăng đều trong vài phút, nên các UUID sinh liên tiếp trong một câu `INSERT ... SELECT` gần như tăng dần: InnoDB vẫn ghi vào cuối cây. **Kết quả đã đo:** 3.094 trang lá trước và 3.093 sau `OPTIMIZE`, tức không có phân mảnh; bảng chỉ to hơn do khóa dài. UUID thật sự ngẫu nhiên (dạng MD5 trong thí nghiệm, tương tự UUID phiên bản 4 do ứng dụng sinh) thì có 4.413 trang lá. Bài học: điều quan trọng là khóa có **tăng đơn điệu** không (tr. 178), không phải nhãn "UUID".

[về mục 2.2](../chapter-07.md#s2-2)

<a id="q2-2-5"></a>

### Câu 2.2.5 — Thay `(A)` bằng `(A, B)` an toàn thế nào

*Bảng có index `(A)`; một đồng nghiệp đề xuất thay bằng `(A, B)` để cover thêm truy vấn. Truy vấn nào có thể chậm đi, vì sao, và bạn triển khai thay đổi này an toàn thế nào trên MySQL 8.0?*

- Có thể chậm đi:
  - `WHERE A = ? ORDER BY ID`: trong InnoDB, `(A)` thực chất là `(A, ID)` nên trả theo thứ tự PK sẵn; `(A, B)` thành `(A, B, ID)` nên phải filesort (tr. 185). **Kết quả đã đo:** với `(state_id)`, `ORDER BY id LIMIT 10` đọc 10 mục; với `(state_id, city, address)` ép dùng thì đọc 9.835 mục rồi filesort.
  - Truy vấn chỉ cần `A` (như `COUNT(*) WHERE A = ?`) phải đọc index rộng hơn, nhất là khi `B` là `VARCHAR` dài (tr. 183–184; Bảng 7-2: 108,55 xuống 100,33 QPS).
  - Mọi lần ghi phải bảo trì index lớn hơn (tr. 185).
- Triển khai an toàn:
  1. Thu thập truy vấn thật dùng `A` (từ Performance Schema hoặc slow log) và kiểm tra `EXPLAIN` cho cả `(A)` lẫn `(A, B)`; sách gợi ý *pt-upgrade* (tr. 185).
  2. Thêm `(A, B)` trước, giữ `(A)`.
  3. Chuyển `(A)` sang **invisible** (tr. 185), theo dõi response time và `EXPLAIN` qua một chu kỳ nghiệp vụ; có thể thử trước trong một phiên bằng `optimizer_switch='use_invisible_indexes=on'` (**Bổ sung ngoài sách**).
  4. Nếu ổn thì xóa `(A)`; nếu có truy vấn hỏng thì cho hiện lại ngay, không phải dựng lại. Nếu cả hai truy vấn đều quan trọng, giữ cả hai index như Bảng 7-2 gợi ý (tr. 184).

[về mục 2.2](../chapter-07.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — InnoDB bị hỏng là bằng chứng của điều gì

*Vì sao sách nói InnoDB bị hỏng là "bằng chứng" của một vấn đề khác, và việc đầu tiên cần làm là gì? Vì sao không nên chỉ chạy lệnh sửa rồi thôi?*

- InnoDB được thiết kế rất bền trước hỏng hóc và "không nên hỏng" (tr. 186). Nếu hỏng, đó là bằng chứng của lỗi phần cứng như RAM hay đĩa (hay gặp), lỗi quản trị như thao tác file database từ bên ngoài, ví dụ backup bằng `rsync` (hay gặp), hoặc bug của InnoDB (hiếm). Không có câu truy vấn nào làm hỏng InnoDB mà lại là lỗi của bạn (tr. 186–187).
- Việc đầu tiên: **tìm nguyên nhân** (tr. 187). Đồng thời xác định phạm vi bằng `CHECK TABLE` (tr. 186).
- Chỉ sửa rồi thôi thì nguyên nhân vẫn còn và hỏng sẽ quay lại (tr. 187). Ví dụ RAM lỗi sẽ tiếp tục ghi sai page. Cách sửa: dựng lại bảng bằng `ALTER TABLE ... ENGINE=InnoDB`, dump và nạp lại, khôi phục từ backup, hoặc dùng `innodb_force_recovery` để lấy dữ liệu ra (tr. 186–187).

[về mục 2.3](../chapter-07.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Cardinality được tính thế nào và 8.0 khác gì

*InnoDB tính `Cardinality` thế nào? Vì sao `SHOW INDEX` của sách cho 200 với `last_name` dù họ có trùng? Trên MySQL 8.0, những gì đã khác so với mô tả ở tr. 188?*

- InnoDB **lấy mẫu vài page ngẫu nhiên** của index và giả định phần còn lại giống vậy (tr. 188). `Cardinality` là số giá trị phân biệt **ước lượng** (tr. 188).
- Ví dụ của sách: bảng `actor` có 200 hàng, họ 'HOPPER' có 2 hàng (tr. 179–180), nên số họ thực tế nhỏ hơn 200. Con số 200 cho thấy ước lượng có thể lệch, nhất là với bảng nhỏ (**Bổ sung ngoài sách**, quan sát).
- Khác biệt trên 8.0 (**Bổ sung ngoài sách**, biến đã kiểm tra trên 8.0.42):
  - `innodb_stats_sample_pages` không còn; dùng `innodb_stats_persistent_sample_pages` (20) và `innodb_stats_transient_sample_pages` (8), hoặc `STATS_SAMPLE_PAGES` của từng bảng;
  - statistics mặc định **bền vững** trong `mysql.innodb_index_stats`, không tính lại chỉ vì mở bảng, và tự tính lại ở nền khi khoảng 10% số hàng thay đổi;
  - `innodb_stats_on_metadata` mặc định `OFF`, nên `SHOW INDEX` không còn kích hoạt lấy mẫu;
  - `INFORMATION_SCHEMA.STATISTICS` đọc từ cache trong data dictionary, làm mới theo `information_schema_stats_expiry` hoặc khi `ANALYZE TABLE`.
- **Kết quả đã đo:** mẫu 20 page cho 1.974 họ (thực tế 2.000); 200 page cho 2.046. Sau khi InnoDB tự tính lại, `INFORMATION_SCHEMA` vẫn trả giá trị cũ cho tới khi `ANALYZE TABLE` hoặc đặt expiry bằng 0.

[về mục 2.3](../chapter-07.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Cardinality không thấy độ lệch

*30% số hàng bị đổi thành cùng một giá trị nhưng cardinality gần như không đổi. Vì sao? Truy vấn nào vẫn được ước lượng đúng, truy vấn nào có thể bị ước lượng sai?*

- Cardinality chỉ đếm **số giá trị phân biệt**. Đổi 150.000 hàng sang một họ đã có hay một họ mới chỉ làm số họ phân biệt thay đổi rất ít (khoảng 2.000), nên "số hàng trung bình mỗi giá trị" vẫn khoảng 250. Nó không mô tả phân bố.
- Được ước lượng đúng hơn (**Kết quả đã đo**, **Bổ sung ngoài sách** về cơ chế): truy vấn có **hằng số** cụ thể, vì optimizer làm index dive, đi xuống cây để đếm khoảng (tối đa `eq_range_index_dive_limit` = 200 khoảng). Đo được: ước lượng 247.757 hàng cho 'Zzguest' (thực tế 150.000) và 186 cho 'Allen' (thực tế 186). Thứ tự độ lớn đúng, nên plan hợp lý.
- Có thể bị ước lượng sai: khi giá trị **chỉ biết lúc chạy**, như cột join (`ref` từ bảng khác) hoặc danh sách `IN` dài hơn giới hạn dive. Khi đó optimizer dùng trung bình từ cardinality (khoảng 250 hàng) cho cả giá trị 'Zzguest'. Đây chính là cơ chế đằng sau các ví dụ "giá trị đặc biệt" của sách (tr. 169–170).
- Phòng tránh: xử lý giá trị đặc biệt ở tầng ứng dụng như sách khuyên (tr. 170); trên 8.0 có thể xem thêm histogram cho cột không có index (**Bổ sung ngoài sách**).

[về mục 2.3](../chapter-07.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — Ba loại phân mảnh dữ liệu

*Nêu ba loại phân mảnh dữ liệu và thao tác mà mỗi loại làm chậm. Vì sao `OPTIMIZE TABLE` gần như không đổi số trang của bảng `t_uuid` (khóa từ `UUID()` của MySQL)?*

- Theo sách (tr. 189):
  - *Row fragmentation*: một hàng bị chia thành nhiều mảnh ở nhiều nơi; làm chậm cả truy vấn chỉ cần một hàng.
  - *Intra-row fragmentation* (theo mô tả: page hay hàng liền nhau về logic nhưng không liền nhau trên đĩa): làm chậm quét toàn bảng và quét khoảng trên clustered index.
  - *Free space fragmentation*: page có nhiều chỗ trống; server đọc nhiều dữ liệu không cần.
- Ngoài ra index cũng phân mảnh: page đầy kém hay không tuần tự, làm quét khoảng và quét toàn index chậm nhiều lần, nhất là truy vấn covering (tr. 188).
- `t_uuid` gần như không phân mảnh từ đầu: `UUID()` phiên bản 1 sinh liên tiếp gần như tăng dần, nên các page được lấp đầy giống chèn tuần tự (**Bổ sung ngoài sách**). `OPTIMIZE TABLE` chỉ dựng lại cùng bố cục: 3.094 trang lá trước, 3.093 sau (**Kết quả đã đo**). Ngược lại, bảng khóa `INT` ngẫu nhiên giảm từ 4.095 xuống 2.440 trang lá. Bài học: đo trước rồi mới quyết định chạy `OPTIMIZE`, vì trên InnoDB nó chép lại toàn bảng.

[về mục 2.3](../chapter-07.md#s2-3)

---

<a id="q2-4-1"></a>

### Câu 2.4.1 — Ba nguyên tắc, ba sao và dấu hiệu `EXPLAIN`

*Ánh xạ ba nguyên tắc ở Summary sang ba sao và sang các dấu hiệu bạn đọc được từ `EXPLAIN` hoặc `EXPLAIN ANALYZE`.*

| Nguyên tắc (tr. 189–190) | Sao | Dấu hiệu khi **thiếu** (**Bổ sung ngoài sách**) |
| --- | --- | --- |
| Truy cập từng hàng lẻ là chậm | Sao 1: hàng liên quan nằm cạnh nhau | `rows` hay số hàng thật ở bước quét lớn hơn nhiều so với số hàng trả về; `type: ALL`; `Using where` lọc bỏ phần lớn; `key_len` ngắn hơn số cột so bằng |
| Truy cập khoảng theo thứ tự là nhanh | Sao 2: đúng thứ tự | `Using filesort`, `Using temporary`; trong `EXPLAIN ANALYZE` có nút `Sort` |
| Chỉ đọc index là nhanh | Sao 3: đủ cột | Không có `Using index`; mỗi hàng ở bước index cần một lần tra clustered index (thấy qua page buffer pool tăng mạnh) |

Checklist response time của sách (tr. 190) chính là bốn câu hỏi tương ứng: đọc quá nhiều hàng, sắp xếp hay bảng tạm sau khi lấy hàng, random I/O, tra hàng để lấy cột không có trong index.

[về mục 2.4](../chapter-07.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — Ví dụ "không thể có index ba sao" trên MySQL 8.0

*Sách nêu ví dụ không thể có index ba sao. Ví dụ đó còn đúng trên MySQL 8.0 không? Nếu có index phù hợp, vẫn còn rủi ro gì?*

- Ví dụ của sách: truy vấn sắp theo hai cột, một tăng một giảm (tr. 190); tr. 181 cũng nói index có các cột đều tăng nên không dùng được cho `ORDER BY inventory_id DESC, customer_id ASC`. Chú thích 10 (tr. 180) gợi ý mẹo lưu giá trị đảo hay đổi dấu.
- **Bổ sung ngoài sách:** MySQL 8.0 có descending index, nên có thể tạo `(rental_date, inventory_id DESC, customer_id)` khớp đúng chiều sắp xếp. Ví dụ không còn "không thể" về mặt cấu trúc.
- Rủi ro còn lại (**Kết quả đã đo**): optimizer chỉ dùng index mới khi bị ép; để tự chọn, kể cả có `LIMIT 10`, nó vẫn dùng index cũ cộng filesort, vì ước lượng hàng của index mới cao hơn chút (1.084 so với 996). Ngoài ra thêm index là thêm chi phí ghi (tr. 185). Vậy vẫn phải kiểm tra `EXPLAIN`, có thể cần index hint hoặc bỏ index cũ nếu nó thừa.

[về mục 2.4](../chapter-07.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — `WHERE a = ? AND b > ? ORDER BY c LIMIT 10`

*Truy vấn `WHERE a = ? AND b > ? ORDER BY c LIMIT 10` có thể có index ba sao không? Phân tích hai lựa chọn `(a, b, c)` và `(a, c, b)`.*

- Không có index nào đạt đủ cả sao 1 và sao 2 cùng lúc, vì cột khoảng `b` và cột sắp xếp `c` khác nhau (**Bổ sung ngoài sách**, theo cách dựng ba sao).
- `(a, b, c)`: lát cắt hẹp nhất (`a` bằng, `b` khoảng), nhưng trong lát cắt đó hàng sắp theo `b` rồi `c`, không theo `c`; phải đọc **toàn bộ** lát cắt rồi filesort, `LIMIT 10` không giúp dừng sớm (tr. 182: khoảng ở cột trước làm mất khả năng sắp xếp bằng cột sau). Tốt khi `b > ?` rất chọn lọc.
- `(a, c, b)`: trong một giá trị `a`, mục đã sắp theo `c`, nên đọc theo thứ tự và **dừng khi đủ 10 hàng** thỏa `b > ?`; `b` được kiểm tra ngay trên mục index (ICP). Tốt khi `b > ?` khớp nhiều hàng, vì 10 hàng đầu đến sớm; tệ khi `b > ?` hiếm, vì có thể phải duyệt gần hết các mục của `a`.
- Thêm các cột `SELECT` vào cuối để có sao 3 nếu hợp lý. Quyết định dựa trên phân bố thật của `b` và giá trị đặc biệt của `a` (tr. 168–170), và kiểm chứng bằng `EXPLAIN ANALYZE` với cả tham số phổ biến lẫn tham số xấu nhất.

[về mục 2.4](../chapter-07.md#s2-4)
