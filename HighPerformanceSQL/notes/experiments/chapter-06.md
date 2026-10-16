# Thực hành — Chương 6: Thiết kế và quản lý schema

[← Về ghi chú chương 6](../chapter-06.md) · [Đáp án tự kiểm tra](../answers/chapter-06.md)

Sáu thí nghiệm kiểm chứng các nhận định về kiểu dữ liệu, identifier, JSON, bảng tạm và thay đổi schema của chương 6. SQL, dữ liệu và kịch bản đều là **Ví dụ minh họa do AI xây dựng**. Mọi số liệu dưới đây là **Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định trừ khi ghi khác, dữ liệu tổng hợp tới 1.000.000 hàng); output được rút gọn khi ghi "đã rút gọn". Thời gian đo trên laptop Windows có nhiều nhiễu, không phải benchmark production; chỉ nên đọc theo tỷ lệ.

## Mục lục

- [Môi trường](#moi-truong)
- [E1. Kiểm chứng nhanh các hành vi phụ thuộc phiên bản](#e1)
- [E2. Kích thước bảng và index theo kiểu dữ liệu](#e2)
- [E3. Identifier: AUTO_INCREMENT, UUID chuỗi, UUID nhị phân có và không đổi chỗ](#e3)
- [E4. JSON so với cột thường](#e4)
- [E5. VARCHAR khai báo dài trong bảng tạm và sort](#e5)
- [E6. DDL native: thuật toán được phép và rollback do online alter log](#e6)
- [Phần chưa thực thi](#chua-thuc-thi)

<a id="moi-truong"></a>

## Môi trường

| Mục | Giá trị |
| --- | --- |
| Phiên bản | MySQL Community Server **8.0.42** (`C:\Program Files\MySQL\MySQL Server 8.0\bin\`) |
| Hệ điều hành | Windows 11 Home, laptop, RAM 31 GB |
| Instance | Riêng, cổng **3316**, `--bind-address=127.0.0.1`, `--no-defaults` (không đọc `my.ini`), datadir trong thư mục tạm của phiên làm việc |
| Cấu hình | Mặc định 8.0.42: `innodb_buffer_pool_size` 128 MB, page 16 KB, row format `DYNAMIC`, `utf8mb4` / `utf8mb4_0900_ai_ci`, `explicit_defaults_for_timestamp=ON`, `internal_tmp_mem_storage_engine=TempTable`, `tmp_table_size` = `max_heap_table_size` = 16 MB, `sort_buffer_size` 256 KB, `max_sort_length` 1024, `innodb_change_buffering=all`, `sql_mode` mặc định có `STRICT_TRANS_TABLES`, `NO_ZERO_DATE`, `NO_ZERO_IN_DATE` |
| Thay đổi tạm thời | E3: tăng buffer pool lên 768 MB **chỉ trong pha đếm page**, rồi trả về 128 MB. E5: đổi `internal_tmp_mem_storage_engine` theo session. E6: hạ `innodb_online_alter_log_max_size` rồi trả về 128 MB. Mỗi chỗ đều ghi rõ |
| Client | `mysql` 8.0.42; trên Windows client kết nối với `character_set_client=cp850`. Dữ liệu thí nghiệm toàn ASCII nên không ảnh hưởng; muốn tránh, thêm `--default-character-set=utf8mb4` |

Dựng và chạy instance (Git Bash; `<SP>` là thư mục làm việc riêng):

```bash
BIN="/c/Program Files/MySQL/MySQL Server 8.0/bin"
"$BIN/mysqld" --no-defaults --initialize-insecure --datadir="<SP>/hpm/mysql/ch06/main/data" --console
"$BIN/mysqld" --no-defaults --datadir="<SP>/hpm/mysql/ch06/main/data" --port=3316 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=ch06.sock --console > "<SP>/hpm/mysql/ch06/main/mysqld.log" 2>&1 &
"$BIN/mysql" -h127.0.0.1 -P3316 -uroot -e "SELECT @@port, @@datadir, @@version"
# ... chạy thí nghiệm ...
"$BIN/mysqladmin" -h127.0.0.1 -P3316 -uroot shutdown
```

Bảng số `e2.seq(n)` từ 1 tới 1.000.000 được dùng lại ở E2–E6:

```sql
CREATE DATABASE e2; USE e2;
CREATE TABLE digits (d TINYINT UNSIGNED NOT NULL PRIMARY KEY);
INSERT INTO digits VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9);
CREATE TABLE seq (n INT UNSIGNED NOT NULL PRIMARY KEY);
INSERT INTO seq
SELECT 1 + a.d + 10*b.d + 100*c.d + 1000*d.d + 10000*e.d + 100000*f.d
FROM digits a, digits b, digits c, digits d, digits e, digits f;
```

---

<a id="e1"></a>

## E1. Kiểm chứng nhanh các hành vi phụ thuộc phiên bản

**Mục tiêu.** Kiểm tra trên 8.0.42 các nhận định nhỏ của sách về kiểu dữ liệu (tr. 127–146), nhất là những chỗ sách nói hành vi "đã thay đổi qua các phiên bản".

**Giả thuyết (Dự đoán).** Các ví dụ của sách về `CHAR`, `ENUM`, `BIT` cho kết quả giống hệt; hành vi mặc định của `TIMESTAMP` và độ rộng số nguyên khác sách vì thay đổi của 8.0; `SMALLBLOB` không tồn tại.

**Chuẩn bị và các bước** (trích, đủ để chạy lại; chạy với `mysql --force --show-warnings`):

```sql
CREATE DATABASE e1; USE e1;
-- Alias
CREATE TABLE alias_test (a INTEGER, b BOOL, c NUMERIC(10,2), d REAL, e DEC(5,1));
SHOW CREATE TABLE alias_test\G
-- Độ rộng hiển thị, precision của số thực
CREATE TABLE width_test (a INT(11), b INT(1) ZEROFILL, c FLOAT(7,4), d DOUBLE(10,2), e FLOAT(30));
SHOW WARNINGS; SHOW CREATE TABLE width_test\G
INSERT INTO width_test VALUES (123456789, 7, 123.45678, 1.005, 1.5);
SELECT * FROM width_test;
-- CHAR và VARCHAR
CREATE TABLE char_test (char_col CHAR(10), varchar_col VARCHAR(10));
INSERT INTO char_test VALUES ('string1','string1'), (' string2',' string2'), ('string3 ','string3 ');
SELECT CONCAT("'", char_col, "'"), CONCAT("'", varchar_col, "'"), LENGTH(char_col), LENGTH(varchar_col) FROM char_test;
-- BINARY
CREATE TABLE bin_test (b BINARY(5), vb VARBINARY(5));
INSERT INTO bin_test VALUES ('ab','ab');
SELECT HEX(b), LENGTH(b), HEX(vb), LENGTH(vb), b = 'ab', b = 'ab\0\0\0' FROM bin_test;
-- ENUM
CREATE TABLE enum_test (e ENUM('fish','apple','dog') NOT NULL);
INSERT INTO enum_test VALUES ('fish'),('dog'),('apple');
SELECT e, e + 0 FROM enum_test ORDER BY e;
SELECT e FROM enum_test ORDER BY FIELD(e,'apple','dog','fish');
-- BIT
CREATE TABLE bittest (a BIT(8)); INSERT INTO bittest VALUES (b'00111001');
SELECT a, a + 0, HEX(a) FROM bittest;
-- TIMESTAMP mặc định, múi giờ, giới hạn 2038
CREATE TABLE ts_test (id INT, ts1 TIMESTAMP, ts2 TIMESTAMP, dt DATETIME);
SHOW CREATE TABLE ts_test\G
INSERT INTO ts_test (id) VALUES (1); SELECT * FROM ts_test;
CREATE TABLE tz_test (ts TIMESTAMP NULL, dt DATETIME NULL);
SET time_zone = '+00:00'; INSERT INTO tz_test VALUES ('2021-11-17 12:00:00','2021-11-17 12:00:00');
SET time_zone = '-05:00'; SELECT ts, dt FROM tz_test;
SET time_zone = '+07:00'; SELECT ts, dt FROM tz_test;
SET time_zone = '+00:00';
INSERT INTO tz_test VALUES ('2038-01-19 03:14:07', '2038-01-19 03:14:07');
INSERT INTO tz_test VALUES ('2038-01-19 03:14:08', '2038-01-19 03:14:08');
-- Ngày toàn số 0, SMALLBLOB, khóa ngoại lệch kiểu
CREATE TABLE zero_dt (dt DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00');
CREATE TABLE smallblob_test (a SMALLBLOB);
CREATE TABLE smalltext_test (a SMALLTEXT);
CREATE TABLE parent_u (id INT UNSIGNED PRIMARY KEY);
CREATE TABLE child_s (pid INT, FOREIGN KEY (pid) REFERENCES parent_u(id));
CREATE TABLE parent_v (code VARCHAR(20) PRIMARY KEY);
CREATE TABLE child_v (code VARCHAR(50), FOREIGN KEY (code) REFERENCES parent_v(code));
-- 61 và 62 bảng trong một join (sinh câu lệnh động)
CREATE TABLE one (x INT); INSERT INTO one VALUES (1);
SET SESSION group_concat_max_len = 100000;
SET @sql = CONCAT('SELECT COUNT(*) FROM one t1', (
  WITH RECURSIVE n(i) AS (SELECT 2 UNION ALL SELECT i+1 FROM n WHERE i < 62)
  SELECT GROUP_CONCAT(CONCAT(' JOIN one t', i, ' USING (x)') ORDER BY i SEPARATOR '') FROM n));
PREPARE s FROM @sql;          -- đổi 62 thành 61 rồi chạy lại để so sánh
-- JSON và khoảng trắng; UUID; IP; NULL trong index
SELECT JSON_STORAGE_SIZE('{"a":"x","b":1}'), JSON_STORAGE_SIZE('{   "a"  :   "x" ,     "b"  :   1    }');
SELECT UUID(), SUBSTRING(UUID(),15,1) AS version_char;
SELECT HEX(UUID_TO_BIN('6ccd780c-baba-1026-9564-5b8c656024db')),
       HEX(UUID_TO_BIN('6ccd780c-baba-1026-9564-5b8c656024db', 1));
SELECT INET_ATON('192.168.10.25'), INET_NTOA(3232238105), LENGTH(INET6_ATON('2001:db8::1'));
CREATE TABLE null_idx (id INT PRIMARY KEY, v INT NULL, KEY (v));
INSERT INTO null_idx VALUES (1,NULL),(2,NULL),(3,5),(4,6),(5,7),(6,8),(7,9),(8,10);
ANALYZE TABLE null_idx; EXPLAIN SELECT id FROM null_idx WHERE v IS NULL;
-- max_sort_length theo collation
CREATE TABLE msl (id INT PRIMARY KEY, t_nopad TEXT, t_pad TEXT COLLATE utf8mb4_general_ci,
                  v_pad VARCHAR(100) COLLATE utf8mb4_general_ci);
INSERT INTO msl VALUES (1,'aaaaZ','aaaaZ','aaaaZ'),(2,'aaaaY','aaaaY','aaaaY'),(3,'aaaaA','aaaaA','aaaaA');
SET SESSION max_sort_length = 4;
SELECT id FROM msl ORDER BY t_pad;  SELECT id FROM msl ORDER BY t_nopad;  SELECT id FROM msl ORDER BY v_pad;
SELECT COUNT(*) FROM (SELECT t_pad FROM msl GROUP BY t_pad) x;
SET SESSION max_sort_length = 1024;
```

**Kết quả đã đo** (MySQL 8.0.42; đã rút gọn):

| Kiểm tra | Kết quả | So với sách |
| --- | --- | --- |
| Alias | `INTEGER, BOOL, NUMERIC(10,2), REAL, DEC(5,1)` hiện thành `int, tinyint(1), decimal(10,2), double, decimal(5,1)` | Khớp (tr. 127) |
| `INT(11)`, `INT(1) ZEROFILL` | Cảnh báo 1681 "Integer display width is deprecated…" và "The ZEROFILL attribute is deprecated…"; `SHOW CREATE TABLE` ghi `int` (bỏ độ rộng) và `int(1) unsigned zerofill` | Sách không nói về deprecation (từ 8.0.17) |
| `FLOAT(7,4)`, `DOUBLE(10,2)`, `FLOAT(30)` | Hai cảnh báo 1681 "Specifying number of digits for floating point data types is deprecated…"; lưu 123.45678 thành `123.4568`, 1.005 thành `1.00`; `FLOAT(30)` thành `double` | Khớp ý "âm thầm đổi kiểu hoặc làm tròn" (tr. 128) |
| `CHAR(10)` và `VARCHAR(10)` | `'string3'` (dài 7) và `'string3 '` (dài 8); khoảng trắng đầu giữ ở cả hai | Khớp (tr. 130) |
| `BINARY(5)` nhận `'ab'` | `HEX` = `6162000000`, `LENGTH` = 5; `= 'ab'` → 0, `= 'ab\0\0\0'` → 1 | Khớp chú thích 2 (tr. 130) |
| ENUM | `e + 0` = 1, 2, 3 cho fish, apple, dog; `ORDER BY e` = fish, apple, dog; `FIELD()` cho thứ tự chữ cái | Khớp (tr. 132–133) |
| `BIT(8)` = `b'00111001'` | `a` = `9`, `a + 0` = 57, `HEX(a)` = 39 | Khớp (tr. 137) |
| `TIMESTAMP` khai báo trần | `timestamp NULL DEFAULT NULL` cho cả hai cột; insert không có giá trị → NULL | **Khác** tr. 135–136 (do `explicit_defaults_for_timestamp=ON` từ 8.0.2) |
| Múi giờ | Ghi 12:00 ở UTC: `TIMESTAMP` hiện 07:00 ở −05:00 và 19:00 ở +07:00; `DATETIME` luôn 12:00; `UNIX_TIMESTAMP(ts)` = 1637150400 | Khớp (tr. 135) |
| Giới hạn 2038 (session UTC) | `03:14:07` ghi được; `03:14:08` → ERROR 1292 "Incorrect datetime value"; `DATETIME` nhận `9999-12-31 23:59:59` | Khớp (tr. 135) |
| `DEFAULT '0000-00-00 00:00:00'` | ERROR 1067 "Invalid default value for 'dt'" | `sql_mode` mặc định đã chặn (tr. 146) |
| `SMALLBLOB`, `SMALLTEXT` | ERROR 1064 lỗi cú pháp cho cả hai | **Sách sai** (tr. 131) |
| Khóa ngoại `INT` → `INT UNSIGNED` | ERROR 3780 "Referencing column 'pid' and referenced column 'id' in foreign key constraint 'child_s_ibfk_1' are incompatible." | Khớp ý chú thích 5; mã lỗi khác (tr. 142) |
| Khóa ngoại `VARCHAR(50)` → `VARCHAR(20)` | Tạo được | Khớp chú thích 5 |
| 62 bảng / 61 bảng | ERROR 1116 "Too many tables; MySQL can only use 61 tables in a join" / chạy, trả 1 | Khớp (tr. 145) |
| `JSON_STORAGE_SIZE` gọn / nhiều khoảng trắng | 23 / 23 byte (văn bản có khoảng trắng dài 38 ký tự; dạng chuẩn hóa `{"a": "x", "b": 1}`) | **Khác** ý "khoảng trắng" ở tr. 140 |
| `UUID()` | Ví dụ `e01b8f35-c932-11f1-a091-00090ffe0001`, ký tự phiên bản = `1` | — |
| `UUID_TO_BIN` | không đổi chỗ `6CCD780CBABA1026…`, đổi chỗ `1026BABA6CCD780C…` | — |
| IP | `INET_ATON` = 3232238105; `INET_NTOA` trả lại `192.168.10.25`; `INET6_ATON('2001:db8::1')` dài 16 byte | Khớp (tr. 144) |
| `WHERE v IS NULL` | `type=ref`, `key=v`, `Using where; Using index` | Khớp (tr. 146) |
| `max_sort_length=4` | `ORDER BY t_pad` → 1, 2, 3 (sai thứ tự); `ORDER BY t_nopad` → 3, 2, 1 (đúng); `ORDER BY v_pad` → 1, 2, 3; `GROUP BY t_pad` → 3 nhóm | Sách chỉ nói BLOB/TEXT (tr. 131) |

**Diễn giải và đối chiếu với sách.** Phần lớn ví dụ của sách tái hiện được nguyên văn. Ba chỗ cần cập nhật cho 8.0: hành vi mặc định của `TIMESTAMP` (chú thích 4, tr. 136, đã cảnh báo rằng quy tắc thay đổi), việc deprecate độ rộng số nguyên và precision số thực (càng củng cố lời khuyên của Summary, tr. 154), và JSON không lưu khoảng trắng. Một lỗi thật của sách: `SMALLBLOB`/`SMALLTEXT` không tồn tại. `max_sort_length` trong 8.0.42 đi theo collation: áp dụng cho collation PAD SPACE (cả `VARCHAR`) và bị bỏ qua với collation NO PAD.

**Dọn dẹp.** `DROP DATABASE e1;` Các biến session (`time_zone`, `max_sort_length`, `group_concat_max_len`) mất khi đóng kết nối.

---

<a id="e2"></a>

## E2. Kích thước bảng và index theo kiểu dữ liệu

**Mục tiêu.** Đo bằng số "nhỏ hơn thường tốt hơn" (tr. 126): cùng dữ liệu, đổi kiểu thì số page của clustered index và index phụ đổi bao nhiêu. Kiểm tra cả dung lượng `DATETIME` (sách ghi 8 byte, tr. 135).

**Giả thuyết (Dự đoán).** `BIGINT` làm cả PRIMARY lẫn index phụ lớn hơn `INT`, index phụ tăng mạnh hơn vì chứa cả khóa chính. `CHAR(32)` utf8mb4 tốn như `VARCHAR(32)` vì InnoDB lưu `CHAR` nhiều byte như cột độ dài thay đổi. `DATETIME` lớn hơn `TIMESTAMP` 1 byte (không phải 4).

**Chuẩn bị.** 13 bảng, mỗi bảng `id INT UNSIGNED` (hoặc `BIGINT` ở `a_bigint`) làm khóa chính và một cột thử nghiệm `NOT NULL`, nạp 1.000.000 hàng theo thứ tự khóa chính; index phụ `k` trên cột thử nghiệm được dựng **sau khi nạp** (sorted index build) để mọi bảng có cùng điều kiện.

```sql
CREATE DATABASE IF NOT EXISTS e2; USE e2;
CREATE TABLE a_int    (id INT UNSIGNED NOT NULL PRIMARY KEY,    v INT UNSIGNED NOT NULL);
CREATE TABLE a_bigint (id BIGINT UNSIGNED NOT NULL PRIMARY KEY, v BIGINT UNSIGNED NOT NULL);
INSERT INTO a_int    SELECT n, (n * 2654435761) % 4294967296 FROM seq ORDER BY n;
INSERT INTO a_bigint SELECT n, (n * 2654435761) % 4294967296 FROM seq ORDER BY n;

CREATE TABLE b_char32_ascii   (id INT UNSIGNED NOT NULL PRIMARY KEY, h CHAR(32) CHARACTER SET ascii NOT NULL);
CREATE TABLE b_char32_utf8mb4 (id INT UNSIGNED NOT NULL PRIMARY KEY, h CHAR(32) NOT NULL);
CREATE TABLE b_varchar32      (id INT UNSIGNED NOT NULL PRIMARY KEY, h VARCHAR(32) NOT NULL);
CREATE TABLE b_binary16       (id INT UNSIGNED NOT NULL PRIMARY KEY, h BINARY(16) NOT NULL);
INSERT INTO b_char32_ascii   SELECT n, MD5(n) FROM seq ORDER BY n;   -- tương tự cho utf8mb4, varchar32
INSERT INTO b_binary16       SELECT n, UNHEX(MD5(n)) FROM seq ORDER BY n;

CREATE TABLE c_char40_utf8mb4 (id INT UNSIGNED NOT NULL PRIMARY KEY, s CHAR(40) NOT NULL);
CREATE TABLE c_char40_latin1  (id INT UNSIGNED NOT NULL PRIMARY KEY, s CHAR(40) CHARACTER SET latin1 NOT NULL);
CREATE TABLE c_varchar40      (id INT UNSIGNED NOT NULL PRIMARY KEY, s VARCHAR(40) NOT NULL);
INSERT INTO c_varchar40 SELECT n, LEFT(CONCAT(MD5(n), MD5(n+1)), 1 + (n * 7919) % 40) FROM seq ORDER BY n;
-- (cùng biểu thức cho hai bảng CHAR(40); độ dài trung bình 20,5 ký tự)

CREATE TABLE d_datetime  (id INT UNSIGNED NOT NULL PRIMARY KEY, t DATETIME NOT NULL);
CREATE TABLE d_timestamp (id INT UNSIGNED NOT NULL PRIMARY KEY, t TIMESTAMP NOT NULL);
CREATE TABLE d_datetime6 (id INT UNSIGNED NOT NULL PRIMARY KEY, t DATETIME(6) NOT NULL);
CREATE TABLE d_int_epoch (id INT UNSIGNED NOT NULL PRIMARY KEY, t INT UNSIGNED NOT NULL);
SET time_zone = '+00:00';
INSERT INTO d_datetime  SELECT n, TIMESTAMP('2020-01-01') + INTERVAL ((n * 37) % 63072000) SECOND FROM seq ORDER BY n;
-- tương tự cho d_timestamp; d_datetime6 cộng thêm (n % 1000000) MICROSECOND;
-- d_int_epoch dùng UNIX_TIMESTAMP('2020-01-01 00:00:00') + ((n * 37) % 63072000)

ALTER TABLE a_int ADD INDEX k (v);   -- lặp cho mọi bảng, trên cột thử nghiệm
ANALYZE TABLE a_int, a_bigint;       -- lặp cho mọi bảng
```

**Các bước và lệnh đo.**

```sql
SELECT table_name, index_name, stat_name, stat_value FROM mysql.innodb_index_stats
WHERE database_name = 'e2' AND stat_name IN ('n_leaf_pages','size');
SELECT table_name, data_length, index_length FROM information_schema.tables WHERE table_schema = 'e2';
```

**Kết quả đã đo** (1.000.000 hàng mỗi bảng; "B/entry" = 16.384 / số entry mỗi leaf page, tức đã gồm overhead của page; **Bổ sung ngoài sách**, phép tính):

| Bảng | Leaf PRIMARY | Hàng / leaf | B/hàng | Leaf index `k` | Entry / leaf | B/entry |
| --- | --- | --- | --- | --- | --- | --- |
| `a_int` (INT, INT) | 1.743 | 573,7 | 28,6 | 832 | 1.201,9 | 13,6 |
| `a_bigint` (BIGINT, BIGINT) | 2.269 | 440,7 | 37,2 | 1.325 | 754,7 | 21,7 |
| `b_char32_ascii` | 3.585 | 278,9 | 58,7 | 2.558 | 390,9 | 41,9 |
| `b_char32_utf8mb4` | 3.651 | 273,9 | 59,8 | 2.618 | 382,0 | 42,9 |
| `b_varchar32` (utf8mb4) | 3.651 | 273,9 | 59,8 | 2.618 | 382,0 | 42,9 |
| `b_binary16` | 2.533 | 394,8 | 41,5 | 1.570 | 636,9 | 25,7 |
| `c_char40_latin1` | 4.116 | 243,0 | 67,4 | 3.049 | 328,0 | 50,0 |
| `c_char40_utf8mb4` | 4.185 | 238,9 | 68,6 | 3.116 | 320,9 | 51,1 |
| `c_varchar40` (utf8mb4) | 2.894 | 345,5 | 47,4 | 1.910 | 523,6 | 31,3 |
| `d_timestamp` | 1.743 | 573,7 | 28,6 | 832 | 1.201,9 | 13,6 |
| `d_int_epoch` | 1.743 | 573,7 | 28,6 | 832 | 1.201,9 | 13,6 |
| `d_datetime` | 1.809 | 552,8 | 29,6 | 893 | 1.119,8 | 14,6 |
| `d_datetime6` | 2.005 | 498,8 | 32,8 | 1.078 | 927,6 | 17,7 |

`information_schema.tables` (MB, đã rút gọn): `a_int` 27,5 + 15,5; `a_bigint` 35,6 + 24,6; `b_binary16` 40,6 + 28,6; `b_char32_utf8mb4` 57,6 + 47,6; `c_varchar40` 45,6 + 34,6; `c_char40_utf8mb4` 65,6 + 56,7; `d_datetime` 28,6 + 16,5; `d_timestamp` 27,5 + 15,5.

**Diễn giải và đối chiếu với sách.**

- `BIGINT` so với `INT`: PRIMARY +30%, index phụ +59%; mỗi entry index phụ tăng đúng khoảng 8 byte (13,6 → 21,7) vì cả cột được index lẫn khóa chính đều dài thêm 4 byte. Khớp "nhỏ hơn thường tốt hơn" (tr. 126).
- `CHAR(32)` utf8mb4 và `VARCHAR(32)` utf8mb4 giống hệt nhau đến từng page; bản `ascii` nhỏ hơn đúng khoảng 1 byte mỗi hàng (không còn byte độ dài). Ưu điểm "CHAR không có byte độ dài" của sách (tr. 129) chỉ còn với bộ ký tự một byte.
- `BINARY(16)` cho cùng giá trị băm: PRIMARY nhỏ hơn 29%, index phụ nhỏ hơn 39% so với `CHAR(32)` ascii. Khớp lời khuyên lưu UUID/băm dạng nhị phân (tr. 144).
- Chuỗi dài thay đổi (trung bình 20,5 ký tự): `VARCHAR(40)` nhỏ hơn `CHAR(40)` khoảng 30% ở PRIMARY và 38% ở index phụ. Khớp điều kiện "độ dài tối đa lớn hơn nhiều so với trung bình" (tr. 129).
- Thời gian: `TIMESTAMP` = `INT UNSIGNED` (4 byte); `DATETIME` lớn hơn đúng 1 byte mỗi entry (5 byte); `DATETIME(6)` thêm 3 byte nữa. **Sách ghi `DATETIME` 8 byte (tr. 135–136) là định dạng trước 5.6.4.**

**Dọn dẹp.** Giữ `e2.seq` cho các thí nghiệm sau; cuối cùng `DROP DATABASE e2;`

---

<a id="e3"></a>

## E3. Identifier: AUTO_INCREMENT, UUID chuỗi, UUID nhị phân có và không đổi chỗ

**Mục tiêu.** Đo các hệ quả sách nêu cho identifier ngẫu nhiên (tr. 143–144): page split, truy cập đĩa ngẫu nhiên, phân mảnh clustered index, cache kém; so sánh với số nguyên tự tăng và với `UUID_TO_BIN` có và không đổi chỗ (**Bổ sung ngoài sách**).

**Giả thuyết (Dự đoán).** Khi các hàng đến rải rác theo thời gian, khóa `CHAR(36)` và `BINARY(16)` không đổi chỗ sẽ để page đầy khoảng 50–70%, gây nhiều lần đọc đĩa khi index vượt buffer pool; `UUID_TO_BIN(u, 1)` hành xử gần như `AUTO_INCREMENT`, chỉ lớn hơn vì khóa 16 byte. Ngoài ra, `UUID()` sinh liên tiếp trong một câu lệnh sẽ tăng dần.

**Chuẩn bị.**

1. *Kiểm tra burst:* 200.000 giá trị `UUID()` thật sinh trong một câu lệnh, đếm số lần giá trị sau nhỏ hơn giá trị trước.
2. *Mô phỏng hàng đến theo thời gian:* lấy mốc thời gian t0 từ một `UUID()` thật, rồi dựng UUID v1 cho hàng thứ n với thời điểm t0 + n × 2,592 giây (1 triệu hàng trải trên 30 ngày), giữ nguyên `clock_seq` và `node`. Cách dựng theo đúng bố cục `time_low-time_mid-time_hi_and_version-clock_seq-node`; kiểm tra bằng `IS_UUID()` và `BIN_TO_UUID(UUID_TO_BIN(u,1),1) = u`.

```sql
CREATE DATABASE e3; USE e3;
SET GLOBAL innodb_monitor_enable = 'module_index';
CREATE TABLE burst (seqno INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY, u CHAR(36) CHARACTER SET ascii NOT NULL);
INSERT INTO burst (u) SELECT UUID() FROM e2.seq WHERE n <= 200000 ORDER BY n;
SELECT COUNT(*), SUM(u < prev) FROM (SELECT u, LAG(u) OVER (ORDER BY seqno) AS prev FROM burst) x;

SET @u0 = UUID();
SET @t0 = (CONV(SUBSTRING(@u0,16,3),16,10) << 48) | (CONV(SUBSTRING(@u0,10,4),16,10) << 32) | CONV(SUBSTRING(@u0,1,8),16,10);
SET @tail = SUBSTRING(@u0, 20);
SET @step = 25920000;   -- 2,592 giây tính bằng đơn vị 100 ns
CREATE TABLE src (n INT UNSIGNED NOT NULL PRIMARY KEY, u CHAR(36) CHARACTER SET ascii NOT NULL,
  user_id INT UNSIGNED NOT NULL, created_at DATETIME NOT NULL, amount DECIMAL(10,2) NOT NULL, note VARCHAR(40) NOT NULL);
INSERT INTO src
SELECT n,
  CONCAT(LPAD(HEX((@t0 + n*@step) & 0xFFFFFFFF), 8, '0'), '-',
         LPAD(HEX(((@t0 + n*@step) >> 32) & 0xFFFF), 4, '0'), '-1',
         LPAD(HEX(((@t0 + n*@step) >> 48) & 0xFFF), 3, '0'), '-', @tail),
  1 + (n * 2654435761) % 100000,
  TIMESTAMP('2026-01-01') + INTERVAL (n * 2592 DIV 1000) SECOND,
  ((n * 7919) % 100000) / 100,
  LEFT(MD5(n), 10 + n % 21)
FROM e2.seq ORDER BY n;
```

3. *Bốn bảng đích,* cùng 4 cột dữ liệu và index phụ `k_user(user_id)`, khác nhau ở khóa chính:

```sql
-- cols = user_id INT UNSIGNED NOT NULL, created_at DATETIME NOT NULL, amount DECIMAL(10,2) NOT NULL,
--        note VARCHAR(40) NOT NULL, KEY k_user (user_id)
CREATE TABLE id_ai         (id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY, /* cols */ ...);
CREATE TABLE id_char36     (id CHAR(36) NOT NULL PRIMARY KEY, /* cols */ ...);
CREATE TABLE id_bin16      (id BINARY(16) NOT NULL PRIMARY KEY, /* cols */ ...);
CREATE TABLE id_bin16_swap (id BINARY(16) NOT NULL PRIMARY KEY, /* cols */ ...);
INSERT INTO id_ai (user_id, created_at, amount, note) SELECT user_id, created_at, amount, note FROM src ORDER BY n;
INSERT INTO id_char36     SELECT LOWER(u),          user_id, created_at, amount, note FROM src ORDER BY n;
INSERT INTO id_bin16      SELECT UUID_TO_BIN(u),    user_id, created_at, amount, note FROM src ORDER BY n;
INSERT INTO id_bin16_swap SELECT UUID_TO_BIN(u, 1), user_id, created_at, amount, note FROM src ORDER BY n;
```

**Các bước và lệnh đo.** Với mỗi bảng: `DROP` rồi `CREATE`, `SET GLOBAL innodb_monitor_reset = 'module_index'`, ghi `Innodb_buffer_pool_reads` trước và sau, đo thời gian bằng `NOW(6)`, đọc `COUNT_RESET` của `index_page_splits`, `index_page_merge_attempts`, `index_page_reorg_attempts` trong `information_schema.INNODB_METRICS`. Lặp 4 lần (vòng 1–3 đọc nhầm cột `COUNT` tích lũy, nên số split của vòng 1–3 được tính bằng hiệu giữa các lần đo; vòng 4 đọc đúng `COUNT_RESET`). Sau vòng 4: tạm tăng buffer pool lên 768 MB, đọc toàn bộ từng index (`SELECT COUNT(*) … FORCE INDEX`), rồi đếm page trong buffer pool:

```sql
SELECT INDEX_NAME, COUNT(*) AS pages, SUM(NUMBER_RECORDS) AS records,
       ROUND(100*AVG(DATA_SIZE)/16384,1) AS avg_fill_pct
FROM information_schema.INNODB_BUFFER_PAGE
WHERE TABLE_NAME = '`e3`.`id_char36`' AND PAGE_TYPE = 'INDEX' GROUP BY INDEX_NAME;
```

Sau đó `ALTER TABLE … FORCE` từng bảng, đếm lại, rồi trả buffer pool về 128 MB.

**Kết quả đã đo.**

*Burst:* 200.000 giá trị `UUID()` liên tiếp, **0 lần giảm** (từ `a344f7b5-c933-…` tới `a4ebf156-c933-…`, cùng `time_mid`). *Mô phỏng 30 ngày:* thứ tự chuỗi theo thứ tự đến giảm **6.035 lần** (mỗi vòng `time_low`); thứ tự sau `UUID_TO_BIN(u, 1)` giảm 0 lần.

*Insert* (buffer pool 128 MB; thời gian các vòng 1, 2, 3, 4):

| Bảng | Thời gian (s) | Trung vị | `Innodb_buffer_pool_reads` | Page split (vòng 4) | Reorg (vòng 4) |
| --- | --- | --- | --- | --- | --- |
| `id_ai` | 10,21 · 10,91 · 17,22 · 16,50 | 13,7 | 522 · 122 · 179 · 110 | 6.102 | 0 |
| `id_char36` | 58,85 · 58,26 · 73,66 · 69,20 | 64,0 | 111.644 · 111.671 · 113.507 · 114.250 | 12.367 | 8.215 |
| `id_bin16` | 23,73 · 32,36 · 33,79 · 29,57 | 31,0 | 17.033 · 19.733 · 18.565 · 17.748 | 8.210 | 6.143 |
| `id_bin16_swap` | 11,85 · 18,02 · 17,23 · 17,16 | 17,2 | 540 · 127 · 124 · 213 | 6.637 | 0 |

Số split vòng 2 và 3 (tính bằng hiệu): 6.103 · 12.435 (12.401) · 8.211 · 6.638 (6.640), khớp vòng 4.

*Số page thật* (đếm trong `INNODB_BUFFER_PAGE`, gồm cả vài page không phải leaf):

| Bảng | PRIMARY sau khi nạp | Độ đầy | `k_user` sau khi nạp | Độ đầy | PRIMARY sau rebuild | `k_user` sau rebuild |
| --- | --- | --- | --- | --- | --- | --- |
| `id_ai` | 4.055 | 91,9% | 2.050 | 50,6% | 4.054 | 1.082 |
| `id_char36` | 8.229 | 67,0% | 4.098 | 68,5% | 5.975 | 2.877 |
| `id_bin16` | 6.162 | 68,5% | 2.050 | 74,5% | 4.583 | 1.575 |
| `id_bin16_swap` | 4.589 | 91,9% | 2.054 | 74,5% | 4.588 | 1.575 |

`n_leaf_pages` trong `mysql.innodb_index_stats` sau `ANALYZE TABLE` cho số khớp (4.049; 8.192; 6.144; 4.580 cho PRIMARY; 2.048; 4.096; 2.048; 2.048 cho `k_user`). *Đọc 1000 hàng mới nhất:* `SELECT … FROM id_ai ORDER BY id DESC LIMIT 1000` có `Handler_read_prev` = 999; `EXPLAIN FORMAT=TREE` cho "Index scan on id_bin16_swap using PRIMARY (reverse)".

**Diễn giải và đối chiếu với sách.**

- **Page split và phân mảnh clustered index** (tr. 143): khóa rải rác để PRIMARY đầy 67–69%, cần nhiều hơn 34–38% page so với cùng dữ liệu sau rebuild. Khóa tăng dần (`AUTO_INCREMENT`, `UUID_TO_BIN(u, 1)`) đầy 92% và rebuild không thu nhỏ được gì.
- **Truy cập đĩa ngẫu nhiên, cache kém** (tr. 143–144): PRIMARY của `id_char36` (8.229 page, khoảng 129 MiB) vượt buffer pool 128 MB, nên insert gây hơn 111.000 lần đọc page; `id_bin16` nhỏ hơn (6.162 page, khoảng 96 MiB) nên đọc khoảng 17.000–20.000 lần; khóa tăng dần chỉ vài trăm.
- **Khóa chính lớn làm index phụ lớn** (tr. 134, 142): sau rebuild `k_user` là 1.082 / 1.575 / 2.877 page cho khóa 8 / 16 / 36 byte.
- **Chỉ số `index_page_splits`** tăng cả khi chỉ nối page mới vào cuối index: tổng split xấp xỉ tổng số page của hai index trong mọi trường hợp. Dấu hiệu phân biệt là độ đầy page và số lần reorg (0 khi chèn tăng dần).
- `k_user` sau khi nạp có cùng 2.050 page ở ba bảng dù kích thước entry khác nhau (độ đầy 50,6% so với 74,5%). Index này nhận insert theo thứ tự `user_id` ngẫu nhiên như nhau ở mọi bảng và đi qua change buffer; người viết ghi chú **chưa giải thích được** vì sao số page trùng nhau. Kết luận về index phụ vì vậy dựa trên số sau rebuild.
- **Giới hạn:** một luồng ghi; không kiểm chứng được chú thích 6 về điểm nóng khi nhiều luồng ghi (tr. 143). Thời gian dao động tới khoảng 70% giữa các vòng.

**Dọn dẹp.** `SET GLOBAL innodb_buffer_pool_size = 134217728;` (đã làm trong kịch bản), `SET GLOBAL innodb_monitor_disable = 'module_index';`, `DROP DATABASE e3;`

---

<a id="e4"></a>

## E4. JSON so với cột thường

**Mục tiêu.** Tái hiện thí nghiệm JSON của sách (tr. 139–142) trên dữ liệu tổng hợp cùng cấu trúc (không tải dữ liệu NASA), ở 202 hàng như sách và ở 100.000 hàng; kiểm tra cột sinh và toán tử `->`/`->>`.

**Giả thuyết (Dự đoán).** Bảng JSON lớn hơn khoảng 1,5–3 lần, đọc một trường chậm hơn khoảng 2 lần; cột sinh có index đưa tra cứu một hàng về gần bằng cột thường; cột sinh định nghĩa bằng `->` giữ dấu nháy.

**Chuẩn bị.**

```sql
CREATE DATABASE e4; USE e4;
CREATE TABLE gen (n INT UNSIGNED NOT NULL PRIMARY KEY, doc JSON NOT NULL);
INSERT INTO gen
SELECT n, JSON_OBJECT(
  'designation',    CONCAT(LPAD(n, 6, '0'), ' (', 1990 + n % 32, ' ',
                           CHAR(65 + n % 26 USING utf8mb4), CHAR(65 + (n DIV 26) % 26 USING utf8mb4), n % 100, ')'),
  'discovery_date', CONCAT(DATE_FORMAT(DATE('1990-01-01') + INTERVAL (n * 37) % 11000 DAY, '%Y-%m-%d'), 'T00:00:00.000'),
  'h_mag',          FORMAT(14 + (n * 7) % 120 / 10, 1),
  'moid_au',        FORMAT(((n * 13) % 500) / 1000, 3),
  'q_au_1',         FORMAT(0.1 + ((n * 17) % 150) / 100, 2),
  'q_au_2',         FORMAT(1 + ((n * 19) % 900) / 100, 2),
  'period_yr',      FORMAT(0.5 + ((n * 23) % 2000) / 100, 2),
  'i_deg',          FORMAT(((n * 29) % 7000) / 100, 2),
  'pha',            IF(n % 7 = 0, 'Y', 'N'),
  'orbit_class',    ELT(1 + n % 5, 'Apollo', 'Aten', 'Amor', 'Jupiter-family Comet', 'Halley-type Comet'))
FROM e2.seq WHERE n <= 100000 ORDER BY n;
-- Giống sách: không khai báo khóa chính; mọi giá trị là chuỗi như dữ liệu mẫu của sách
CREATE TABLE asteroids_json (json_data JSON);
CREATE TABLE asteroids_sql (designation VARCHAR(30), discovery_date DATE, h_mag FLOAT, moid_au FLOAT,
  q_au_1 FLOAT, q_au_2 FLOAT, period_yr FLOAT, i_deg FLOAT, pha CHAR(3), orbit_class VARCHAR(30));
INSERT INTO asteroids_json SELECT doc FROM gen ORDER BY n;
INSERT INTO asteroids_sql
SELECT doc->>'$.designation', LEFT(doc->>'$.discovery_date', 10), doc->>'$.h_mag', doc->>'$.moid_au',
       doc->>'$.q_au_1', doc->>'$.q_au_2', doc->>'$.period_yr', doc->>'$.i_deg', doc->>'$.pha', doc->>'$.orbit_class'
FROM gen ORDER BY n;
-- Bản 202 hàng: asteroids_json_202, asteroids_sql_202 với WHERE n <= 202
ANALYZE TABLE asteroids_json, asteroids_sql;
```

Lưu ý khi dựng dữ liệu: `CHAR(65)` không có `USING` trả về **chuỗi nhị phân**, và `JSON_OBJECT()` khi đó lưu trường thành giá trị nhị phân mờ (`base64:type15:…`); lần chạy đầu gặp lỗi này nên đã thêm `USING utf8mb4`.

**Các bước và lệnh đo.**

1. Kích thước: `information_schema.tables`, `mysql.innodb_index_stats`, `AVG(JSON_STORAGE_SIZE(json_data))`.
2. Đọc một trường trên mọi hàng (đã nằm trong buffer pool), 7 lần mỗi loại, lấy `TIMER_WAIT` từ `performance_schema.events_statements_history` của chính session: `SELECT SUM(LENGTH(designation)) FROM asteroids_sql` và `SELECT SUM(LENGTH(json_data->>'$.designation')) FROM asteroids_json` (dùng hàm tổng hợp để server tính biểu thức trên mọi hàng mà không gửi 100.000 hàng về client). Chạy thêm biến thể `MAX()`.
3. Tra một hàng chưa có index: `FLUSH STATUS`, truy vấn, `SHOW SESSION STATUS` cho `Handler_read_%`.
4. Thêm index: `ALTER TABLE asteroids_sql ADD INDEX (designation);` và cột sinh như sách: `ALTER TABLE asteroids_json ADD COLUMN designation VARCHAR(30) GENERATED ALWAYS AS (json_data->"$.designation"), ADD INDEX (designation);` rồi thêm biến thể `designation_u … AS (json_data->>'$.designation') VIRTUAL` có index. `EXPLAIN` và đo thời gian tra một hàng 7 lần.

**Kết quả đã đo.**

| Kích thước | `asteroids_sql` | `asteroids_json` | Sách (202 hàng) |
| --- | --- | --- | --- |
| 202 hàng: `Data_length` | 49.152 (3 page; 2 leaf) | 98.304 (6 page; 5 leaf) | 49.152 / 81.920 |
| 202 hàng: `Avg_row_length` | 243 | 486 | 243 / 405 |
| 100.000 hàng: leaf page | 579 | 1.786 | — |
| 100.000 hàng: `Data_length` | 9.977.856 | 29.949.952 | — |
| `JSON_STORAGE_SIZE` trung bình | — | 240,5 byte (văn bản 235,5 ký tự) | — |

| Truy vấn (100.000 hàng) | Cột thường | JSON |
| --- | --- | --- |
| `SUM(LENGTH(...))` trên mọi hàng, trung vị 7 lần | 83,2 ms | 146,0 ms |
| `MAX(...)` trên mọi hàng, trung vị 7 lần | 118,8 ms | 152,6 ms |
| Tra một hàng, chưa có index | `Handler_read_rnd_next` = 100.001 | `Handler_read_rnd_next` = 100.001; khoảng 141 ms (trung vị 5 lần) |
| Tra một hàng qua index, trung vị 7 lần | 0,24 ms | 0,32 ms (cột sinh `designation_u`) |

Cột sinh theo cách của sách (`->`): `SELECT designation … LIMIT 2` trả `"000001 (1991 BA1)"` **kèm dấu nháy**; `WHERE designation = '"050000 (2006 CZ0)"'` dùng `type=ref` (`Handler_read_key` = 1, `Handler_read_next` = 1); `WHERE designation = '050000 (2006 CZ0)'` trả **0 hàng**. Với cột `designation_u` (`->>`), cả `WHERE designation_u = '…'` lẫn `WHERE json_data->>'$.designation' = '…'` đều được `EXPLAIN` cho `type=ref` trên `designation_u`; Note 1003 cho thấy điều kiện thứ hai đã được viết lại thành `designation_u = '050000 (2006 CZ0)'`. Sau khi thêm hai cột sinh ảo có index: `Data_length` của bảng JSON vẫn 29.949.952; `Index_length` 8.421.376.

`SHOW CREATE TABLE` ghi biểu thức là `json_extract(json_data, _cp850'$.designation')`: tiền tố `_cp850` đến từ bộ ký tự của client Windows (xem [Môi trường](#moi-truong)), không ảnh hưởng kết quả.

**Diễn giải và đối chiếu với sách.**

- Ở 202 hàng, tỷ lệ kích thước (6 so với 3 page) rất gần sách (5 so với 3). Ở 100.000 hàng, JSON lớn gấp khoảng 3,1 lần: phần tốn thêm là tên khóa lặp lại (77 byte tên trường mỗi hàng), metadata nhị phân, và số lưu dạng chuỗi; **không** phải khoảng trắng (E1). Khớp kết luận "dung lượng là lý do chính" (tr. 142), nhưng sửa giải thích ở tr. 140.
- Đọc một trường trên mọi hàng chậm hơn khoảng 1,75 lần (đo bằng `SUM(LENGTH())`), cùng xu hướng "gần gấp đôi" của sách (tr. 141). Với `MAX()`, phép so sánh chuỗi theo collation chiếm phần lớn thời gian nên tỷ lệ chỉ 1,3 lần: cách đo ảnh hưởng tỷ lệ.
- Tra một hàng: cột sinh có index gần bằng cột thường (0,32 so với 0,24 ms; sách 0,4 so với 0,33 ms). Không có index thì phải quét toàn bảng, chậm hơn khoảng 590 lần ở 100.000 hàng; ở 202 hàng của sách, chênh lệch chỉ là 0,58 so với 0,33 ms vì bảng quá nhỏ.

**Dọn dẹp.** `DROP DATABASE e4;`

---

<a id="e5"></a>

## E5. VARCHAR khai báo dài trong bảng tạm và sort

**Mục tiêu.** Kiểm chứng hộp "Generosity Can Be Unwise" (tr. 131) và Summary (tr. 154) trên 8.0.42: độ dài khai báo của `VARCHAR` có làm bảng tạm và sort tốn bộ nhớ hơn không, khi dữ liệu thật như nhau?

**Giả thuyết (Dự đoán).** Engine bảng tạm MEMORY (hàng độ dài cố định) bị ảnh hưởng rõ; TempTable (mặc định 8.0) ít hoặc không bị ảnh hưởng. Sort phụ thuộc collation: PAD SPACE dùng sort key cố định nên bị ảnh hưởng, NO PAD thì không.

**Chuẩn bị.**

```sql
CREATE DATABASE e5; USE e5;
CREATE TABLE v10       (id INT UNSIGNED NOT NULL PRIMARY KEY, s VARCHAR(10)   NOT NULL);
CREATE TABLE v1000     (id INT UNSIGNED NOT NULL PRIMARY KEY, s VARCHAR(1000) NOT NULL);
CREATE TABLE v10_pad   (id INT UNSIGNED NOT NULL PRIMARY KEY, s VARCHAR(10)   COLLATE utf8mb4_general_ci NOT NULL);
CREATE TABLE v1000_pad (id INT UNSIGNED NOT NULL PRIMARY KEY, s VARCHAR(1000) COLLATE utf8mb4_general_ci NOT NULL);
INSERT INTO v10 SELECT n, LEFT(MD5(n), 10) FROM e2.seq WHERE n <= 200000 ORDER BY n;
INSERT INTO v1000 SELECT * FROM v10;  INSERT INTO v10_pad SELECT * FROM v10;  INSERT INTO v1000_pad SELECT * FROM v10;
ANALYZE TABLE v10, v1000, v10_pad, v1000_pad;
```

200.000 giá trị khác nhau, mỗi giá trị đúng 10 ký tự; cả bốn bảng có `Data_length` 7.880.704 byte. `utf8mb4_0900_ai_ci` là NO PAD, `utf8mb4_general_ci` là PAD SPACE (`information_schema.COLLATIONS`).

**Các bước và lệnh đo.**

*Bảng tạm* (20.000 nhóm; mỗi tổ hợp engine × bảng chạy trong session mới):

```sql
SET SESSION internal_tmp_mem_storage_engine = 'MEMORY';   -- hoặc 'TempTable'
TRUNCATE TABLE performance_schema.memory_summary_global_by_event_name;
FLUSH STATUS;
SELECT s, COUNT(*) FROM v1000 WHERE id <= 20000 GROUP BY s HAVING COUNT(*) > 1;
SELECT VARIABLE_NAME, VARIABLE_VALUE FROM performance_schema.session_status
WHERE VARIABLE_NAME IN ('Created_tmp_tables','Created_tmp_disk_tables');
SELECT EVENT_NAME, HIGH_NUMBER_OF_BYTES_USED FROM performance_schema.memory_summary_global_by_event_name
WHERE EVENT_NAME IN ('memory/temptable/physical_ram','memory/temptable/physical_disk') OR EVENT_NAME LIKE 'memory/memory/%';
```

`EXPLAIN FORMAT=TREE` của truy vấn: "Aggregate using temporary table" trên `Index range scan on v1000 using PRIMARY over (id <= 20000)`. Lần thử đầu với 200.000 nhóm làm mọi trường hợp đều tràn trần 16 MB nên không phân biệt được; đã giảm còn 20.000 nhóm.

*Sort* (200.000 hàng; `LIMIT 1 OFFSET 199999` buộc sắp toàn bộ mà không gửi nhiều hàng về client; trace xác nhận không dùng priority queue):

```sql
SET optimizer_trace = 'enabled=on'; SET optimizer_trace_max_mem_size = 10000000;
FLUSH STATUS;
SELECT id, s FROM v1000_pad ORDER BY s LIMIT 1 OFFSET 199999;
SELECT JSON_EXTRACT(TRACE, '$**.filesort_summary') FROM information_schema.OPTIMIZER_TRACE;
SHOW SESSION STATUS WHERE Variable_name IN ('Sort_merge_passes','Sort_rows');
```

Thời gian: mỗi truy vấn 5 lần, `TIMER_WAIT` từ `events_statements_history`.

**Kết quả đã đo.**

| `GROUP BY` 20.000 nhóm | `Created_tmp_disk_tables` | Bộ nhớ đỉnh | Thời gian (trung vị 5) |
| --- | --- | --- | --- |
| TempTable, `v10` | 0 | 8.388.736 byte (`temptable/physical_ram`) | 29,1 ms |
| TempTable, `v1000` | 0 | 8.388.736 byte | 30,1 ms |
| MEMORY, `v10` | 0 | 1.922.340 byte (`memory/memory/%`) | 23,4 ms |
| MEMORY, `v1000` | **1** | 16.892.080 byte (chạm trần) | **131,4 ms** |

Kết quả với `v10_pad` và `v1000_pad` giống hệt cặp NO PAD tương ứng (bảng tạm không phụ thuộc collation ở đây).

| `ORDER BY s`, 200.000 hàng | `sort_mode` (trace) | `max_rows_per_buffer` | Chunk tràn đĩa | `Sort_merge_passes` | Thời gian (trung vị 5) |
| --- | --- | --- | --- | --- | --- |
| `v10` (NO PAD) | `<varlen_sort_key, packed_additional_fields>` | 1.208 | 43 | 7 | 228,8 ms |
| `v1000` (NO PAD) | `<varlen_sort_key, packed_additional_fields>` | 13 | 43 | 7 | 228,8 ms |
| `v10_pad` (PAD SPACE) | `<fixed_sort_key, packed_additional_fields>` | 3.799 | 40 | 7 | 202,4 ms |
| `v1000_pad` (PAD SPACE) | `<fixed_sort_key, packed_additional_fields>` | 52 | **807** | **134** | **1.080,6 ms** |

`peak_memory_used` của filesort khoảng 263–284 KB trong mọi trường hợp (bằng `sort_buffer_size`).

**Diễn giải và đối chiếu với sách.**

- Lời khuyên của sách **đúng có điều kiện** trên 8.0.42. Với mặc định (TempTable, `utf8mb4_0900_ai_ci`), `VARCHAR(1000)` không tốn hơn `VARCHAR(10)` khi dữ liệu thật như nhau.
- Với engine MEMORY (hàng độ dài cố định), `VARCHAR(1000)` utf8mb4 làm bảng tạm vượt trần 16 MB, chuyển sang đĩa và chậm khoảng 5,6 lần: đúng "cấp phát khối kích thước cố định" của tr. 131.
- Với collation PAD SPACE, sort key cố định theo độ dài khai báo: mỗi buffer chứa ít hàng hơn khoảng 73 lần, số chunk tăng khoảng 20 lần, merge pass từ 7 lên 134, thời gian khoảng 5,3 lần: đúng "filesort" của tr. 131 và "cấp phát bi quan theo độ dài tối đa khi sắp xếp" của tr. 154.
- Với NO PAD, `max_rows_per_buffer` trong trace vẫn giảm (1.208 xuống 13) vì là ước tính theo độ dài tối đa, nhưng số chunk thực tế không đổi vì key được đóng gói theo độ dài thật.

**Dọn dẹp.** Các biến session tự mất; `DROP DATABASE e5;`

---

<a id="e6"></a>

## E6. DDL native: thuật toán được phép và rollback do online alter log

**Mục tiêu.** Đối chiếu danh sách "vẫn chưa làm native được với INPLACE" của sách (đổi khóa chính, đổi charset, mã hóa theo bảng, thêm hoặc bỏ khóa ngoại; tr. 149–150) với 8.0.42, và tái hiện cảnh báo "rollback khi log thay đổi quá lớn" (tr. 150).

**Giả thuyết (Dự đoán).** Đổi charset và mã hóa bị từ chối với `INPLACE`; đổi khóa chính (bỏ và thêm trong một câu) và khóa ngoại (khi tắt `foreign_key_checks`) chạy được với `INPLACE` theo tài liệu 8.0; thêm cột và thêm phần tử ENUM ở cuối chạy được với `INSTANT`. Hạ `innodb_online_alter_log_max_size` sẽ làm ALTER online thất bại khi có DML đồng thời.

**Chuẩn bị.**

```sql
CREATE DATABASE e6; USE e6;
CREATE TABLE parent (id INT NOT NULL PRIMARY KEY);
INSERT INTO parent SELECT n FROM e2.seq WHERE n <= 1000;
CREATE TABLE t (id INT NOT NULL PRIMARY KEY, a INT NOT NULL, p INT NOT NULL,
                s VARCHAR(20) NOT NULL, e ENUM('x','y') NOT NULL, c CHAR(10) NOT NULL);
INSERT INTO t SELECT n, n % 1000, 1 + n % 1000, LEFT(MD5(n), 12), IF(n % 2, 'x', 'y'), LEFT(MD5(n+1), 10)
FROM e2.seq WHERE n <= 200000;
```

**Các bước.** Chạy lần lượt (với `mysql --force`), mỗi câu chỉ định `ALGORITHM` và `LOCK`:

```sql
ALTER TABLE t ADD COLUMN x1 INT NULL, ALGORITHM=INSTANT;
ALTER TABLE t ADD COLUMN x2 INT NULL AFTER a, ALGORITHM=INSTANT;
ALTER TABLE t DROP COLUMN x2, ALGORITHM=INSTANT;
SELECT NAME, INSTANT_COLS, TOTAL_ROW_VERSIONS FROM information_schema.INNODB_TABLES WHERE NAME = 'e6/t';
ALTER TABLE t MODIFY s VARCHAR(60) NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t MODIFY s VARCHAR(100) NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t MODIFY e ENUM('x','y','z') NOT NULL, ALGORITHM=INSTANT;
ALTER TABLE t MODIFY e ENUM('w','x','y','z') NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t MODIFY a BIGINT NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t CONVERT TO CHARACTER SET latin1, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t DEFAULT CHARACTER SET latin1, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t DROP PRIMARY KEY, ADD PRIMARY KEY (a, id), ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t DROP PRIMARY KEY, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t ADD CONSTRAINT fk_p FOREIGN KEY (p) REFERENCES parent(id), ALGORITHM=INPLACE, LOCK=NONE;
SET SESSION foreign_key_checks = 0;
ALTER TABLE t ADD CONSTRAINT fk_p FOREIGN KEY (p) REFERENCES parent(id), ALGORITHM=INPLACE, LOCK=NONE;
SET SESSION foreign_key_checks = 1;
ALTER TABLE t DROP FOREIGN KEY fk_p, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t ENCRYPTION='Y', ALGORITHM=INPLACE;
ALTER TABLE t ADD INDEX k_s (s), ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t FORCE, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE t MODIFY a BIGINT NOT NULL, ALGORITHM=COPY, LOCK=NONE;
ALTER TABLE t MODIFY a BIGINT NOT NULL, ALGORITHM=COPY, LOCK=SHARED;
```

*Phần rollback (E6b).* Bảng `big(n, user_id, note)` 1.000.000 hàng chép từ `e3.src`. Với mỗi giá trị của `innodb_online_alter_log_max_size` (65.536 rồi 134.217.728): phiên A chạy `ALTER TABLE big FORCE, ALGORITHM=INPLACE, LOCK=NONE`; phiên B `DO SLEEP(1)`, xác nhận ALTER đang chạy qua `information_schema.PROCESSLIST`, rồi chạy 10 câu `UPDATE big SET note = CONCAT('u', LEFT(note, 50)) WHERE n BETWEEN …` mỗi câu 5.000 hàng. Cuối cùng đặt lại biến về 134.217.728.

**Kết quả đã đo** (thông báo lỗi giữ nguyên văn; đã rút gọn):

| Câu | Kết quả |
| --- | --- |
| Thêm cột ở cuối, thêm cột ở giữa, bỏ cột (`INSTANT`) | OK; `INSTANT_COLS` = 0, `TOTAL_ROW_VERSIONS` = 3 |
| `VARCHAR(20)` → `VARCHAR(60)` (`INPLACE`) | OK |
| `VARCHAR(60)` → `VARCHAR(100)` (`INPLACE`) | ERROR 1846: "Cannot change column type INPLACE. Try ALGORITHM=COPY." |
| ENUM thêm `'z'` ở cuối (`INSTANT`) | OK |
| ENUM chèn `'w'` ở đầu (`INPLACE`) | ERROR 1846: "Cannot change column type INPLACE…" |
| `INT` → `BIGINT` (`INPLACE`) | ERROR 1846: "Cannot change column type INPLACE…" |
| `CONVERT TO CHARACTER SET latin1` (`INPLACE`) | ERROR 1846: "Cannot change column type INPLACE…" |
| `DEFAULT CHARACTER SET latin1` (`INPLACE`) | OK |
| `DROP PRIMARY KEY, ADD PRIMARY KEY (a, id)` (`INPLACE, LOCK=NONE`) | OK |
| `DROP PRIMARY KEY` (`INPLACE`) | ERROR 1846: "Dropping a primary key is not allowed without also adding a new primary key. Try ALGORITHM=COPY." |
| `ADD FOREIGN KEY`, `foreign_key_checks=1` (`INPLACE`) | ERROR 1846: "Adding foreign keys needs foreign_key_checks=OFF. Try ALGORITHM=COPY." |
| `ADD FOREIGN KEY`, `foreign_key_checks=0` (`INPLACE`) | OK |
| `DROP FOREIGN KEY` (`INPLACE`) | OK |
| `ENCRYPTION='Y'` (`INPLACE`) | ERROR 1846: "Cannot alter encryption attribute by inplace algorithm.. Try ALGORITHM=COPY." |
| `ADD INDEX` (`INPLACE, LOCK=NONE`) | OK |
| `FORCE` (`INPLACE, LOCK=NONE`) | OK |
| `INT` → `BIGINT` với `COPY, LOCK=NONE` | ERROR 1846: "LOCK=NONE is not supported. Reason: COPY algorithm requires a lock. Try LOCK=SHARED." |
| `INT` → `BIGINT` với `COPY, LOCK=SHARED` | OK |

*E6b:* với 64 KB, `alter_running_threads` = 1 khi phiên B bắt đầu cập nhật, và phiên A nhận **ERROR 1799 (HY000): "Creating index 'PRIMARY' required more than 'innodb_online_alter_log_max_size' bytes of modification log. Please try again."** Với 128 MB, cùng kịch bản: "A: ALTER finished OK", 3,99 giây.

**Diễn giải và đối chiếu với sách.**

- **Khớp sách:** đổi charset của dữ liệu và bật mã hóa không làm được với `INPLACE`.
- **Khác sách:** đổi khóa chính (bỏ và thêm trong cùng câu) chạy được `INPLACE, LOCK=NONE` (có rebuild); thêm khóa ngoại chạy được `INPLACE` khi tắt `foreign_key_checks`, và bỏ khóa ngoại luôn `INPLACE`. Danh sách của tr. 149–150 nên hiểu là "có điều kiện", đúng như chính sách khuyên tra tài liệu (chú thích 7).
- **Ngoài danh sách nhưng đáng biết:** đổi kiểu cột và nới `VARCHAR` qua ngưỡng 255 byte cần `COPY` có khóa; đây chính là lúc công cụ ngoài như gh-ost hay pt-online-schema-change cần thiết (tr. 150).
- **Rollback:** lỗi 1799 là cơ chế của cảnh báo ở tr. 150. Trong E6b chỉ cần 50.000 hàng thay đổi để vượt 64 KB; trên production với mặc định 128 MB, một ALTER nhiều giờ trên bảng ghi nhiều vẫn có thể vượt.

**Dọn dẹp.** Đã đặt lại `innodb_online_alter_log_max_size = 134217728`; `DROP DATABASE e6;`

---

<a id="chua-thuc-thi"></a>

## Phần chưa thực thi

| Thí nghiệm đề xuất | Lý do chưa chạy | Quy trình gợi ý |
| --- | --- | --- |
| So sánh pt-online-schema-change và gh-ost trên cùng bảng có tải ghi | Phải tải công cụ từ internet, không được phép trong môi trường này | Dựng source và replica (như chương 9), tạo bảng có tải ghi đều, chạy cùng thay đổi bằng hai công cụ; đo thời gian, độ trễ ghi của ứng dụng, replication lag; thử bảng có khóa ngoại để thấy gh-ost dừng (tr. 151) |
| Pipeline schema với Skeema và CI | Cần tải Skeema và hệ thống CI | Repo mỗi cluster, `skeema lint` trong pull request, quy tắc cấm khóa ngoại và charset (tr. 149, 152–153) |
| Điểm nóng ở cuối index với nhiều luồng ghi (chú thích 6, tr. 143) | Cần công cụ sinh tải nhiều luồng (ví dụ sysbench), không có sẵn | Lặp E3 với 16–64 luồng insert; so sánh độ trễ p99 và chờ latch giữa `AUTO_INCREMENT` và khóa rải rác |

## Dọn dẹp chung

Sau khi chạy xong: `DROP DATABASE` cho `e1`…`e6` (hoặc xóa cả datadir riêng), rồi `mysqladmin -h127.0.0.1 -P3316 -uroot shutdown`. Instance của thí nghiệm này đã được tắt sau khi đo.
