# Đáp án — Chương 6: Thiết kế và quản lý schema

[← Về ghi chú chương 6](../chapter-06.md) · Số trang "tr. N" là trang in (trang PDF = N + 22). Số liệu "đã đo" lấy từ [bài thực hành chương 6](../experiments/chapter-06.md) (MySQL 8.0.42, Windows 11, instance riêng).

<a id="q2-1-1"></a>

### Câu 2.1.1 — "Cứ khai báo VARCHAR(255) cho thoải mái"

*Một đồng nghiệp nói: "`VARCHAR(255)` và `VARCHAR(50)` tốn cùng chỗ trên đĩa với chuỗi 20 ký tự, nên cứ khai báo 255 cho thoải mái." Phân tích lập luận này theo sách và theo kết quả đo trên MySQL 8.0.42: điều gì đúng, điều gì thiếu, và còn rủi ro nào khác ngoài bộ nhớ?*

**Điều đúng.** Trên đĩa, `VARCHAR` chỉ dùng chỗ cho dữ liệu thật cộng byte độ dài (tr. 129); sách cũng nói `'hello'` tốn cùng chỗ trong `VARCHAR(5)` và `VARCHAR(200)` (tr. 131). E5 xác nhận: hai bảng `VARCHAR(10)` và `VARCHAR(1000)` chứa cùng dữ liệu có cùng `Data_length`.

**Điều thiếu.** Hộp "Generosity Can Be Unwise" nói cột dài hơn có thể dùng **nhiều bộ nhớ hơn hẳn**, vì MySQL thường cấp phát khối kích thước cố định khi xử lý bên trong, đặc biệt khi sắp xếp và dùng bảng tạm (tr. 131); Summary nhắc lại (tr. 154). Trên 8.0.42, mức độ phụ thuộc cấu hình:

- Bảng tạm TempTable (mặc định) và collation NO PAD (mặc định `utf8mb4_0900_ai_ci`): không thấy khác biệt (E5).
- Bảng tạm MEMORY: `GROUP BY` trên `VARCHAR(1000)` tràn ra đĩa, chậm khoảng 5,6 lần (E5).
- Collation PAD SPACE (như `utf8mb4_general_ci`): sort key cố định, 807 chunk thay vì 40, chậm khoảng 5,3 lần (E5).

Vậy lập luận "thoải mái" đặt cược vào cấu hình mặc định và collation hiện tại; đổi một trong hai là vấn đề quay lại.

**Rủi ro khác** (**Bổ sung ngoài sách**):

- Với utf8mb4, `VARCHAR(255)` có thể dài 1.020 byte nên cần **2 byte độ dài**, trong khi `VARCHAR(50)` (200 byte) chỉ cần 1.
- Độ dài khai báo là một **ràng buộc dữ liệu**: khai báo 255 cho một mã 20 ký tự là để rác lọt vào.
- Nới độ dài sau này vượt ngưỡng 255 byte thì cần chép bảng (E6), nên "chừa thật rộng" cũng không giúp gì cho thay đổi sau.

