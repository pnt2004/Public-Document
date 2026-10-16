# Thực hành — Chương 7: Indexing for High Performance

[← Về ghi chú chương 7](../chapter-07.md) · [Đáp án tự kiểm tra](../answers/chapter-07.md)

Năm thí nghiệm kiểm chứng các nhận định về B-tree, selectivity, thứ tự cột, clustered và covering index, sắp xếp bằng index, thứ tự chèn primary key, statistics và adaptive hash index của chương 7. Sách dùng Sakila; ở đây **không** tải Sakila mà tự sinh dữ liệu có cấu trúc tương tự bằng SQL. Schema, dữ liệu và truy vấn là **Ví dụ minh họa do AI xây dựng**. Mọi số liệu là **Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định trừ khi ghi khác, dữ liệu tổng hợp 300.000 tới 500.000 hàng mỗi bảng). Output được rút gọn khi ghi "đã rút gọn". Thời gian đo trên laptop Windows có nhiều nhiễu, không phải benchmark production; chỉ nên đọc theo tỉ lệ và luôn kèm chỉ số tất định (số mục đọc, số page, plan).

## Mục lục

- [Môi trường](#moi-truong)
- [E1. B-tree, leftmost prefix và các giới hạn](#e1)
- [E2. Selectivity, prefix index và thứ tự cột](#e2)
- [E3. Clustered index, covering index, index thừa và không dùng](#e3)
- [E4. Sắp xếp bằng index so với filesort](#e4)
- [E5. Thứ tự chèn PK, bảo trì bảng, statistics và adaptive hash index](#e5)
- [Dọn dẹp chung](#don-dep)

<a id="moi-truong"></a>

## Môi trường

| Mục | Giá trị |
| --- | --- |
| Phiên bản | MySQL Community Server **8.0.42** (`C:\Program Files\MySQL\MySQL Server 8.0\bin\`) |
| Hệ điều hành | Windows 11 Home, laptop |
| Instance | Riêng, cổng **3317**, `--no-defaults`, `--bind-address=127.0.0.1`, `--mysqlx=OFF`, datadir trong thư mục tạm của phiên làm việc |
| Cấu hình | Mặc định 8.0.42: `innodb_buffer_pool_size` 128 MB, page 16 KB, `utf8mb4_0900_ai_ci`, `innodb_flush_method=unbuffered` (mặc định trên Windows), `innodb_adaptive_hash_index=ON` (8 phân vùng), `innodb_stats_persistent=ON`, `innodb_stats_persistent_sample_pages=20`, `innodb_stats_on_metadata=OFF`, `information_schema_stats_expiry=86400`, `eq_range_index_dive_limit=200`, `innodb_autoinc_lock_mode=2`, `optimizer_switch` mặc định (có `skip_scan=on`, `index_condition_pushdown=on`, các cờ `index_merge*` bật) |
| Thay đổi tạm thời | E5 phần I/O-bound: khởi động lại với `--innodb-buffer-pool-size=16M`, rồi trả về mặc định. Các thay đổi `optimizer_switch`, `eq_range_index_dive_limit`, `information_schema_stats_expiry` chỉ ở mức session. `innodb_adaptive_hash_index` đổi ở mức global trong E5 rồi trả về `ON` |
| Biến không tồn tại | `SHOW VARIABLES LIKE 'innodb_stats_sample_pages'` trả về rỗng trên 8.0.42 (biến sách nhắc ở tr. 188) |

Dựng và chạy instance (Git Bash; `<SP>` là thư mục làm việc riêng):

```bash
BIN="/c/Program Files/MySQL/MySQL Server 8.0/bin"
"$BIN/mysqld" --no-defaults --initialize-insecure --datadir="<SP>/hpm/mysql/ch07/main/data" --console
"$BIN/mysqld" --no-defaults --datadir="<SP>/hpm/mysql/ch07/main/data" --port=3317 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=ch07.sock --console > "<SP>/hpm/mysql/ch07/main/mysqld.log" 2>&1 &
"$BIN/mysql" -h127.0.0.1 -P3317 -uroot -e "SELECT @@port, @@datadir, @@version"
# ... chạy thí nghiệm ...
"$BIN/mysqladmin" -h127.0.0.1 -P3317 -uroot shutdown
```

Bảng số `hpm7.seq(n)` từ 1 tới 1.000.000 được dùng cho mọi thí nghiệm. Dữ liệu được sinh **tất định** bằng `CRC32` thay vì `RAND()`, nên chạy lại cho cùng phân bố.

```sql
CREATE DATABASE IF NOT EXISTS hpm7;
USE hpm7;
CREATE TABLE digits (d TINYINT NOT NULL PRIMARY KEY);
INSERT INTO digits VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9);
CREATE TABLE seq (n INT NOT NULL PRIMARY KEY);
INSERT INTO seq (n)
SELECT 1 + a.d + 10*b.d + 100*c.d + 1000*e.d + 10000*f.d + 100000*g.d
FROM digits a, digits b, digits c, digits e, digits f, digits g;
```

**Cách đo chung.** Với mỗi truy vấn: `EXPLAIN`, rồi `FLUSH STATUS`, chạy truy vấn, `SHOW SESSION STATUS WHERE Variable_name IN ('Handler_read_key','Handler_read_next', ...)`. Để giữ output nhỏ mà vẫn buộc engine đọc hàng, nhiều truy vấn được bọc bằng `SELECT COUNT(*), MAX(cột_ngoài_index)`; plan truy cập giống truy vấn `SELECT *` tương ứng. Thời gian lấy từ `performance_schema.events_statements_history` (`TIMER_WAIT`) hoặc `EXPLAIN ANALYZE`.

---

<a id="e1"></a>

## E1. B-tree, leftmost prefix và các giới hạn

**Mục tiêu.** Kiểm chứng các kiểu truy vấn dùng được B-tree và ba giới hạn ở tr. 159–161 bằng chỉ số tất định; xem MySQL 8.0 thay đổi gì (ICP, skip scan).

**Giả thuyết (Dự đoán).**
- Full value, leftmost prefix, column prefix, range, "bằng + khoảng" dùng index; `key_len` tăng theo số cột dùng.
- `WHERE first_name = ...` không dùng được index để thu hẹp; trên 8.0.13+ có thể dùng skip scan nếu chỉ cần cột của index.
- Cột sau điều kiện khoảng không thu hẹp phạm vi quét; thay khoảng bằng `IN` thì thu hẹp được.

**Chuẩn bị** (bảng giống `People` ở tr. 158, thêm PK và cột đệm để truy vấn `SELECT *` không được cover):

```sql
USE hpm7;
CREATE TABLE people (
  id INT NOT NULL AUTO_INCREMENT,
  last_name  VARCHAR(50) NOT NULL,
  first_name VARCHAR(50) NOT NULL,
  dob DATE NOT NULL,
  bio CHAR(100) NOT NULL,
  PRIMARY KEY (id),
  KEY idx_name_dob (last_name, first_name, dob)
) ENGINE=InnoDB;

-- 2.000 họ (40 âm đầu x 50 âm cuối, có 'Allen', 'Smith'), 300 tên (có 'Kirl', 'Bill', 'Jon'...)
INSERT INTO people (last_name, first_name, dob, bio)
SELECT
  CONCAT(
    ELT(1 + (CRC32(CONCAT('l', n)) % 2000) % 40,
        'Al','Ak','Ba','Be','Bo','Ca','Ch','Co','Da','De','Ed','Fa','Fo','Ga','Gr','Ha','He','Ho','Ja','Jo',
        'Ka','Ke','La','Le','Ma','Mo','Na','No','Pa','Pe','Ra','Ro','Sa','Sm','St','Ta','To','Va','Wa','Wi'),
    ELT(1 + (CRC32(CONCAT('l', n)) % 2000) DIV 40,
        'len','royd','ker','ley','son','ton','man','well','ford','wood','ridge','more','ith','ard','ett','ins','ers','ock','ing','ish',
        'ster','field','berg','stein','ow','ay','ez','el','an','on','er','by','ham','ville','ent','dale','gate','hill','lock','mont',
        'ney','quist','rick','sby','thorpe','ux','vin','wick','yard','zer')),
  CONCAT(
    ELT(1 + (CRC32(CONCAT('f', n)) % 300) % 30,
        'Ki','Ka','Bi','Cu','Ja','Jo','Me','Ma','An','Al','Be','Ch','Da','De','El','Fr','Ge','Ha','Il','Iv',
        'Le','Li','Mi','Na','Ol','Pa','Ro','Sa','Th','Vi'),
    ELT(1 + (CRC32(CONCAT('f', n)) % 300) DIV 30, 'm','rl','ll','ba','ck','ne','ra','na','tt','n')),
  DATE '1940-01-01' + INTERVAL (CRC32(CONCAT('d', n)) % 25000) DAY,
  REPEAT('x', 100)
FROM seq WHERE n <= 500000;
ANALYZE TABLE people;
```

Thực tế: 500.000 hàng, 2.000 họ, 300 tên, 153.954 cặp họ và tên, 499.876 bộ ba. `mysql.innodb_index_stats`: `idx_name_dob` có 1.040 trang lá (1.251 page cấp phát), `PRIMARY` 4.546 trang lá; thống kê `n_diff_pfx04` của `idx_name_dob` có mô tả `last_name,first_name,dob,id`.

**Các bước và lệnh đo** (mẫu cho mỗi truy vấn):

```sql
EXPLAIN SELECT COUNT(*), MAX(bio) FROM people WHERE last_name='Allen' AND first_name LIKE 'K%';
FLUSH STATUS;
SELECT COUNT(*), MAX(bio) FROM people WHERE last_name='Allen' AND first_name LIKE 'K%';
SHOW SESSION STATUS WHERE Variable_name IN ('Handler_read_key','Handler_read_next','Handler_read_rnd_next');
-- So sánh khi tắt ICP:
SET SESSION optimizer_switch='index_condition_pushdown=off';
-- Khoảng quét thật:
SET optimizer_trace='enabled=on';
SELECT ...;
SELECT JSON_EXTRACT(trace, '$**.range_scan_alternatives') FROM information_schema.OPTIMIZER_TRACE;
```

**Kết quả** (`key_len`: 202 = một cột `VARCHAR(50)` utf8mb4, 404 = hai cột, 407 = ba cột):

| # | Điều kiện | `type`, `key_len`, `Extra` | `rows` ước lượng | `Handler_read_key` / `_next` / `_rnd_next` | Hàng khớp |
| --- | --- | --- | --- | --- | --- |
| Q1 | `last_name='Allen' AND first_name='Kirl' AND dob='1981-03-21'` | `ref`, 407 | 1 | 1 / 1 / 0 | 1 |
| Q2 | `last_name='Allen'` | `ref`, 202 | 245 | 1 / 245 / 0 | 245 |
| Q3 | `last_name LIKE 'Al%'` | `range`, 202, `Using index condition` | 23.804 | 1 / 12.636 / 0 | 12.636 |
| Q4 | `last_name BETWEEN 'Allen' AND 'Baker'` | `range`, 202, `Using index condition` | 23.840 | 1 / 12.645 / 0 | 12.645 |
| Q5 | `last_name='Allen' AND first_name LIKE 'K%'` | `range`, 404, `Using index condition` | 15 | 1 / 15 / 0 | 15 |
| Q6 | như Q2 nhưng chỉ lấy `dob`, `id` | `ref`, 202, `Using index` | 245 | 1 / 245 / 0 | 245 |
| Q7 | `first_name='Bill'`, cần `bio` | `ALL`, `Using where` | 495.514 | 1 / 0 / 500.001 | 1.680 |
| Q7b | `first_name='Bill'`, chỉ cột của index | `range`, 404, `Using where; Using index for skip scan` | 49.551 | 4.001 / 1.680 / 0 | 1.680 |
| Q8 | `last_name='Smith' AND dob='1965-11-27'` | `ref`, 202, `Using index condition` | 255 | ICP bật: 1 / 1; ICP tắt: 1 / **255** | 1 |
| Q9 | `last_name='Smith' AND first_name LIKE 'J%' AND dob='1965-11-27'` | `range`, **407**, `Using index condition` | 23 | ICP bật: 1 / 1; ICP tắt: 1 / **23** | 1 |
| Q10 | như Q9 nhưng `first_name IN (20 tên J)` | `range`, 407, `Using index condition` | 20 | **20** / 1 (bật hay tắt ICP như nhau) | 1 |
| Q11a | `last_name='Allen' ORDER BY first_name, dob` | `ref`, 202, không filesort | 245 | | |
| Q11b | `last_name='Allen' ORDER BY dob` | `ref`, 202, `Using filesort` | 245 | | |

Optimizer trace của Q9 (đã rút gọn):

```text
"index": "idx_name_dob", "rows": 23, "chosen": true,
"ranges": [ "last_name = 'Smith' AND 'J' <= first_name <= 'J?????...'" ]
```

`EXPLAIN ANALYZE` (mỗi số một lần chạy): Q7b `Covering index skip scan on people using idx_name_dob over first_name = 'Bill'`, actual time 0,416..29,9 ms, 1.680 hàng. Q7 `Table scan on people`, 500.000 hàng, tổng 275 ms. Q10: `Index range scan ... over (last_name = 'Smith' AND first_name = 'Jaba' AND dob = '1965-11-27') OR (...) OR (18 more)`.

**Diễn giải và đối chiếu với sách.**
- Sáu kiểu truy vấn ở tr. 159–160 đều dùng index như sách nói; Q11 xác nhận index dùng được cho `ORDER BY` cùng tiêu chí (tr. 160).
- Giới hạn 1 (tr. 160): Q7 quét toàn bảng. Nhưng 8.0.13+ có skip scan (Q7b) khi chỉ cần cột của index; đó là phần "tương lai" mà chú thích 3 (tr. 159) nhắc.
- Giới hạn 2: Q8 chỉ dùng cột đầu để thu hẹp (255 mục), `dob` được kiểm bằng ICP. "Mục đích khác" ở tr. 161 chính là ICP.
- Giới hạn 3: Q9 báo `key_len` 407 nhưng trace và bộ đếm khi tắt ICP (23 mục) cho thấy `dob` không thu hẹp khoảng. `key_len` của `range` có thể "nói quá".
- Cách lách bằng điều kiện bằng (tr. 161): Q10 làm 20 lần tra, chỉ đọc 1 mục.
- Bẫy đo: `Handler_read_next` không đếm mục bị ICP loại (Q8, Q9 khi bật ICP chỉ là 1).

**Dọn dẹp.** Giữ bảng `people` cho E5; trả `optimizer_switch` về mặc định (`SET SESSION optimizer_switch='index_condition_pushdown=on'`).

---

<a id="e2"></a>

## E2. Selectivity, prefix index và thứ tự cột

**Mục tiêu.** Lặp lại phân tích `city_demo` (tr. 162–165) và `payment` (tr. 168–170) trên dữ liệu có phân bố lệch rõ hơn; đo kích thước và plan của prefix index; đo ảnh hưởng thứ tự cột và index merge.

**Giả thuyết (Dự đoán).**
- Selectivity của tiền tố tăng dần rồi bão hòa; tiền tố phổ biến nhất ở độ dài ngắn có nhiều hàng hơn hẳn tên đầy đủ.
- Prefix index không cover được, không dùng được cho `ORDER BY` hay `GROUP BY`.
- Với điều kiện bằng trên hai cột, thứ tự cột không đổi số mục đọc; với điều kiện khoảng, cột so bằng nên đứng trước.

**Chuẩn bị:**

```sql
USE hpm7;
-- 600 tên thành phố, 200 tên có 10 tiền tố chung
CREATE TABLE city_list (k INT PRIMARY KEY, city VARCHAR(50) NOT NULL);
INSERT INTO city_list
SELECT n-1,
  CASE WHEN n-1 < 200 THEN
    CONCAT(ELT(1 + (n-1) DIV 20, 'San ','Santa ','New ','Port ','Saint ','South ','North ','Fort ','Lake ','Mount '),
           ELT(1 + (n-1) % 20, 'Felipe','Jose','Juan','Pedro','Miguel','Carlos','Luis','Marcos','Pablo','Rafael',
               'Antonio','Andres','Bernardo','Cristobal','Diego','Fernando','Gabriel','Isidro','Lorenzo','Mateo'))
  ELSE
    CONCAT(ELT(1 + (n-1) % 25, 'Ka','Lo','Mi','Ba','Te','Ro','Shi','Va','Po','Da','Gu','Ha','Ki','Ne','Ol','Pe','Qu','Ra','Su','To','Ur','Vi','Wa','Ya','Zo'),
           ELT(1 + ((n-1) DIV 25) % 16, 'ra','lo','mi','ba','na','ko','sa','ti','ve','du','ga','he','ji','mu','po','ri'),
           ELT(1 + (n-1) % 3, 'ville','burg','pur'))
  END
FROM seq WHERE n <= 600;

CREATE TABLE city_demo (id INT AUTO_INCREMENT PRIMARY KEY, city VARCHAR(50) NOT NULL) ENGINE=InnoDB;
INSERT INTO city_demo (city)
SELECT c.city FROM seq s JOIN city_list c ON c.k = CRC32(CONCAT('c', s.n)) % 600 WHERE s.n <= 300000;

-- Khóa session dạng hex dài (tr. 165)
CREATE TABLE sessions (id INT AUTO_INCREMENT PRIMARY KEY, sid CHAR(64) NOT NULL, data CHAR(50) NOT NULL) ENGINE=InnoDB;
INSERT INTO sessions (sid, data) SELECT SHA2(CONCAT('sess', n), 256), REPEAT('d', 50) FROM seq WHERE n <= 300000;

-- payment: 400.000 hàng, 10.000 khách, 2 nhân viên, khách 1 là "guest" chiếm khoảng 10% hàng
CREATE TABLE payment (
  payment_id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL, staff_id TINYINT NOT NULL,
  amount DECIMAL(5,2) NOT NULL, payment_date DATETIME NOT NULL, note CHAR(60) NOT NULL
) ENGINE=InnoDB;
INSERT INTO payment (customer_id, staff_id, amount, payment_date, note)
SELECT IF(CRC32(CONCAT('g', n)) % 10 = 0, 1, 2 + CRC32(CONCAT('c', n)) % 9999),
       1 + CRC32(CONCAT('s', n)) % 2,
       (CRC32(CONCAT('a', n)) % 1000) / 100,
       TIMESTAMP '2020-01-01 00:00:00' + INTERVAL (CRC32(CONCAT('t', n)) % 31536000) SECOND,
       REPEAT('p', 60)
FROM seq WHERE n <= 400000;
ANALYZE TABLE city_demo, sessions, payment;
```

**Các bước và lệnh đo.**

```sql
-- A: selectivity
SELECT COUNT(*) AS c, city FROM city_demo GROUP BY city ORDER BY c DESC LIMIT 5;
SELECT COUNT(DISTINCT city)/COUNT(*) FROM city_demo;
SELECT COUNT(DISTINCT LEFT(city,3))/COUNT(*) AS sel3, COUNT(DISTINCT LEFT(city,4))/COUNT(*) AS sel4,
       /* ... tới 10 ... */ COUNT(DISTINCT LEFT(city,10))/COUNT(*) AS sel10 FROM city_demo;
SELECT COUNT(*) AS c, LEFT(city,4) AS pref FROM city_demo GROUP BY pref ORDER BY c DESC LIMIT 5;
-- B: prefix index
ALTER TABLE city_demo ADD KEY idx_city_full (city), ADD KEY idx_city_p9 (city(9));
ALTER TABLE sessions ADD KEY idx_sid_full (sid), ADD KEY idx_sid_p8 (sid(8));
ANALYZE TABLE city_demo, sessions;
SELECT table_name, index_name, stat_value AS pages FROM mysql.innodb_index_stats
WHERE database_name='hpm7' AND table_name IN ('city_demo','sessions') AND stat_name='size';
EXPLAIN SELECT city FROM city_demo USE INDEX (idx_city_full) WHERE city LIKE 'San F%';
EXPLAIN SELECT city FROM city_demo FORCE INDEX (idx_city_p9) WHERE city LIKE 'San F%';
EXPLAIN SELECT id, city FROM city_demo FORCE INDEX (idx_city_p9) ORDER BY city LIMIT 10;
EXPLAIN SELECT city, COUNT(*) FROM city_demo FORCE INDEX (idx_city_p9) GROUP BY city;
-- C: thứ tự cột
SELECT SUM(staff_id = 2), SUM(customer_id = 584), SUM(customer_id = 1) FROM payment;
SELECT COUNT(DISTINCT staff_id)/COUNT(*), COUNT(DISTINCT customer_id)/COUNT(*), COUNT(*) FROM payment;
ALTER TABLE payment ADD KEY idx_sc (staff_id, customer_id), ADD KEY idx_cs (customer_id, staff_id);
ANALYZE TABLE payment;
-- mỗi truy vấn chạy với FORCE INDEX (idx_sc) rồi (idx_cs), đo Handler_read_*, có lúc tắt ICP
SELECT COUNT(*), MAX(note) FROM payment FORCE INDEX (idx_sc) WHERE staff_id = 2 AND customer_id BETWEEN 500 AND 600;
-- C7: index merge với index một cột
ALTER TABLE payment ADD KEY idx_s (staff_id), ADD KEY idx_c (customer_id);
EXPLAIN SELECT payment_id FROM payment IGNORE INDEX (idx_sc, idx_cs) WHERE customer_id = 584 OR staff_id = 3;
ALTER TABLE payment DROP KEY idx_s, DROP KEY idx_c;
```

**Kết quả.**

A. Selectivity của `city_demo` (300.000 hàng, 600 tên):

| Độ dài | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | Cả cột |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Số giá trị phân biệt | 345 | 408 | 432 | 475 | 551 | 585 | 600 | 600 | 600 |
| Selectivity | 0,0012 | 0,0014 | 0,0014 | 0,0016 | 0,0018 | 0,0020 | 0,0020 | 0,0020 | 0,0020 |

- Tên đầy đủ phổ biến nhất: New Rafael 571, Mount Fernando 563, South Miguel 553, Sugaburg 553, San Rafael 552.
- Tiền tố 4 phổ biến nhất: `San ` 10.076, `Sout` 10.047, `Nort` 9.999, `Moun` 9.998, `Sant` 9.985.
- Tiền tố 8 phổ biến nhất: `South Fe` 1.047, `South Ma` 1.041, `Mount An` 1.034, `Santa An` 1.033, `Saint Fe` 1.030.

B. Prefix index:

| Bảng, index | Số page (`size`) | Ghi chú |
| --- | --- | --- |
| `city_demo.PRIMARY` | 673 | |
| `city_demo.idx_city_full` | 417 | |
| `city_demo.idx_city_p9` | 417 | tên ngắn nên không tiết kiệm |
| `sessions.PRIMARY` | 2.788 | |
| `sessions.idx_sid_full` (`CHAR(64)`) | 1.640 | |
| `sessions.idx_sid_p8` | 417 | selectivity của 8 ký tự đầu: 1,0000 (6 ký tự: 0,9911) |

- Tra bằng qua prefix index: `city = 'San Felipe'` với `idx_city_p9` cho `ref`, `key_len` 38 (= 9 × 4 + 2), `Using where`; `sid = SHA2('sess12345', 256)` với `idx_sid_p8` cho `ref`, `key_len` 32, `rows` 1, `Using where`, 1 lần tra và 1 mục đọc. `INFORMATION_SCHEMA.STATISTICS` cho `SUB_PART` = 8.
- `SELECT city ... LIKE 'San F%'`: index cả cột có `Using where; Using index`; prefix index chỉ có `Using where` (không cover).
- `ORDER BY city LIMIT 10`: index cả cột cho `type: index`, `rows` 10, `Using index`; ép `idx_city_p9` thì optimizer quét toàn bảng (`ALL`), `Using filesort`.
- `GROUP BY city`: index cả cột cho `type: index`, `Using index`; ép `idx_city_p9` thì `ALL`, `Using temporary`.
- Phát hiện phụ: với `FORCE INDEX`, `EXPLAIN` báo `rows: 1` cho `LIKE 'San F%'`; với `USE INDEX` thì `rows: 1031`, đúng bằng số hàng thật. **Bổ sung ngoài sách:** theo tài liệu MySQL 8.0, optimizer bỏ index dive khi có `FORCE INDEX` trên truy vấn một bảng, nên đừng đọc `rows` trong trường hợp này.

C. Thứ tự cột trên `payment`:

| Phép đo | Kết quả |
| --- | --- |
| `SUM(staff_id = 2)`, `SUM(customer_id = 584)`, `SUM(customer_id = 1)` | 200.001; 40; 39.948 |
| `SUM(staff_id = 2)` trong các hàng của khách 584 | 15 |
| Selectivity `staff_id` / `customer_id` | 0,0000 / 0,0250 trên 400.000 hàng |

| Truy vấn | `idx_sc (staff_id, customer_id)` | `idx_cs (customer_id, staff_id)` |
| --- | --- | --- |
| C3. `staff_id = 2 AND customer_id = 584` | `ref`, `key_len` 5, `rows` 19; đọc 15 | `ref`, `key_len` 5, `rows` 19; đọc 15 |
| C4. `customer_id = 584` (bỏ `idx_cs`) | optimizer chọn `ALL`, `Handler_read_rnd_next` 400.001 | `ref`, `key_len` 4, `rows` 41 |
| C5. `staff_id = 2 AND customer_id BETWEEN 500 AND 600`, ICP tắt | đọc **1.664** mục | đọc **3.634** mục |
| C5, ICP bật | 1.664 | `Handler_read_next` 1.664 (mục bị loại trong engine không được đếm) |
| C5, optimizer tự chọn | chọn `idx_sc`, `rows` 1.664 | (`USE INDEX` cho `rows` 3.634) |
| C6. `staff_id = 2 AND customer_id = 1` (guest) | optimizer chọn `idx_sc`: `rows` ước lượng 74.506, đọc 38.958, `EXPLAIN ANALYZE` 182 ms | không đo riêng |
| C6 so sánh: cùng truy vấn với khách 584 | `EXPLAIN ANALYZE` 0,11 ms | |

C7. Index merge với hai index một cột `idx_c(customer_id)`, `idx_s(staff_id)`:

| Truy vấn | Plan |
| --- | --- |
| `customer_id = 584 OR staff_id = 3` | `index_merge`, `key: idx_c,idx_s`, `key_len: 4,1`, `Using union(idx_c,idx_s); Using where` |
| `customer_id = 584 OR customer_id = 777` | `range` trên `idx_c`, `rows` 78, `Using where; Using index` |
| `customer_id = 584 AND staff_id = 2` | `ref` trên `idx_c`, `rows` 40, `filtered` 50, `Using where` (không intersection) |

**Diễn giải và đối chiếu với sách.**
- Selectivity bão hòa dần như ở tr. 164; trường hợp xấu nhất của tiền tố 4 (10.076 hàng) gấp 20 lần một tên đầy đủ trung bình (khoảng 500), đúng cảnh báo ở tr. 165 về "San" và "New".
- Hai nhược điểm của prefix index ở tr. 165 được xác nhận bằng plan. Lợi ích kích thước chỉ có khi giá trị dài: đúng trường hợp khóa hex mà sách nêu.
- Ví dụ `payment` của sách (tr. 168–169) chỉ có điều kiện bằng; đo cho thấy với điều kiện bằng hai thứ tự như nhau, còn điều kiện khoảng mới làm khác biệt, và lúc đó cột kém chọn lọc đặt trước lại tốt hơn. Điều này cụ thể hóa câu "sắp xếp, nhóm, điều kiện khoảng tạo khác biệt lớn hơn nhiều" (tr. 170).
- Giá trị đặc biệt (guest) làm cùng một câu truy vấn chậm hơn khoảng 1.600 lần, như ví dụ `Message` (tr. 169–170).
- Index merge xuất hiện với `OR` trên hai cột khác nhau, như ví dụ `film_actor` (tr. 166).

**Dọn dẹp.** Giữ `payment` cho E4 mục 4.6; xóa index thử nghiệm nếu không dùng tiếp: `ALTER TABLE city_demo DROP KEY idx_city_full, DROP KEY idx_city_p9;`.

---

<a id="e3"></a>

## E3. Clustered index, covering index, index thừa và không dùng

**Mục tiêu.** Tái hiện thí nghiệm `userinfo` (tr. 183–185, Bảng 7-2, 7-3) ở quy mô 500.000 hàng; kiểm chứng PK nằm trong secondary index, bẫy `(A)` thành `(A, B)`, warning của index trùng, view `sys`, invisible index và điều kiện reset bộ đếm index không dùng.

**Giả thuyết (Dự đoán).**
- Q2 nhanh hơn nhiều với index covering; Q1 chậm đi chút với index rộng hơn; có cả hai index thì cả hai truy vấn đều nhanh.
- `SELECT id ... WHERE state_id = 5` được `KEY(state_id)` cover.
- `WHERE state_id = 5 ORDER BY id` không cần filesort với `KEY(state_id)`, nhưng cần với `(state_id, city, address)`.

**Chuẩn bị** (bảng theo tr. 183–184, bỏ cột `gender`):

```sql
USE hpm7;
CREATE TABLE userinfo (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT,
  name VARCHAR(64) NOT NULL DEFAULT '', email VARCHAR(64) NOT NULL DEFAULT '',
  password VARCHAR(64) NOT NULL DEFAULT '', dob DATE DEFAULT NULL,
  address VARCHAR(255) NOT NULL DEFAULT '', city VARCHAR(64) NOT NULL DEFAULT '',
  state_id TINYINT UNSIGNED NOT NULL DEFAULT '0', zip VARCHAR(8) NOT NULL DEFAULT '',
  country_id SMALLINT UNSIGNED NOT NULL DEFAULT '0', account_type VARCHAR(32) NOT NULL DEFAULT '',
  verified TINYINT NOT NULL DEFAULT '0', allow_mail TINYINT UNSIGNED NOT NULL DEFAULT '0',
  parent_account INT UNSIGNED NOT NULL DEFAULT '0', closest_airport VARCHAR(3) NOT NULL DEFAULT '',
  PRIMARY KEY (id), UNIQUE KEY email (email), KEY country_id (country_id), KEY state_id (state_id)
) ENGINE=InnoDB;
INSERT INTO userinfo (name, email, password, dob, address, city, state_id, zip, country_id, account_type, verified, allow_mail, parent_account, closest_airport)
SELECT CONCAT('user', s.n), CONCAT('user', s.n, '@example.com'), MD5(CONCAT('pw', s.n)),
       DATE '1950-01-01' + INTERVAL (CRC32(CONCAT('b', s.n)) % 20000) DAY,
       CONCAT(1 + CRC32(CONCAT('h', s.n)) % 9999, ' ', c2.city, ' Street, Apt ', CRC32(CONCAT('ap', s.n)) % 500),
       c.city, 1 + CRC32(CONCAT('st', s.n)) % 50, LPAD(CRC32(CONCAT('z', s.n)) % 99999, 5, '0'),
       1 + CRC32(CONCAT('co', s.n)) % 200, ELT(1 + s.n % 3, 'standard', 'premium', 'trial'),
       s.n % 2, s.n % 2, 0, 'SGN'
FROM seq s
JOIN city_list c  ON c.k  = CRC32(CONCAT('ci', s.n)) % 600
JOIN city_list c2 ON c2.k = CRC32(CONCAT('sn', s.n)) % 600
WHERE s.n <= 500000;
ALTER TABLE userinfo ADD KEY state_id_2 (state_id, city, address);
ANALYZE TABLE userinfo;
```

Thực tế: 500.000 hàng, `state_id = 5` khớp 9.835 hàng, `address` dài trung bình 29,8 ký tự, `city` 9,1 ký tự. Số page: `PRIMARY` 5.351, `email` 1.830, `country_id` 673, `state_id` 481, `state_id_2` 1.831.

**Các bước và lệnh đo.** Ba cấu hình được tạo bằng invisible index (thao tác chỉ đổi metadata):

```sql
-- A: chỉ state_id
ALTER TABLE userinfo ALTER INDEX state_id VISIBLE, ALTER INDEX state_id_2 INVISIBLE;
-- B: chỉ state_id_2
ALTER TABLE userinfo ALTER INDEX state_id INVISIBLE, ALTER INDEX state_id_2 VISIBLE;
-- C: cả hai
ALTER TABLE userinfo ALTER INDEX state_id VISIBLE, ALTER INDEX state_id_2 VISIBLE;

-- Q1 và Q2 (Q2 được bọc để output nhỏ; plan giống SELECT state_id, city, address ...)
SELECT COUNT(*) FROM userinfo WHERE state_id=5;
SELECT COUNT(*), MAX(city), MAX(address) FROM (SELECT state_id, city, address FROM userinfo WHERE state_id=5) t;

-- số page logic một lần chạy
SELECT VARIABLE_VALUE INTO @rr0 FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests';
-- <truy vấn>
SELECT VARIABLE_VALUE - @rr0 FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests';
-- thời gian: chạy 5 lần rồi đọc 5 câu gần nhất
SELECT ROUND(TIMER_WAIT/1e9,2) AS ms, ROWS_EXAMINED FROM performance_schema.events_statements_history
WHERE THREAD_ID=PS_CURRENT_THREAD_ID() AND SQL_TEXT LIKE 'SELECT COUNT(*)%' ORDER BY EVENT_ID DESC LIMIT 5;
```

Cả ba cấu hình được chạy trọn vẹn hai lần (hai phiên).

**Kết quả.**

D0. Q1 và Q2 theo cấu hình (thời gian: trung vị 5 lần, phiên 1 / phiên 2; page: một lần chạy, phiên 1 / phiên 2):

| Cấu hình | Truy vấn | `key`, `Extra` | `rows` ước lượng | ms (trung vị) | Page buffer pool |
| --- | --- | --- | --- | --- | --- |
| A | Q1 | `state_id`, `Using index` | 9.835 | 3,85 / 2,92 | 176 / 177 |
| A | Q2 | `state_id`, (không) | 9.835 | 81,66 / 61,08 | 42.853 / 32.477 |
| B | Q1 | `state_id_2`, `Using index` | 19.338 | 3,47 / 3,56 | 213 / 204 |
| B | Q2 | `state_id_2`, `Using index` | 19.338 | 5,78 / 6,07 | 207 / 207 |
| C | Q1 | `state_id`, `Using index` | 9.835 | 3,12 / 4,02 | 192 / 1.316 |
| C | Q2 | **`state_id`**, (không) | 9.835 | 41,68 / 34,58 | 32.119 / 29.785 |

Giá trị thô phiên 1, cấu hình A, Q2 (ms): 194,16; 40,63; 41,42; 122,57; 81,66 (đã sắp lại: 40,63 / 41,42 / 81,66 / 122,57 / 194,16). `ROWS_EXAMINED` luôn là 9.835.

Optimizer trace của Q2 ở cấu hình C (đã rút gọn):

```text
phiên chọn state_id:   state_id   cost 4090.27 rows 9835  in_memory 0.650  index_only false  -> chosen
                       state_id_2 cost 4697.24 rows 19338 in_memory 0      index_only true   -> "cause": "cost"
phiên khác:            state_id   cost 4733.87 rows 9835  in_memory 0.702
                       state_id_2 cost 4697.24 rows 19338 in_memory 0      -> được chọn
```

Thống kê `n_diff_pfx01`: `state_id` 51, `state_id_2` 48 (thực tế 50).

D1–D3. PK trong secondary index và bẫy mở rộng index:

| Truy vấn | Plan | Bộ đếm |
| --- | --- | --- |
| `SELECT id ... IGNORE INDEX (state_id_2) WHERE state_id = 5` | `ref` `state_id`, `Using index` | |
| `SELECT * ... IGNORE INDEX (state_id_2) WHERE state_id = 5 ORDER BY id LIMIT 10` | `ref` `state_id`, không filesort | `Handler_read_key` 1, `_next` 9, `Sort_rows` 0 |
| Cùng truy vấn, `IGNORE INDEX (state_id)` | `index` trên `PRIMARY`, `rows` 255, `filtered` 3,91, `Using where` | `Handler_read_next` 420 |
| Cùng truy vấn, `FORCE INDEX (state_id_2)` | `ref` `state_id_2`, `Using filesort` | `Handler_read_next` 9.835, `Sort_rows` 10 |
| `SELECT id, city ... WHERE state_id = 5 ORDER BY id` chỉ với `state_id_2` | `Using index; Using filesort` | |

D4. Index trùng:

```sql
CREATE TABLE test_dup (ID INT NOT NULL PRIMARY KEY, A INT NOT NULL, B INT NOT NULL, UNIQUE(ID), INDEX(ID)) ENGINE=InnoDB;
SHOW WARNINGS;   -- rỗng; bảng có ba index PRIMARY, ID (unique), ID_2
CREATE TABLE test_dup2 (a INT NOT NULL, KEY k1 (a), KEY k2 (a)) ENGINE=InnoDB;
SHOW WARNINGS;
-- Warning | 1831 | Duplicate index 'k2' defined on the table 'hpm7.test_dup2'.
--   This is deprecated and will be disallowed in a future release.
```

D5. `sys.schema_redundant_indexes` (schema `hpm7`): `city_demo.idx_city_p9` (trội: `idx_city_full`), `sessions.idx_sid_p8` (trội: `idx_sid_full`), `test_dup.ID` (trội: `PRIMARY`), `test_dup.ID_2` (trội: `ID` và `PRIMARY`), `userinfo.state_id` (trội: `state_id_2`).

D6. `sys.schema_unused_indexes` sau E2 và E3: `payment.idx_sc`, `payment.idx_cs`, `sessions.idx_sid_full`, `test_dup.ID_2`, `userinfo.country_id`. Hai index của `payment` **đã được dùng** ở E2 nhưng vẫn bị liệt kê. Kiểm chứng nguyên nhân (sau khi khởi động lại):

```sql
SELECT COUNT(*) FROM userinfo WHERE state_id = 7;             -- 10.080
SELECT INDEX_NAME, COUNT_STAR FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA='hpm7' AND OBJECT_NAME='userinfo';        -- state_id: 10080
ALTER TABLE userinfo ADD KEY tmp_zip (zip);
-- chạy lại truy vấn trên performance_schema: mọi index, kể cả state_id, về 0
ALTER TABLE userinfo DROP KEY tmp_zip;
SELECT index_name FROM sys.schema_unused_indexes WHERE object_schema='hpm7' AND object_name='userinfo';
-- country_id, state_id, state_id_2
```

D7. Invisible index: ẩn cả `state_id` và `state_id_2` thì Q1 thành `ALL`, `rows` 494.612; `SET SESSION optimizer_switch='use_invisible_indexes=on'` thì quay về `ref` trên `state_id`, `Using index`.

D8. Chi phí ghi: chép 200.000 hàng (`INSERT INTO ... SELECT * FROM userinfo WHERE id <= 200000 ORDER BY id`) vào bảng rỗng cùng cấu trúc:

| Bảng đích | Lần 1 | Lần 2 | Lần 3 | Trung vị |
| --- | --- | --- | --- | --- |
| Chỉ `state_id` | 7,43 s (lần đầu, nhiễu khởi động) | 3,83 s | 4,16 s | **4,16 s** |
| `state_id` + `state_id_2` | 5,43 s | 5,84 s | 5,42 s | **5,43 s** |

**Diễn giải và đối chiếu với sách.**
- Covering index cho Q2 nhanh hơn 10–14 lần và đọc ít hơn 150–200 lần số page: cùng chiều với Bảng 7-2 (12,12 lên 28,04 QPS), mạnh hơn vì đo theo page logic thay vì QPS.
- Q1 với index rộng hơn đọc nhiều page hơn khoảng 15–20% nhưng thời gian chênh nằm trong nhiễu; sách đo được khoảng 8% QPS.
- Khác sách: ở cấu hình C, optimizer chọn index không covering cho Q2 trong hai trên ba phiên. Ước lượng `rows` qua `state_id_2` gấp đôi thực tế, `in_memory` bằng 0, chi phí hai phương án sát nhau. Đúng lời nhắc ở chú thích 4 và 12: luôn kiểm tra `EXPLAIN`.
- Bẫy `(A, ID)` ở tr. 185 được tái hiện đầy đủ (D2, D3). Warning ở tr. 182 chỉ xuất hiện khi index trùng cùng loại.
- Bộ đếm của `sys.schema_unused_indexes` bị đặt lại bởi `ALTER TABLE`, không chỉ bởi khởi động lại; phải tính đến khi đọc view này (tr. 185).
- Chi phí ghi tăng khoảng 30% với index thứ hai (sách: 70%, Bảng 7-3).

**Dọn dẹp.** `DROP TABLE test_dup, test_dup2;` và đưa mọi index của `userinfo` về `VISIBLE` (đã làm ở cuối mỗi script).

---

<a id="e4"></a>

## E4. Sắp xếp bằng index so với filesort

**Mục tiêu.** Chạy lại danh sách truy vấn ở tr. 181–182 trên 8.0.42; kiểm tra chú thích 12; đo trade-off "đọc theo index không covering so với quét bảng rồi sắp xếp" (tr. 180); thử descending index; đo tình huống A của ghi chú.

**Giả thuyết (Dự đoán).**
- Các truy vấn "dùng được" và "không dùng được" cho kết quả như sách.
- Với khoảng rộng, optimizer có thể bỏ index vì chi phí, không phải vì không dùng được.
- Đọc toàn bảng theo index không covering chậm hơn quét bảng cộng filesort.

**Chuẩn bị.** `rental_date` dùng kiểu `DATE` (Sakila dùng `DATETIME`) để điều kiện bằng một ngày khớp nhiều hàng; đây là lựa chọn của người viết ghi chú.

```sql
USE hpm7;
CREATE TABLE rental (
  rental_id INT NOT NULL AUTO_INCREMENT,
  rental_date DATE NOT NULL,
  inventory_id MEDIUMINT UNSIGNED NOT NULL,
  customer_id SMALLINT UNSIGNED NOT NULL,
  return_date DATETIME DEFAULT NULL,
  staff_id TINYINT UNSIGNED NOT NULL,
  note CHAR(80) NOT NULL DEFAULT '',
  PRIMARY KEY (rental_id),
  UNIQUE KEY rental_date (rental_date, inventory_id, customer_id),
  KEY idx_fk_inventory_id (inventory_id),
  KEY idx_fk_customer_id (customer_id),
  KEY idx_fk_staff_id (staff_id)
) ENGINE=InnoDB;
INSERT IGNORE INTO rental (rental_date, inventory_id, customer_id, return_date, staff_id, note)
SELECT DATE '2005-05-24' + INTERVAL (CRC32(CONCAT('rd', n)) % 365) DAY,
       1 + CRC32(CONCAT('inv', n)) % 5000, 1 + CRC32(CONCAT('cus', n)) % 2000,
       NULL, 1 + n % 2, REPEAT('r', 80)
FROM seq WHERE n <= 400000;

CREATE TABLE film (film_id SMALLINT UNSIGNED NOT NULL PRIMARY KEY, title VARCHAR(128) NOT NULL,
  description TEXT, KEY idx_title (title)) ENGINE=InnoDB;
INSERT INTO film SELECT n, CONCAT('FILM ', LPAD(n, 4, '0')), REPEAT('d', 200) FROM seq WHERE n <= 1000;
CREATE TABLE film_actor (actor_id SMALLINT UNSIGNED NOT NULL, film_id SMALLINT UNSIGNED NOT NULL,
  last_update TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (actor_id, film_id), KEY idx_fk_film_id (film_id)) ENGINE=InnoDB;
INSERT IGNORE INTO film_actor (actor_id, film_id)
SELECT 1 + CRC32(CONCAT('a', n)) % 200, 1 + CRC32(CONCAT('f', n)) % 1000 FROM seq WHERE n <= 5500;
ANALYZE TABLE rental, film, film_actor;
```

Thực tế: `rental` 399.907 hàng (một số bị bỏ do trùng khóa unique), 996 hàng ngày 2005-05-25, 24.246 hàng sau 2006-05-01; `film_actor` 5.103 hàng.

**Các bước và lệnh đo.** `EXPLAIN SELECT rental_id, staff_id FROM rental <điều kiện>` cho từng dòng của bảng kết quả; thêm các biến thể `FORCE INDEX FOR ORDER BY`, `LIMIT 10`, descending index và join như ghi ở bảng.

**Kết quả.**

| # | Điều kiện | `type`, `key` | `rows` | `Extra` |
| --- | --- | --- | --- | --- |
| S1 | `= '2005-05-25' ORDER BY inventory_id, customer_id` | `ref`, `rental_date` | 996 | (không) |
| S2 | `= '2005-05-25' ORDER BY inventory_id DESC` | `ref`, `rental_date` | 996 | `Backward index scan` |
| S3 | `> '2005-05-25' ORDER BY rental_date, inventory_id` | `ALL` | 397.005 | `Using where; Using filesort` |
| S3n | `> '2006-05-01' ORDER BY rental_date, inventory_id` | `range`, `rental_date` | 48.116 | `Using index condition` |
| S3 + `FORCE INDEX FOR ORDER BY (rental_date)` | | `range`, `rental_date` | 198.502 | `Using index condition` |
| S3 + `LIMIT 10` | | `range`, `rental_date` | 198.502 | `Using index condition`; đo được `Handler_read_key` 1, `_next` 9, `Sort_rows` 0 |
| S4 | `= '2005-05-25' ORDER BY inventory_id DESC, customer_id ASC` | `ref` | 996 | `Using filesort` |
| S5 | `= '2005-05-25' ORDER BY inventory_id, staff_id` | `ref` | 996 | `Using filesort` |
| S6 | `= '2005-05-25' ORDER BY customer_id` | `ref` | 996 | `Using filesort` |
| S7 | `> '2006-05-01' ORDER BY inventory_id, customer_id` | `range` | 48.116 | `Using index condition; Using filesort` |
| S8 | `= '2005-05-25' AND inventory_id IN (1,2) ORDER BY customer_id` | `range`, `key_len` 6 | 2 | `Using index condition; Using filesort` |
| S9 | S4 sau khi thêm `idx_mixed (rental_date, inventory_id DESC, customer_id)` | `ref`, `rental_date` | 996 | `Using filesort` (optimizer không chọn index mới) |
| S9 | ép `FORCE INDEX (idx_mixed)` | `ref`, `idx_mixed` | 1.084 | (không filesort) |
| S9 | để optimizer tự chọn, thêm `LIMIT 10` | `ref`, `rental_date` | 996 | `Using filesort` |

Join (J1):

| Truy vấn | Bảng 1 | Bảng 2 |
| --- | --- | --- |
| `SELECT actor_id, title FROM film_actor INNER JOIN film USING(film_id) ORDER BY actor_id` | `film`: `index` trên `idx_title`, `Using index; Using temporary; Using filesort` | `film_actor`: `ref` trên `idx_fk_film_id`, `Using index` |
| Thêm `STRAIGHT_JOIN` | `film_actor`: `index` trên `PRIMARY`, `Using index` | `film`: `eq_ref` |
| Thêm `LIMIT 10` (không `STRAIGHT_JOIN`) | `film_actor`: `index` trên `PRIMARY`, `rows` 10, `Using index` | `film`: `eq_ref` |

T1. Sắp gần toàn bảng theo cột có index nhưng cần cột ngoài index (`SELECT rental_id, note FROM rental ORDER BY customer_id LIMIT 399000, 1`):

| Cách | Plan | Page buffer pool | Bộ đếm | Thời gian (3 lần) | Trung vị |
| --- | --- | --- | --- | --- | --- |
| Optimizer tự chọn | `ALL`, `Using filesort` | 4.022 | `Handler_read_rnd_next` 399.908; `Sort_rows` 399.001; `Sort_merge_passes` 30 | 346,3 / 419,8 / 439,5 ms | **419,8 ms** |
| `FORCE INDEX (idx_fk_customer_id)` | `index` trên `idx_fk_customer_id` | 1.230.777 | `Handler_read_next` 399.000; `Sort_rows` 0 | 911,5 / 928,6 / 951,0 ms | **928,6 ms** |

<a id="e4-6"></a>

**E4.6 Tình huống A của ghi chú** (bảng `payment` của E2; thêm hai index rồi dùng invisible index để chỉ một index được thấy mỗi lần; `idx_sc` bị ẩn suốt phần này):

```sql
ALTER TABLE payment ADD KEY idx_cd (customer_id, payment_date), ADD KEY idx_cda (customer_id, payment_date, amount);
ANALYZE TABLE payment;
-- cấu hình 1: chỉ idx_cs; 2: chỉ idx_cd; 3: chỉ idx_cda
SELECT payment_id, amount, payment_date INTO @a, @b, @c
FROM payment WHERE customer_id = 1 ORDER BY payment_date DESC LIMIT 19, 1;   -- 1 lần làm nóng + 5 lần đo
```

| Index | Khách | `Extra` | ms (5 lần, đã sắp) | `ROWS_EXAMINED` | `SORT_ROWS` |
| --- | --- | --- | --- | --- | --- |
| `idx_cs` | 584 | `Using filesort` | 0,226; 0,226; 0,227; 0,243; 0,259 | 60 | 20 |
| `idx_cs` | 1 | `Using filesort` | 64,2; 67,3; 78,3; 97,4; 101,7 | 39.968 | 20 |
| `idx_cd` | 584 | `Backward index scan` | 0,306; 0,313; 0,317; 0,342; 0,395 | 20 | 0 |
| `idx_cd` | 1 | `Backward index scan` | 13,6; 16,5; 17,2; 19,3; 24,4 | 20 | 0 |
| `idx_cda` | 584 | `Backward index scan; Using index` | 0,16; 0,166; 0,174; 0,212; 0,227 | 20 | 0 |
| `idx_cda` | 1 | `Backward index scan; Using index` | 1,10; 1,75; 1,79; 1,79; 1,92 | 20 | 0 |

`EXPLAIN ANALYZE` cấu hình 1, khách 1: `Index lookup on payment using idx_cs (customer_id=1)` 39.948 hàng, rồi `Sort: payment.payment_date DESC, limit input to 20 row(s) per chunk`, tổng 77,9 ms; khoảng 151.000 page buffer pool.

Quan sát chưa giải thích hết: với `idx_cd` và khách 1, thời gian khoảng 17 ms dù chỉ 20 hàng được đếm. Đặt `SET SESSION eq_range_index_dive_limit = 1` (optimizer dùng statistics, ước lượng 39 hàng thay vì 81.108) thì cùng plan chỉ còn 0,16–0,19 ms (truy vấn chỉ lấy `payment_id`; 1,6–2,3 ms khi để index dive). `EXPLAIN` một mình chỉ tốn khoảng 0,2 ms. **Dự đoán** (chưa kiểm chứng): khi ước lượng số hàng lớn, phía thực thi chuẩn bị hoặc đọc trước nhiều hơn mức `LIMIT` cần.

**Diễn giải và đối chiếu với sách.**
- S1, S2 và S4–S8 khớp đúng danh sách ở tr. 181–182. S2 cho thấy 8.0 đọc index ngược (`Backward index scan`).
- Chú thích 12: trên 8.0.42, với khoảng phủ gần toàn bảng, optimizer chọn quét bảng và filesort; với khoảng hẹp hoặc có `LIMIT`, nó tự dùng index. T1 cho thấy lựa chọn quét bảng là **đúng** về chi phí khi phải lấy cột ngoài index: nhanh hơn 2,2 lần, ít page hơn khoảng 300 lần. Đây là bằng chứng cho câu ở tr. 180.
- J1 tái hiện đúng output ở tr. 182; `LIMIT 10` là trường hợp sắp xếp bằng index quan trọng nhất mà sách nhắc ở cuối mục.
- Descending index (Bổ sung ngoài sách) giải được trường hợp hai chiều, nhưng optimizer không tự chọn nó ở đây.
- E4.6: với giá trị đặc biệt, index ba sao nhanh hơn index một sao khoảng 40 lần.

**Dọn dẹp.** `ALTER TABLE rental DROP INDEX idx_mixed;` (đã làm). Đưa mọi index của `payment` về `VISIBLE` (đã làm ở cuối script).

---

<a id="e5"></a>

## E5. Thứ tự chèn PK, bảo trì bảng, statistics và adaptive hash index

**Mục tiêu.** Tái hiện Bảng 7-1 ở quy mô nhỏ (trong và ngoài bộ nhớ); xem `OPTIMIZE TABLE` trên InnoDB; quan sát statistics lấy mẫu, tự tính lại và cache của `INFORMATION_SCHEMA`; kiểm chứng adaptive hash index (roadmap mục 8).

**Giả thuyết (Dự đoán).**
- Khóa ngẫu nhiên tạo nhiều trang lá hơn; khi bảng lớn hơn buffer pool, chèn chậm hơn nhiều lần vì đọc page từ đĩa.
- `OPTIMIZE TABLE` đưa số trang về như chèn tuần tự.
- Statistics từ 20 page mẫu sai số nhỏ với dữ liệu đều; cardinality không phản ánh độ lệch.
- AHI phục vụ một phần lượt tìm khi tra lặp lại khóa nóng.

**Chuẩn bị và các bước** — phần 1, chèn (buffer pool 128 MB):

```sql
USE hpm7;
SET GLOBAL innodb_monitor_enable = 'index_page_splits';
CREATE TABLE t_seq       (id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY, pad CHAR(100) NOT NULL) ENGINE=InnoDB;
CREATE TABLE t_rand_int  (id INT UNSIGNED NOT NULL PRIMARY KEY, pad CHAR(100) NOT NULL) ENGINE=InnoDB;
CREATE TABLE t_uuid      (id CHAR(36) NOT NULL PRIMARY KEY, pad CHAR(100) NOT NULL) ENGINE=InnoDB;
CREATE TABLE t_uuid_ord  (id BINARY(16) NOT NULL PRIMARY KEY, pad CHAR(100) NOT NULL) ENGINE=InnoDB;
CREATE TABLE t_uuid_rand (id CHAR(36) NOT NULL PRIMARY KEY, pad CHAR(100) NOT NULL) ENGINE=InnoDB;

SET @t0 = NOW(6);
INSERT INTO t_seq (pad) SELECT REPEAT('a', 100) FROM seq WHERE n <= 300000 ORDER BY n;
SELECT TIMESTAMPDIFF(MICROSECOND, @t0, NOW(6))/1e6 AS sec;
-- n * 2654435761 mod 2^32 là một hoán vị của số 32 bit: khóa duy nhất, thứ tự ngẫu nhiên
INSERT INTO t_rand_int (id, pad) SELECT (n * 2654435761) % 4294967296, REPEAT('a', 100) FROM seq WHERE n <= 300000 ORDER BY n;
INSERT INTO t_uuid (id, pad) SELECT UUID(), REPEAT('a', 100) FROM seq WHERE n <= 300000 ORDER BY n;
INSERT INTO t_uuid_ord (id, pad) SELECT UUID_TO_BIN(UUID(), 1), REPEAT('a', 100) FROM seq WHERE n <= 300000 ORDER BY n;
-- UUID ngẫu nhiên thật sự (giống phiên bản 4) dựng từ MD5
INSERT INTO t_uuid_rand (id, pad)
SELECT CONCAT_WS('-', SUBSTR(m,1,8), SUBSTR(m,9,4), SUBSTR(m,13,4), SUBSTR(m,17,4), SUBSTR(m,21,12)), REPEAT('a', 100)
FROM (SELECT n, MD5(CONCAT('u', n)) AS m FROM seq WHERE n <= 300000) x ORDER BY n;

ANALYZE TABLE t_seq, t_rand_int, t_uuid, t_uuid_ord, t_uuid_rand;
SELECT table_name, MAX(IF(stat_name='n_leaf_pages', stat_value, NULL)) AS leaf_pages,
       MAX(IF(stat_name='size', stat_value, NULL)) AS total_pages
FROM mysql.innodb_index_stats
WHERE database_name='hpm7' AND table_name LIKE 't\_%' AND index_name='PRIMARY' GROUP BY table_name;
OPTIMIZE TABLE t_rand_int, t_uuid, t_uuid_rand;
```

Phần 2, I/O-bound: tắt server, khởi động lại với `--innodb-buffer-pool-size=16M`, `TRUNCATE` rồi chèn lại `t_seq`, `t_rand_int`, `t_uuid_rand` ba lần, đo thời gian cùng hiệu `Innodb_buffer_pool_reads` (page đọc từ đĩa) và `Innodb_buffer_pool_pages_flushed`. Sau đó khởi động lại với cấu hình mặc định.

**Kết quả phần 1** (một lần chạy mỗi bảng):

| Bảng | Thời gian chèn | Bộ đếm `index_page_splits` | Trang lá | Page cấp phát | Sau `OPTIMIZE`: trang lá / cấp phát |
| --- | --- | --- | --- | --- | --- |
| `t_seq` | 4,63 s | 2.444 | 2.440 | 2.468 | |
| `t_rand_int` | 6,01 s | 6.541 | 4.095 | 4.709 | 2.440 / 2.852 |
| `t_uuid` | 6,62 s | 9.643 | 3.094 | 3.115 | 3.093 / 3.563 |
| `t_uuid_ord` | 4,80 s | 12.326 | 2.680 | 2.726 | |
| `t_uuid_rand` | 10,97 s | 16.753 | 4.413 | 5.105 | 3.093 / 3.563 |

- Ba UUID đầu của `t_uuid` sau khi sắp: `a02b3fb0-c934-11f1-…`, `a02b5e31-c934-11f1-…`, `a02b5f97-c934-11f1-…`; ba lần gọi `UUID()` liên tiếp: `c20a6e21-…`, `c20a6e29-…`, `c20a6e2b-…` (phần đầu tăng dần).
- Ba UUID đầu của `t_uuid_rand`: `0000198c-b303-…`, `000020f2-2fa9-…`, `00002355-01e8-…`.
- `OPTIMIZE TABLE` trả về `note: Table does not support optimize, doing recreate + analyze instead` rồi `status: OK`.
- Bộ đếm `index_page_splits` tăng cả khi chèn tuần tự (khoảng một lần mỗi page mới) và có giá trị khó diễn giải cho từng bảng (ví dụ `t_uuid_ord` cao dù bố cục gần tuần tự); chỉ số đáng tin hơn là số trang lá.

**Kết quả phần 2** (buffer pool 16 MB, 3 lần):

| Bảng | Thời gian (s) | Trung vị | Page đọc từ đĩa | Page được flush |
| --- | --- | --- | --- | --- |
| `t_seq` | 3,62 / 4,40 / 4,43 | **4,40** | 1.226 / 1.226 / 1.628 | 2.481 / 2.063 / 3.252 |
| `t_rand_int` | 60,90 / 64,84 / 63,65 | **63,65** | 194.515 / 192.987 / 193.106 | 198.088 / 196.258 / 196.174 |
| `t_uuid_rand` | 66,82 / 62,96 / 61,61 | **62,96** | 169.583 / 168.609 / 168.557 | 176.097 / 175.661 / 174.311 |

**Phần 3, statistics** (bản sao `people_s` của bảng `people` ở E1):

```sql
CREATE TABLE people_s LIKE people; INSERT INTO people_s SELECT * FROM people; ANALYZE TABLE people_s;
SELECT index_name, stat_name, stat_value, sample_size FROM mysql.innodb_index_stats
WHERE database_name='hpm7' AND table_name='people_s' AND stat_name LIKE 'n_diff%';
SHOW INDEX FROM people_s;
ALTER TABLE people_s STATS_SAMPLE_PAGES=200; ANALYZE TABLE people_s;   -- rồi trả về DEFAULT và ANALYZE lại
UPDATE people_s SET last_name='Zzguest' WHERE id <= 150000;           -- 30% số hàng
SELECT SLEEP(20);
-- so mysql.innodb_index_stats, INFORMATION_SCHEMA.STATISTICS (cache), và cùng truy vấn với
SET SESSION information_schema_stats_expiry = 0;
EXPLAIN SELECT * FROM people_s WHERE last_name='Zzguest';
ANALYZE TABLE people_s;
CHECK TABLE people_s;
```

| Thống kê | Thực tế | 20 page mẫu | 200 page mẫu |
| --- | --- | --- | --- |
| `PRIMARY` `n_diff_pfx01` | 500.000 | 495.514 | 495.400 |
| `idx_name_dob` `n_diff_pfx01` (`last_name`) | 2.000 | 1.974 | 2.046 |
| `n_diff_pfx02` (`last_name, first_name`) | 153.954 | 154.388 | 154.798 |
| `n_diff_pfx03` (thêm `dob`) | 499.876 | 512.460 | 503.328 |
| `n_diff_pfx04` (thêm `id`) | 500.000 | 489.112 | 495.892 |

- `SHOW INDEX` và `INFORMATION_SCHEMA.STATISTICS` cho `last_name` 1.974, `first_name` 154.388, `dob` 495.514 (bị chặn ở ước lượng số hàng, dù `n_diff_pfx03` là 512.460).
- Sau `UPDATE` và 20 giây chờ: `mysql.innodb_index_stats` đã tự cập nhật (`n_diff_pfx01` = 2.097, `last_update` đổi); `INFORMATION_SCHEMA.STATISTICS` vẫn trả 1.922 (giá trị cache cũ); với `information_schema_stats_expiry = 0` trả 2.097; sau `ANALYZE TABLE` trả 1.942.
- `EXPLAIN ... last_name='Zzguest'`: `ref`, `rows` 247.757 (thực tế 150.000). `last_name='Allen'`: `rows` 186 (thực tế 186).
- `CHECK TABLE people_s`: `status OK`.

**Phần 4, adaptive hash index:**

```sql
DELIMITER //
CREATE PROCEDURE hot_lookups(IN cnt INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  DECLARE v CHAR(100);
  WHILE i < cnt DO
    SELECT bio INTO v FROM people WHERE id = 1000 + (i % 200);
    SELECT bio INTO v FROM people WHERE last_name = 'Allen' AND first_name = 'Kirl' AND dob = '1981-03-21';
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
SET GLOBAL innodb_monitor_enable = 'module_adaptive_hash';
-- mỗi lần: SET GLOBAL innodb_adaptive_hash_index = ON hoặc OFF; CALL hot_lookups(2000) để làm nóng;
-- chụp adaptive_hash_searches, adaptive_hash_searches_btree trong INFORMATION_SCHEMA.INNODB_METRICS;
-- CALL hot_lookups(20000); chụp lại và lấy hiệu
```

| Chế độ | Thời gian 3 lần (s) | Trung vị | Lượt tìm qua hash | Lượt tìm qua B-tree |
| --- | --- | --- | --- | --- |
| AHI tắt | 2,169 / 2,062 / 1,851 | 2,062 | 0 | 100.000 mỗi lần |
| AHI bật | 2,058 / 2,048 / 2,037 | 2,048 | 40.000 mỗi lần | 60.000 mỗi lần |

`SHOW ENGINE INNODB STATUS` (mục `INSERT BUFFER AND ADAPTIVE HASH INDEX`, đã rút gọn) có 8 dòng `Hash table size 34679, node heap has … buffer(s)` (8 phân vùng) và dòng tốc độ `2578.69 hash searches/s, 7654.98 non-hash searches/s` tại thời điểm chụp. Lưu ý `innodb_monitor_reset_all` không đặt lại các bộ đếm này khi chúng đang bật, nên phải lấy hiệu trước và sau.

**Diễn giải và đối chiếu với sách.**
- Bảng 7-1 (tr. 175) được tái hiện ở quy mô nhỏ: vừa bộ nhớ thì khóa ngẫu nhiên chậm 1,3–2,4 lần; vượt bộ nhớ thì chậm khoảng 14 lần, với gần 200.000 lần đọc page từ đĩa so với khoảng 1.200. Ba hệ quả ở tr. 177 (đọc lại page đã flush, page split, page thưa) đều thấy được qua số đo.
- Mật độ page: chèn tuần tự cho khoảng 123 hàng mỗi trang lá (300.000 / 2.440); chèn ngẫu nhiên chỉ khoảng 73 hàng (300.000 / 4.095), tức page đầy khoảng 60%. **Bổ sung ngoài sách (phép tính):** mỗi hàng khoảng 125 byte (4 + 100 byte dữ liệu, cộng header bản ghi, transaction ID 6 byte và rollback pointer 7 byte), nên 123 hàng chiếm khoảng 15 KB, gần mức 15/16 của page 16 KB mà sách nêu (tr. 176).
- `OPTIMIZE TABLE` trên InnoDB là dựng lại bảng (tr. 177, 189) và đưa số trang lá về như chèn tuần tự.
- `UUID()` của MySQL (phiên bản 1) không gây phân mảnh trong một lần nạp nhanh; bảng chỉ to hơn vì khóa dài. Hình phạt của sách ứng với UUID ngẫu nhiên.
- Statistics là ước lượng từ mẫu (tr. 188); trên 8.0 chúng bền vững, tự tính lại và được cache ở `INFORMATION_SCHEMA`. `innodb_stats_sample_pages` mà sách nhắc không còn.
- AHI tự hoạt động và tắt được khi đang chạy (tr. 159); nó phục vụ 40% lượt tìm nhưng không đổi thời gian ở workload này.

**Dọn dẹp.** `DROP PROCEDURE hot_lookups; DROP TABLE t_seq, t_rand_int, t_uuid, t_uuid_ord, t_uuid_rand, people_s;` `SET GLOBAL innodb_adaptive_hash_index = ON;` (đã trả về). `SET GLOBAL innodb_monitor_disable = 'module_adaptive_hash'; SET GLOBAL innodb_monitor_disable = 'index_page_splits';` Khởi động lại instance với cấu hình mặc định sau phần I/O-bound.

---

<a id="don-dep"></a>

## Dọn dẹp chung

- Mọi thay đổi ở mức session tự mất khi đóng kết nối; thay đổi global (`innodb_adaptive_hash_index`, `innodb_monitor_enable`) chỉ có trên instance riêng và mất khi khởi động lại.
- Xóa toàn bộ dữ liệu thí nghiệm: `DROP DATABASE hpm7;`, hoặc tắt instance rồi xóa thư mục datadir riêng.
- Tắt instance: `mysqladmin -h127.0.0.1 -P3317 -uroot shutdown`. Đã tắt sau khi hoàn tất.
