# Thực hành — Chương 10: Backup and Recovery

Quay về ghi chú: [Chương 10 — Sao lưu và phục hồi](../chapter-10.md) · Đáp án tự kiểm tra: [answers/chapter-10.md](../answers/chapter-10.md)

> **Nhãn.** Schema, dữ liệu, SQL và kịch bản trong tệp này là **Ví dụ minh họa do AI xây dựng**. Mỗi thí nghiệm ghi **Dự đoán** trước khi chạy và **Kết quả đã đo** sau khi chạy. Output được chép từ lần chạy thật, có chỗ **đã rút gọn** (ghi rõ). Phần [Chưa thực thi](#chua-thuc-thi) chỉ là quy trình, chưa chạy.

<a id="moi-truong"></a>

## Môi trường

| Mục | Giá trị |
| --- | --- |
| MySQL | Community **8.0.42** (Win64), các binary trong `C:\Program Files\MySQL\MySQL Server 8.0\bin\` |
| Máy | Laptop Windows 11, 16 luồng CPU, SSD; nhiều tiến trình khác chạy song song: **có nhiễu, không phải benchmark production** |
| Instance nguồn | Cổng **3320**, datadir riêng trong thư mục tạm, khởi tạo bằng `--initialize-insecure`, chạy với `--no-defaults --bind-address=127.0.0.1 --mysqlx=OFF --gtid-mode=ON --enforce-gtid-consistency=ON --server-id=1020` |
| Instance phục hồi | Cổng **3335**, datadir riêng, khởi tạo lại trước mỗi lần phục hồi; thí nghiệm 2 chạy với GTID tắt, thí nghiệm 3 và 4 với GTID bật |
| Cấu hình còn lại | Mặc định của 8.0.42: `innodb_buffer_pool_size` 128 MB, `binlog_format` ROW, `sync_binlog` 1, `innodb_flush_log_at_trx_commit` 1, `binlog_expire_logs_seconds` 2592000 |
| Client | Mọi client (`mysql`, `mysqldump`, `mysqlpump`, `mysqlbinlog`, `mysqladmin`, `mysqlcheck`) chạy với `--no-defaults -h127.0.0.1 -P<cổng> -uroot` để không đọc cấu hình của MySQL trên máy và không chạm vào instance ở cổng 3306 |
| Đo thời gian | Thời gian thực bằng `date +%s.%N` (Git Bash) hay `time.perf_counter()` (Python); lặp 3 lần, báo trung vị kèm khoảng min–max |

Khởi động và tắt một instance (Git Bash; `$BIN` là thư mục binary, `$DATADIR` là datadir riêng):

```bash
"$BIN/mysqld" --no-defaults --initialize-insecure --datadir="$DATADIR" --console
"$BIN/mysqld" --no-defaults --datadir="$DATADIR" --port=3320 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=ch10.sock --gtid-mode=ON --enforce-gtid-consistency=ON \
  --server-id=1020 --console > mysqld.log 2>&1 &
# chờ tới khi lệnh sau thành công
"$BIN/mysql" --no-defaults -h127.0.0.1 -P3320 -uroot -e "SELECT @@port, @@datadir"
# luôn tắt khi xong
"$BIN/mysqladmin" --no-defaults -h127.0.0.1 -P3320 -uroot shutdown
```

Trong các lệnh dưới, `my_src` là viết tắt của `"$BIN/mysql" --no-defaults -h127.0.0.1 -P3320 -uroot`, `my_rst` tương tự với cổng 3335, `dump_src` là `mysqldump` với cùng các tham số kết nối tới 3320.

---

<a id="exp1"></a>

## Thí nghiệm 1 — `mysqldump` khi có ghi đồng thời: nhất quán và mức gián đoạn

**Mục tiêu.** Kiểm chứng rằng backup "online" có thể gián đoạn dịch vụ tùy yêu cầu nhất quán (tr. 262), và so sánh ba chế độ của `mysqldump`: không khóa, `--single-transaction`, mặc định (`LOCK TABLES`). Liên quan mục [2.3.1](../chapter-10.md#online-offline) và [2.6.1](../chapter-10.md#single-transaction).

**Giả thuyết (Dự đoán).**

- `--skip-lock-tables` (không transaction): mỗi bảng được đọc ở một thời điểm khác nhau, nên một bất biến xuyên bảng sẽ **sai** trong bản dump.
- `--single-transaction`: mọi bảng InnoDB đọc từ cùng một snapshot MVCC, bất biến **đúng**, luồng ghi gần như không bị ảnh hưởng.
- Mặc định: `LOCK TABLES ... READ` cho cả database, bất biến **đúng**, nhưng luồng ghi bị **chặn** trong lúc dump.

**Chuẩn bị.** Hai bảng 200 000 hàng, mỗi hàng số dư 1000. Một thủ tục chuyển 1 đơn vị từ `checking` sang `savings` của cùng `id`, mỗi lần một transaction, nên tổng hai bảng luôn là 400 000 000 tại mọi điểm commit.

```sql
DROP DATABASE IF EXISTS bank;
CREATE DATABASE bank;
USE bank;
CREATE TABLE checking (
  id      INT PRIMARY KEY,
  balance INT NOT NULL,
  pad     CHAR(100) NOT NULL
) ENGINE=InnoDB;
CREATE TABLE savings LIKE checking;

SET SESSION cte_max_recursion_depth = 1000;
INSERT INTO checking (id, balance, pad)
WITH RECURSIVE d AS (SELECT 0 AS n UNION ALL SELECT n + 1 FROM d WHERE n < 999)
SELECT a.n * 200 + b.n + 1, 1000, REPEAT('x', 100)
FROM d AS a JOIN d AS b
WHERE a.n < 1000 AND b.n < 200;
INSERT INTO savings SELECT * FROM checking;

DELIMITER //
CREATE PROCEDURE transfer_loop(IN secs INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  DECLARE r INT;
  DECLARE end_ts DATETIME(6) DEFAULT SYSDATE(6) + INTERVAL secs SECOND;
  WHILE SYSDATE(6) < end_ts DO
    SET r = (i * 7919) % 200000 + 1;
    START TRANSACTION;
    UPDATE checking SET balance = balance - 1 WHERE id = r;
    UPDATE savings  SET balance = balance + 1 WHERE id = r;
    COMMIT;
    SET i = i + 1;
  END WHILE;
  SELECT i AS transfers_done;
END//
DELIMITER ;
```

> Ghi chú: thủ tục dùng `SYSDATE(6)` vì `NOW()` trong stored routine cố định tại lúc bắt đầu câu `CALL`. Binlog dạng ROW nên tính không tất định của `SYSDATE()` không ảnh hưởng replication hay PITR ở đây (chương 5, tr. 122 cảnh báo điều này với binlog dạng statement).

**Các bước và lệnh đo.** Với mỗi chế độ, lặp 3 lần:

1. Chạy nền `CALL bank.transfer_loop(10)`.
2. Sau 1 giây, đo tốc độ cập nhật nền trong 2 giây bằng hiệu số của `Innodb_rows_updated` (đọc từ `performance_schema.global_status`).
3. Đo `Innodb_rows_updated` và thời gian ngay trước và ngay sau lệnh dump; lần lặp đầu bật general log để ghi lại câu lệnh `mysqldump` gửi.
4. Dump: `dump_src <tùy chọn chế độ> --set-gtid-purged=OFF bank > dump.sql`, với tùy chọn là `--skip-lock-tables`, `--single-transaction`, hoặc không thêm gì.
5. Chờ luồng ghi xong. Nạp bản dump vào một database tạm với `--init-command="SET SESSION sql_log_bin=0"`, rồi tính `SUM(checking.balance) + SUM(savings.balance)`.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance 3320, dữ liệu tổng hợp 2 × 200 000 hàng; mỗi bản dump khoảng 46 MB):

| Chế độ | Lần | Thời gian dump (s) | Hàng cập nhật/s trước dump | Hàng cập nhật/s trong lúc dump | Tổng sau restore | Lệch |
| --- | --- | --- | --- | --- | --- | --- |
| `--skip-lock-tables` | 1 | 0,65 | 953 | 1004 | 400 000 132 | +132 |
| `--skip-lock-tables` | 2 | 1,20 | 900 | 729 | 400 000 204 | +204 |
| `--skip-lock-tables` | 3 | 1,22 | 893 | 810 | 400 000 257 | +257 |
| `--single-transaction` | 1 | 1,31 | 929 | 849 | 400 000 000 | 0 |
| `--single-transaction` | 2 | 1,24 | 958 | 855 | 400 000 000 | 0 |
| `--single-transaction` | 3 | 1,15 | 1015 | 897 | 400 000 000 | 0 |
| Mặc định (`LOCK TABLES`) | 1 | 1,46 | 782 | 113 | 400 000 000 | 0 |
| Mặc định (`LOCK TABLES`) | 2 | 1,03 | 955 | 163 | 400 000 000 | 0 |
| Mặc định (`LOCK TABLES`) | 3 | 1,33 | 864 | 109 | 400 000 000 | 0 |

Câu lệnh `mysqldump` gửi tới server (general log, lần 1, **đã rút gọn** các truy vấn metadata):

| Chế độ | Trình tự chính |
| --- | --- |
| `--skip-lock-tables` | `show tables` → `SELECT /*!40001 SQL_NO_CACHE */ * FROM checking` → … → `SELECT ... FROM savings` |
| `--single-transaction` | `SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ` → `START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */` → `UNLOCK TABLES` → `SAVEPOINT sp` → `SELECT ... FROM checking` → `ROLLBACK TO SAVEPOINT sp` → `SELECT ... FROM savings` → … |
| Mặc định | `show tables` → `LOCK TABLES checking READ /*!32311 LOCAL */, savings READ /*!32311 LOCAL */` → `SELECT ... FROM checking` → `SELECT ... FROM savings` → … |

Khi dump một database trên server bật GTID mà không có `--set-gtid-purged=OFF` và không có tùy chọn nhất quán, `mysqldump` 8.0.42 in cảnh báo (đã rút gọn): bản dump có thể không nhất quán, hãy dùng `--single-transaction`, `--lock-all-tables` hay `--master-data`.

**Diễn giải và đối chiếu với sách.**

- Độ lệch dương ở chế độ không khóa bằng đúng số lần chuyển tiền đã commit giữa lúc câu `SELECT` của `checking` bắt đầu và lúc câu `SELECT` của `savings` bắt đầu: `checking` được chụp sớm (chưa bị trừ), `savings` được chụp muộn (đã được cộng). Mỗi bảng tự nó vẫn nhất quán, vì một câu `SELECT` trong InnoDB đọc từ một snapshot; lỗi chỉ lộ ra qua bất biến **xuyên bảng**.
- `--single-transaction` cho lệch 0 trong khi luồng ghi vẫn chạy khoảng 850–900 hàng/giây: snapshot MVCC ([Chương 1](../chapter-01.md#mvcc)) cho phép đọc nhất quán mà không chặn ghi.
- Chế độ mặc định cũng đúng, nhưng tốc độ ghi trong cửa sổ đo rơi xuống khoảng 110–160 hàng/giây. Cửa sổ đo gồm cả phần trước `LOCK TABLES` và sau khi nhả, nên trong lúc khóa thực sự luồng ghi gần như đứng hẳn. Đây là minh họa cụ thể cho câu "backup online vẫn có thể gián đoạn dịch vụ đáng kể" (tr. 262).
- Sách không bàn các tùy chọn này; phần so sánh là **Bổ sung ngoài sách**.

**Dọn dẹp.** `DROP DATABASE v_check` (database tạm, tạo với `sql_log_bin = 0`). Database `bank` được giữ cho thí nghiệm 4.

---

<a id="exp2"></a>

## Thí nghiệm 2 — Thời gian backup và restore logic theo kích thước dữ liệu

**Mục tiêu.** Đo thời gian dump và nạp lại so với kích thước dữ liệu (tr. 264, 286), tác động của secondary index và binlog lên thời gian restore (tr. 264, 282), và lợi ích của dump song song (tr. 269–272).

**Giả thuyết (Dự đoán).**

- Restore chậm hơn dump **nhiều lần** (parse SQL, dựng index, ghi redo và binlog).
- Restore bảng không có secondary index nhanh hơn rõ rệt; tắt binlog cho phiên nạp cũng nhanh hơn.
- Dump file nhỏ hơn file `.ibd` khi bảng có index.
- Dump song song (`mysqlpump` 4 luồng) nhanh hơn `mysqldump` khi có nhiều bảng.

**Chuẩn bị.** Trên 3320, bốn database, mỗi database một bảng `events` (khóa chính BIGINT, 5 cột, 2 secondary index trừ biến thể `_noidx`). Giá trị tất định theo `n`.

```sql
SET SESSION sql_log_bin = 0;
SET SESSION cte_max_recursion_depth = 2000;
DROP DATABASE IF EXISTS seq; CREATE DATABASE seq;
CREATE TABLE seq.k1000 (n INT PRIMARY KEY);
INSERT INTO seq.k1000
WITH RECURSIVE d AS (SELECT 0 AS n UNION ALL SELECT n + 1 FROM d WHERE n < 999) SELECT n FROM d;
CREATE TABLE seq.k2000 (n INT PRIMARY KEY);
INSERT INTO seq.k2000
WITH RECURSIVE d AS (SELECT 0 AS n UNION ALL SELECT n + 1 FROM d WHERE n < 1999) SELECT n FROM d;

DELIMITER //
CREATE PROCEDURE seq.make_events(IN db VARCHAR(64), IN nrows INT, IN with_idx BOOL)
BEGIN
  SET @ddl = CONCAT('CREATE TABLE `', db, '`.events (',
    ' id BIGINT NOT NULL PRIMARY KEY,',
    ' user_id INT NOT NULL,',
    ' created_at DATETIME NOT NULL,',
    ' status TINYINT NOT NULL,',
    ' amount DECIMAL(10,2) NOT NULL,',
    ' note VARCHAR(64) NOT NULL',
    IF(with_idx, ', KEY idx_user_created (user_id, created_at), KEY idx_status (status)', ''),
    ') ENGINE=InnoDB');
  SET @s = CONCAT('DROP DATABASE IF EXISTS `', db, '`'); PREPARE st FROM @s; EXECUTE st;
  SET @s = CONCAT('CREATE DATABASE `', db, '`'); PREPARE st FROM @s; EXECUTE st;
  PREPARE st FROM @ddl; EXECUTE st;
  SET @s = CONCAT('INSERT INTO `', db, '`.events ',
    'SELECT n, (n * 7919) % 50000 + 1, ',
    '  ''2021-01-01'' + INTERVAL ((n * 104729) % 31536000) SECOND, ',
    '  (n * 31) % 5, ((n * 613) % 100000) / 100, ',
    '  CONCAT(''note-'', LPAD(CONV((n * 2654435761) % 4294967296, 10, 36), 7, ''0''), ''-'', n % 997) ',
    'FROM (SELECT a.n * 1000 + b.n + 1 AS n FROM seq.k2000 a CROSS JOIN seq.k1000 b ',
    '      WHERE a.n * 1000 + b.n < ', nrows, ' ORDER BY 1) x');
  PREPARE st FROM @s; EXECUTE st;
  DEALLOCATE PREPARE st;
END//
DELIMITER ;

CALL seq.make_events('sz250k', 250000, TRUE);
CALL seq.make_events('sz1m', 1000000, TRUE);
CALL seq.make_events('sz2m', 2000000, TRUE);
CALL seq.make_events('sz1m_noidx', 1000000, FALSE);
```

Sinh dữ liệu mất khoảng 1 phút 50 giây.

**Các bước và lệnh đo** (một script Python gọi các client, đo bằng `time.perf_counter()`):

1. Với mỗi database: dump 3 lần bằng `mysqldump --single-transaction --set-gtid-purged=OFF --databases <db>`.
2. Nạp file dump vào instance 3335 3 lần (`DROP DATABASE` trước mỗi lần, không tính vào thời gian), đo thời gian và lượng binlog sinh ra trên 3335 (tổng `File_size` của `SHOW BINARY LOGS` trước và sau).
3. Với `sz1m`: nạp thêm 3 lần với `--init-command="SET SESSION sql_log_bin=0"`.
4. So sánh `CHECKSUM TABLE` giữa nguồn và đích; ghi kích thước `.ibd` ở nguồn và ở đích.
5. Với ba database `sz250k`, `sz1m`, `sz1m_noidx` (tổng 2,25 triệu hàng): so `mysqldump` với `mysqlpump --single-transaction --set-gtid-purged=OFF --default-parallelism=4`, rồi nạp lại cả hai file.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, dữ liệu tổng hợp; trung vị 3 lần, khoảng min–max trong ngoặc):

| Database | Số hàng | `.ibd` nguồn | File dump | Dump (s) | Restore (s) | Restore/dump | Hàng/s khi nạp | Binlog sinh ra | `.ibd` sau restore | Checksum |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `sz250k` | 250 000 | 44 MB | 15,3 MB | 0,44 (0,44–0,62) | 6,69 (5,99–7,15) | 15,4 | 37 346 | 10,0 MB | 40 MB | khớp |
| `sz1m` | 1 000 000 | 140 MB | 61,5 MB | 2,07 (2,03–2,17) | 22,53 (17,73–26,35) | 10,9 | 44 382 | 40,1 MB | 124 MB | khớp |
| `sz2m` | 2 000 000 | 268 MB | 124,0 MB | 2,72 (2,70–2,80) | 35,04 (33,57–35,94) | 12,9 | 57 073 | 80,3 MB | 236 MB | khớp |
| `sz1m_noidx` | 1 000 000 | 80 MB | 61,5 MB | 0,99 (0,78–1,30) | 16,12 (9,87–16,76) | 16,3 | 62 024 | 40,1 MB | 68 MB | khớp |

| Biến thể | Kết quả |
| --- | --- |
| `sz1m`, nạp với `sql_log_bin = 0` | 13,60 s (13,42–14,06), binlog sinh ra 0 MB |
| 3 database, `mysqldump` | dump 3,84 s (3,79–4,10), file 138,2 MB; nạp lại 48,78 s (48,42–52,92) |
| 3 database, `mysqlpump` 4 luồng | dump 1,98 s (1,91–2,13), file 138,6 MB; nạp lại 42,37 s (40,28–47,71) |

Quan sát thêm về file dump:

- `sz1m.sql` của `mysqldump` có 63 câu `INSERT` nhiều hàng, trung bình khoảng 1,02 MB mỗi câu, cùng `LOCK TABLES ... WRITE` và `ALTER TABLE ... DISABLE KEYS` đúng như listing ở tr. 271.
- `mysqlpump` 8.0.42 in `WARNING: mysqlpump is deprecated and will be removed in a future version. Use mysqldump instead.` File của nó tắt `UNIQUE_CHECKS`, `FOREIGN_KEY_CHECKS`, đặt `SET @@SESSION.SQL_LOG_BIN= 0`, tạo bảng **không** có secondary index rồi thêm index bằng `ALTER TABLE ... ADD KEY` **sau** khi nạp dữ liệu, và các câu `INSERT` của các bảng **xen kẽ** nhau (do nhiều luồng ghi chung một file).

**Diễn giải và đối chiếu với sách.**

- Restore chậm hơn dump 11–16 lần ở mọi kích thước: đúng chiều với cảnh báo "backup vài giờ, restore nhiều tuần" (tr. 286) và với nhận định chi phí lớn nhất của logical backup là nạp lại qua SQL (tr. 264).
- File dump nhỏ hơn `.ibd`: 61,5 MB so với 140 MB khi có 2 secondary index, và vẫn nhỏ hơn 80 MB khi không có index. Sách nói logical thường nhỏ hơn raw khi có nhiều index (tr. 264). Sau khi nạp lại, `.ibd` nhỏ đi một chút (124 so với 140 MB), một dấu hiệu tablespace nguồn có chỗ trống (tr. 264).
- Bỏ secondary index làm dump nhanh hơn khoảng 2 lần và nạp nhanh hơn khoảng 28% theo trung vị, nhưng độ dao động của lần nạp lớn (9,9–16,8 s), nên chỉ kết luận được chiều hướng.
- Tắt binlog cho phiên nạp làm nhanh khoảng 40% (22,5 → 13,6 s) và tránh 40 MB binlog: đúng lý do sách gợi ý (tr. 282).
- Dump song song nhanh gần gấp đôi với 3 bảng: phù hợp lập luận của sách về mydumper (tr. 269). Thời gian nạp file `mysqlpump` ngắn hơn khoảng 13%, nhưng **có nhiều yếu tố gộp lại** (tắt binlog, tắt unique check, thêm index sau): không quy được cho riêng việc "thêm index sau".
- Tốc độ nạp tính theo hàng/giây **tăng** từ 37 000 lên 57 000 khi dữ liệu lớn lên tới 2 triệu hàng (có lẽ chi phí cố định và hiệu ứng khởi động chiếm tỷ trọng lớn ở file nhỏ). Thí nghiệm **chưa** tái hiện được hiện tượng restore chậm hẳn đi khi dữ liệu vượt bộ nhớ (tr. 264); muốn kiểm phải dùng dữ liệu lớn hơn nhiều lần buffer pool.

**Dọn dẹp.** Các database thử trên 3335 bị xóa khi khởi tạo lại datadir cho thí nghiệm 3. Các database `sz*` trên 3320 được giữ cho thí nghiệm 4.

---

<a id="exp3"></a>

## Thí nghiệm 3 — Point-in-time recovery sau một `DROP TABLE` lỡ tay

**Mục tiêu.** Chạy trọn bước 8 "xem xét và phát lại binary log" của quy trình phục hồi (tr. 281), theo đúng yêu cầu PITR ở tr. 261 và 269: restore backup rồi phát lại binlog tới ngay trước lệnh hỏng. Thêm điều kiện thực tế: ứng dụng vẫn ghi vào bảng khác **sau** sự cố.

**Giả thuyết (Dự đoán).**

- Restore dump rồi phát lại binlog từ tọa độ trong dump tới **ngay trước** sự kiện `DROP`, rồi phát lại tiếp phần **sau** sự kiện đó, cho ra đúng trạng thái mong đợi của cả hai bảng.
- Cách theo GTID (`--exclude-gtids`) cho cùng kết quả; transaction đã có trong `GTID_PURGED` của dump được tự bỏ qua; phát lại lần hai không đổi gì.
- Phát lại **toàn bộ** binlog thì bảng lại mất: đây cũng là điều một replica làm.

**Chuẩn bị** (trên 3320, GTID bật):

```sql
DROP DATABASE IF EXISTS shop;
CREATE DATABASE shop;
USE shop;
CREATE TABLE customers (
  id     INT AUTO_INCREMENT PRIMARY KEY,
  name   VARCHAR(50) NOT NULL,
  credit INT NOT NULL
) ENGINE=InnoDB;
CREATE TABLE orders (
  id          INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  amount      DECIMAL(10,2) NOT NULL,
  KEY idx_customer (customer_id)
) ENGINE=InnoDB;

SET SESSION cte_max_recursion_depth = 10000;
INSERT INTO customers (name, credit)
WITH RECURSIVE s AS (SELECT 1 AS n UNION ALL SELECT n + 1 FROM s WHERE n < 1000)
SELECT CONCAT('cust-', n), 100 FROM s;
INSERT INTO orders (customer_id, amount)
WITH RECURSIVE s AS (SELECT 1 AS n UNION ALL SELECT n + 1 FROM s WHERE n < 5000)
SELECT (n * 37) % 1000 + 1, (n * 13) % 500 + 0.99 FROM s;

DELIMITER //
CREATE PROCEDURE place_orders(IN k INT, IN seed INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < k DO
    START TRANSACTION;
    INSERT INTO orders (customer_id, amount) VALUES ((seed + i * 7) % 1000 + 1, (seed + i) % 300 + 0.5);
    UPDATE customers SET credit = credit - 1 WHERE id = (seed + i * 7) % 1000 + 1;
    COMMIT;
    SET i = i + 1;
  END WHILE;
END//
CREATE PROCEDURE add_customers(IN k INT, IN seed INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < k DO
    INSERT INTO customers (name, credit) VALUES (CONCAT('late-', seed, '-', i), 50);
    UPDATE customers SET credit = credit + 5 WHERE id = (seed + i * 11) % 1000 + 1;
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
```

**Các bước và lệnh đo.**

```bash
# 1. Full backup logic: snapshot nhất quán + tọa độ binlog + GTID, bắt đầu binlog mới
dump_src --single-transaction --source-data=2 --flush-logs --set-gtid-purged=ON \
  --routines --events --triggers --databases shop > full.sql
# 2. Ghi tiếp sau backup
my_src shop -e "CALL place_orders(300, 1)"
my_src shop -e "CALL add_customers(50, 2)"
my_src -e "CHECKSUM TABLE shop.orders"            # trạng thái mong đợi của orders
# 3. Sự cố
my_src -e "DROP TABLE shop.orders"
# 4. Ứng dụng vẫn ghi vào bảng khác
my_src shop -e "CALL add_customers(100, 3)"
my_src -e "CHECKSUM TABLE shop.customers"         # trạng thái mong đợi của customers
# 5. Sao lưu binlog qua mạng sang thư mục backup
"$BIN/mysqlbinlog" --no-defaults --read-from-remote-server --host=127.0.0.1 --port=3320 --user=root \
  --raw --to-last-log --result-file=binlogs/ binlog.000003
# 6. Tìm sự kiện hỏng
my_src -e "SHOW BINLOG EVENTS IN 'binlog.000003'"
# 7A. Phục hồi theo vị trí (3335 vừa khởi tạo, GTID bật)
my_rst < full.sql
"$BIN/mysqlbinlog" --no-defaults --start-position=197 --stop-position=159525 binlogs/binlog.000003 | my_rst
"$BIN/mysqlbinlog" --no-defaults --start-position=159736 binlogs/binlog.000003 | my_rst
# 7B. Phục hồi theo GTID (3335 khởi tạo lại)
my_rst < full.sql
"$BIN/mysqlbinlog" --no-defaults --exclude-gtids='<server_uuid>:43466' binlogs/binlog.000003 | my_rst
# 7C. Đối chứng: phát lại toàn bộ (3335 khởi tạo lại)
my_rst < full.sql
"$BIN/mysqlbinlog" --no-defaults binlogs/binlog.000003 | my_rst
```

Các vị trí 197, 159525, 159736 và GTID `:43466` là giá trị của lần chạy này, đọc từ file dump và từ `SHOW BINLOG EVENTS`; chạy lại sẽ ra số khác.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11; `server_uuid` nguồn `9a2c6499-c932-11f1-a8b5-00090ffe0001`, viết tắt `<uuid>` dưới đây):

Phần đầu file dump (đã rút gọn):

```text
SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
SET @@SESSION.SQL_LOG_BIN= 0;
SET @@GLOBAL.GTID_PURGED=/*!80000 '+'*/ '<uuid>:1-43065';
-- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000003', MASTER_LOG_POS=197;
```

Trình tự câu lệnh của `mysqldump` (general log, đã rút gọn): `FLUSH /*!40101 LOCAL */ TABLES` → `FLUSH TABLES WITH READ LOCK` → lệnh Refresh (xoay vòng log, do `--flush-logs`) → `SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ` → `START TRANSACTION WITH CONSISTENT SNAPSHOT` → `SELECT @@GLOBAL.GTID_EXECUTED` → `SHOW MASTER STATUS` → `UNLOCK TABLES` → dump từng bảng. Dump mất 0,16 s.

Trạng thái mong đợi: `orders` 5 300 hàng, checksum 535405811 (ngay trước `DROP`); `customers` 1 150 hàng, checksum 2790939615 (cuối cùng).

Ba sự kiện quanh sự cố trong `SHOW BINLOG EVENTS` (binlog.000003 có tổng 3 604 sự kiện):

| Pos | Event_type | End_log_pos | Info |
| --- | --- | --- | --- |
| 159525 | Gtid | 159602 | `SET @@SESSION.GTID_NEXT= '<uuid>:43466'` |
| 159602 | Query | 159736 | ``DROP TABLE `shop`.`orders` /* generated by server */ /* xid=304663 */`` |
| 159736 | Gtid | 159815 | `SET @@SESSION.GTID_NEXT= '<uuid>:43467'` |

| Cách | Thời gian (s) | `orders` | `customers` | `gtid_executed` sau phục hồi |
| --- | --- | --- | --- | --- |
| 7A theo vị trí | nạp dump 0,22; phát lại trước `DROP` 1,00; sau `DROP` 0,53 | 5 300 hàng, checksum **khớp** | 1 150 hàng, checksum **khớp** | `<uuid>:1-43465:43467-43666` |
| 7B theo GTID | nạp dump 0,18; phát lại 1,58 | **khớp** | **khớp** | `<uuid>:1-43465:43467-43666` |
| 7B, phát lại lần hai | | vẫn khớp | vẫn khớp | không đổi |
| 7C toàn bộ binlog | | lỗi 1146: bảng `shop.orders` không tồn tại | **khớp** | |

**Diễn giải và đối chiếu với sách.**

- Backup + binlog qua được phép thử `DROP TABLE` (tr. 286) với RPO bằng 0 cho cả hai bảng, kể cả các ghi hợp lệ sau sự cố.
- Cách theo vị trí cần hai lần gọi `mysqlbinlog` và phải chọn đúng biên: `--stop-position` là vị trí **bắt đầu** của sự kiện GTID của `DROP` (159525, không phải vị trí câu Query), và phần sau bắt đầu ở `End_log_pos` của câu `DROP` (159736). Cách theo GTID chỉ cần biết một GTID; transaction có sẵn trong `GTID_PURGED` (nạp từ dump) bị server tự bỏ qua, nên có thể đưa cả file vào. Đây là ví dụ cụ thể cho lời khuyên dùng GTID của sách (tr. 268).
- "Lỗ" `:43466` trong `gtid_executed` là dấu vết thấy được của transaction đã bị cố ý bỏ qua. Nếu server phục hồi này sau đó được nối làm replica của source cũ, lỗ đó sẽ làm nó cố lấy lại đúng transaction `DROP`; cần xử lý trước khi nối (**Bổ sung ngoài sách**, chưa thử).
- 7C cho thấy vì sao replica không phải backup (tr. 260): một bản sao áp toàn bộ dòng thay đổi sẽ lặp lại cả lệnh phá hoại.
- `mysqldump --source-data` của 8.0.42 vẫn ghi dòng tọa độ dạng `CHANGE MASTER TO MASTER_LOG_FILE`; script phân tích file dump phải chấp nhận cả dạng này lẫn dạng `SOURCE_LOG_FILE`.
- Dump kèm GTID tự tắt binlog cho phiên nạp (`SQL_LOG_BIN= 0`) và đặt `GTID_PURGED` thay vào đó.

**Dọn dẹp.** Mỗi lần phục hồi dùng một datadir 3335 vừa khởi tạo; tắt 3335 sau mỗi lần. Trên 3320, `shop` được giữ (chỉ còn `customers`).

---

<a id="exp4"></a>

## Thí nghiệm 4 — Raw backup chạy được trên Windows: transportable tablespace và clone

**Mục tiêu.** Không có XtraBackup và LVM, nên dùng hai cơ chế raw có sẵn trong MySQL 8.0 để kiểm chứng hai ý của sách: raw restore nhanh hơn logical vì không phải chạy SQL hay dựng index (tr. 264), và câu "chỉ restore được `.ibd` vào đúng server đã backup" (tr. 284). Cả hai cơ chế là **Bổ sung ngoài sách**.

**Giả thuyết (Dự đoán).**

- 4A: `FLUSH TABLES ... FOR EXPORT` + `IMPORT TABLESPACE` chuyển được bảng sang một instance **khác** trên 8.0.42, nhanh hơn nhiều lần so với nạp logical cùng bảng (22,5 s ở thí nghiệm 2).
- 4B: `CLONE LOCAL` trong lúc có ghi cho bản sao nhất quán (bất biến tổng đúng) mà không chặn ghi.

**Chuẩn bị.** Dùng lại `sz1m.events` và `bank` từ thí nghiệm 1, 2. Cài clone plugin trên 3320: `INSTALL PLUGIN clone SONAME 'mysql_clone.dll'` (tên file trên Windows).

**Các bước và lệnh đo.**

4A (3335 vừa khởi tạo, `server_uuid` khác 3320):

```sql
-- Trên 3320, phiên này phải mở suốt lúc chép file (bảng chỉ đọc trong lúc đó)
FLUSH TABLES sz1m.events FOR EXPORT;
-- (chép events.ibd và events.cfg từ datadir/sz1m/ ra thư mục backup)
UNLOCK TABLES;

-- Trên 3335
CREATE DATABASE sz1m;
-- CREATE TABLE sz1m.events (...) đúng định nghĩa lấy từ SHOW CREATE TABLE ở nguồn
ALTER TABLE sz1m.events DISCARD TABLESPACE;
-- (chép events.ibd và events.cfg vào datadir/sz1m/ của 3335)
ALTER TABLE sz1m.events IMPORT TABLESPACE;
CHECKSUM TABLE sz1m.events;
CHECK TABLE sz1m.events;
```

4B:

```sql
-- Trên 3320, trong lúc một phiên khác chạy CALL bank.transfer_loop(20)
CLONE LOCAL DATA DIRECTORY = '<thư mục chưa tồn tại>';
SELECT * FROM performance_schema.clone_status\G
SELECT STAGE, STATE, DATA FROM performance_schema.clone_progress;
```

Sau đó khởi động một `mysqld` ở cổng 3335 với `--datadir=<thư mục clone>`, kiểm tra bất biến tổng số dư và chạy `mysqlcheck --databases bank shop sz1m sz2m sz250k`.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11):

4A:

| Đo | Giá trị |
| --- | --- |
| `server_uuid` nguồn / đích | `9a2c6499-...` / `ba897812-...` (hai instance khác nhau) |
| File xuất | `events.ibd` 146 800 640 byte, `events.cfg` 1 225 byte |
| Giữ khóa export và chép ra | 0,65 s (gồm thời gian chờ `.cfg` xuất hiện) |
| Chép vào đích và `IMPORT TABLESPACE` | 2,47 s |
| Checksum nguồn / đích | 3392977300 / 3392977300: **khớp** |
| Số hàng, `CHECK TABLE` | 1 000 000; `status OK` |

4B:

| Đo | Giá trị |
| --- | --- |
| Thời gian `CLONE LOCAL` | 1,09 s, trong khi luồng ghi chạy (luồng ghi hoàn thành 9 555 lần chuyển trong 20 s) |
| `clone_progress` | FILE COPY khoảng 745,6 MB; PAGE COPY 1,4 MB; REDO COPY 0,7 MB; các giai đoạn RESTART, RECOVERY "Not Started" (clone cục bộ) |
| `clone_status` | `Completed`, SOURCE `LOCAL INSTANCE`; các cột BINLOG_FILE, GTID_EXECUTED để trống |
| Kích thước | datadir nguồn khoảng 955 MB, bản clone khoảng 749 MB |
| Nội dung bản clone | `#clone`, `#innodb_redo`, `ibdata1`, `mysql.ibd`, `undo_001`, `undo_002`, `ib_buffer_pool` và các thư mục database; **không có** file binlog, `auto.cnf`, file `.pem` |
| Khởi động instance trên bản clone | 3,25 s tới lúc nhận kết nối; error log: "No existing UUID has been found... Generating a new UUID", "CA certificate ca.pem is self signed" |
| Bất biến tổng số dư | 400 000 000: **đúng** |
| `mysqlcheck` 5 database | mọi bảng `OK`, 14,74 s |
| Biến clone liên quan | `clone_block_ddl = OFF`, `clone_max_data_bandwidth = 0` (không giới hạn) |

**Diễn giải và đối chiếu với sách.**

- 4A bác bỏ, **trên MySQL 8.0.42**, ràng buộc "chỉ restore `.ibd` vào đúng server đã backup" ở tr. 284: ràng buộc đó đúng với các bản cũ, trước khi có file `.cfg` và transportable tablespace (5.6). Các ràng buộc khác vẫn còn: cùng page size, định nghĩa bảng khớp, bảng nguồn chỉ đọc trong lúc export.
- 4A còn minh họa lợi thế raw của tr. 264: 2,5 s so với 22,5 s nạp logical cùng bảng, khoảng 9 lần, vì không parse SQL và không dựng lại hai secondary index.
- 4B là một raw backup trực tuyến: chép file, chép page và redo phát sinh trong lúc chép, giống tinh thần XtraBackup (tr. 270). Bất biến đúng dù có ghi đồng thời.
- Bản clone thiếu binlog, `auto.cnf`, chứng chỉ: đúng loại "dữ liệu dễ quên" của tr. 265–266; muốn PITR từ bản clone vẫn phải có binlog được backup riêng. Việc server tự sinh UUID mới cũng là điều mong muốn khi dùng bản sao để dựng server khác.
- Error log ở mức mặc định không in thông điệp mức Note, nên không thấy dòng crash recovery; không kết luận được recovery có chạy hay không.
- `mysqlcheck` mất gần 15 giây: kiểm tra backup theo cách sách khuyên (tr. 265) là chi phí thời gian cần tính vào RTO hay vào lịch diễn tập.

**Dọn dẹp.** Tắt instance 3335 chạy trên bản clone. Plugin clone vẫn cài trên instance 3320 riêng (không ảnh hưởng gì khác). Thư mục clone và datadir 3335 nằm trong thư mục tạm, có thể xóa.

---

<a id="exp5"></a>

## Thí nghiệm 5 — Hết hạn binlog, `log_status` và khóa backup

**Mục tiêu.** Kiểm chứng ba chi tiết: (1) `binlog_expire_logs_seconds` chỉ có hiệu lực khi khởi động hoặc xoay vòng binlog và dựa vào thời gian sửa file (tr. 269); (2) biến cũ `expire_logs_days` của tr. 261 trên 8.0.42; (3) nguồn tọa độ mà output XtraBackup nhắc tới (`p_s.log_status`) và khóa backup của MySQL 8.0 (tr. 278).

**Giả thuyết (Dự đoán).** Đặt thời hạn ngắn lúc runtime không xóa gì cho tới lần `FLUSH BINARY LOGS` kế tiếp; khi xoay vòng, file có mtime cũ bị xóa, file vừa sửa thì không. `LOCK INSTANCE FOR BACKUP` chặn DDL, không chặn DML.

**Chuẩn bị.** Instance 3320 sau các thí nghiệm trên.

**Các bước và lệnh đo.**

```sql
SHOW BINARY LOGS;
FLUSH BINARY LOGS; FLUSH BINARY LOGS;          -- tạo thêm hai file
-- chờ 4 giây
SET GLOBAL binlog_expire_logs_seconds = 2;
SHOW BINARY LOGS;                              -- chưa xoay vòng
INSERT INTO shop.customers (name, credit) VALUES ('exp5', 1);
SHOW BINARY LOGS;
FLUSH BINARY LOGS;                             -- xoay vòng
SHOW BINARY LOGS;
SET GLOBAL binlog_expire_logs_seconds = 2592000;
SET GLOBAL expire_logs_days = 7;               -- thử biến cũ
SET GLOBAL expire_logs_days = 0; SHOW WARNINGS;
SELECT SERVER_UUID, LOCAL, REPLICATION, STORAGE_ENGINES FROM performance_schema.log_status\G
-- Phiên A: LOCK INSTANCE FOR BACKUP; SELECT SLEEP(6); UNLOCK INSTANCE;
-- Phiên B (trong lúc A giữ khóa): một INSERT, rồi
--   SET SESSION lock_wait_timeout = 2; CREATE TABLE shop.ddl_probe (id INT PRIMARY KEY);
```

**Kết quả đã đo** (MySQL 8.0.42; output **đã rút gọn**):

| Thời điểm | Danh sách binlog |
| --- | --- |
| Ban đầu | `binlog.000001` (80 462 031 B), `000002`, `000003` |
| Sau hai lần `FLUSH` | thêm `000004` (241 B), `000005` (197 B) |
| Sau `SET GLOBAL binlog_expire_logs_seconds = 2` (đã chờ hơn 4 giây) | **không đổi**: vẫn `000001` tới `000005` |
| Sau một `INSERT` | không đổi, `000005` lớn lên 491 B |
| Sau `FLUSH BINARY LOGS` | chỉ còn `binlog.000005` (535 B) và `binlog.000006` (197 B) |

- Đặt `expire_logs_days = 7` khi `binlog_expire_logs_seconds` khác 0: `ERROR 3683 (HY000): The option expire_logs_days and binlog_expire_logs_seconds cannot be used together. Please use binlog_expire_logs_seconds to set the expire time (expire_logs_days is deprecated)`.
- Đặt `expire_logs_days = 0`: thành công kèm `Warning 1287 '@@expire_logs_days' is deprecated and will be removed in a future release. Please use binlog_expire_logs_seconds instead.`
- Giá trị ban đầu: `binlog_expire_logs_auto_purge = 1`, `max_binlog_size = 1073741824`.
- `performance_schema.log_status`: `LOCAL` = `{"gtid_executed": "<uuid>:1-53222", "binary_log_file": "binlog.000006", "binary_log_position": 197}`; `REPLICATION` = `{"channels": []}`; `STORAGE_ENGINES` = `{"InnoDB": {"LSN": 1507760731, "LSN_checkpoint": 1507760731}}` (đọc ở một lần truy vấn sau đó).
- Khóa backup: `INSERT` xong trong 0,08 s; `CREATE TABLE` chờ 2,08 s rồi `ERROR 1205 (HY000): Lock wait timeout exceeded`.

**Diễn giải và đối chiếu với sách.**

- Khớp tr. 269: đặt thời hạn chưa đủ, phải có xoay vòng. `binlog.000005` được giữ vì vừa bị sửa (bởi câu `INSERT`) chưa tới 2 giây trước lúc xoay vòng, còn các file cũ hơn bị xóa: việc chọn file dựa vào thời gian sửa file.
- Trên một server ít ghi, file binlog có thể không bao giờ đạt 1 GB, nên không bao giờ xoay vòng và không bao giờ bị xóa: đúng trường hợp sách cảnh báo. Cách xử lý: `FLUSH BINARY LOGS` định kỳ (cũng có ích cho sao lưu binlog theo file), hoặc `PURGE BINARY LOGS` có kiểm soát (**Bổ sung ngoài sách**).
- `expire_logs_days` (tr. 261) vẫn tồn tại trên 8.0.42 nhưng deprecated và xung đột với biến mới: nên đọc khuyến nghị ở tr. 261 theo nghĩa "đặt `binlog_expire_logs_seconds` đủ dài".
- `log_status` trả về cùng lúc GTID, vị trí binlog và LSN; đây là dữ liệu XtraBackup đọc ở bước "Selecting LSN and binary log position from p_s.log_status" (tr. 278).
- `LOCK INSTANCE FOR BACKUP` là tương đương trên MySQL 8.0 của `LOCK TABLES FOR BACKUP` thấy trong output tr. 278 (lệnh của Percona Server).

**Dọn dẹp.** Đã trả `binlog_expire_logs_seconds` về 2592000; xóa bảng `shop.ddl_probe` nếu có. Tắt cả hai instance:

```bash
"$BIN/mysqladmin" --no-defaults -h127.0.0.1 -P3320 -uroot shutdown
"$BIN/mysqladmin" --no-defaults -h127.0.0.1 -P3335 -uroot shutdown   # nếu còn chạy
```

Đã xác nhận sau khi chạy: cả cổng 3320 và 3335 đều không còn nhận kết nối.

---

<a id="chua-thuc-thi"></a>

## Chưa thực thi

Các quy trình dưới đây theo sách, chưa chạy vì máy thực hành là Windows và không có các công cụ tương ứng. Lệnh viết theo mô tả của sách; luôn kiểm lại manual của phiên bản bạn dùng (tr. 279).

### A. Percona XtraBackup: backup, prepare, restore, kiểm tra (tr. 278–286)

**Chưa thực thi** — không có XtraBackup (công cụ chủ yếu cho Linux).

1. Backup streaming có nén từ một **replica** có GTID: `xtrabackup --backup --compress --stream=xbstream` chuyển ra một file có ngày trong tên (tr. 280). Ghi lại thời gian, kích thước, tải CPU và I/O trên replica.
2. Trên server phục hồi: `xbstream -x --decompress` vào thư mục trống (tr. 284).
3. `xtrabackup --prepare --target-dir=<thư mục>`; đo thời gian (đây là một phần RTO, tr. 285).
4. Dừng `mysqld` đích, `xtrabackup --copy-back --target-dir=<thư mục>`, sửa owner và quyền sang user `mysql` (tr. 285).
5. Khởi động với `--skip-networking` và một socket riêng (tr. 282), theo dõi error log, chạy `SHOW TABLE STATUS` ở từng database (tr. 286), rồi `mysqlcheck`.
6. Đọc `xtrabackup_binlog_info` để có tọa độ binlog và GTID, rồi phát lại binlog như thí nghiệm 3.
7. **Tiêu chí:** checksum khớp với nguồn tại thời điểm cuối của backup; thời gian từ bước 2 tới bước 5 so với RTO.

### B. mydumper so với mysqldump (tr. 269–272)

**Chưa thực thi** — không có mydumper.

Dump cùng tập dữ liệu thí nghiệm 2 bằng mydumper với 1, 2, 4, 8 luồng, rồi nạp bằng công cụ nạp song song đi kèm; so với `mysqldump` và `mysql` đơn luồng. **Dự đoán:** dump và nạp đều nhanh hơn khi có nhiều bảng; với **một** bảng lớn thì lợi ích phụ thuộc việc công cụ có chia nhỏ bảng theo khoảng khóa hay không.

### C. Snapshot LVM không khóa (tr. 273–277)

**Chưa thực thi** — không có LVM trên Windows.

1. Điều kiện: MySQL 8.0, chỉ InnoDB, GTID bật, `sync_binlog = 1`, `innodb_flush_log_at_trx_commit = 1`; datadir và mọi file InnoDB trên **một** logical volume; volume group còn chỗ trống (tr. 274, 276).
2. Ước lượng dung lượng copy-on-write bằng `iostat` hay `vmstat` và thời gian chép (công thức ở tr. 277).
3. `lvcreate --snapshot` với dung lượng đã tính; mount chỉ đọc; chép ra nơi khác; theo dõi `Snap%` bằng `lvs` trong lúc chép (tr. 275–276).
4. `umount`, `lvremove` (tr. 276).
5. Kiểm tra: khởi động `mysqld` trên bản chép ở máy khác, để InnoDB crash recovery chạy, rồi `CHECK TABLES` (tr. 265).
6. Thí nghiệm âm: cố tình dành quá ít chỗ copy-on-write và chạy ghi nặng để thấy snapshot hỏng giữa chừng (tr. 273). **Dự đoán:** job chép lỗi I/O.

### D. Delayed replica cứu `DROP TABLE` (tr. 268)

**Chưa thực thi** ở chương này — cần topology replication của [Chương 9](../chapter-09.md#delayed-replication). Quy trình: replica với `SOURCE_DELAY = 3600`; chạy `DROP TABLE` trên source; trên replica `STOP REPLICA SQL_THREAD`, rồi `START REPLICA SQL_THREAD UNTIL SQL_BEFORE_GTIDS = '<GTID của DROP>'`; kiểm bảng còn nguyên; trích bảng hay thăng replica. **Dự đoán:** thời gian phục hồi bằng thời gian áp phần relay log trước sự cố, ngắn hơn nhiều so với restore + phát lại binlog cả ngày.