[về mục 2.1](../chapter-06.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Vì sao ENUM nhanh khi join với ENUM, chậm khi join với VARCHAR

*Bảng `webservicecalls` của sách đổi hai cột `VARCHAR` trong khóa chính sang `ENUM`. Giải thích vì sao join ENUM với ENUM nhanh hơn, join ENUM với VARCHAR lại chậm hơn cả VARCHAR với VARCHAR, và vì sao việc khóa chính nhỏ đi một nửa còn có lợi cho cả các index khác.*

- **ENUM với ENUM (3,5 truy vấn mỗi giây):** ENUM lưu số nguyên 1–2 byte chỉ vị trí phần tử (tr. 132), nên so sánh là so sánh **số nguyên nhỏ**, rẻ hơn so sánh chuỗi phải đi qua character set và collation (tr. 126).
- **ENUM với VARCHAR (1,7–1,8):** để so sánh với chuỗi, MySQL phải **tra bảng để đổi số thành chuỗi** rồi mới so sánh theo collation (tr. 133). Mỗi cặp hàng chịu cả chi phí chuyển đổi lẫn chi phí so sánh chuỗi, nên chậm hơn cả VARCHAR với VARCHAR (2,6) (Bảng 6-1, tr. 134).
- **Khóa chính nhỏ đi một nửa** (tr. 134): trong InnoDB, mỗi entry của index phụ chứa **khóa chính** để tìm về hàng (**Bổ sung ngoài sách**, chi tiết ở [chương 7](../chapter-07.md#secondary-index)), nên khóa chính nhỏ làm mọi index phụ nhỏ theo, đúng như sách nói. E2 và E3 cho thấy hiệu ứng này bằng số: index phụ với khóa `BIGINT` nhỏ hơn 2,7 lần so với khóa `CHAR(36)`.
- **Bài học thiết kế:** dùng cùng kiểu cho cột join (tr. 142, 154), hoặc dùng lookup table với khóa số nguyên (tr. 134).

[về mục 2.1](../chapter-06.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Khóa chính CHAR(36) chứa UUID() làm insert chậm dần

*Một bảng có khóa chính `CHAR(36)` chứa giá trị `UUID()` và đang chậm dần khi insert. Dự đoán điều gì xảy ra trong InnoDB (page, buffer pool, index phụ) và đề xuất hai thay đổi theo thứ tự ưu tiên, nêu điều kiện để mỗi thay đổi có tác dụng.*

**Điều xảy ra** (tr. 143–144, kèm số đo E3):

1. Khi các hàng đến rải rác theo thời gian, giá trị UUID v1 dạng chuỗi rơi vào **vị trí ngẫu nhiên** của clustered index, gây **page split** ở giữa page; page chỉ còn đầy khoảng 67% (E3), tức clustered index **phân mảnh** và lớn hơn khoảng 38% so với bản đã rebuild.
2. Khi clustered index lớn hơn buffer pool, mỗi lần chèn có xác suất cao chạm page không có trong bộ nhớ, nên phải **đọc đĩa ngẫu nhiên**: E3 đo hơn 111.000 lần đọc page cho 1 triệu insert, so với vài trăm khi khóa tăng dần. Đây là lý do "chậm dần": khi bảng còn nhỏ, mọi thứ vừa bộ nhớ; khi lớn hơn working set, cache miss tăng (tr. 144).
3. Mỗi index phụ mang theo khóa chính 36 ký tự, nên lớn gấp khoảng 2,7 lần so với khóa `BIGINT` (E3), làm working set càng lớn.

**Hai thay đổi theo thứ tự ưu tiên:**

1. **Đổi khóa chính sang số nguyên tự tăng** (tr. 143), giữ UUID làm cột `UNIQUE` phụ nếu ứng dụng cần. Hiệu quả lớn nhất, nhưng điều kiện là ứng dụng chấp nhận để database sinh khóa, và các bảng con tham chiếu phải đổi theo (cùng kiểu, tr. 142).
2. Nếu bắt buộc giữ UUID: lưu `BINARY(16)` (tr. 144). Riêng với UUID v1 của `UUID()`, dùng `UUID_TO_BIN(u, 1)` để giá trị **tăng theo thời gian** (**Bổ sung ngoài sách**): E3 cho thấy cách này đưa độ đầy page về khoảng 92% và số lần đọc đĩa về vài trăm. Điều kiện: UUID phải có phần thời gian. Với UUID v4 hoàn toàn ngẫu nhiên thì `BINARY(16)` chỉ làm khóa nhỏ lại (E3: 6.162 so với 8.229 page) chứ không hết ngẫu nhiên.

**Lưu ý:** với burst insert rất nhanh, `UUID()` dạng chuỗi lại gần như tăng dần (E3: 200.000 giá trị liên tiếp không giảm lần nào), nên hiện tượng chỉ rõ khi insert rải theo thời gian. Ngoài ra chú thích 6 nhắc rằng ở bảng rất lớn với nhiều luồng ghi, giá trị giả ngẫu nhiên đôi khi giúp tránh điểm nóng (tr. 143), nên cần đo trên chính workload.

[về mục 2.1](../chapter-06.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — JSON lớn hơn vì đâu, và vì sao cần dấu nháy trong WHERE

*Trong thí nghiệm JSON của sách, bảng JSON lớn hơn và đọc chậm hơn bảng cột thường. Nguyên nhân gốc của chênh lệch kích thước là gì với kiểu `JSON` native của MySQL (sách có một chỗ diễn đạt chưa chính xác), và vì sao truy vấn `WHERE designation='"(2010 GW62)"'` của sách phải có cặp dấu nháy kép bên trong?*

**Kích thước.** Sách nói JSON tốn thêm chỗ cho ký tự định nghĩa cấu trúc và khoảng trắng (tr. 140). Với kiểu JSON native, MySQL lưu dạng **nhị phân chuẩn hóa**, không giữ khoảng trắng: E1 đo `JSON_STORAGE_SIZE()` như nhau cho bản viết gọn và bản nhiều khoảng trắng (**Bổ sung ngoài sách**). Nguyên nhân thật:

- **tên khóa lặp lại** trong từng tài liệu (với dữ liệu của sách là 10 tên trường mỗi hàng);
- metadata của định dạng nhị phân (offset, nhãn kiểu);
- dữ liệu của sách lưu **số dưới dạng chuỗi** (đã xem trên ảnh trang PDF 161), trong khi bảng SQL dùng `FLOAT` 4 byte.

E4 tái hiện tỷ lệ: 6 page so với 3 page ở 202 hàng (sách: 5 so với 3), và khoảng 3,1 lần ở 100.000 hàng.

**Dấu nháy.** Cột sinh của sách định nghĩa bằng `json_data->"$.designation"` (tr. 141). Toán tử `->` là `JSON_EXTRACT()`, trả về **giá trị JSON**; một chuỗi JSON được biểu diễn kèm dấu nháy kép, nên cột sinh chứa `"(2010 GW62)"` cả dấu nháy. Sách giải thích ngắn: trường được đặt trong dấu nháy trong JSON, nên phải tìm kèm dấu nháy (tr. 141). E4 cho thấy tìm không kèm dấu nháy trả 0 hàng. Cách gọn hơn (**Bổ sung ngoài sách**): định nghĩa cột sinh bằng `->>` (`JSON_UNQUOTE(JSON_EXTRACT())`), khi đó tìm bằng chuỗi thường, và optimizer 8.0 còn tự dùng index đó khi `WHERE` viết bằng biểu thức `->>` khớp định nghĩa (E4).

[về mục 2.1](../chapter-06.md#s2-1)

<a id="q2-1-5"></a>

### Câu 2.1.5 — DATETIME, TIMESTAMP hay INT cho created_at

*Một ứng dụng chạy ở nhiều múi giờ lưu thời điểm đặt hàng. So sánh `DATETIME`, `TIMESTAMP` và `INT UNSIGNED` (epoch UTC) theo: dung lượng thật trong MySQL 8.0, miền giá trị, hành vi khi session đổi `time_zone`, và hành vi mặc định khi insert không có giá trị. Bạn chọn gì cho cột `created_at` của bảng 500 triệu hàng có index trên cột đó?*

| Tiêu chí | `DATETIME` | `TIMESTAMP` | `INT UNSIGNED` epoch |
| --- | --- | --- | --- |
| Dung lượng (8.0) | 5 byte (+0–3 byte phần lẻ giây); sách ghi 8 byte là định dạng cũ (tr. 135; E2) | 4 byte (+0–3) (tr. 135; E2) | 4 byte (E2: bằng đúng `TIMESTAMP`) |
| Miền giá trị | Năm 1000–9999 (tr. 135) | 1970 tới 19/01/2038 (tr. 135); E1: 03:14:08 UTC ngày đó bị từ chối | Tới năm 2106 (tr. 136) |
| Khi đổi `time_zone` | Giữ nguyên chữ (tr. 135); E1: vẫn 12:00 | Hiển thị theo múi giờ session (tr. 135); E1: 07:00 ở −05:00, 19:00 ở +07:00 | Không đổi; ứng dụng tự chuyển |
| Insert không có giá trị (8.0 mặc định) | NULL nếu cột nullable | NULL; không còn tự lấy giờ hiện tại vì `explicit_defaults_for_timestamp=ON` (E1; khác mô tả ở tr. 135–136) | Theo `DEFAULT` khai báo |

**Lựa chọn và lý do** (**Bổ sung ngoài sách**, dựa trên bốn câu hỏi của hộp tr. 136):

- Với **500 triệu hàng**, chênh 1 byte giữa `DATETIME` và `TIMESTAMP`/`INT` là khoảng 500 MB mỗi nơi lưu cột đó (clustered index và index trên `created_at`), chưa kể overhead page. Đáng kể nhưng không quyết định.
- Ứng dụng nhiều múi giờ cần một **mốc thời điểm tuyệt đối**. `TIMESTAMP` cho đúng điều đó và nhỏ, nhưng **hết hạn năm 2038**: dữ liệu đơn hàng sống lâu hoặc có ngày tương lai (giao hàng, bảo hành) sẽ chạm giới hạn.
- Lựa chọn hợp lý: `DATETIME` (hoặc `DATETIME(3)` nếu cần mili giây) với quy ước **luôn ghi giờ UTC**, khai báo tường minh `NOT NULL DEFAULT CURRENT_TIMESTAMP` (và đặt `time_zone='+00:00'` cho session ghi). Cách này tránh 2038, vẫn dùng được các hàm ngày giờ của MySQL, chỉ tốn thêm 1 byte. `INT UNSIGNED` epoch là phương án hợp lệ nếu đội muốn dồn toàn bộ xử lý thời gian vào code (tr. 136), đổi lại mất khả năng đọc trực tiếp và các hàm ngày giờ.
- Dù chọn gì, kiểm tra `SHOW CREATE TABLE` sau khi tạo, như chú thích 4 khuyên (tr. 136).

[về mục 2.1](../chapter-06.md#s2-1)

<a id="q2-2-1"></a>

### Câu 2.2.1 — Bảng 300 cột, truy vấn chỉ đọc 3 cột, CPU cao

*Một bảng `users` có 300 cột, nhưng truy vấn nóng nhất chỉ đọc `id`, `email`, `status`. CPU của server cao bất thường. Theo cơ chế sách mô tả, chi phí nằm ở đâu, và hai hướng giảm chi phí là gì?*

**Chi phí nằm ở ranh giới server và storage engine.** Storage engine API chép hàng từ engine lên server dưới dạng row buffer, rồi server giải mã buffer thành các cột. Với InnoDB, bước chuyển đổi này **luôn** xảy ra và chi phí tăng theo **số cột** của hàng, không theo số cột truy vấn cần (tr. 145). Sách kể đúng tình huống này: khách hàng có bảng hàng trăm cột, CPU cao dù chỉ vài cột được dùng (tr. 145). InnoDB lưu theo dòng nên đọc một hàng từ clustered index là đọc cả bản ghi (**Bổ sung ngoài sách**).

**Hai hướng giảm** (**Bổ sung ngoài sách**, dựa trên lập luận của sách):

1. **Không đọc hàng rộng:** tạo **covering index** `(status, email)` hay tương tự để truy vấn được trả lời từ index phụ, vốn chứa cột được index và khóa chính, không cần quay về hàng 300 cột ([chương 7](../chapter-07.md#covering-index)). Điều kiện: truy vấn chỉ cần các cột nằm trong index.
2. **Tách bảng theo mẫu truy cập** (*vertical partitioning*): giữ các cột nóng trong một bảng hẹp, đưa cột ít dùng sang bảng phụ cùng khóa chính. Đánh đổi: truy vấn cần cả hai phần phải join thêm, và ứng dụng phải ghi hai bảng.

Đây là ví dụ "tránh thiết kế cực đoan" của Summary (tr. 154). Cần đo trước khi kết luận: CPU cao có thể do nguyên nhân khác (truy vấn đọc quá nhiều hàng, [chương 8](../chapter-08.md#examined-too-much)).

[về mục 2.2](../chapter-06.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — EAV: giới hạn cứng, giới hạn mềm và mất lợi ích kiểu dữ liệu

*Vì sao một thiết kế EAV vừa chạm giới hạn cứng của MySQL, vừa làm hỏng lợi ích đã học ở mục chọn kiểu dữ liệu? Giới hạn nào là cứng, giới hạn nào là "mềm" theo sách?*

- **Giới hạn cứng:** MySQL chỉ cho **61 bảng mỗi join** (tr. 145). EAV dựng lại một "hàng" logic bằng cách self-join bảng thuộc tính một lần cho mỗi thuộc tính, nên thực thể có nhiều thuộc tính sẽ chạm giới hạn; sách đã thấy nhiều database EAV vượt nó (tr. 145). Thực hành E1 xác nhận lỗi 1116 với 62 bảng.
- **Giới hạn mềm:** ngay cả với ít join hơn nhiều, **chi phí lập kế hoạch và tối ưu** đã thành vấn đề; quy tắc ngón tay cái là **khoảng một tá bảng trở xuống** mỗi truy vấn nếu cần nhanh với concurrency cao (tr. 145).
- **Mất lợi ích kiểu dữ liệu:** cột `value` của EAV thường là chuỗi chung cho mọi thuộc tính, nên ngày tháng, số, IP… đều thành chuỗi. Điều này đi ngược nguyên tắc "đơn giản là tốt" (so sánh số rẻ hơn chuỗi, lưu ngày bằng kiểu thời gian, IP bằng số nguyên; tr. 126, 144) và "nhỏ hơn thường tốt hơn" (tr. 126). Hộp về schema tự sinh cũng chê ORM lưu mỗi thuộc tính một hàng (tr. 144).

[về mục 2.2](../chapter-06.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Hằng ma thuật −1 làm AVG ra số âm

*Đội của bạn có cột `discount_pct INT NOT NULL DEFAULT -1`, trong đó −1 nghĩa là "chưa áp dụng chính sách". Một báo cáo tính `AVG(discount_pct)` ra số âm. Phân tích theo mục "NULL Not Invented Here" và đề xuất cách sửa, kèm đánh đổi.*

**Phân tích.** Đây đúng là trường hợp sách cảnh báo: chọn một giá trị trong miền của kiểu (−1) làm "chưa biết" làm code phức tạp và sinh bug (tr. 146). Hàm tổng hợp không biết −1 là đặc biệt nên tính nó như một mức giảm giá âm. Mọi truy vấn đều phải nhớ thêm `WHERE discount_pct <> -1`, và chỉ cần quên một lần là sai.

**Cách sửa.** Cho phép NULL: `discount_pct INT NULL`, dùng NULL cho "chưa áp dụng". Hàm tổng hợp như `AVG()` bỏ qua NULL, nên báo cáo đúng mà không cần điều kiện thêm (**Bổ sung ngoài sách**, hành vi chuẩn của SQL).

**Đánh đổi.** NULL có chi phí nhỏ (bitmap NULL, xử lý đặc biệt, so sánh ba giá trị; tr. 126) và cần cẩn thận với `= NULL` so với `IS NULL`, `NOT IN` với NULL. Nhưng sách kết luận xử lý NULL thường **tốt hơn** hằng ma thuật (tr. 146). Nếu "chưa áp dụng" là trạng thái nghiệp vụ có ý nghĩa riêng, một cột trạng thái riêng (ví dụ `discount_status ENUM('none','applied')`) còn rõ hơn nữa. Đổi kiểu cột trên bảng lớn là thay đổi schema, nên đi qua pipeline của [mục 2.3](../chapter-06.md#s2-3).

[về mục 2.2](../chapter-06.md#s2-2)

<a id="q2-3-1"></a>

### Câu 2.3.1 — Vì sao quản lý schema là vấn đề tổ chức

*Vì sao sách coi quản lý thay đổi schema là vấn đề tổ chức (tốc độ của các đội) chứ không chỉ là vấn đề kỹ thuật DDL? Nêu ba quyết định cụ thể trong pipeline của sách xuất phát từ mục tiêu đó.*

Sách đặt mục tiêu là **tốc độ của kỹ sư** và thời gian từ thiết kế tới production (tr. 147). Khi có hàng chục, hàng trăm instance, nếu mỗi thay đổi schema phải qua tay một vài người của đội database thì cả tổ chức chờ họ (tr. 146–147). Vì vậy đội database phải là người **tạo điều kiện**, không phải cổng kiểm soát (tr. 147). DDL native hay gh-ost chỉ giải quyết "chạy an toàn"; câu hỏi lớn hơn là **ai** được chạy và **qua quy trình nào**.

Ba quyết định xuất phát từ mục tiêu đó:

1. **Dùng cùng công cụ CI** với triển khai code, để quy trình quen thuộc với các đội (Lưu ý tr. 148; điều rút ra đầu tiên tr. 149).
2. **Một repository cho mỗi cluster**, không phải một repo cho tất cả, để mỗi đội đi theo nhịp riêng và có lint riêng (tr. 149, 152).
3. **Pipeline linh hoạt theo đội**: tự động đẩy khi test qua, hoặc cần người phê duyệt, tùy mức quan trọng của database (tr. 153). Kèm theo là **cấu hình nền an toàn** và repository template để "làm đúng thành làm dễ" (tr. 153).

[về mục 2.3](../chapter-06.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Bảng 2 TB có khóa ngoại: thêm index và đổi INT sang BIGINT

*Bảng `orders` 2 TB, 5.000 ghi mỗi giây, có khóa ngoại tới `customers`. Cần thêm một index và đổi một cột `INT` sang `BIGINT`. Với từng thay đổi, chọn cách chạy (DDL native, gh-ost, pt-online-schema-change) và giải thích theo các đánh đổi sách nêu.*

**Thêm index.** Với 8.0, `ADD INDEX` chạy được bằng `ALGORITHM=INPLACE, LOCK=NONE` (đã đo trong [E6](../experiments/chapter-06.md#e6)), không chép cả bảng. Nhưng sách cảnh báo hai rủi ro với bảng rất lớn: log thay đổi nội bộ có thể quá lớn làm **rollback** sau nhiều giờ, và DDL native không **điều tiết** được tốc độ (tr. 150). Với 5.000 ghi mỗi giây trong nhiều giờ, cần ước lượng lượng thay đổi so với `innodb_online_alter_log_max_size` (**Bổ sung ngoài sách**), và nhớ rằng replica sẽ phải chạy lại cùng ALTER (**Bổ sung ngoài sách**, gây replication lag; xem [chương 9](../chapter-09.md)). Nếu rủi ro chấp nhận được thì native là lựa chọn rẻ nhất.

**Đổi `INT` sang `BIGINT`.** Đổi kiểu cột không làm được với `INPLACE` (đo trong E6: bị từ chối, phải `COPY` với khóa chia sẻ), nên native nghĩa là chặn ghi trong suốt lúc chép 2 TB: không chấp nhận được. Phải dùng công cụ ngoài, mà công cụ ngoài luôn chép cả bảng (Cảnh báo tr. 150).

- **gh-ost** sẽ **từ chối** vì bảng có khóa ngoại (tr. 151).
- **pt-online-schema-change** hỗ trợ khóa ngoại ở mức nào đó nhưng phải chọn giữa uptime và nhất quán (tr. 151–152), và trigger có cái giá hiệu năng ở 5.000 ghi mỗi giây; cần cấu hình để công cụ hủy sớm khi tải cao (tr. 151).

Sách sẽ chọn pt-online-schema-change cho database cũ đã có khóa ngoại khó gỡ (tr. 152), hoặc về lâu dài bỏ khóa ngoại để dùng gh-ost (tr. 151–153). Đây là ví dụ cho bài học của [2.1.1](../chapter-06.md#whole-numbers): chọn kiểu nguyên đủ lớn ngay từ đầu rẻ hơn nhiều so với nới rộng sau (tr. 126).

[về mục 2.3](../chapter-06.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — ALTER INPLACE chạy 6 giờ rồi rollback

*Một `ALTER TABLE ... ALGORITHM=INPLACE, LOCK=NONE` trên bảng rất lớn chạy 6 giờ rồi thất bại và rollback. Theo sách, nguyên nhân có thể là gì? Bạn sẽ kiểm tra biến nào, và có những cách nào để tránh lặp lại?*

**Nguyên nhân theo sách.** Với bảng rất lớn, DDL native có thể rollback nếu **log thay đổi** mà InnoDB giữ trong lúc ALTER chạy quá lớn, làm mất nhiều giờ công việc (tr. 150).

**Biến cần kiểm tra (Bổ sung ngoài sách).** Trong lúc ALTER online, các thay đổi DML đồng thời được ghi vào một log tạm; kích thước tối đa là `innodb_online_alter_log_max_size` (mặc định 128 MB). Vượt ngưỡng thì ALTER thất bại với lỗi 1799 ("… required more than 'innodb_online_alter_log_max_size' bytes of modification log"). Thí nghiệm [E6](../experiments/chapter-06.md#e6) tái hiện đúng lỗi này với ngưỡng 64 KB.

**Cách tránh:**

1. Chạy vào lúc ít ghi, hoặc tăng `innodb_online_alter_log_max_size` (đánh đổi: tốn thêm đĩa tạm, và pha áp log cuối ALTER dài hơn).
2. Dùng công cụ ngoài như gh-ost để **điều tiết** tốc độ và tự hủy an toàn khi tải cao, đúng lý do thứ hai sách nêu (tr. 150–151); đánh đổi là chép cả bảng (tr. 150).
3. Nếu thay đổi làm được bằng `INSTANT` (thêm cột, thêm phần tử ENUM ở cuối…), dùng `ALGORITHM=INSTANT` để không có pha chép dữ liệu nào.

[về mục 2.3](../chapter-06.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — Trigger hay binary log: hệ quả

*gh-ost và pt-online-schema-change khác nhau thế nào ở cách theo dõi thay đổi trong lúc chép bảng? Hệ quả của khác biệt đó với hiệu năng, với khóa ngoại và với yêu cầu về binary log là gì?*

| | pt-online-schema-change | gh-ost |
| --- | --- | --- |
| Theo dõi thay đổi | **Trigger** trên bảng gốc chép mỗi thay đổi sang bảng mới (tr. 150) | Kết nối **như một replica**, đọc **binary log dạng row** làm changelog (tr. 151) |
| Hiệu năng | Có trigger là có chi phí trên mỗi lần ghi; quan trọng ở TPS rất cao (tr. 151) | Không trigger, nên không có cái giá đó (tr. 152) |
| Giới hạn trigger | Bảng đã có trigger cùng loại có thể gây lỗi (sách nói trước 8.0); không chạy nhiều migration cùng bảng (tr. 150–151) | Không áp dụng |
| Khóa ngoại | Hỗ trợ ở mức nào đó, kèm đánh đổi uptime và nhất quán (tr. 151–152) | **Dừng hẳn** nếu bảng có khóa ngoại (tr. 151) |
| Binary log | Không cần đọc | **Bắt buộc** đọc được; không có thì pt-online-schema-change là lựa chọn còn lại (tr. 152) |
| Phụ thuộc phiên bản | Gắn với giới hạn trigger của phiên bản MySQL | Ít phụ thuộc hơn nhiều, thậm chí chạy với statement-based replication kèm điều kiện (tr. 152) |

Cả hai đều **chép toàn bộ bảng** (Cảnh báo tr. 150). Vì vậy sách khuyên đội mới chọn gh-ost nếu cam kết không dùng khóa ngoại (tr. 152).

[về mục 2.3](../chapter-06.md#s2-3)

<a id="q2-4-1"></a>

### Câu 2.4.1 — Kiểu lệch nhau trong join: hai cơ chế gây hại

*Summary nói "dùng cùng kiểu cho giá trị liên quan, nhất là trong điều kiện join". Nêu hai cơ chế khác nhau khiến kiểu lệch nhau gây hại, một về hiệu năng và một về tính đúng đắn, dựa trên các mục trước của chương.*

1. **Hiệu năng.** So sánh hai kiểu khác nhau buộc MySQL **chuyển đổi** một phía cho mỗi cặp hàng. Bảng 6-1 cho thấy join ENUM với VARCHAR chỉ đạt 1,7–1,8 truy vấn mỗi giây, trong khi ENUM với ENUM đạt 3,5 và VARCHAR với VARCHAR đạt 2,6 (tr. 134). **Bổ sung ngoài sách:** khi chuyển đổi đặt lên cột được index, index trên cột đó có thể không dùng được để tra cứu, biến tra cứu theo index thành quét.
2. **Tính đúng đắn.** Chuyển kiểu ngầm khi so sánh có thể tạo **lỗi khó tìm**, xuất hiện rất lâu sau khi bạn đã quên rằng mình đang so sánh hai kiểu khác nhau (tr. 142). Với InnoDB, khóa ngoại cũng có thể không tạo được nếu kiểu không khớp chính xác, kể cả `UNSIGNED` (chú thích 5, tr. 142); E1 đo được lỗi 3780 ở 8.0.42.

[về mục 2.4](../chapter-06.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — "Tránh NULL" và "đừng dùng hằng ma thuật": có mâu thuẫn?

*Hai lời khuyên "tránh NULL" và "đừng dùng hằng ma thuật thay NULL" có mâu thuẫn không? Phát biểu một quy tắc quyết định gộp được cả hai.*

Không mâu thuẫn; chúng nhắm vào hai tình huống khác nhau.

- "Tránh NULL" nhắm vào cột **nullable theo mặc định** dù ứng dụng không bao giờ lưu NULL (tr. 126). Ở đó NULL không mang ý nghĩa gì mà vẫn có chi phí, nên khai báo `NOT NULL`.
- "NULL Not Invented Here" nhắm vào trường hợp **thật sự có giá trị chưa biết**; khi đó NULL thường tốt hơn hằng ma thuật như −1 hay `'0000-00-00'` (tr. 146).

**Quy tắc gộp:** khai báo `NOT NULL` khi dữ liệu luôn có giá trị; khi "chưa biết" là một trạng thái có thật, dùng NULL thay vì mượn một giá trị hợp lệ của kiểu. Summary nói đúng ý này: tránh NULL **trừ khi đó đúng là cách mô hình hóa thực tế dữ liệu** (tr. 154). Lợi ích hiệu năng khi đổi NULL thành NOT NULL thường nhỏ, nên đừng hy sinh tính đúng đắn để đổi lấy nó (tr. 126).

[về mục 2.4](../chapter-06.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Nguyên tắc nào của Summary đã đổi với MySQL 8.0

*Trong bảy nguyên tắc của Summary, nguyên tắc nào đã bị MySQL 8.0 làm "nhẹ đi" hoặc thay đổi một phần, và nguyên tắc nào vẫn nguyên giá trị? Dùng kết quả thực hành để trả lời.*

**Bị làm nhẹ hoặc thay đổi một phần:**

- *Cẩn thận chuỗi độ dài thay đổi trong bảng tạm và sort* (tr. 154). Với mặc định của 8.0 (engine bảng tạm TempTable, collation NO PAD `utf8mb4_0900_ai_ci`), E5 không thấy khác biệt giữa `VARCHAR(10)` và `VARCHAR(1000)`. Nhưng với engine MEMORY, `GROUP BY` trên `VARCHAR(1000)` tràn ra đĩa và chậm khoảng 5,6 lần; với collation PAD SPACE, sort chậm khoảng 5,3 lần và có 134 merge pass thay vì 7. Nguyên tắc vẫn đúng, nhưng giờ **có điều kiện**.
- *Tránh thói quen cũ như precision cho số thực, độ rộng cho số nguyên* (tr. 154). 8.0.17 đã deprecate cả hai; E1 thấy cảnh báo 1681 và `SHOW CREATE TABLE` bỏ độ rộng của `INT`. Nguyên tắc vẫn đúng, giờ server tự nhắc.
- *Cẩn thận ENUM và SET* (tr. 154). Phần chi phí thay đổi đã nhẹ đi: thêm phần tử ở cuối là `INSTANT` (E6). Phần bẫy ngữ nghĩa (sắp theo vị trí, hằng là chữ số) vẫn nguyên (E1).
- Một chi tiết của nguyên tắc "tránh NULL": cột `TIMESTAMP` trong 8.0 **mặc định cho phép NULL** (E1), ngược với mô tả ở tr. 136.

**Vẫn nguyên giá trị:**

- *Kiểu nhỏ, đơn giản*: E2 cho thấy `BIGINT` thay `INT` làm PRIMARY lớn thêm 30% và index phụ thêm 59%; `BINARY(16)` nhỏ hơn hẳn `CHAR(32)` cho cùng giá trị băm.
- *Số nguyên cho identifier*: E3 cho thấy khóa `CHAR(36)` làm insert chậm khoảng 4,7 lần, đọc đĩa hơn 111.000 lần, index phụ lớn gấp 2,7 lần.
- *Cùng kiểu cho cột liên quan*: E1 gặp lỗi 3780 khi khóa ngoại lệch `UNSIGNED`.
- *Tránh thiết kế cực đoan*: E1 xác nhận giới hạn 61 bảng; *tránh BIT*: E1 tái hiện kết quả `'9'` và 57 gây nhầm lẫn.

[về mục 2.4](../chapter-06.md#s2-4)
