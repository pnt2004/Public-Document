# Thực hành — Phụ lục A: Nâng cấp MySQL và thay đổi schema tức thì

Về ghi chú: [Phụ lục A](../appendix-a.md) · Đáp án tự kiểm tra: [answers/appendix-a.md](../answers/appendix-a.md)

Phụ lục A chủ yếu là quy trình vận hành. Phần chạy được trên một máy là tính năng mà sách lấy làm ví dụ cho lợi ích của nâng cấp: **thêm cột tức thì** trong MySQL 8.0 (tr. 348), cùng một số kiểm tra trước nâng cấp chạy được trên một phiên bản. Nâng cấp thật giữa hai phiên bản và thử hạ cấp **chưa thực thi** vì máy chỉ có một phiên bản MySQL.

<a id="moi-truong"></a>

## Môi trường

> **Kết quả đã đo (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định trừ khi ghi khác, dữ liệu tổng hợp 1.000.000 hàng).**
>
> - MySQL Community Server 8.0.42 for Win64, cài sẵn tại `C:\Program Files\MySQL\MySQL Server 8.0\bin\`. Laptop Windows 11 Home, nhiều tiến trình nền: số đo thời gian chỉ để so sánh tương đối, **không phải benchmark production**.
> - Instance riêng: `--no-defaults` (không đọc `my.ini` của máy), cổng **3324**, `--bind-address=127.0.0.1`, `--mysqlx=OFF`, datadir riêng trong thư mục scratchpad, khởi tạo bằng `--initialize-insecure`. Không kết nối hay thay đổi MySQL của người dùng ở cổng 3306.
> - Cấu hình mặc định của 8.0.42: `innodb_buffer_pool_size` = 134217728 (128 MiB), `sql_mode` mặc định của 8.0. Riêng TN1 bật thêm các instrument stage lúc chạy (xem bên dưới). TN4 có một lần khởi động với `--upgrade=FORCE` và một tùy chọn deprecated, ghi rõ ở đó.
> - Bảng `lab.orders`: 1.000.000 hàng, tablespace `lab/orders` có `FILE_SIZE` = 121634816 byte (116 MiB) lúc tạo; tạo và nạp trong khoảng 16 giây.

**Khởi tạo, chạy, tắt instance** (Git Bash; `D` là thư mục riêng của bài thực hành):

```bash
BIN="C:/Program Files/MySQL/MySQL Server 8.0/bin"
D="<thư mục thực hành>/appA/exp"

# Khởi tạo một lần
"$BIN/mysqld" --no-defaults --initialize-insecure --datadir="$D/data" --console

# Chạy (để chạy nền)
"$BIN/mysqld" --no-defaults --datadir="$D/data" --port=3324 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=appA.sock --console > "$D/mysqld.log" 2>&1 &

# Chờ server sẵn sàng rồi kiểm tra đúng instance
for i in $(seq 1 30); do "$BIN/mysql" -h127.0.0.1 -P3324 -uroot -e "SELECT 1" >/dev/null 2>&1 && break; sleep 1; done
"$BIN/mysql" -h127.0.0.1 -P3324 -uroot -e "SELECT @@port, @@datadir, @@version"

# Luôn tắt khi xong, kể cả khi thí nghiệm lỗi
"$BIN/mysqladmin" -h127.0.0.1 -P3324 -uroot shutdown
```

Output kiểm tra đã chạy: `@@port` = 3324, `@@datadir` trỏ tới thư mục riêng, `@@version` = 8.0.42.

**Dữ liệu chung** (`01_setup.sql`):

```sql
CREATE DATABASE IF NOT EXISTS lab;
USE lab;
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
  id          INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id INT UNSIGNED NOT NULL,
  status      TINYINT UNSIGNED NOT NULL,
  amount      DECIMAL(10,2) NOT NULL,
  created_at  DATETIME NOT NULL,
  note        VARCHAR(100) NOT NULL,
  KEY idx_customer (customer_id)
) ENGINE=InnoDB;
SET SESSION cte_max_recursion_depth = 1000000;
INSERT INTO orders (customer_id, status, amount, created_at, note)
WITH RECURSIVE seq (n) AS (
  SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 1000000
)
SELECT n % 50000,
       n % 5,
       (n % 10000) / 100,
       TIMESTAMP '2024-01-01 00:00:00' + INTERVAL (n % 31536000) SECOND,
       REPEAT('x', 40)
FROM seq;
ANALYZE TABLE orders;
SELECT COUNT(*) AS row_count FROM orders;
SELECT NAME, FILE_SIZE, ALLOCATED_SIZE
FROM information_schema.INNODB_TABLESPACES WHERE NAME = 'lab/orders';
```

Output: `row_count` = 1000000; `lab/orders` có `FILE_SIZE` = `ALLOCATED_SIZE` = 121634816.

---

<a id="tn1"></a>

## TN1. INSTANT, INPLACE và COPY khi thêm một cột

**Mục tiêu.** Kiểm chứng câu của sách: từ MySQL 8.0, InnoDB thêm cột tức thì, không cần dựng lại cả bảng (tr. 348). So với hai cách cũ là rebuild tại chỗ (INPLACE) và chép bảng (COPY).

**Giả thuyết (Dự đoán).**

- INSTANT chỉ sửa metadata: thời gian gần như hằng số, không có stage `alter table (...)`, `TABLE_ID` và `SPACE` không đổi, hầu như không ghi dữ liệu.
- INPLACE dựng lại bảng: thời gian tỉ lệ với kích thước bảng, `TABLE_ID` và `SPACE` đổi, ghi lượng dữ liệu cỡ kích thước bảng, nhưng ít redo (index được dựng bằng sắp xếp rồi nạp hàng loạt).
- COPY chép từng hàng qua tầng handler: chậm nhất, `Handler_write` cỡ số hàng, nhiều redo.

**Chuẩn bị.** Bảng `lab.orders` ở trên. Bật instrument stage (`02_ps_setup.sql`):

```sql
UPDATE performance_schema.setup_instruments
   SET ENABLED = 'YES', TIMED = 'YES'
 WHERE NAME LIKE 'stage/innodb/alter%' OR NAME = 'stage/sql/copy to tmp table';
UPDATE performance_schema.setup_consumers
   SET ENABLED = 'YES'
 WHERE NAME LIKE 'events_stages%';
