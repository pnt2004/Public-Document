# Thực hành — Chương 3: Performance Schema

Về ghi chú chương: [chapter-03.md](../chapter-03.md) · Đáp án tự kiểm tra: [answers/chapter-03.md](../answers/chapter-03.md)

> **Nhãn.** Mọi schema, dữ liệu, workload và SQL ở đây là **Ví dụ minh họa do AI xây dựng**. Các khối "Kết quả đã đo" là output thật, có rút gọn ở chỗ ghi "đã rút gọn". "Dự đoán" là điều người viết kỳ vọng **trước** khi chạy.

## Môi trường

| Mục | Giá trị |
| --- | --- |
| Máy | Laptop Windows 11 Home, nhiều tiến trình khác chạy song song (nhiễu cao, **không** phải benchmark production) |
| MySQL | Community Server **8.0.42** (Win64) |
| Instance | Instance riêng, datadir tạm, cổng **3313**, `--no-defaults` (không đọc file cấu hình nào), `--bind-address=127.0.0.1`, `--mysqlx=OFF`; mọi biến khác ở giá trị mặc định (buffer pool 128 MB) |
| Dữ liệu | Tự sinh bằng SQL: `orders` 300.000 hàng, `customers` 20.000 hàng, `regions` 50 hàng, `seq` 1.000 hàng |
| Thời gian chạy | Mỗi thí nghiệm vài giây tới khoảng 2 phút (riêng phần tạo 3.000 bảng ở thí nghiệm 4) |

**Khởi tạo và chạy instance** (đổi `<bin>`, `<datadir>`, `<log>` theo máy của bạn; tuyệt đối không dùng instance đang phục vụ ứng dụng):

```bash
"<bin>/mysqld" --no-defaults --initialize-insecure --datadir="<datadir>" --console
"<bin>/mysqld" --no-defaults --datadir="<datadir>" --port=3313 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=ch03.sock --console > "<log>" 2>&1 &
mysql -h127.0.0.1 -P3313 -uroot -e "SELECT @@port, @@datadir, @@version"
# ... chạy thí nghiệm ...
mysqladmin -h127.0.0.1 -P3313 -uroot shutdown
```

**Quy đổi đơn vị.** Các cột `TIMER_*`, `LOCK_TIME`, `SUM_TIMER_*` tính bằng picosecond: chia 10⁹ ra mili giây, chia 10¹² ra giây (kiểm chứng ở thí nghiệm 2).

## Mục lục