```

**Các bước và lệnh đo.** Mẫu `tn1_alter.sql.tpl`, thay `__ALGO__` bằng `INSTANT`, `INPLACE` hoặc `COPY`. Cuối mỗi lần chạy, cột vừa thêm được xóa bằng INSTANT để bảng trở lại như cũ.

```sql
USE lab;
TRUNCATE TABLE performance_schema.events_stages_history_long;
SELECT t.TABLE_ID, t.SPACE, t.TOTAL_ROW_VERSIONS, ROUND(s.FILE_SIZE / 1048576, 1) AS file_mib
  FROM information_schema.INNODB_TABLES t
  JOIN information_schema.INNODB_TABLESPACES s ON s.SPACE = t.SPACE
 WHERE t.NAME = 'lab/orders';
SELECT VARIABLE_VALUE INTO @dw0 FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_data_written';
SELECT VARIABLE_VALUE INTO @lw0 FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_os_log_written';
FLUSH STATUS;
SET @t0 = NOW(6);
ALTER TABLE orders ADD COLUMN c_new INT NULL, ALGORITHM=__ALGO__;
SELECT TIMESTAMPDIFF(MICROSECOND, @t0, NOW(6)) / 1e6 AS alter_sec;
SHOW SESSION STATUS LIKE 'Handler_write';
SELECT ROUND(((SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_data_written') - @dw0) / 1048576, 2) AS data_written_mib,
       ROUND(((SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_os_log_written') - @lw0) / 1048576, 2) AS redo_written_mib;
SELECT t.TABLE_ID, t.SPACE, t.TOTAL_ROW_VERSIONS, ROUND(s.FILE_SIZE / 1048576, 1) AS file_mib
  FROM information_schema.INNODB_TABLES t
  JOIN information_schema.INNODB_TABLESPACES s ON s.SPACE = t.SPACE
 WHERE t.NAME = 'lab/orders';
SELECT EVENT_NAME, COUNT(*) AS n, MAX(WORK_ESTIMATED) AS max_work_estimated,
       ROUND(SUM(TIMER_WAIT) / 1e12, 2) AS total_sec
  FROM performance_schema.events_stages_history_long
 WHERE THREAD_ID = PS_CURRENT_THREAD_ID()
   AND (EVENT_NAME LIKE 'stage/innodb/alter%' OR EVENT_NAME = 'stage/sql/copy to tmp table')
 GROUP BY EVENT_NAME;
ALTER TABLE orders DROP COLUMN c_new, ALGORITHM=INSTANT;
```

Chạy bằng `mysql -vvv` để client in "rows affected". Lặp 3 vòng, mỗi vòng theo thứ tự INSTANT, INPLACE, COPY:

```bash
for r in 1 2 3; do for a in instant inplace copy; do
  "$BIN/mysql" -h127.0.0.1 -P3324 -uroot -vvv < tn1_$a.sql > tn1_r${r}_$a.txt 2>&1
done; done
```

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định, 1.000.000 hàng).

| Lần | Thuật toán | `alter_sec` | Rows affected | `Handler_write` | Data ghi (MiB) | Redo ghi (MiB) | `TABLE_ID`, `SPACE` trước → sau | `TOTAL_ROW_VERSIONS` trước → sau |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | INSTANT | 0,020414 | 0 | 24 | 0,04 | 0,02 | 1066, 4 → 1066, 4 | 1 → 2 |
| 1 | INPLACE | 7,609527 | 0 | 25 | 111,29 | 0,28 | 1066, 4 → 1067, 5 | 3 → 0 |
| 1 | COPY | 10,899179 | 1000000 | 1000025 | 212,30 | 136,35 | 1067, 5 → 1068, 6 | 1 → 0 |
| 2 | INSTANT | 0,026237 | 0 | 24 | 0,96 | 0,02 | 1068, 6 → 1068, 6 | 1 → 2 |
| 2 | INPLACE | 6,843210 | 0 | 25 | 137,57 | 0,28 | 1068, 6 → 1069, 7 | 3 → 0 |
| 2 | COPY | 11,003326 | 1000000 | 1000025 | 221,43 | 136,27 | 1069, 7 → 1070, 8 | 1 → 0 |
| 3 | INSTANT | 0,021528 | 0 | 24 | 0,03 | 0,02 | 1070, 8 → 1070, 8 | 1 → 2 |
| 3 | INPLACE | 6,723026 | 0 | 25 | 138,56 | 0,28 | 1070, 8 → 1071, 9 | 3 → 0 |
| 3 | COPY | 10,663712 | 1000000 | 1000025 | 218,34 | 136,49 | 1071, 9 → 1072, 10 | 1 → 0 |

Trung vị thời gian: **INSTANT 0,0215 s · INPLACE 6,84 s · COPY 10,90 s**. `file_mib` = 116,0 ở mọi lần đo, trước và sau.

Stage ghi nhận trong `events_stages_history_long` (luồng hiện tại, đã rút gọn, vòng 2):

| Thuật toán | Stage | Số lần | `MAX(WORK_ESTIMATED)` | Tổng giây |
| --- | --- | --- | --- | --- |
| INSTANT | *(không có dòng nào: "Empty set")* | | | |
| INPLACE | `stage/innodb/alter table (read PK and internal sort)` | 4 | 23566 | 6,75 |
| INPLACE | `stage/innodb/alter table (merge sort)` | 1 | 23566 | 0 |
| INPLACE | `stage/innodb/alter table (insert)` | 1 | 26218 | 0 |
| INPLACE | `stage/innodb/alter table (log apply table)` | 2 | 26986 | 0,04 |
| INPLACE | `stage/innodb/alter table (end)` | 5 | 31482 | 0 |
| COPY | `stage/sql/copy to tmp table` | 1 | 994840 | 10,79 |

Lần chạy thử đầu tiên (trước 3 vòng trên, không tính vào trung vị): INSTANT 0,11 s, INPLACE 5,44 s, COPY 11,01 s. Lần INSTANT đầu chậm hơn các lần sau, có thể do mở bảng lần đầu; người viết chưa kiểm chứng nguyên nhân.

**Diễn giải và đối chiếu với sách.**

- **INSTANT không rebuild bảng**, đúng như tr. 348: `TABLE_ID` và `SPACE` giữ nguyên (cùng một tablespace), không có stage `alter table (...)` nào, dữ liệu ghi gần như bằng 0. Thay đổi duy nhất thấy được là `TOTAL_ROW_VERSIONS` tăng 1: InnoDB ghi nhận một phiên bản schema mới cho hàng (cơ chế row version là **Bổ sung ngoài sách**, xem [ghi chú §2.5](../appendix-a.md#instant-add-column)). Khoảng 24 lần `Handler_write` có lẽ là các hàng của data dictionary được cập nhật (suy luận, chưa kiểm chứng chi tiết).
- **INPLACE là rebuild**: `TABLE_ID` và `SPACE` mới, tức một bảng và tablespace mới thay chỗ bảng cũ; stage cho thấy đọc primary key, sắp xếp, nạp, rồi áp "row log" (các thay đổi đồng thời ghi trong lúc rebuild). Dữ liệu ghi xấp xỉ kích thước bảng. Redo rất ít (0,28 MiB) so với COPY: phù hợp với việc InnoDB dựng index bằng nạp hàng loạt rồi flush trang trực tiếp (**Bổ sung ngoài sách**, suy luận từ số đo).
- **COPY đi qua tầng handler** như một chuỗi `INSERT`: `Handler_write` = 1.000.025, rows affected = 1.000.000, redo khoảng 136 MiB. Đây là cách chậm nhất trong ba cách.
- **Hệ quả vận hành** (tr. 348): với công ty thêm cột thường xuyên, khác biệt giữa vài phần trăm giây và vài giây (trên bảng 116 MiB) sẽ thành khác biệt giữa vài phần trăm giây và nhiều giờ trên bảng hàng trăm GB. **Dự đoán**: thời gian INPLACE và COPY tăng gần tuyến tính theo kích thước bảng; chưa đo trên nhiều kích thước. Bằng chứng gián tiếp: ở TN2, INSTANT trên bảng 3 hàng cũng mất khoảng 0,02 s, như trên bảng 1 triệu hàng.

**Dọn dẹp và khôi phục.** Mỗi lần chạy đã tự `DROP COLUMN c_new`. Bảng vẫn dùng tiếp cho TN2, TN3.

---

<a id="tn2"></a>

## TN2. Điều kiện của INSTANT và giới hạn 64 row version

**Mục tiêu.** Xác định trên 8.0.42 thao tác nào chạy được với `ALGORITHM=INSTANT`, điều gì xảy ra khi chạm giới hạn row version, và câu `ADD COLUMN` không ghi `ALGORITHM` cư xử thế nào khi đó.

**Giả thuyết (Dự đoán).** Theo tài liệu MySQL (**Bổ sung ngoài sách**): thêm, xóa, đổi tên cột, đổi giá trị mặc định là INSTANT; đổi kiểu dữ liệu, thêm index thì không. Một bảng có tối đa 64 row version. Khi không ghi `ALGORITHM`, MySQL tự chọn thuật toán rẻ nhất mà thao tác cho phép, nên có thể chuyển sang rebuild mà không báo lỗi.

**Chuẩn bị và các bước.**

*(a) Ma trận thao tác* (`tn2_matrix.sql`, chạy với `mysql -vvv --force` để đi tiếp sau lỗi):

```sql
USE lab;
DROP TABLE IF EXISTS t2, t2_ft, t2_zip;
CREATE TABLE t2 (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id INT UNSIGNED NOT NULL,
  status ENUM('new','paid') NOT NULL DEFAULT 'new',
  note VARCHAR(100) NOT NULL DEFAULT ''
) ENGINE=InnoDB;
INSERT INTO t2 (customer_id) VALUES (1),(2),(3);
CREATE TABLE t2_ft (id INT PRIMARY KEY, body TEXT, FULLTEXT KEY ft_body (body)) ENGINE=InnoDB;
CREATE TABLE t2_zip (id INT PRIMARY KEY, v INT) ENGINE=InnoDB ROW_FORMAT=COMPRESSED;

ALTER TABLE t2 ADD COLUMN a1 INT NULL, ALGORITHM=INSTANT;                         -- 1
ALTER TABLE t2 ADD COLUMN a2 TINYINT NOT NULL DEFAULT 7, ALGORITHM=INSTANT;       -- 2
SELECT id, a2 FROM t2;
ALTER TABLE t2 ADD COLUMN a3 INT NULL AFTER customer_id, ALGORITHM=INSTANT;       -- 3
ALTER TABLE t2 DROP COLUMN a1, ALGORITHM=INSTANT;                                 -- 4
ALTER TABLE t2 RENAME COLUMN a3 TO a3_renamed, ALGORITHM=INSTANT;                 -- 5
ALTER TABLE t2 ALTER COLUMN a2 SET DEFAULT 9, ALGORITHM=INSTANT;                  -- 6
ALTER TABLE t2 MODIFY COLUMN status ENUM('new','paid','refunded') NOT NULL DEFAULT 'new', ALGORITHM=INSTANT; -- 7
ALTER TABLE t2 RENAME TO t2r, ALGORITHM=INSTANT;                                  -- 8
ALTER TABLE t2r RENAME TO t2, ALGORITHM=INSTANT;
ALTER TABLE t2 MODIFY COLUMN customer_id BIGINT UNSIGNED NOT NULL, ALGORITHM=INSTANT; -- 9
ALTER TABLE t2 MODIFY COLUMN note VARCHAR(200) NOT NULL DEFAULT '', ALGORITHM=INSTANT; -- 10
ALTER TABLE t2 ADD INDEX idx_c (customer_id), ALGORITHM=INSTANT;                  -- 11
ALTER TABLE t2 ADD COLUMN a4 INT NULL, ADD INDEX idx_a4 (a4), ALGORITHM=INSTANT;  -- 12
ALTER TABLE t2_ft ADD COLUMN a1 INT NULL, ALGORITHM=INSTANT;                      -- 13
ALTER TABLE t2_zip ADD COLUMN a1 INT NULL, ALGORITHM=INSTANT;                     -- 14
CREATE TEMPORARY TABLE t2_tmp (id INT PRIMARY KEY) ENGINE=InnoDB;
ALTER TABLE t2_tmp ADD COLUMN a1 INT NULL, ALGORITHM=INSTANT;                     -- 15
SELECT NAME, TOTAL_ROW_VERSIONS FROM information_schema.INNODB_TABLES WHERE NAME LIKE 'lab/t2%';
```

*(b) Giới hạn row version trên bảng nhỏ* (`tn2_rowversions.sql`, sinh bằng vòng lặp bash): tạo `t3 (id INT PRIMARY KEY, v INT)` với 2 hàng, chạy 65 câu `ALTER TABLE t3 ADD COLUMN c<i> INT NULL, ALGORITHM=INSTANT` với i = 1..65, rồi `ALTER TABLE t3 ADD COLUMN c65 INT NULL` **không ghi** `ALGORITHM`, rồi thêm `c66` bằng INSTANT. Sau mỗi giai đoạn đọc `TABLE_ID`, `TOTAL_ROW_VERSIONS`.

```bash
{ echo "USE lab;"; echo "DROP TABLE IF EXISTS t3;"
  echo "CREATE TABLE t3 (id INT PRIMARY KEY, v INT) ENGINE=InnoDB;"
  echo "INSERT INTO t3 VALUES (1,1),(2,2);"
  Q="SELECT TABLE_ID, TOTAL_ROW_VERSIONS FROM information_schema.INNODB_TABLES WHERE NAME='lab/t3';"
  echo "$Q"
  for i in $(seq 1 65); do echo "ALTER TABLE t3 ADD COLUMN c$i INT NULL, ALGORITHM=INSTANT;"; done
  echo "$Q"; echo "ALTER TABLE t3 ADD COLUMN c65 INT NULL;"; echo "$Q"
  echo "ALTER TABLE t3 ADD COLUMN c66 INT NULL, ALGORITHM=INSTANT;"; echo "$Q"
} > tn2_rowversions.sql
```

*(c) Âm thầm rebuild trên bảng lớn* (`tn2_silent_rebuild.sql`): trên `lab.orders` (đang có `TOTAL_ROW_VERSIONS` = 0 sau một lần rebuild), thêm 64 cột `z1..z64` bằng INSTANT, rồi đo `ALTER TABLE orders ADD COLUMN w INT NULL` không ghi `ALGORITHM`. Dọn bằng một câu `DROP COLUMN w, DROP COLUMN z1, ..., DROP COLUMN z64, ALGORITHM=INPLACE`.

*(d) Mở rộng `VARCHAR` bằng INPLACE trên bảng lớn:* `ALTER TABLE orders MODIFY COLUMN note VARCHAR(120) NOT NULL, ALGORITHM=INPLACE`, đo thời gian và `TABLE_ID`; rồi thu về `VARCHAR(100)`.

**Kết quả đã đo** (MySQL 8.0.42; output đã rút gọn).

*(a) Ma trận thao tác:*

| # | Thao tác với `ALGORITHM=INSTANT` | Kết quả trên 8.0.42 |
| --- | --- | --- |
| 1 | Thêm cột `INT NULL` ở cuối | OK, 0,02 s |
| 2 | Thêm cột `TINYINT NOT NULL DEFAULT 7` | OK, 0,02 s; `SELECT id, a2` trả 7 cho cả 3 hàng cũ |
| 3 | Thêm cột ở giữa (`AFTER customer_id`) | OK |
| 4 | Xóa cột | OK |
| 5 | Đổi tên cột | OK |
| 6 | Đổi giá trị mặc định | OK |
| 7 | Thêm phần tử vào cuối `ENUM` | OK |
| 8 | Đổi tên bảng | OK |
| 9 | Đổi `INT UNSIGNED` sang `BIGINT UNSIGNED` | `ERROR 1846 (0A000)`: ALGORITHM=INSTANT is not supported. Reason: Need to rebuild the table to change column type. Try ALGORITHM=COPY/INPLACE. |
| 10 | `VARCHAR(100)` sang `VARCHAR(200)` | `ERROR 1845 (0A000)`: ALGORITHM=INSTANT is not supported for this operation. Try ALGORITHM=COPY/INPLACE. |
| 11 | Thêm secondary index | `ERROR 1845`, như trên |
| 12 | Thêm cột kèm index trong cùng câu | `ERROR 1845`, như trên |
| 13 | Thêm cột vào bảng có `FULLTEXT` index | `ERROR 1846`, lý do in ra: "InnoDB presently supports one FULLTEXT index creation at a time" |
| 14 | Thêm cột vào bảng `ROW_FORMAT=COMPRESSED` | `ERROR 1845`, như trên |
| 15 | Thêm cột vào bảng tạm | `ERROR 1845`, gợi ý "Try ALGORITHM=COPY." |

Sau ma trận: `lab/t2` có `TOTAL_ROW_VERSIONS` = 4; `lab/t2_ft` và `lab/t2_zip` = 0.

*(b) Giới hạn row version:*

```text
TABLE_ID 1088, TOTAL_ROW_VERSIONS 0                      -- bảng mới
ALTER ... ADD COLUMN c1 ... c64, ALGORITHM=INSTANT       -- 64 lần Query OK
ALTER TABLE t3 ADD COLUMN c65 INT NULL, ALGORITHM=INSTANT
ERROR 4092 (HY000) at line 70: Maximum row versions reached for table lab/t3.
No more columns can be added or dropped instantly. Please use COPY/INPLACE.
TABLE_ID 1088, TOTAL_ROW_VERSIONS 64
ALTER TABLE t3 ADD COLUMN c65 INT NULL                   -- không ghi ALGORITHM
Query OK, 0 rows affected (0.06 sec)
TABLE_ID 1089, TOTAL_ROW_VERSIONS 0                      -- đã rebuild
ALTER TABLE t3 ADD COLUMN c66 INT NULL, ALGORITHM=INSTANT
TABLE_ID 1089, TOTAL_ROW_VERSIONS 1
```

*(c) Âm thầm rebuild trên `lab.orders` (1.000.000 hàng):*

```text
TABLE_ID 1091, TOTAL_ROW_VERSIONS 0
64 x ALTER TABLE orders ADD COLUMN z<i> INT NULL, ALGORITHM=INSTANT   -- đều Query OK
TABLE_ID 1091, TOTAL_ROW_VERSIONS 64
ALTER TABLE orders ADD COLUMN w INT NULL                               -- không ghi ALGORITHM
Query OK, 0 rows affected (13.42 sec)
TABLE_ID 1092, TOTAL_ROW_VERSIONS 0
ALTER TABLE orders DROP COLUMN w, DROP COLUMN z1, ..., ALGORITHM=INPLACE
Query OK, 0 rows affected (7.58 sec)
TABLE_ID 1093, TOTAL_ROW_VERSIONS 0
```

*(d) `VARCHAR` trên `lab.orders`:* `MODIFY COLUMN note VARCHAR(120) ... ALGORITHM=INPLACE` mất **0,013 s**, `TABLE_ID` không đổi. Thu về `VARCHAR(100)` bằng INPLACE bị từ chối (`ERROR 1846`: Cannot change column type INPLACE. Try ALGORITHM=COPY.); bằng COPY mất **11,03 s**, 1.000.000 rows affected, `TABLE_ID` đổi.

> Ghi chú trung thực: lần thử đầu của (c) dùng vòng lặp thêm rồi xóa cùng một cột `z`. Bảng khi đó đang có sẵn 1 row version nên lần xóa cuối chạm giới hạn và báo lỗi 4092, để lại cột `z`. Lần đo trong bảng trên được làm lại từ đầu sau khi rebuild bảng về 0 row version.

**Diễn giải và đối chiếu với sách.**

- Sách chỉ nói "thêm cột tức thì" (tr. 348). Trên 8.0.42, phạm vi rộng hơn (thêm ở giữa, xóa, đổi tên cột) nhưng cũng có điều kiện rõ ràng. Mọi thao tác phải đổi định dạng lưu trữ của dữ liệu đang có (đổi kiểu cột, dựng index) đều không thể "tức thì", vì chúng buộc phải đọc và ghi lại dữ liệu.
- `TOTAL_ROW_VERSIONS` = 4 ở `t2` cho thấy chỉ thêm và xóa cột mới tốn row version (a1, a2, a3 thêm; a1 xóa). Đổi tên, đổi mặc định, thêm phần tử `ENUM`, đổi tên bảng chỉ sửa metadata và không cần phiên bản hàng mới.
- Lý do in ra cho bảng `FULLTEXT` (#13) không khớp với thao tác đang làm; nên đọc mã lỗi (1846) và tài liệu thay vì tin vào câu lý do.
- **Điểm quan trọng nhất cho vận hành:** khi hết row version, câu `ADD COLUMN` không ghi `ALGORITHM` **không báo lỗi** mà rebuild bảng (13,42 s trên 1 triệu hàng, sẽ là nhiều giờ trên bảng lớn). Ghi rõ `ALGORITHM=INSTANT` biến tình huống này thành một lỗi tức thì, dễ xử lý hơn.
- Mở rộng `VARCHAR` trong cùng nhóm độ dài (cả 100 và 120 ký tự `utf8mb4` đều cần 2 byte lưu độ dài) là thay đổi chỉ-metadata nhưng được xếp loại INPLACE, không phải INSTANT. Thu nhỏ `VARCHAR` thì phải chép bảng.

**Dọn dẹp và khôi phục.** Lệnh dọn: `DROP TABLE lab.t2, lab.t2_ft, lab.t2_zip, lab.t3;` (bảng tạm `t2_tmp` tự mất khi phiên kết thúc). Trong lần chạy này, bốn bảng được giữ lại tới hết TN4 (chúng xuất hiện trong output `mysqlcheck`) và vẫn nằm trong datadir riêng. `lab.orders` đã trở về cấu trúc ban đầu sau bước dọn ở (c) và (d).

---

<a id="tn3"></a>

## TN3. INSTANT vẫn cần metadata lock

**Mục tiêu.** Kiểm tra xem một thay đổi "tức thì" có thể bị chặn và gây chặn hay không. Điều này quan trọng khi đưa INSTANT vào quy trình triển khai (tr. 348 nói về lợi ích; sách không nói về khóa, phần này là **Bổ sung ngoài sách** có kiểm chứng).

**Giả thuyết (Dự đoán).** ALTER cần exclusive metadata lock (MDL) trên bảng, dù chỉ trong chốc lát. Một transaction đang mở và đã đọc bảng giữ shared MDL tới khi kết thúc, nên ALTER phải chờ. Trong lúc ALTER chờ, các câu đọc mới xếp hàng sau yêu cầu exclusive đó cũng phải chờ.

**Chuẩn bị.** Bảng `lab.orders`. Bốn phiên chạy song song từ một script bash:

```bash
M() { "$BIN/mysql" -h127.0.0.1 -P3324 -uroot -vvv lab "$@"; }
M --force < tn3_A.sql > tn3_A.txt 2>&1 &   # phiên A: giữ transaction mở 25 s
sleep 2
M --force < tn3_B.sql > tn3_B.txt 2>&1 &   # phiên B: ALTER INSTANT, lock_wait_timeout = 10
sleep 2
M --force < tn3_D.sql > tn3_D.txt 2>&1 &   # phiên D: một SELECT bình thường
sleep 2
M --force < tn3_C.sql > tn3_C.txt 2>&1     # phiên C: quan sát
wait
```

```sql
-- tn3_A.sql
START TRANSACTION;
SELECT COUNT(*) FROM orders WHERE id <= 10;
SELECT SLEEP(25);
COMMIT;

-- tn3_B.sql
SET SESSION lock_wait_timeout = 10;
SET @t0 = NOW(6);
ALTER TABLE orders ADD COLUMN m INT NULL, ALGORITHM=INSTANT;
SELECT TIMESTAMPDIFF(MICROSECOND, @t0, NOW(6)) / 1e6 AS alter_wait_sec;

-- tn3_D.sql
SET @t0 = NOW(6);
SELECT COUNT(*) FROM orders WHERE id <= 10;
SELECT TIMESTAMPDIFF(MICROSECOND, @t0, NOW(6)) / 1e6 AS select_wait_sec;

-- tn3_C.sql
SELECT ml.OBJECT_NAME, ml.LOCK_TYPE, ml.LOCK_STATUS, t.PROCESSLIST_ID,
       LEFT(t.PROCESSLIST_INFO, 60) AS info
  FROM performance_schema.metadata_locks ml
  JOIN performance_schema.threads t ON t.THREAD_ID = ml.OWNER_THREAD_ID
 WHERE ml.OBJECT_SCHEMA = 'lab' AND ml.OBJECT_NAME = 'orders'
 ORDER BY t.PROCESSLIST_ID;
SELECT ID, TIME, STATE, LEFT(INFO, 60) AS info FROM information_schema.PROCESSLIST
 WHERE DB = 'lab' AND COMMAND <> 'Sleep' ORDER BY ID;
```

**Kết quả đã đo** (MySQL 8.0.42; output đã rút gọn).

Phiên C, khoảng giây thứ 6:

```text
+-------------+-------------------+-------------+----------------+-------------------------------------------------------------+
| OBJECT_NAME | LOCK_TYPE         | LOCK_STATUS | PROCESSLIST_ID | info                                                        |
+-------------+-------------------+-------------+----------------+-------------------------------------------------------------+
| orders      | SHARED_READ       | GRANTED     |             28 | SELECT SLEEP(25)                                            |
| orders      | SHARED_UPGRADABLE | GRANTED     |             29 | ALTER TABLE orders ADD COLUMN m INT NULL, ALGORITHM=INSTANT |
| orders      | EXCLUSIVE         | PENDING     |             29 | ALTER TABLE orders ADD COLUMN m INT NULL, ALGORITHM=INSTANT |
| orders      | SHARED_READ       | PENDING     |             30 | SELECT COUNT(*) FROM orders WHERE id <= 10                  |
+-------------+-------------------+-------------+----------------+-------------------------------------------------------------+

+----+------+---------------------------------+--------------------------------------------------------------+
| ID | TIME | STATE                           | info                                                         |
+----+------+---------------------------------+--------------------------------------------------------------+
| 28 |    7 | User sleep                      | SELECT SLEEP(25)                                             |
| 29 |    4 | Waiting for table metadata lock | ALTER TABLE orders ADD COLUMN m INT NULL, ALGORITHM=INSTANT  |
| 30 |    2 | Waiting for table metadata lock | SELECT COUNT(*) FROM orders WHERE id <= 10                   |
+----+------+---------------------------------+--------------------------------------------------------------+
```

Phiên B: `ERROR 1205 (HY000) at line 3: Lock wait timeout exceeded; try restarting transaction`, `alter_wait_sec` = **10,014002**.
Phiên D: `COUNT(*)` = 10, nhưng `select_wait_sec` = **7,949628** (một truy vấn đọc 10 hàng theo primary key).
Phiên A: `SLEEP(25)` chạy hết 25,02 s rồi `COMMIT`.

Truy vấn `INFORMATION_SCHEMA.PROCESSLIST` ở phiên C sinh 1 cảnh báo; TN4 cho thấy đó là cảnh báo deprecated.

**Diễn giải và đối chiếu với sách.**

- `ALGORITHM=INSTANT` không chép dữ liệu, nhưng **vẫn phải lấy exclusive MDL** để đổi định nghĩa bảng. Phiên A chỉ đọc 10 hàng nhưng transaction còn mở nên vẫn giữ `SHARED_READ` tới khi `COMMIT`.
- Yêu cầu `EXCLUSIVE` đang chờ của B làm câu đọc mới của D cũng phải chờ (`SHARED_READ ... PENDING`). D chỉ chạy được khi B bỏ cuộc lúc hết `lock_wait_timeout`. Nếu không đặt timeout nhỏ, B chờ theo mặc định của `lock_wait_timeout` (31536000 giây, tức một năm, **Bổ sung ngoài sách**), và mọi truy vấn trên bảng sẽ dồn lại phía sau.
- Chương 3 mô tả đúng tình huống này: metadata lock được giữ tới hết transaction, DDL chờ ở trạng thái chờ metadata lock, và bảng `metadata_locks` cho biết ai giữ, ai chờ (tr. 63–64). Thí nghiệm này nối kiến thức đó với lời khuyên nâng cấp để dùng INSTANT (tr. 348).
- Kết luận vận hành: chạy DDL tức thì với `lock_wait_timeout` nhỏ và thử lại, kiểm tra transaction dài trước, tránh giờ cao điểm.

**Dọn dẹp và khôi phục.** ALTER của B thất bại nên bảng không đổi. Không cần dọn.

---

<a id="tn4"></a>

## TN4. Kiểm tra trước nâng cấp chạy được trên một phiên bản

**Mục tiêu.** Kiểm chứng ba ý của phụ lục bằng những gì làm được trên một phiên bản duy nhất:

- `mysql_upgrade` đã được chuyển vào lúc server khởi động từ 8.0 (chú thích 2, tr. 347);
- phiên bản mới hay đổi tên hoặc bỏ biến cấu hình, nên phải rà tệp cấu hình (tr. 345);
- bước "Verify target" dựa vào `read_only` (tr. 346), và `super_read_only` phải tắt cho `mysql_upgrade` ở bản cũ (tr. 347).

**Giả thuyết (Dự đoán).** Trên 8.0.42, `mysql_upgrade` chỉ còn là vỏ thông báo; server có cơ chế tự nâng cấp lúc khởi động và có thể ép chạy lại. Tùy chọn đã bị bỏ làm server từ chối khởi động; tùy chọn deprecated chỉ sinh cảnh báo.

**Chuẩn bị.** Dùng tiếp instance và database `lab` của TN1–TN3. Không cần dữ liệu thêm. Bước (d) cần tắt và khởi động lại instance.

**Các bước và lệnh đo.**

```bash
# (a) Client mysql_upgrade và kiểm tra bảng
"$BIN/mysql_upgrade" -h127.0.0.1 -P3324 -uroot
"$BIN/mysqlcheck"   -h127.0.0.1 -P3324 -uroot --check-upgrade --databases lab

# (b) Cảnh báo deprecated lúc chạy
"$BIN/mysql" -h127.0.0.1 -P3324 -uroot -vvv -e "SHOW GLOBAL STATUS LIKE 'Deprecated%'; SELECT COUNT(*) FROM information_schema.PROCESSLIST; SHOW WARNINGS; SHOW GLOBAL STATUS LIKE 'Deprecated%'; SET GLOBAL expire_logs_days = 7"

# (c) Kiểm tra tĩnh tùy chọn cấu hình, không khởi động server
for opt in --query_cache_size=0 --query_cache_type=0 --innodb_log_file_size=256M \
           --expire_logs_days=7 --default-authentication-plugin=mysql_native_password \
           --innodb_buffer_pool_size=256M; do
  "$BIN/mysqld" --no-defaults --validate-config $opt; echo "exit=$?"
done

# (d) Tắt instance, khởi động lại với --upgrade=FORCE và một tùy chọn deprecated
"$BIN/mysqladmin" -h127.0.0.1 -P3324 -uroot shutdown
"$BIN/mysqld" --no-defaults --datadir="$D/data" --port=3324 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=appA.sock --upgrade=FORCE \
  --default-authentication-plugin=mysql_native_password --log-error-verbosity=3 \
  --console > "$D/mysqld_run2_upgrade_force.log" 2>&1 &
grep -i "upgrad\|deprecat" "$D/mysqld_run2_upgrade_force.log"
```

*(e) Bước "Verify target"* (`tn4_verify_target.sql`, **Ví dụ minh họa do AI xây dựng**; so sánh chuỗi phiên bản ở đây là đơn giản hóa):

```sql
SET @target_version = '8.4.0';
SELECT @@hostname AS host, @@port AS port, @@version AS current_version,
       @@read_only AS read_only, @@super_read_only AS super_read_only,
       CASE
         WHEN @@read_only = 0 THEN 'SKIP: writable, may be taking production writes'
         WHEN @@version = @target_version THEN 'SKIP: already upgraded'
         ELSE 'OK to upgrade'
       END AS decision;
SET GLOBAL super_read_only = ON;
SELECT @@read_only AS read_only, @@super_read_only AS super_read_only,
       CASE
         WHEN @@read_only = 0 THEN 'SKIP: writable, may be taking production writes'
         WHEN @@version = @target_version THEN 'SKIP: already upgraded'
         ELSE 'OK to upgrade'
       END AS decision;
CREATE TABLE lab.should_fail (id INT PRIMARY KEY);
SET GLOBAL read_only = OFF;
SELECT @@read_only AS read_only, @@super_read_only AS super_read_only;
```

**Kết quả đã đo** (MySQL 8.0.42; output đã rút gọn).

*(a)* `mysql_upgrade` in thông báo (đã rút gọn): client đã deprecated, việc nâng cấp nay do server làm; muốn nâng cấp thì khởi động binary mới với datadir cũ; dùng `--upgrade=NONE` để tránh nâng cấp ngoài ý muốn, `--upgrade=FORCE` để chạy lại quy trình nâng cấp theo yêu cầu, `--upgrade=MINIMAL` khi nâng cấp lỗi liên tục. Mã thoát 0, không làm gì thêm. `mysqlcheck --check-upgrade` báo `OK` cho cả 5 bảng của `lab`.

*(b)*

```text
Deprecated_use_i_s_processlist_count  1     -- trước (đã có từ phiên C của TN3)
SELECT COUNT(*) FROM information_schema.PROCESSLIST  -> 1 row in set, 1 warning
Warning | 1287 | 'INFORMATION_SCHEMA.PROCESSLIST' is deprecated and will be removed in a future
release. Please use performance_schema.processlist instead
Deprecated_use_i_s_processlist_count  2     -- sau
SET GLOBAL expire_logs_days = 7
ERROR 3683 (HY000): The option expire_logs_days and binlog_expire_logs_seconds cannot be used
together. Please use binlog_expire_logs_seconds to set the expire time (expire_logs_days is deprecated)
```

*(c)*

| Tùy chọn | Kết quả `--validate-config` |
| --- | --- |
| `--query_cache_size=0` | `[ERROR] [MY-000067] [Server] unknown variable 'query_cache_size=0'.` rồi `Aborting`, exit=1 |
| `--query_cache_type=0` | `unknown variable 'query_cache_type=0'.`, `Aborting`, exit=1 |
| `--innodb_log_file_size=256M` | Không in gì, exit=0 |
| `--expire_logs_days=7` | Không in gì, exit=0 |
| `--default-authentication-plugin=mysql_native_password` | Không in gì, exit=0 |
| `--innodb_buffer_pool_size=256M` | Không in gì, exit=0 |

*(d)* Error log (đã rút gọn):

```text
[Warning] [MY-010918] [Server] 'default_authentication_plugin' is deprecated and will be removed in a future release. Please use authentication_policy instead.
[System]  [MY-013381] [Server] Server upgrade from '80042' to '80042' started.
[Note]    [MY-013386] [Server] Running queries to upgrade MySQL server.
[Note]    [MY-013387] [Server] Upgrading system table data.
[Note]    [MY-013385] [Server] Upgrading the sys schema.
[Note]    [MY-013400] [Server] Upgrade of help tables started.
[Note]    [MY-013400] [Server] Upgrade of help tables completed.
[System]  [MY-013381] [Server] Server upgrade from '80042' to '80042' completed.
[Warning] [MY-013360] [Server] Plugin mysql_native_password reported: ''mysql_native_password' is deprecated and will be removed in a future release. Please use caching_sha2_password instead'
```

Từ "started" (07:30:56.59) tới "completed" (07:31:01.99) khoảng **5,4 giây**, trên datadir gần như trống. Sau đó `SELECT @@port, @@datadir, @@version` vẫn trả 3324, datadir riêng, 8.0.42. Tài khoản `root` dùng plugin `caching_sha2_password`.

*(e)*

```text
current_version 8.0.42 | read_only 0 | super_read_only 0 | decision: SKIP: writable, may be taking production writes
SET GLOBAL super_read_only = ON
read_only 1 | super_read_only 1 | decision: OK to upgrade
CREATE TABLE lab.should_fail (...)
ERROR 1290 (HY000): The MySQL server is running with the --super-read-only option so it cannot execute this statement
SET GLOBAL read_only = OFF
read_only 0 | super_read_only 0
```

**Diễn giải và đối chiếu với sách.**

- *(a), (d)* khớp với chú thích 2 (tr. 347): trên 8.0.42, nâng cấp bảng hệ thống, `sys` và bảng help do server làm lúc khởi động; `mysql_upgrade` không còn vai trò. Mốc chính xác là 8.0.16 (**Bổ sung ngoài sách**). Với playbook trên 8.0, bước 7 thành "đọc error log tìm dòng `Server upgrade ... completed`". `--upgrade=NONE` là cách chặn nâng cấp ngoài ý muốn ở tầng server, bổ sung cho bước "Verify target" (tr. 346).
- *(b), (c), (d)* minh họa câu "phiên bản mới hay đổi tên hoặc bỏ hẳn biến" (tr. 345). Tùy chọn **đã bị bỏ** (query cache, bị bỏ trong 8.0, **Bổ sung ngoài sách**) làm server **không khởi động được**; `--validate-config` bắt được việc này mà không cần khởi động. Tùy chọn **deprecated** thì `--validate-config` bỏ qua; chúng chỉ hiện thành cảnh báo trong error log hoặc cảnh báo 1287 khi chạy, và biến trạng thái `Deprecated_use_*` đếm số lần dùng. Đây là danh sách việc cần làm trước lần nâng cấp sau.
- *(e)* khớp với bước 1 (tr. 346) và chú thích 6 của chương 9 (tr. 240): bật `super_read_only` kéo theo `read_only`, tắt `read_only` kéo theo tắt `super_read_only`. `super_read_only` chặn cả `root`, nên với bản cũ hơn 8.0 phải tạm tắt nó cho `mysql_upgrade` (tr. 347).
- Giới hạn: thí nghiệm "nâng cấp" từ 8.0.42 lên chính 8.0.42 chỉ cho thấy **cơ chế**, không cho thấy thay đổi dữ liệu dictionary hay hành vi giữa hai phiên bản khác nhau.

**Dọn dẹp và khôi phục.** `super_read_only` và `read_only` đã về 0. Instance đã được tắt bằng `mysqladmin shutdown` sau khi kiểm tra. Lần khởi động sau nên bỏ `--upgrade=FORCE` và `--default-authentication-plugin`.

---

<a id="tn5"></a>

## TN5. Nâng cấp thật, thử hạ cấp và pt-upgrade (Chưa thực thi)

**Lý do chưa thực thi.** Máy chỉ có một phiên bản MySQL (8.0.42), môi trường không cho tải phần mềm từ internet, và không có Percona Toolkit. Dưới đây là quy trình để chạy lại khi có binary thứ hai (ví dụ một bản 8.4 LTS), cùng các **Dự đoán** cần kiểm chứng.

**Mục tiêu.**

1. Quan sát nâng cấp in-place thật (tr. 346) và các bước nâng cấp tự động lúc khởi động (tr. 347).
2. Kiểm chứng cảnh báo của sách: không hạ cấp được về bản cũ hơn (tr. 345).
3. So kết quả truy vấn giữa hai phiên bản theo cách pt-upgrade làm (tr. 346).

**Giả thuyết (Dự đoán).** Binary mới khởi động trên datadir 8.0.42 sẽ tự nâng cấp data dictionary và bảng hệ thống, ghi các dòng "Server upgrade from ..." vào error log. Binary 8.0.42 khởi động lại trên datadir đã nâng cấp sẽ bị từ chối. Phần lớn truy vấn đơn giản cho cùng kết quả trên hai phiên bản; khác biệt, nếu có, dễ nằm ở plan hơn là ở kết quả.

**Chuẩn bị.** Một binary MySQL thứ hai (ví dụ 8.4 LTS) đặt ở thư mục riêng, không cài đè lên bản 8.0.42; datadir của TN1–TN4 đã tắt sạch.

**Quy trình đề xuất (Ví dụ minh họa do AI xây dựng).**

1. **Đường lui trước tiên** (tr. 345): tắt instance 8.0.42, chép nguyên thư mục datadir sang `data_backup_80042` (đây là bản "backup" vật lý lạnh cho phòng thí nghiệm). Tạo thêm một bản logic: `mysqldump --single-transaction --routines --events --databases lab > lab_80042.sql`.
2. **Rà cấu hình** cho bản đích: chạy `mysqld --no-defaults --validate-config <các tùy chọn đang dùng>` bằng **binary mới**.
3. **Nâng cấp in-place** (tr. 346): khởi động binary mới với `--no-defaults --datadir=<datadir 8.0.42> --port=3324 ...`. Đọc error log, tìm dòng "Server upgrade from '80042' to '...' started/completed". **Dự đoán:** dòng này xuất hiện và thời gian tăng theo số đối tượng trong data dictionary.
4. **Xác nhận** (tr. 347): `SELECT 1`, `SELECT @@version`, `CHECK TABLE lab.orders`, đếm lại số hàng, chạy lại TN2 để xem giới hạn row version và điều kiện INSTANT có đổi không.
5. **Thử hạ cấp** (tr. 345): tắt server, khởi động lại **binary 8.0.42** trên datadir vừa nâng cấp, `--console`. **Dự đoán:** server từ chối khởi động, error log báo datadir có phiên bản data dictionary mới hơn binary. Không cố sửa; khôi phục bằng cách chép lại `data_backup_80042` (bước 1), hoặc khởi tạo datadir mới rồi nạp `lab_80042.sql`, đúng như sách nói về export và import (tr. 345).
6. **So sánh kiểu pt-upgrade** (tr. 346): dựng hai instance từ cùng bản sao dữ liệu (một 8.0.42, một bản mới), bật slow query log với `long_query_time = 0` trên instance cũ khi chạy một bộ truy vấn mẫu, rồi chạy `pt-upgrade` với slow log làm đầu vào và hai DSN làm đích. Nếu không có Percona Toolkit: chạy cùng tệp truy vấn trên hai instance, lưu output, so bằng `diff`, kèm `EXPLAIN FORMAT=TREE` của từng truy vấn.

**Tiêu chí đánh giá.** Không có lỗi mới; số hàng và dữ liệu hàng giống nhau; khác biệt plan được ghi lại và giải thích được; thời gian khởi động và nâng cấp được ghi lại; thử hạ cấp cho kết quả đúng như sách (không hạ cấp tại chỗ được).

**Dọn dẹp.** Xóa các datadir thử nghiệm; giữ `data_backup_80042` tới khi kết thúc bài.

---

## Dọn dẹp chung

- Instance cổng 3324 **đã được tắt** bằng `mysqladmin -h127.0.0.1 -P3324 -uroot shutdown` (exit 0) sau TN4.
- Dữ liệu thử nghiệm nằm trong datadir riêng ở thư mục scratchpad. Muốn làm lại từ đầu: tắt instance rồi xóa cả thư mục datadir và khởi tạo lại; hoặc chạy `DROP DATABASE lab;`.
- Không có thay đổi nào trên MySQL của người dùng (cổng 3306) hay trong `C:\Program Files`, `C:\ProgramData`.