- [Thí nghiệm 1: Instrument và consumer bật mặc định; tùy chọn khởi động](#e1)
- [Thí nghiệm 2: Workload tốt và tệ; digest, index, stage, đọc/ghi](#e2)
- [Thí nghiệm 3: Metadata lock đang chặn với ba session](#e3)
- [Thí nghiệm 4: Bộ nhớ theo thành phần, theo thread, và của chính Performance Schema](#e4)
- [Thí nghiệm 5: Lỗi, biến, prepared statement, stored routine, database mặc định](#e5)
- [Tổng kết và dọn dẹp](#tong-ket)

---

<a id="e1"></a>

## Thí nghiệm 1: Instrument và consumer bật mặc định; tùy chọn khởi động

**Mục tiêu.** Đo trên chính 8.0.42 những gì mục Defaults của sách mô tả ([2.2.7](../chapter-03.md#defaults), tr. 53), và kiểm chứng luật ưu tiên của `performance-schema-instrument` ([2.2.2](../chapter-03.md#enable-instruments), tr. 50).

**Giả thuyết (Dự đoán).**

- Bật: instrument statement, transaction, metadata lock, memory; consumer global, thread, statement, transaction, digest (tr. 53).
- Tắt: phần lớn `wait/*`, `stage/*`.
- 15 consumer (tr. 50), khoảng 110 bảng (tr. 44).
- Với hai tùy chọn khởi động, chuỗi dài hơn thắng bất kể thứ tự (tr. 50); thay đổi bằng `UPDATE` mất sau restart (tr. 49).

**Chuẩn bị.** Instance mới khởi tạo, chưa đổi gì.

**Các bước và lệnh đo** (rút từ script đã chạy):

```sql
SELECT NAME, ENABLED FROM performance_schema.setup_consumers;

SELECT SUBSTRING_INDEX(NAME, '/', 1) AS type, COUNT(*) AS total,
       SUM(ENABLED = 'YES') AS enabled, SUM(TIMED = 'YES') AS timed
FROM performance_schema.setup_instruments GROUP BY type ORDER BY type;

SELECT SUBSTRING_INDEX(NAME, '/', 3) AS class, COUNT(*) AS total, SUM(ENABLED = 'YES') AS enabled
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'wait/%' GROUP BY class ORDER BY class;

SELECT NAME, ENABLED FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/%' AND ENABLED = 'NO';

SELECT NAME FROM performance_schema.setup_instruments
WHERE NAME LIKE 'stage/%' AND ENABLED = 'YES';

SELECT * FROM performance_schema.setup_actors;
SELECT * FROM performance_schema.setup_objects;
SELECT COUNT(*), SUM(ENABLED = 'YES'), SUM(HISTORY = 'YES') FROM performance_schema.setup_threads;

SELECT VARIABLE_NAME, VARIABLE_VALUE FROM performance_schema.global_variables
WHERE VARIABLE_NAME LIKE 'performance_schema%' OR VARIABLE_NAME = 'max_digest_length';

SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'performance_schema';
SELECT table_type, COUNT(*) FROM information_schema.tables WHERE table_schema = 'sys' GROUP BY table_type;
SELECT COUNT(*) FROM information_schema.routines WHERE routine_schema = 'sys';

SELECT NAME, THREAD_ID, PROCESSLIST_ID, THREAD_OS_ID, TYPE
FROM performance_schema.threads WHERE TYPE = 'FOREGROUND' OR NAME = 'thread/sql/main';
SELECT CONNECTION_ID() AS my_processlist_id, PS_CURRENT_THREAD_ID() AS my_thread_id;

SELECT VARIABLE_NAME FROM performance_schema.global_variables
WHERE VARIABLE_NAME IN ('tx_isolation', 'transaction_isolation', 'show_compatibility_56');
SELECT NAME FROM performance_schema.setup_instruments
WHERE NAME IN ('stage/sql/Sending data', 'stage/sql/executing');
```

Phần khởi động lại: tắt instance, chạy lại với các tùy chọn sau (lưu ý dòng `select=ON` đặt **trước** wildcard):

```bash
"<bin>/mysqld" --no-defaults ... \
  "--performance-schema-instrument=statement/sql/select=ON" \
  "--performance-schema-instrument=statement/sql/%=OFF" \
  "--performance-schema-instrument=wait/synch/mutex/innodb/autoinc_mutex=COUNTED" \
  --performance-schema-consumer-events-statements-history-long=ON
```

rồi đọc lại `setup_instruments`, `setup_consumers` và đếm dòng trong bảng digest.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định; đã rút gọn).

Consumer (16 dòng):

| Consumer | ENABLED |
| --- | --- |
| `events_stages_current`, `_history`, `_history_long` | NO, NO, NO |
| `events_statements_cpu` | NO |
| `events_statements_current`, `_history` | YES, YES |
| `events_statements_history_long` | **NO** |
| `events_transactions_current`, `_history` | YES, YES |
| `events_transactions_history_long` | NO |
| `events_waits_current`, `_history`, `_history_long` | NO, NO, NO |
| `global_instrumentation`, `thread_instrumentation`, `statements_digest` | YES, YES, YES |

Instrument theo loại:

| Loại | Tổng | Bật | Timed |
| --- | --- | --- | --- |
| error | 1 | 1 | NULL |
| idle | 1 | 1 | 1 |
| memory | 504 | 504 | NULL |
| stage | 132 | 16 | 16 |
| statement | 213 | 198 | 198 |
| transaction | 1 | 1 | 1 |
| wait | 368 | 53 | 53 |

Chi tiết: `statement/sql` 159/159, `statement/com` 33/33, `statement/abstract` 4/4, `statement/scheduler` 1/1, `statement/sp` **1/16** (chỉ `stmt`; 15 cái tắt gồm `hpush_jump`, `hpop`, `hreturn`, `jump`, `jump_if_not`, `cpush`, `copen`, `cfetch`, `cclose`, `cpop`, `set`, `set_trigger_field`, `freturn`, `error`, `set_case_expr`). `wait/io/file` 50/50, `wait/io/table` 1/1, `wait/io/socket` 0/3, `wait/lock/metadata` 1/1, `wait/lock/table` 1/1, `wait/synch/cond` 0/48, `wait/synch/mutex` 0/219, `wait/synch/rwlock` 0/28, `wait/synch/sxlock` 0/15, `wait/synch/prlock` 0/2. 16 stage bật gồm 12 `stage/innodb/*` (alter table, alter tablespace, buffer pool load, clone) và 4 `stage/sql/*` ("copy to tmp table", ba "Applying batch of row changes"). Có 170/1.220 instrument có `DOCUMENTATION`.

Các bảng setup và biến:

- `setup_actors`: một dòng `%`, `%`, `%`, `YES`, `YES`.
- `setup_objects`: 20 dòng; mỗi loại `EVENT`, `FUNCTION`, `PROCEDURE`, `TABLE`, `TRIGGER` có `mysql`, `performance_schema`, `information_schema` là `NO`, `%` là `YES`.
- `setup_threads`: 55 dòng, đều `ENABLED = YES`, `HISTORY = YES`.
- `_history_size` = 10; `_history_long_size` = 10000; `performance_schema_digests_size` = 10000; `max_sql_text_length` = `max_digest_length` = 1024; `max_memory_classes` = 450; `max_statement_classes` = 219; 18 biến bằng -1.
- `performance_schema` có **111** bảng. sys có 100 view, 1 bảng thường, 48 routine.
- 35 thread (3 tiền cảnh, 32 nền). `thread/sql/main` có `THREAD_ID` 1, `PROCESSLIST_ID` NULL; `thread/sql/event_scheduler` có `THREAD_ID` 40, `PROCESSLIST_ID` 5. Kết nối thử: `CONNECTION_ID()` = 9, `PS_CURRENT_THREAD_ID()` = 45.
- Biến: chỉ có `transaction_isolation`; **không** có `tx_isolation`, `show_compatibility_56`. Instrument: có `stage/sql/executing`, **không** có `stage/sql/Sending data`.
- `sys.memory_global_total` lúc này: 431,26 MiB.

Sau khi khởi động lại với tùy chọn:

| Instrument / consumer | ENABLED | TIMED |
| --- | --- | --- |
| `statement/sql/select` | YES | YES |
| `statement/sql/insert`, `statement/sql/update` | NO | NO |
| Tổng `statement/sql/*` bật | 1 / 159 | |
| `wait/synch/mutex/innodb/autoinc_mutex` | YES | **NO** |
| `stage/sql/executing`, `statement/sp/hpush_jump` (trước đó đã bật bằng `UPDATE`) | NO | NO |
| `events_statements_history_long` | YES | |
| Số dòng trong bảng digest ngay sau restart | 6 | |

**Diễn giải và đối chiếu với sách.**

- Hướng chung khớp tr. 53: statement, transaction, MDL, memory bật; phần lớn wait và stage tắt. Sách không nhắc rằng `wait/io/file/*`, `wait/io/table/sql/handler`, `wait/lock/table/sql/handler` cũng bật sẵn, và stage báo tiến độ của InnoDB cũng bật.
- Consumer `_history_long` mặc định tắt, nên các truy vấn mẫu của sách trên `events_statements_history_long`, `events_stages_history_long` cần bật trước.
- Listing tr. 48 (`statement/sql/select` = NO) không phải mặc định của 8.0.42.
- 16 consumer và 111 bảng so với 15 và 110 của sách: khác do phiên bản (8.0.25 so với 8.0.42).
- Luật "chuỗi dài thắng bất kể thứ tự" (tr. 50) đúng; `COUNTED` cho `TIMED = NO`; thay đổi bằng `UPDATE` không sống qua restart (tr. 49); dữ liệu digest cũng mất sau restart.
- `THREAD_ID` khác `PROCESSLIST_ID` (tr. 47) thấy ngay ở kết nối thử (45 so với 9).

**Dọn dẹp.** Không cần; restart không kèm tùy chọn để trở về mặc định.

---

<a id="e2"></a>

## Thí nghiệm 2: Workload tốt và tệ; digest, index, stage, đọc/ghi

**Mục tiêu.** Dùng các công cụ ở [2.3.1](../chapter-03.md#examining-statements) và [2.3.2](../chapter-03.md#read-write) để tìm câu tốn nhất theo digest, câu không dùng index, câu tạo bảng tạm; kiểm tra đơn vị thời gian; xem stage; đánh giá đọc/ghi.

**Giả thuyết (Dự đoán).**

1. Các câu quét toàn bảng 300.000 hàng đứng đầu theo tổng thời gian, dù chạy ít lần hơn câu tra theo khóa.
2. Mọi câu quét toàn bảng có `NO_INDEX_USED > 0` và xuất hiện trong `sys.statements_with_full_table_scans`.
3. Nếu `TIMER_WAIT` tính bằng picosecond, `SELECT SLEEP(1)` cho khoảng 10¹².
4. Trên 8.0.42, stage chiếm thời gian của câu tệ là `executing` (không phải "Sending data").
5. `LOCK_TIME` nhỏ hơn nhiều so với `TIMER_WAIT`.

**Chuẩn bị: schema và dữ liệu.**

```sql
CREATE DATABASE IF NOT EXISTS hpm3;
USE hpm3;
CREATE TABLE seq (n INT PRIMARY KEY);
SET SESSION cte_max_recursion_depth = 1000;
INSERT INTO seq
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM s WHERE n < 1000)
SELECT n FROM s;

CREATE TABLE regions (
  country CHAR(2) NOT NULL,      -- cố ý không có index
  region  VARCHAR(20) NOT NULL
) ENGINE=InnoDB;
INSERT INTO regions
SELECT CONCAT(CHAR(65 + (n-1) DIV 26), CHAR(65 + (n-1) MOD 26)),
       ELT(1 + n MOD 5, 'APAC', 'EMEA', 'NA', 'LATAM', 'ANZ')
FROM seq WHERE n <= 50;

CREATE TABLE customers (
  id INT PRIMARY KEY,
  name VARCHAR(50) NOT NULL,
  country CHAR(2) NOT NULL,      -- không index
  segment VARCHAR(10) NOT NULL
) ENGINE=InnoDB;
INSERT INTO customers
SELECT n, CONCAT('cust', n),
       CONCAT(CHAR(65 + ((n*7) MOD 50) DIV 26), CHAR(65 + ((n*7) MOD 50) MOD 26)),
       ELT(1 + n MOD 3, 'retail', 'sme', 'corp')
FROM (SELECT (a.n - 1) * 1000 + b.n AS n FROM seq a JOIN seq b WHERE a.n <= 20) t;

CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer_id INT NOT NULL,
  status VARCHAR(10) NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  created_at DATETIME NOT NULL,
  note VARCHAR(100) NOT NULL,
  KEY idx_customer (customer_id)
) ENGINE=InnoDB;
INSERT INTO orders
SELECT n,
       1 + (n * 7919) MOD 20000,
       ELT(1 + n MOD 4, 'new', 'paid', 'shipped', 'cancelled'),
       ROUND(((n * 37) MOD 100000) / 100, 2),
       '2024-01-01' + INTERVAL (n MOD 525600) MINUTE,
       CONCAT('note-', MD5(n))
FROM (SELECT (a.n - 1) * 1000 + b.n AS n FROM seq a JOIN seq b WHERE a.n <= 300) t;
ANALYZE TABLE regions, customers, orders;
```

Kế hoạch thực thi của các mẫu truy vấn (`EXPLAIN FORMAT=TREE`, đã rút gọn): tra theo khóa chính là "Rows fetched before execution"; tra theo `customer_id` là "Index lookup on orders using idx_customer"; lọc theo `amount`, `created_at`, `note` là "Table scan on orders" (ước lượng 329.588 hàng); `GROUP BY status ORDER BY c DESC` là "Aggregate using temporary table" rồi "Sort"; join `customers` với `regions` theo `country` là "Inner hash join" với hai "Table scan".

**Workload** (sinh bằng Bash rồi chạy trong **một** kết nối; 489 câu):

```bash
{
echo "USE hpm3;"
for i in $(seq 1 200); do echo "SELECT * FROM orders WHERE id = $(( (i*1543) % 300000 + 1 ));"; done
for i in $(seq 1 200); do echo "SELECT COUNT(*), SUM(amount) FROM orders WHERE customer_id = $(( (i*97) % 20000 + 1 ));"; done
for i in $(seq 1 10); do echo "SELECT COUNT(*) FROM orders WHERE amount BETWEEN $((i*50)) AND $((i*50+20));"; done
for i in $(seq 1 10); do echo "SELECT status, COUNT(*) AS c FROM orders WHERE created_at >= '2024-0$(( i % 9 + 1 ))-01' GROUP BY status ORDER BY c DESC;"; done
for i in $(seq 1 10); do echo "SELECT r.region, COUNT(*) FROM customers c JOIN regions r ON r.country = c.country WHERE c.segment = '$( [ $((i%2)) -eq 0 ] && echo retail || echo corp )' GROUP BY r.region;"; done
for i in $(seq 1 5); do echo "SELECT id, amount FROM orders WHERE note LIKE '%ab$i%' ORDER BY amount DESC LIMIT 10;"; done
for i in $(seq 1 20); do echo "INSERT INTO orders VALUES ($((300000+i)), $i, 'new', 10.00, NOW(), 'workload');"; done
for i in $(seq 1 20); do echo "UPDATE orders SET status = 'paid' WHERE id = $(( i*101 ));"; done
for i in $(seq 1 3); do echo "UPDATE orders SET note = CONCAT(note, '') WHERE amount = $((i*100)).99;"; done
for i in $(seq 1 10); do echo "DELETE FROM orders WHERE id = $((300000+i));"; done
echo "SELECT SLEEP(1);"
} > e2_workload.sql
```

**Các bước và lệnh đo.**

Bước 1, chuẩn bị đo (session quản trị):

```sql
UPDATE performance_schema.setup_consumers SET ENABLED = 'YES'
 WHERE NAME IN ('events_statements_history_long', 'events_stages_current',
                'events_stages_history', 'events_stages_history_long');
UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES' WHERE NAME LIKE 'stage/%';
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
TRUNCATE TABLE performance_schema.events_statements_history_long;
TRUNCATE TABLE performance_schema.events_stages_history_long;
CREATE TABLE hpm3.handler_before AS
SELECT VARIABLE_NAME, CAST(VARIABLE_VALUE AS UNSIGNED) AS v
FROM performance_schema.global_status WHERE VARIABLE_NAME LIKE 'Handler%';
```

Bước 2: `mysql -h127.0.0.1 -P3313 -uroot < e2_workload.sql` (khoảng 7,7 s).

Bước 3, phân tích:

```sql
-- A. top digest theo tổng thời gian
SELECT LEFT(DIGEST_TEXT, 70) AS digest_text, COUNT_STAR AS cnt,
       ROUND(SUM_TIMER_WAIT/1e12, 3) AS total_s, ROUND(AVG_TIMER_WAIT/1e9, 2) AS avg_ms,
       SUM_ROWS_EXAMINED AS rows_exam, SUM_ROWS_SENT AS rows_sent, SUM_ROWS_AFFECTED AS rows_aff,
       SUM_NO_INDEX_USED AS no_idx, SUM_CREATED_TMP_TABLES AS tmp, SUM_CREATED_TMP_DISK_TABLES AS tmp_disk,
       SUM_SELECT_FULL_JOIN AS full_join, SUM_SELECT_SCAN AS sel_scan,
       SUM_SORT_ROWS AS sort_rows, SUM_SORT_SCAN AS sort_scan
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'hpm3' ORDER BY SUM_TIMER_WAIT DESC LIMIT 12;

-- B, C. sys
SELECT LEFT(query, 60) AS query, exec_count, total_latency, avg_latency,
       rows_examined_avg, full_scan, tmp_tables, rows_sorted
FROM sys.statement_analysis WHERE db = 'hpm3' LIMIT 6;
SELECT LEFT(query, 60) AS query, exec_count, no_index_used_count, no_good_index_used_count,
       no_index_used_pct, rows_examined_avg
FROM sys.statements_with_full_table_scans WHERE db = 'hpm3';

-- D. truy vấn kiểu sách trên history_long
SELECT THREAD_ID, LEFT(SQL_TEXT, 70) AS sql_text, ROWS_SENT, ROWS_EXAMINED,
       CREATED_TMP_TABLES, NO_INDEX_USED
FROM performance_schema.events_statements_history_long
WHERE CURRENT_SCHEMA = 'hpm3' AND (NO_INDEX_USED > 0 OR NO_GOOD_INDEX_USED > 0)
ORDER BY EVENT_ID LIMIT 4;

-- E. đơn vị thời gian
SELECT SQL_TEXT, TIMER_WAIT, ROUND(TIMER_WAIT/1e12, 4) AS seconds, LOCK_TIME
FROM performance_schema.events_statements_history_long WHERE SQL_TEXT LIKE 'SELECT SLEEP%';
SELECT COUNT(*) FROM performance_schema.events_statements_history_long
WHERE CURRENT_SCHEMA = 'hpm3' AND TIMER_WAIT > 5000000000;
SELECT COUNT(*) FROM performance_schema.events_statements_history_long
WHERE CURRENT_SCHEMA = 'hpm3' AND TIMER_WAIT > 5000000000000;

-- F. đọc so với ghi
SELECT EVENT_NAME, COUNT(EVENT_NAME) AS n, ROUND(SUM(TIMER_WAIT)/1e9, 1) AS total_ms,
       ROUND(SUM(LOCK_TIME)/1e9, 3) AS lock_ms
FROM performance_schema.events_statements_history_long
WHERE CURRENT_SCHEMA = 'hpm3' GROUP BY EVENT_NAME ORDER BY n DESC;
SELECT b.VARIABLE_NAME, CAST(g.VARIABLE_VALUE AS UNSIGNED) - b.v AS delta
FROM hpm3.handler_before b JOIN performance_schema.global_status g USING (VARIABLE_NAME)
WHERE CAST(g.VARIABLE_VALUE AS UNSIGNED) - b.v > 0 ORDER BY delta DESC;

-- G. stage (join cả THREAD_ID lẫn EVENT_ID)
SELECT sg.EVENT_NAME, COUNT(*) AS n, ROUND(SUM(sg.TIMER_WAIT)/1e9, 1) AS total_ms
FROM performance_schema.events_stages_history_long sg
JOIN performance_schema.events_statements_history_long st
  ON sg.NESTING_EVENT_ID = st.EVENT_ID AND sg.THREAD_ID = st.THREAD_ID
WHERE st.CURRENT_SCHEMA = 'hpm3' GROUP BY sg.EVENT_NAME ORDER BY total_ms DESC LIMIT 8;
```

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định, dữ liệu tổng hợp 300.000 hàng; đã rút gọn).

A. Top digest:

| digest_text (rút gọn) | cnt | total_s | avg_ms | rows_exam | rows_sent | rows_aff | no_idx | tmp | full_join | sel_scan | sort_rows | sort_scan |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `SELECT STATUS , COUNT ( * ) AS c FROM orders WHERE created_at >= …` | 10 | 2,139 | 213,86 | 3.000.032 | 32 | 0 | 10 | 10 | 0 | 10 | 32 | 10 |
| `UPDATE orders SET note = CONCAT ( note , ? ) WHERE amount = …` | 3 | 1,517 | 505,52 | 900.060 | 0 | 0 | **0** | 0 | 0 | **0** | 0 | 0 |
| `SELECT id , amount FROM orders WHERE note LIKE ? ORDER BY amo…` | 5 | 1,394 | 278,8 | 1.500.050 | 50 | 0 | 5 | 0 | 0 | 5 | 50 | 5 |
| `SELECT COUNT ( * ) FROM orders WHERE amount BETWEEN ? AND ?` | 10 | 1,116 | 111,58 | 3.000.000 | 10 | 0 | 10 | 0 | 0 | 10 | 0 | 0 |
| `SELECT SLEEP (?)` | 1 | 1,01 | 1009,66 | 1 | 1 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| `SELECT r . region , COUNT ( * ) FROM customers c JOIN regions …` | 10 | 0,178 | 17,76 | 200.500 | 50 | 0 | 10 | 10 | 10 | 10 | 0 | 0 |
| `SELECT COUNT ( * ) , SUM ( amount ) FROM orders WHERE customer_id …` | 200 | 0,074 | 0,37 | 3.000 | 200 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| `UPDATE orders SET STATUS = ? WHERE id = ?` | 20 | 0,048 | 2,41 | 20 | 0 | 15 | 0 | 0 | 0 | 0 | 0 | 0 |
| `SELECT * FROM orders WHERE id = ?` | 200 | 0,042 | 0,21 | 200 | 200 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| `INSERT INTO orders VALUES ( ?, ... , NOW ( ) , ? )` | 20 | 0,04 | 1,99 | 0 | 0 | 20 | 0 | 0 | 0 | 0 | 0 | 0 |
| `DELETE FROM orders WHERE id = ?` | 10 | 0,022 | 2,16 | 10 | 0 | 10 | 0 | 0 | 0 | 0 | 0 | 0 |

`CREATED_TMP_DISK_TABLES` bằng 0 ở mọi dòng.

B. `sys.statement_analysis`: cùng thứ tự (2.14 s, 1.52 s, 1.39 s, 1.12 s, 1.01 s, 177.62 ms); cột `full_scan` có `*` ở bốn câu `SELECT` tệ, **trống** ở câu `UPDATE` quét toàn bảng.

C. `sys.statements_with_full_table_scans`: đúng bốn dòng (câu `GROUP BY status`, câu join `regions`, câu `LIKE`, câu `BETWEEN`), mỗi dòng `no_index_used_pct` = 100. **Không** có câu `UPDATE … WHERE amount = ?`.

D. Truy vấn kiểu sách trên `history_long`: 35 câu `statement/sql/select` có `NO_INDEX_USED > 0`; ví dụ dòng đầu: thread 58, `SELECT COUNT(*) FROM orders WHERE amount BETWEEN 50 AND 70`, `ROWS_SENT` 1, `ROWS_EXAMINED` 300.000.

E. Đơn vị:

| Kiểm tra | Kết quả |
| --- | --- |
| `SELECT SLEEP(1)`: `TIMER_WAIT` | 1.009.658.600.000 (= 1,0097 s nếu là picosecond); `LOCK_TIME` 0 |
| Số câu có `TIMER_WAIT > 5000000000` | 42 |
| Số câu có `TIMER_WAIT > 5000000000000` | 0 |

F. Đọc/ghi:

| EVENT_NAME | n | total_ms (`TIMER_WAIT`) | lock_ms (`LOCK_TIME`) |
| --- | --- | --- | --- |
| `statement/sql/select` | 436 | 5.951 | 0,784 |
| `statement/sql/update` | 23 | 1.564,7 | 0,042 |
| `statement/sql/insert` | 20 | 39,9 | 0,052 |
| `statement/sql/delete` | 10 | 21,6 | 0,021 |

Mức tăng `Handler_*` trong lúc chạy workload: `Handler_read_rnd_next` 8.604.027; `Handler_read_key` 1.505.887; `Handler_update` 1.505.204; `Handler_read_next` 3.107; `Handler_external_lock` 1.370; `Handler_commit` 639; `Handler_write` 126; `Handler_prepare` 94; `Handler_read_first` 49; `Handler_read_rnd` 25; `Handler_delete` 10.

G. Stage: một lần chạy câu `GROUP BY status` (`created_at >= '2024-02-01'`): `starting` 0,12 ms, `Opening tables` 0,06 ms, `Creating tmp table` 0,07 ms, `executing` **279,02 ms**, `waiting for handler commit` 0,22 ms, `freeing items` 0,29 ms, các stage khác ≈ 0. Gộp toàn workload:

| Stage | n | total_ms |
| --- | --- | --- |
| `stage/sql/executing` | 436 | 4.833 |
| `stage/sql/updating` | 33 | 1.522 |
| `stage/sql/User sleep` | 1 | 1.009,2 |
| `stage/sql/waiting for handler commit` | 488 | 93,3 |
| `stage/sql/freeing items` | 489 | 44,2 |
| `stage/sql/starting` | 978 | 25,8 |
| `stage/sql/statistics` | 435 | 16,9 |
| `stage/sql/Opening tables` | 489 | 12,9 |

**Diễn giải và đối chiếu với sách.**

- **Giả thuyết 1 đúng.** Sắp theo tổng thời gian, câu tệ chạy 3–10 lần đứng trên câu tốt chạy 200 lần. Tỷ lệ rows examined trên rows sent của câu tệ cỡ 10⁴–10⁵ (ví dụ 3.000.000 trên 10), của câu tốt là 1 (khóa chính) hay 15 (`idx_customer`). Đây là cách dùng bảng digest mà tr. 44–45 và 56–57 gợi ý.
- **Giả thuyết 2 sai một phần.** Câu `UPDATE` quét 300.000 hàng mỗi lần **không** bật `NO_INDEX_USED` hay `SELECT_SCAN`, nên sys bỏ sót nó. Điều kiện `ROWS_EXAMINED > ROWS_AFFECTED` trong điều kiện dài ở tr. 56 bắt được nó (900.060 so với 0). `ROWS_AFFECTED` = 0 vì `CONCAT(note, '')` không đổi giá trị nên InnoDB không coi là hàng bị thay đổi.
- Join không index trên bảng 50 hàng được MySQL 8.0 chạy bằng hash join nhưng vẫn đếm `SELECT_FULL_JOIN` = 1 mỗi lần. Theo Bảng 3-4 (tr. 55) đây là chỉ số mức cao, "trừ khi bảng rất nhỏ"; ở đây `regions` rất nhỏ, nên cần đọc kèm kích thước bảng.
- `CREATED_TMP_TABLES` = 1 cho mỗi câu `GROUP BY`, `CREATED_TMP_DISK_TABLES` = 0: bảng tạm nằm trong bộ nhớ (mức "Trung bình" ở Bảng 3-4).
- **Giả thuyết 3 đúng**: đơn vị picosecond. Điều kiện `TIMER_WAIT > 5000000000` của sách (tr. 56) lọc câu dài hơn 5 ms (42 câu), không phải 5 s (0 câu).
- **Giả thuyết 4 đúng**: không có stage "Sending data"; thời gian của câu tệ nằm ở `executing`. Bảng 3-7 (tr. 61) cần đọc lại với tên stage của 8.0.42. Stage `updating` gom thời gian của `UPDATE` và `DELETE` (phần việc trong storage engine, như tr. 62 nói về `stage/sql/update`).
- **Giả thuyết 5 đúng**: tổng `LOCK_TIME` (0,784 ms cho `SELECT`) không phản ánh latency (5.951 ms), và thứ hạng còn bị đảo: theo `LOCK_TIME`, `insert` (0,052 ms) đứng trên `update` (0,042 ms); theo `TIMER_WAIT`, `update` (1.564,7 ms) gấp gần 40 lần `insert` (39,9 ms). `LOCK_TIME` cùng đơn vị picosecond với `TIMER_WAIT` (mỗi câu có `LOCK_TIME` ≤ `TIMER_WAIT`; nếu đơn vị là micro giây thì tổng chờ khóa của 436 câu `SELECT` sẽ là hàng trăm giây, vô lý so với 7,7 s chạy cả workload), nên `LOCK_TIME/1000000` ở tr. 62 ra micro giây chứ không phải mili giây. Cách xếp hạng latency theo `LOCK_TIME` ở tr. 62 không dùng được cho mục đích đó.
- **Phát hiện thêm.** `Handler_update` tăng 1.505.204 dù chỉ có 15 hàng thật sự được `UPDATE` thay đổi. Phép tính kiểm tra (**Bổ sung ngoài sách**): `created_at` của 300.000 hàng trải từ 01/01 tới khoảng 27/07/2024, nên số hàng đầu vào của 10 câu `GROUP BY` lần lượt là 255.360, 213.600, 168.960, 125.760, 81.120, 37.920, 0, 0, 300.000, 255.360, tổng 1.438.080; cộng khoảng 66.670 hàng đầu vào của 10 câu join `GROUP BY region` thì xấp xỉ 1.504.750, gần bằng mức tăng `Handler_update` và `Handler_read_key`. Vậy mỗi hàng đi qua bảng tạm gộp nhóm tạo một lần tra (`read_key`) và một lần cập nhật (`update`) trên bảng tạm. Số "rows written" tính từ `Handler_*` như ở tr. 63 vì thế gồm cả việc ghi vào bảng tạm nội bộ.

**Dọn dẹp và khôi phục.** Thí nghiệm 3–5 dùng tiếp dữ liệu này. Cuối buổi: `DROP DATABASE hpm3;` và restart để trả consumer, instrument về mặc định.

---

<a id="e3"></a>

## Thí nghiệm 3: Metadata lock đang chặn với ba session

**Mục tiêu.** Tái hiện [2.3.3](../chapter-03.md#metadata-locks) (tr. 63–64): tìm ai giữ, ai chờ metadata lock; thấy rõ `THREAD_ID` khác `PROCESSLIST_ID` khi `KILL`.

**Giả thuyết (Dự đoán).** Session giữ transaction mở có `SHARED_READ` `GRANTED`; `ALTER TABLE` có `EXCLUSIVE` `PENDING`. Một `SELECT` mới vào cùng bảng **vẫn chạy được** vì shared lock không xung đột với shared lock (tr. 63).

**Chuẩn bị.** Dữ liệu của thí nghiệm 2 (`hpm3.customers`). Instrument `wait/lock/metadata/sql/mdl` đã bật mặc định.

**Các bước.** Chạy ba client nền gần như cùng lúc, rồi một client quan sát:

```bash
# Session A: giữ transaction mở 30 s sau khi đọc bảng
mysql -h127.0.0.1 -P3313 -uroot hpm3 -e "START TRANSACTION; SELECT COUNT(*) FROM customers WHERE id < 10; SELECT SLEEP(30); COMMIT;" &
# Session B: sau 1 s, chạy ALTER
mysql -h127.0.0.1 -P3313 -uroot hpm3 -e "DO SLEEP(1); SET SESSION lock_wait_timeout = 60; ALTER TABLE customers ADD COLUMN extra INT; SELECT 'ALTER done';" &
# Session C: sau 2 s, một SELECT theo khóa chính
mysql -h127.0.0.1 -P3313 -uroot hpm3 -e "DO SLEEP(2); SELECT name FROM customers WHERE id = 1;" &
```

Client quan sát (bắt đầu bằng `DO SLEEP(4)`):

```sql
SELECT ID, USER, DB, COMMAND, TIME, STATE, LEFT(INFO, 60) AS info
FROM information_schema.PROCESSLIST WHERE DB = 'hpm3' ORDER BY ID;

SELECT t.PROCESSLIST_ID, ml.OWNER_THREAD_ID, ml.OBJECT_TYPE, ml.OBJECT_NAME,
       ml.LOCK_TYPE, ml.LOCK_DURATION, ml.LOCK_STATUS, ml.SOURCE
FROM performance_schema.metadata_locks ml
JOIN performance_schema.threads t ON ml.OWNER_THREAD_ID = t.THREAD_ID
WHERE ml.OBJECT_SCHEMA = 'hpm3' AND ml.OBJECT_NAME = 'customers'
ORDER BY ml.OWNER_THREAD_ID, ml.LOCK_STATUS;

SELECT object_name, waiting_pid, waiting_lock_type, LEFT(waiting_query, 40) AS waiting_query,
       blocking_pid, blocking_lock_type, sql_kill_blocking_connection
FROM sys.schema_table_lock_waits;

-- câu đã lấy lock của thread đang giữ SHARED_READ
SELECT h.THREAD_ID, h.EVENT_ID, LEFT(h.SQL_TEXT, 60) AS sql_text
FROM performance_schema.events_statements_history h
WHERE h.THREAD_ID = (SELECT OWNER_THREAD_ID FROM performance_schema.metadata_locks
                     WHERE OBJECT_NAME = 'customers' AND LOCK_TYPE = 'SHARED_READ'
                       AND LOCK_STATUS = 'GRANTED' LIMIT 1)
ORDER BY h.EVENT_ID;

SELECT t.PROCESSLIST_ID, tr.THREAD_ID, tr.STATE, ROUND(tr.TIMER_WAIT/1e12, 1) AS open_s,
       tr.ACCESS_MODE, tr.ISOLATION_LEVEL
FROM performance_schema.events_transactions_current tr
JOIN performance_schema.threads t USING (THREAD_ID) WHERE tr.STATE = 'ACTIVE';
```

Cuối cùng thử `KILL 61` (THREAD_ID), rồi lấy `PROCESSLIST_ID` từ `threads` và `KILL` đúng số đó.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định; đã rút gọn).

Process list:

| ID | TIME | STATE | info |
| --- | --- | --- | --- |
| 17 | 4 | User sleep | `SELECT SLEEP(30) AS a_sleep` |
| 18 | 3 | Waiting for table metadata lock | `ALTER TABLE customers ADD COLUMN extra INT` |
| 19 | 2 | Waiting for table metadata lock | `SELECT name AS d_name FROM customers WHERE id = 1` |

`metadata_locks` join `threads`:

| PROCESSLIST_ID | OWNER_THREAD_ID | LOCK_TYPE | LOCK_DURATION | LOCK_STATUS | SOURCE |
| --- | --- | --- | --- | --- | --- |
| 17 | 61 | SHARED_READ | TRANSACTION | GRANTED | sql_parse.cc:6157 |
| 18 | 62 | SHARED_UPGRADABLE | TRANSACTION | GRANTED | sql_parse.cc:6157 |
| 18 | 62 | EXCLUSIVE | TRANSACTION | PENDING | mdl.cc:3763 |
| 19 | 63 | SHARED_READ | TRANSACTION | PENDING | sql_parse.cc:6157 |

`sys.schema_table_lock_waits` (4 dòng): (waiting 18 `EXCLUSIVE`, blocking 17 `SHARED_READ`, `KILL 17`); (waiting 19 `SHARED_READ`, blocking 17, `KILL 17`); (waiting 18, blocking **18** `SHARED_UPGRADABLE`, `KILL 18`); (waiting 19, blocking 18 `SHARED_UPGRADABLE`, `KILL 18`).

`events_statements_history` của thread 61: `select @@version_comment limit 1`, `START TRANSACTION`, `SELECT COUNT(*) AS a_count FROM customers WHERE id < 10`. `events_statements_current` của thread 61 là `SELECT SLEEP(30)`.

`events_transactions_current`: thread 61 (pid 17) `ACTIVE` 4,2 s; thread 62 (pid 18) `ACTIVE` 3,1 s; cả hai `READ WRITE`, `REPEATABLE READ`.

`KILL 61` → `ERROR 1094 (HY000): Unknown thread id: 61`. Lấy `PROCESSLIST_ID` từ `threads` được 17, `KILL 17` thành công. Sau đó: session A báo `Lost connection to MySQL server during query`; session B in `ALTER done`; session C trả `cust1`; cột `extra` đã có; `metadata_locks` của `customers` còn 0 dòng.

**Diễn giải và đối chiếu với sách.**

- Khớp ví dụ tr. 64: kẻ chờ có `EXCLUSIVE` `PENDING`, kẻ giữ có `SHARED_READ` `GRANTED`. Thêm chi tiết: ALTER giữ trước `SHARED_UPGRADABLE`.
- **Giả thuyết về session C sai.** `SELECT` mới bị chặn, vì yêu cầu `EXCLUSIVE` đang chờ được ưu tiên hơn yêu cầu shared mới. Đây là cơ chế khiến một DDL đang chờ làm tắc cả bảng (**Bổ sung ngoài sách** về lý do ưu tiên).
- Kẻ giữ lock **không** hiện câu nào liên quan `customers` trong process list (chỉ `SLEEP`), đúng như sách cảnh báo (tr. 63). `events_statements_history` của thread đó chỉ ra câu đã lấy lock.
- `sys.schema_table_lock_waits` tiện (có sẵn câu `KILL`) nhưng in cả dòng ALTER "chặn chính nó"; cần đọc kỹ trước khi kill.
- `KILL` cần `PROCESSLIST_ID`, đúng cảnh báo tr. 47. Nếu `THREAD_ID` trùng với một `PROCESSLIST_ID` khác đang tồn tại, `KILL` nhầm số sẽ giết nhầm kết nối (**Dự đoán**, không thử).

**Dọn dẹp.** `ALTER TABLE hpm3.customers DROP COLUMN extra;` (hoặc bỏ cả database ở cuối buổi).

---

<a id="e4"></a>

## Thí nghiệm 4: Bộ nhớ theo thành phần, theo thread, và của chính Performance Schema

**Mục tiêu.** Áp dụng [2.3.4](../chapter-03.md#memory-usage) và [2.3.7](../chapter-03.md#pfs-itself); kiểm chứng nhận định "bộ nhớ đã cấp không được trả lại" ([2.1.3](../chapter-03.md#resource-consumption), tr. 45–46, 73).

**Giả thuyết (Dự đoán).**

1. InnoDB (buffer pool 128 MB) là thành phần dùng nhiều bộ nhớ nhất.
2. Một session đặt biến người dùng 50 MB sẽ đứng đầu `sys.memory_by_thread_by_current_bytes`, và bộ nhớ đó được trả khi session ngắt.
3. Bật thêm consumer làm bộ nhớ Performance Schema tăng; tắt và `TRUNCATE` không làm giảm.
4. Tạo nhiều bảng làm các container autoscale tăng; xóa bảng không làm giảm.

**Các bước và lệnh đo.**

Phần 1: một session nền chạy `SET @big = REPEAT('x', 50*1024*1024); SET @small = 'foo'; DO SLEEP(12);`, đồng thời client quan sát chạy:

```sql
SELECT SUBSTRING_INDEX(EVENT_NAME, '/', 2) AS component,
       ROUND(SUM(CURRENT_NUMBER_OF_BYTES_USED)/1024/1024, 2) AS current_mb,
       ROUND(SUM(HIGH_NUMBER_OF_BYTES_USED)/1024/1024, 2) AS high_mb
FROM performance_schema.memory_summary_global_by_event_name
GROUP BY component ORDER BY SUM(CURRENT_NUMBER_OF_BYTES_USED) DESC LIMIT 8;

SELECT EVENT_NAME, ROUND(CURRENT_NUMBER_OF_BYTES_USED/1024/1024, 2) AS current_mb,
       ROUND(HIGH_NUMBER_OF_BYTES_USED/1024/1024, 2) AS high_mb
FROM performance_schema.memory_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'memory/innodb/%' ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC LIMIT 6;

SELECT * FROM sys.memory_global_total;
SELECT thread_id, user, current_allocated, total_allocated
FROM sys.memory_by_thread_by_current_bytes LIMIT 6;

SELECT t.PROCESSLIST_ID, m.THREAD_ID, m.EVENT_NAME,
       ROUND(m.CURRENT_NUMBER_OF_BYTES_USED/1024/1024, 2) AS current_mb
FROM performance_schema.memory_summary_by_thread_by_event_name m
JOIN performance_schema.threads t USING (THREAD_ID)
WHERE t.PROCESSLIST_USER = 'root' AND m.CURRENT_NUMBER_OF_BYTES_USED > 1024*1024
ORDER BY m.CURRENT_NUMBER_OF_BYTES_USED DESC LIMIT 5;

SELECT THREAD_ID, VARIABLE_NAME, LENGTH(VARIABLE_VALUE) AS value_len
FROM performance_schema.user_variables_by_thread;

SELECT SUBSTRING_INDEX(EVENT_NAME, '/', -1) AS event,
       ROUND(CURRENT_NUMBER_OF_BYTES_USED/1024/1024, 2) AS current_mb
FROM performance_schema.memory_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'memory/performance_schema/%'
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC LIMIT 8;
```

và `SHOW ENGINE PERFORMANCE_SCHEMA STATUS`.

Phần 2 (sau khi session 50 MB đã kết thúc): đo tổng `memory/performance_schema/%`, bộ đệm `events_waits_history_long`, và `memory/sql/user_var_entry::value`; bật mọi consumer `events_waits%` và mọi instrument `wait/%`; chạy hai câu quét `orders`; đo lại; tắt consumer và `wait/synch/%`, `TRUNCATE` `events_waits_history_long`, `events_waits_history`, `events_statements_summary_by_digest`; đo lại.

Phần 3: một procedure tạo N bảng InnoDB một cột (và `SELECT COUNT(*)` từ mỗi bảng) hoặc xóa N bảng, dùng prepared statement động:

```sql
DELIMITER //
CREATE PROCEDURE hpm3.make_tables(IN n INT, IN do_drop BOOL)
BEGIN
  DECLARE i INT DEFAULT 1;
  WHILE i <= n DO
    IF do_drop THEN
      SET @s = CONCAT('DROP TABLE hpm3_many.t', i);
      PREPARE st FROM @s; EXECUTE st; DEALLOCATE PREPARE st;
    ELSE
      SET @s = CONCAT('CREATE TABLE hpm3_many.t', i, ' (id INT PRIMARY KEY) ENGINE=InnoDB');
      PREPARE st FROM @s; EXECUTE st; DEALLOCATE PREPARE st;
      SET @s = CONCAT('SELECT COUNT(*) INTO @c FROM hpm3_many.t', i);
      PREPARE st FROM @s; EXECUTE st; DEALLOCATE PREPARE st;
    END IF;
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
```

Lượt 1: tạo 1.500 bảng, đo, xóa 1.500 bảng, đo. Lượt 2: chụp bộ nhớ từng instrument `memory/performance_schema/%` vào một bảng, tạo 3.000 bảng, so từng instrument, xóa 3.000 bảng, đo.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định; đã rút gọn).

Phần 1:

| Thành phần | current_mb | high_mb |
| --- | --- | --- |
| `memory/performance_schema` | **229,59** | 229,59 |
| `memory/innodb` | 193,06 | 195,69 |
| `memory/sql` | 64,22 | 118,86 |
| `memory/mysys` | 8,90 | 17,10 |
| `memory/temptable` | 1,00 | 5,00 |
| `memory/mysqld_openssl` | 0,51 | 2,06 |

Top InnoDB: `buf_buf_pool` 130,82; `ut0link_buf` 24,00; `log_buffer_memory` 16,00; `sync0arr` 6,25; `lock0lock` 4,85; `ut0pool` 4,00 (MB).

`sys.memory_global_total` = 497,29 MiB. `sys.memory_by_thread_by_current_bytes`: thread 68 `root@localhost` 50,10 MiB hiện tại, 100,31 MiB tổng đã cấp; thread 69 `root@localhost` (client quan sát) 8,28 MiB; thread 1 `sql/main` 1,45 MiB; thread 35 `innodb/clone_gtid_thread` 543,63 KiB. Phân rã thread 68: `memory/sql/user_var_entry::value` 50,00 MB. `user_variables_by_thread`: thread 68 có `big` dài 52.428.800 byte và `small` dài 3 byte.

Top Performance Schema: `events_statements_summary_by_digest` 40,28; `events_statements_history_long` 14,42; `events_errors_summary_by_thread_by_error` 12,82; `events_statements_summary_by_thread_by_event_name` 11,55; `events_statements_history_long.sql_text` 9,77; `events_statements_summary_by_digest.digest_text` 9,77; `events_statements_history_long.digest_text` 9,77; `table_handles` 9,06 (MB). `SHOW ENGINE PERFORMANCE_SCHEMA STATUS` trả 248 dòng; vài dòng: `events_waits_current.size` 168, `.count` 1.536; `events_statements_history_long.count` 10.000, `.memory` 15.120.000; dòng cuối `performance_schema.memory` **240.747.592**.

Phần 2:

| Bước | Tổng PFS (MB) | `events_waits_history_long` (MB) | Ghi chú |
| --- | --- | --- | --- |
| Trước (session 50 MB đã ngắt) | 229,59 | 1,60 | `user_var_entry::value` = 0,00 MB |
| Sau khi bật waits và chạy hai câu quét | 229,59 | 1,60 | `events_waits_history_long` có 7.564 dòng |
| Sau khi tắt và `TRUNCATE` | 229,59 | 1,60 | digest vẫn 40,28 MB dù đã `TRUNCATE` |

Phần 3:

| Bước | Tổng PFS (MB) | Số bảng `hpm3_many` thấy trong `objects_summary_global_by_type` |
| --- | --- | --- |
| Trước | 229,59 | 0 |
| Sau tạo 1.500 bảng | 232,36 | 1.500 |
| Sau xóa 1.500 bảng | **232,36** | 0 |
| Sau tạo 3.000 bảng (lượt 2) | 232,94 | |
| Sau xóa 3.000 bảng | **232,94** | |

Instrument thay đổi trong lượt 2: `mutex_instances` 655.360 → 901.120 byte; `rwlock_instances` 720.896 → 1.081.344 byte; `scalable_buffer` 3.344 → 4.016 byte. Lượt 1 không chụp từng instrument nên không biết chính xác phần tăng 2,77 MB nằm ở đâu. Thời gian: lượt 1 khoảng 49 s, lượt 2 khoảng 96 s.

**Diễn giải và đối chiếu với sách.**

- **Giả thuyết 1 sai** trên instance nhỏ cấu hình mặc định: chính Performance Schema (≈ 230 MB) dùng nhiều hơn InnoDB (193 MB). Trên server production với buffer pool hàng chục GB thì tỷ lệ sẽ khác hẳn (**Dự đoán**). Con số của sách ở tr. 72–73 (218.456.400 byte) cùng cỡ với 240.747.592 đo được.
- **Giả thuyết 2 đúng.** Đúng như ví dụ 36 GiB ở tr. 66, view sys cho thấy ngay thread "ngốn" bộ nhớ, và phân rã theo event chỉ ra loại cấp phát. `total_allocated` 100 MiB gấp đôi 50 MiB vì `REPEAT` tạo chuỗi tạm rồi chép vào biến. Bộ nhớ của thread được trả lại khi session kết thúc.
- **Giả thuyết 3 sai một nửa.** Bật thêm consumer **không** làm tăng tổng, vì các bộ đệm lịch sử cố định (10.000 dòng) đã được cấp sẵn lúc khởi động, kể cả khi consumer tắt. Phần "không giảm sau khi tắt và `TRUNCATE`" thì đúng như tr. 45.
- **Giả thuyết 4 đúng.** Container autoscale tăng theo số đối tượng và không giảm sau khi xóa (tr. 45–46). Lượt 2 chỉ tăng 0,58 MB cho gấp đôi số bảng: bộ nhớ đã giữ được **dùng lại**. Đây là lý do tác giả nói đây không phải rò rỉ (tr. 73).

**Dọn dẹp.** `DROP DATABASE hpm3_many; DROP PROCEDURE hpm3.make_tables;` Bộ nhớ Performance Schema chỉ về mức ban đầu sau khi restart.

---

<a id="e5"></a>

## Thí nghiệm 5: Lỗi, biến, prepared statement, stored routine, database mặc định

**Mục tiêu.** Chạy lại các ví dụ ở [2.3.1](../chapter-03.md#prepared-statements) (prepared statement, stored routine), [2.3.5](../chapter-03.md#variables), [2.3.6](../chapter-03.md#errors), [2.3.7](../chapter-03.md#pfs-itself) trên 8.0.42; thêm một kiểm tra về bộ đếm dùng index cho [Chương 7](../chapter-07.md#unused-index).

**Giả thuyết (Dự đoán).**

1. Lỗi được handler bắt sẽ tăng cả `SUM_ERROR_RAISED` lẫn `SUM_ERROR_HANDLED`.
2. Với mặc định 8.0.42, `CALL sp_test(NULL)` chỉ để lại event `statement/sp/stmt`; bật `statement/sp/%` thì ra đủ 3 và 7 event như tr. 60.
3. `prepared_statements_instances` cộng dồn ba lần chạy; `DROP PREPARE` xóa dòng (tr. 59).
4. `tx_isolation` không còn trên 8.0.42.
5. Theo tr. 71, câu chạy khi database mặc định là `performance_schema` sẽ không được ghi.

**Chuẩn bị.**

```sql
TRUNCATE TABLE performance_schema.events_errors_summary_global_by_error;
TRUNCATE TABLE performance_schema.events_errors_summary_by_account_by_error;
USE hpm3;
CREATE TABLE t1 (f1 INT NOT NULL);
DELIMITER //
CREATE PROCEDURE sp_test(val INT)
BEGIN
  DECLARE CONTINUE HANDLER FOR 1364, 1048, 1366
  BEGIN
    INSERT IGNORE INTO t1 VALUES ('Some string');
    GET STACKED DIAGNOSTICS CONDITION 1 @stacked_state = RETURNED_SQLSTATE;
    GET STACKED DIAGNOSTICS CONDITION 1 @stacked_msg = MESSAGE_TEXT;
  END;
  INSERT INTO t1 VALUES (val);
END//
DELIMITER ;
```

(Cùng logic với procedure ở tr. 59–60, nhưng tên không đặt trong nháy đơn.)

**Các bước và lệnh đo.**

Bước 1, gây lỗi (chạy bằng `mysql --force` để không dừng ở lỗi đầu): ba câu `SELEC n` (lỗi cú pháp), bốn câu đọc `no_such_table`, năm câu `INSERT` trùng khóa chính vào `orders`, hai câu `SELECT n/0`.

Bước 2, stored routine và prepared statement (trong một session; `@tid = PS_CURRENT_THREAD_ID()`):

```sql
CALL sp_test(NULL);   -- với mặc định
SELECT EVENT_ID, EVENT_NAME, LEFT(SQL_TEXT, 45), MYSQL_ERRNO, ERRORS, WARNINGS
FROM performance_schema.events_statements_history_long
WHERE THREAD_ID = @tid AND EVENT_NAME LIKE 'statement/sp/%' ORDER BY EVENT_ID;

UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/sp/%';
-- đặt mốc @mark = MAX(EVENT_ID) của thread, CALL sp_test(1), đọc event sau mốc;
-- đặt mốc mới, CALL sp_test(NULL), đọc event sau mốc

PREPARE stmt FROM 'SELECT COUNT(*) FROM hpm3.orders WHERE created_at > ?';
SET @hd = '2024-03-01'; EXECUTE stmt USING @hd;
SET @hd = '2024-05-01'; EXECUTE stmt USING @hd;
SET @hd = '2024-07-01'; EXECUTE stmt USING @hd;
SELECT STATEMENT_NAME, OWNER_THREAD_ID, COUNT_REPREPARE, COUNT_EXECUTE, SUM_TIMER_EXECUTE,
       ROUND(SUM_TIMER_EXECUTE/COUNT_EXECUTE/1e9, 1) AS avg_ms, SUM_ROWS_EXAMINED, SUM_NO_INDEX_USED
FROM performance_schema.prepared_statements_instances;
DROP PREPARE stmt;
SELECT COUNT(*) FROM performance_schema.prepared_statements_instances;
```

Bước 3, đọc bảng lỗi:

```sql
SELECT ERROR_NUMBER, ERROR_NAME, SQL_STATE, SUM_ERROR_RAISED, SUM_ERROR_HANDLED
FROM performance_schema.events_errors_summary_global_by_error
WHERE SUM_ERROR_RAISED > 0 OR SUM_ERROR_HANDLED > 0
ORDER BY SUM_ERROR_RAISED DESC;
SELECT USER, HOST, ERROR_NUMBER, ERROR_NAME, SUM_ERROR_RAISED, SUM_ERROR_HANDLED
FROM performance_schema.events_errors_summary_by_account_by_error
WHERE SUM_ERROR_RAISED > 2 AND USER IS NOT NULL ORDER BY SUM_ERROR_RAISED DESC;
SELECT LEFT(query, 55) AS query, exec_count, errors, error_pct, warnings, warning_pct
FROM sys.statements_with_errors_or_warnings
WHERE db = 'hpm3' OR db IS NULL ORDER BY errors DESC, warnings DESC LIMIT 8;
```

Bước 4, biến: một session nền đặt `transaction_isolation = 'SERIALIZABLE'`, `sort_buffer_size = 4194304`, `@foo = 'bar'`, chạy hai lần `INSERT INTO t1 SELECT n FROM seq` rồi ngủ 10 s. Client quan sát chạy `SET GLOBAL long_query_time = 2` rồi:

```sql
SELECT t.PROCESSLIST_ID, v.THREAD_ID, v.VARIABLE_NAME, v.VARIABLE_VALUE
FROM performance_schema.variables_by_thread v JOIN performance_schema.threads t USING (THREAD_ID)
WHERE v.VARIABLE_NAME = 'transaction_isolation';
SELECT COUNT(*) FROM performance_schema.variables_by_thread WHERE VARIABLE_NAME = 'tx_isolation';

SELECT vt2.THREAD_ID AS TID, vt2.VARIABLE_NAME, vt1.VARIABLE_VALUE AS MY_VALUE, vt2.VARIABLE_VALUE AS OTHER_VALUE
FROM performance_schema.variables_by_thread vt1
JOIN performance_schema.threads t USING (THREAD_ID)
JOIN performance_schema.variables_by_thread vt2 USING (VARIABLE_NAME)
WHERE vt1.VARIABLE_VALUE != vt2.VARIABLE_VALUE AND t.PROCESSLIST_ID = @@pseudo_thread_id
  AND vt2.VARIABLE_NAME NOT IN ('timestamp', 'pseudo_thread_id');

SELECT s.THREAD_ID, t.PROCESSLIST_ID, s.VARIABLE_NAME, s.VARIABLE_VALUE
FROM performance_schema.status_by_thread s JOIN performance_schema.threads t USING (THREAD_ID)
WHERE s.VARIABLE_NAME = 'Handler_write' ORDER BY CAST(s.VARIABLE_VALUE AS UNSIGNED) DESC LIMIT 4;
SELECT THREAD_ID, VARIABLE_NAME, VARIABLE_VALUE FROM performance_schema.user_variables_by_thread;
SELECT VARIABLE_NAME, VARIABLE_SOURCE, SET_TIME IS NOT NULL AS has_set_time, SET_USER, SET_HOST
FROM performance_schema.variables_info WHERE VARIABLE_SOURCE <> 'COMPILED'
ORDER BY VARIABLE_SOURCE, VARIABLE_NAME;
SELECT VARIABLE_SOURCE, COUNT(*) FROM performance_schema.variables_info GROUP BY VARIABLE_SOURCE;
```

Bước 5, database mặc định: một session chạy `USE performance_schema; SELECT 'marker_pfs_db', COUNT(*) FROM threads; USE sys; SELECT 'marker_sys_db'; USE mysql; SELECT 'marker_mysql_db', COUNT(*) FROM user; USE hpm3; SELECT 'marker_hpm3_db';`. Sau đó tìm các câu `marker` trong `events_statements_history_long` và bảng digest.

Bước 6, bộ đếm dùng index (liên quan chương 7, tr. 185): chạy hai câu tra theo khóa chính trên `orders`, đọc `table_io_waits_summary_by_index_usage`, chạy `CREATE INDEX idx_tmp ON orders (status)`, đọc lại; xem `sys.schema_unused_indexes` sau khi tạo một index không dùng.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định; đã rút gọn).

Stored routine:

| Lần | Event `statement/sp/*` |
| --- | --- |
| Mặc định, `sp_test(NULL)` | 4 event `stmt`: `INSERT INTO t1 VALUES (val)` (`MYSQL_ERRNO` 1048, `ERRORS` 1), `INSERT IGNORE … 'Some string'` (`WARNINGS` 1), hai `GET STACKED DIAGNOSTICS`. Không có `hpush_jump`, `hreturn`, `hpop` |
| Bật `statement/sp/%`, `sp_test(1)` | `hpush_jump`, `stmt` (`INSERT INTO t1 VALUES (val)`), `hpop` |
| Bật `statement/sp/%`, `sp_test(NULL)` | `hpush_jump`, `stmt`, `stmt` (`INSERT IGNORE`), `stmt`, `stmt` (hai `GET STACKED DIAGNOSTICS`), `hreturn`, `hpop` |

`@stacked_state` = 23000, `@stacked_msg` = "Column 'f1' cannot be null"; `t1` chứa 0, 1, 0.

Prepared statement: ba lần đếm ra 213.610, 125.770, 37.930; `prepared_statements_instances`: `stmt`, `OWNER_THREAD_ID` 78, `COUNT_REPREPARE` 0, `COUNT_EXECUTE` 3, `SUM_TIMER_EXECUTE` 630.153.200.000, trung bình 210,1 ms, `SUM_ROWS_EXAMINED` 900.030, `SUM_NO_INDEX_USED` 3. Sau `DROP PREPARE`: 0 dòng.

Lỗi (`events_errors_summary_global_by_error`):

| ERROR_NUMBER | ERROR_NAME | SUM_ERROR_RAISED | SUM_ERROR_HANDLED |
| --- | --- | --- | --- |
| 1062 | ER_DUP_ENTRY | 5 | 0 |
| 1146 | ER_NO_SUCH_TABLE | 4 | 0 |
| 1064 | ER_PARSE_ERROR | 3 | 0 |
| 1365 | ER_DIVISION_BY_ZERO | 2 | 0 |
| 1366 | ER_TRUNCATED_WRONG_VALUE_FOR_FIELD | 2 | 0 |
| 1051 | ER_BAD_TABLE_ERROR | 1 | 0 |
| 1305 | ER_SP_DOES_NOT_EXIST | 1 | 0 |
| 1048 | ER_BAD_NULL_ERROR | **0** | **2** |

Theo account (`SUM_ERROR_RAISED > 2`): `root@localhost` 1062 (5), 1146 (4), 1064 (3). `sys.statements_with_errors_or_warnings`: `INSERT INTO orders …` 5 lần, 5 lỗi; `SELECT * FROM no_such_table` 4/4 lỗi; `SELECT ? / ?` 2 lần, 2 cảnh báo; `CALL sp_test (?)` 3 lần, 2 cảnh báo; `DROP TABLE IF EXISTS t1` và `DROP PROCEDURE IF EXISTS sp_test` mỗi câu 1 cảnh báo. Bảng toàn cục có 5.319 dòng, 7 dòng có `SUM_ERROR_RAISED > 0`.

Biến:

| Truy vấn | Kết quả |
| --- | --- |
| `transaction_isolation` theo thread | thread 81 (pid 37) `SERIALIZABLE`; thread 82 (pid 38, client quan sát) `REPEATABLE-READ` |
| `tx_isolation` | 0 dòng |
| Self-join kiểu sách | thread 81: `sort_buffer_size` 262144 so với 4194304; `statement_id`; `transaction_isolation` `REPEATABLE-READ` so với `SERIALIZABLE` |
| `status_by_thread`, `Handler_write` | thread 81: 2000; thread 82: 0 |
| `user_variables_by_thread` | thread 81: `foo` = `bar` |
| `variables_info` khác `COMPILED` | `COMMAND_LINE`: `bind_address`, `datadir`, `port`, `socket`. `DYNAMIC`: `cte_max_recursion_depth`, `lock_wait_timeout`, `long_query_time`, `sort_buffer_size`, `transaction_isolation` (đều `SET_USER` root, `SET_HOST` localhost), `foreign_key_checks` (`SET_USER` NULL) |
| Đếm theo nguồn | `COMPILED` 618, `COMMAND_LINE` 4, `DYNAMIC` 6 |

Database mặc định:

| CURRENT_SCHEMA | Câu | Có trong `history_long`? | Có trong digest? |
| --- | --- | --- | --- |
| performance_schema | `SELECT 'marker_pfs_db' AS m, COUNT(*) FROM threads` | Có | Có (`SCHEMA_NAME` = performance_schema) |
| sys | `SELECT 'marker_sys_db' AS m` | Có | Có |
| mysql | `SELECT 'marker_mysql_db' AS m, COUNT(*) FROM user` | Có | Có |
| hpm3 | `SELECT 'marker_hpm3_db' AS m` | Có | Có |

`table_io_waits_summary_by_table` có dòng cho các bảng `hpm3` nhưng không có dòng nào cho `mysql` hay `performance_schema`.

Bộ đếm dùng index: trước DDL, `PRIMARY` `COUNT_READ` = 2; ngay sau `CREATE INDEX idx_tmp`, `PRIMARY`, `idx_customer`, `idx_tmp` đều 0. Trước đó, với `idx_created` vừa tạo và chưa dùng, `sys.schema_unused_indexes` liệt kê đúng `hpm3.orders.idx_created`.

**Diễn giải và đối chiếu với sách.**

- **Giả thuyết 1 sai.** Lỗi bị handler bắt chỉ tăng `SUM_ERROR_HANDLED`. Bảng gộp đếm cả cảnh báo (1365, 1366) và note (1051, 1305). Ví dụ lọc `SUM_ERROR_RAISED > 10` ở tr. 70 vì vậy bỏ sót lỗi bị nuốt; nên xét cả hai cột.
- **Giả thuyết 2 đúng**, khớp listing tr. 60. Với mặc định vẫn đoán được handler đã chạy (có thêm câu `INSERT IGNORE`, `GET STACKED DIAGNOSTICS`), nhưng muốn thấy cấu trúc điều khiển phải bật `statement/sp/%` như tr. 59 hướng dẫn.
- **Giả thuyết 3 đúng**, khớp tr. 58–59; instrument của Bảng 3-6 bật sẵn trên 8.0.42.
- **Giả thuyết 4 đúng.** Ví dụ `tx_isolation` (tr. 67) phải viết lại thành `transaction_isolation` trên 8.0. Truy vấn self-join (tr. 67–68) chạy được và cho đúng loại thông tin; `status_by_thread` chỉ ra thread ghi nhiều (tr. 68).
- `variables_info` (tr. 69) ghi `DYNAMIC` cả cho biến chỉ đổi ở mức session, kể cả bởi session khác. `foreign_key_checks` với `SET_USER` NULL xuất hiện giống ví dụ của sách.
- **Giả thuyết 5 sai.** Với `setup_actors` mặc định, câu chạy khi database mặc định là `performance_schema` vẫn được ghi. Thứ bị loại là số đo I/O trên bảng của các schema hệ thống, do `setup_objects`. Nhận định ở tr. 71 không tái hiện được trên 8.0.42.
- Bộ đếm `table_io_waits_summary_by_index_usage` (nền của `sys.schema_unused_indexes`, chương 7 tr. 185) **bị đặt lại** khi bảng bị DDL, ngoài việc bị đặt lại khi restart. Trước khi kết luận "index không dùng", phải bảo đảm cửa sổ quan sát đủ dài và không có DDL xen giữa.

**Dọn dẹp.** `SET GLOBAL long_query_time = 10;` (hoặc restart), `DROP DATABASE hpm3;`.

---

<a id="tong-ket"></a>

## Tổng kết và dọn dẹp

| Thí nghiệm | Trạng thái | Kết luận chính |
| --- | --- | --- |
| 1 | Đã chạy | Mặc định 8.0.42 gần đúng tr. 53, có thêm file/table I/O; `_history_long` tắt; luật ưu tiên tùy chọn khởi động đúng |
| 2 | Đã chạy | Digest sắp theo tổng thời gian tìm đúng câu tệ; `UPDATE` quét toàn bảng không bật `NO_INDEX_USED`; picosecond; không còn "Sending data"; `LOCK_TIME` không phải latency; `Handler_*` gồm bảng tạm |
| 3 | Đã chạy | MDL: DDL chờ chặn cả `SELECT` mới; `KILL` theo `PROCESSLIST_ID` |
| 4 | Đã chạy | Performance Schema ≈ 230 MB trên instance mặc định; bộ nhớ không giảm nhưng được dùng lại |
| 5 | Đã chạy | Lỗi bị handler bắt chỉ vào `SUM_ERROR_HANDLED`; `tx_isolation` không còn; database mặc định `performance_schema` vẫn được ghi; DDL đặt lại bộ đếm dùng index |

**Chưa thực thi:** đo chi phí CPU của việc bật `wait/synch/*` (cần benchmark có kiểm soát; laptop nhiều nhiễu không cho số đáng tin).

**Dọn dẹp đã làm:** `DROP DATABASE hpm3_many`, `DROP DATABASE hpm3`, `DROP DATABASE ops` (dùng khi chạy thử SQL ở mục 3 của ghi chú chương); tắt instance bằng `mysqladmin shutdown` sau mỗi lượt. Mọi thay đổi bằng `UPDATE setup_*` và `SET GLOBAL` mất khi restart.
