# Thực hành — Chương 1: Kiến trúc MySQL

Về ghi chú chương: [../chapter-01.md](../chapter-01.md) · Đáp án câu tự kiểm tra: [../answers/chapter-01.md](../answers/chapter-01.md)

Bốn thí nghiệm dưới đây kiểm tra trực tiếp các khẳng định của chương 1 về isolation level, MVCC, khóa, deadlock, transaction trong MySQL và data dictionary của 8.0. Mọi số liệu ghi **Kết quả đã đo** đều lấy từ lần chạy thật; output được **rút gọn** (bỏ khung bảng, bỏ dòng `Query OK, 0 rows affected`).

## Mục lục

- [Môi trường](#moi-truong)
- [Cách chạy nhiều session](#cach-chay)
- [TN1. Isolation level và MVCC](#tn1)
- [TN2. Khóa: row lock, gap lock, khóa ở tầng server](#tn2)
- [TN3. Deadlock](#tn3)
- [TN4. Transaction trong MySQL và data dictionary của 8.0](#tn4)
- [Dọn dẹp chung](#don-dep)

<a id="moi-truong"></a>

## Môi trường

| Mục | Giá trị |
| --- | --- |
| MySQL | Community Server **8.0.42**, Windows 11 Home, laptop |
| Instance | Riêng cho chương 1, khởi tạo bằng `mysqld --no-defaults --initialize-insecure`, chạy với `--no-defaults --port=3311 --bind-address=127.0.0.1 --mysqlx=OFF` |
| Tùy chọn thí nghiệm | `--innodb-print-all-deadlocks=ON` (ghi mọi deadlock vào error log). Mọi thứ khác mặc định |
| Mặc định đã đọc trên instance | `transaction_isolation = REPEATABLE-READ`, `autocommit = 1`, `innodb_lock_wait_timeout = 50`, `innodb_deadlock_detect = ON`, `innodb_rollback_on_timeout = OFF`, `innodb_flush_log_at_trx_commit = 1`, `innodb_buffer_pool_size = 128 MiB`, `binlog_format = ROW`, `log_bin = 1`, `thread_handling = one-thread-per-connection`, `thread_cache_size = 9` |
| Dữ liệu | Tổng hợp bằng SQL: bảng vài hàng cho thí nghiệm khóa; 2.000 hàng (`filler`), 50.000 hàng (`m`), 100.000 hàng (`orders`) |
| Đo thời gian | Client `mysql -vvv` in thời gian mỗi câu (độ phân giải 10 ms); với thí nghiệm MVCC dùng `NOW(6)` trước và sau câu đọc, lặp 3 lần, lấy trung vị. Laptop Windows nhiều nhiễu, **không phải benchmark production** |

> **Cảnh báo.** Chỉ chạy trên instance thử nghiệm. TN3 phần C đổi biến **toàn cục** `innodb_deadlock_detect`; phải bật lại sau khi chạy.

<a id="cach-chay"></a>

## Cách chạy nhiều session

Mỗi thí nghiệm gồm một script chuẩn bị (`prep.sql`) và vài script session (`s1.sql`, `s2.sql`, …). Thứ tự thao tác giữa các session được điều khiển **trong SQL** bằng `DO SLEEP(n)`: mọi session khởi động cùng lúc, session nào cần đi sau thì ngủ trước. Ví dụ một session "giữ khóa" là một transaction mở, ghi một hàng, rồi `DO SLEEP(3)` trước khi `COMMIT`.

Chạy song song bằng shell (**Ví dụ minh họa do AI xây dựng**; trên Windows dùng Git Bash):

```bash
M='mysql -h127.0.0.1 -P3311 -uroot'
$M < prep.sql
for s in s1 s2 s3; do
  $M -vvv -t --unbuffered --force lab < $s.sql > out_$s.txt &
done
wait
```

`--force` để script chạy tiếp sau lỗi (deadlock, timeout); `-vvv` để thấy từng câu và thời gian của nó. Khi chạy thật, người viết ghi chú dùng một script Python nhỏ làm đúng việc này (khởi động các session cùng lúc, lưu output, in bản rút gọn). Có thể làm thủ công bằng ba cửa sổ `mysql` và gõ lệnh theo đúng bảng thời gian, chỉ cần thao tác đủ chậm.

---

<a id="tn1"></a>

## TN1. Isolation level và MVCC

**Mục tiêu.** Quan sát điều mỗi isolation level cho phép thấy (tr. 8–9, Bảng 1-1), khi nào read view được tạo, khi nào transaction có ID thật (tr. 13), và cái giá của việc giữ snapshot lâu (tr. 14).

**Giả thuyết (Dự đoán).**

- A, B: ở REPEATABLE READ, đọc lại một hàng và đếm lại một khoảng sau khi session khác commit cho kết quả **như cũ**; ở READ COMMITTED thì thấy thay đổi.
- C: read view ở RR được tạo ở **lần đọc đầu**, không ở `START TRANSACTION`.
- D: transaction chỉ đọc không có ID thật; ID thật xuất hiện khi ghi.
- E: READ UNCOMMITTED thấy dữ liệu chưa commit; RC và RR đọc không bị chặn; locking read bị chặn.
- F: snapshot cũ phải đi ngược undo nên đọc chậm hơn; history list length tăng khi snapshot còn mở.

**Chuẩn bị** (dùng cho A–E; chạy lại trước mỗi phần):

```sql
CREATE DATABASE IF NOT EXISTS lab;
USE lab;
DROP TABLE IF EXISTS t1;
CREATE TABLE t1 (id INT PRIMARY KEY, v INT NOT NULL) ENGINE=InnoDB;
INSERT INTO t1 VALUES (1,100),(2,200);
```

### Phần A và B: REPEATABLE READ so với READ COMMITTED

Session 1 (phần B thay `REPEATABLE-READ` bằng `READ-COMMITTED`):

```sql
SET SESSION transaction_isolation = 'REPEATABLE-READ';
START TRANSACTION;
SELECT NOW(3) AS t, v FROM t1 WHERE id = 1;
SELECT NOW(3) AS t, COUNT(*) AS n FROM t1 WHERE id BETWEEN 1 AND 100;
DO SLEEP(2);
SELECT NOW(3) AS t, v FROM t1 WHERE id = 1;
SELECT NOW(3) AS t, COUNT(*) AS n FROM t1 WHERE id BETWEEN 1 AND 100;
COMMIT;
SELECT v FROM t1 WHERE id = 1;
SELECT COUNT(*) FROM t1 WHERE id BETWEEN 1 AND 100;
```

Session 2 (autocommit):

```sql
DO SLEEP(1);
UPDATE t1 SET v = v + 1 WHERE id = 1;
INSERT INTO t1 VALUES (5, 500);
```

**Kết quả đã đo.**

| Thời điểm | Sự kiện | REPEATABLE READ (S1 thấy) | READ COMMITTED (S1 thấy) |
| --- | --- | --- | --- |
| t ≈ 0 s | S1 đọc lần 1 | v = 100, n = 2 | v = 100, n = 2 |
| t ≈ 1 s | S2 sửa `id=1` thành 101, chèn `id=5`, tự commit | | |
| t ≈ 2 s | S1 đọc lần 2, vẫn trong transaction | v = **100**, n = **2** | v = **101**, n = **3** |
| sau `COMMIT` | S1 đọc lần 3 | v = 101, n = 3 | v = 101, n = 3 |

**Diễn giải.** RR giữ một snapshot cho cả transaction nên không có nonrepeatable read, và với consistent read cũng không có phantom (Bảng 1-1 ghi phantom "Có" cho RR vì đó là chuẩn ANSI; InnoDB chặn được, tr. 9). RC tạo snapshot mới cho mỗi câu, đúng như định nghĩa "thấy thay đổi đã commit sau khi mình bắt đầu" (tr. 8).

### Phần C: read view được tạo khi nào

Session 1: `START TRANSACTION` (RR), chưa đọc gì, `DO SLEEP(2)`, rồi đọc `v` của `id=1`. Session 3: giống session 1 nhưng mở bằng `START TRANSACTION WITH CONSISTENT SNAPSHOT`. Session 2: `DO SLEEP(1)` rồi `UPDATE t1 SET v = v + 1 WHERE id = 1` (autocommit).

**Kết quả đã đo.** Session 1 đọc được **101**; session 3 đọc được **100**.

**Diễn giải.** Ở RR, read view được tạo ở **câu consistent read đầu tiên** (t ≈ 2 s, sau khi S2 commit), không phải ở `START TRANSACTION`. `WITH CONSISTENT SNAPSHOT` tạo read view ngay. Đây là chi tiết sách gói trong câu "ID được gán lần đầu transaction đọc dữ liệu" (tr. 13).

### Phần D: transaction ID thật được cấp khi nào

Một session:

```sql
START TRANSACTION;
DO SLEEP(0.2);
SELECT COUNT(*) FROM information_schema.innodb_trx
 WHERE trx_mysql_thread_id = CONNECTION_ID();
SELECT v FROM t1 WHERE id = 1;
DO SLEEP(0.2);
SELECT trx_id, trx_state, trx_rows_locked, trx_rows_modified
  FROM information_schema.innodb_trx
 WHERE trx_mysql_thread_id = CONNECTION_ID();
UPDATE t1 SET v = v + 1 WHERE id = 2;
DO SLEEP(0.2);
SELECT trx_id, trx_state, trx_rows_locked, trx_rows_modified
  FROM information_schema.innodb_trx
 WHERE trx_mysql_thread_id = CONNECTION_ID();
ROLLBACK;
```

`DO SLEEP(0.2)` cần thiết vì `innodb_trx` được cache và chỉ làm mới sau khoảng 0,1 s; lần chạy đầu không có `SLEEP` cho kết quả rỗng ở cả ba câu.

**Kết quả đã đo.**

| Thời điểm | `innodb_trx` của session |
| --- | --- |
| Sau `START TRANSACTION` | 0 dòng |
| Sau `SELECT` đầu tiên | `trx_id = 284026416991296`, RUNNING, 0 hàng khóa, 0 hàng sửa |
| Sau `UPDATE` đầu tiên | `trx_id = 1455`, RUNNING, 1 hàng khóa, 1 hàng sửa |

**Diễn giải.** InnoDB chỉ biết tới transaction khi nó chạm dữ liệu lần đầu. ID hiển thị cho transaction chỉ đọc là một số rất lớn không lấy từ bộ đếm; theo manual MySQL (**Bổ sung ngoài sách**), ID thật chỉ được cấp cho transaction ghi hoặc khóa. Số 1455 nằm trong dãy ID thật (bộ đếm `Trx id counter` của `SHOW ENGINE INNODB STATUS` cùng thang).

### Phần E: dirty read, đọc không khóa, locking read

| Session | Thao tác |
| --- | --- |
| S1 | `START TRANSACTION; UPDATE t1 SET v = 999 WHERE id = 2;` rồi `DO SLEEP(3); ROLLBACK;` |
| S2 | `DO SLEEP(1)`, đặt READ-UNCOMMITTED, đọc `v` của `id=2`; `DO SLEEP(3)`, đọc lại |
| S3 | `DO SLEEP(1)`, đọc `v` của `id=2` ở READ-COMMITTED rồi ở REPEATABLE-READ |
| S4 | `DO SLEEP(1)`, `SELECT v FROM t1 WHERE id = 2 FOR SHARE` |

**Kết quả đã đo.**

| Session, level | Giá trị đọc được | Thời gian câu đọc |
| --- | --- | --- |
| S2, READ UNCOMMITTED, khi S1 chưa rollback | **999** | 0,00 s |
| S2, READ UNCOMMITTED, sau rollback | 200 | 0,00 s |
| S3, READ COMMITTED | 200 | 0,00 s |
| S3, REPEATABLE READ | 200 | 0,00 s |
| S4, `FOR SHARE` (autocommit) | 200 | **1,99 s** (chờ tới khi S1 rollback) |

**Diễn giải.** RU đọc bản mới nhất, kể cả bản chưa commit và sau đó bị rollback: dirty read thật (tr. 8, 15). RC và RR đọc bản đã commit **qua undo**, không chờ khóa X của S1: "phần lớn truy vấn đọc không bao giờ xin khóa" (tr. 14). Locking read thì phải chờ như mô hình read/write lock ở tr. 4.

### Phần F: cái giá của snapshot dài

**Chuẩn bị.**

```sql
USE lab;
SET SESSION cte_max_recursion_depth = 100000;
DROP TABLE IF EXISTS m;
CREATE TABLE m (id INT PRIMARY KEY, v INT NOT NULL, pad CHAR(100) NOT NULL) ENGINE=InnoDB;
INSERT INTO m (id, v, pad)
  WITH RECURSIVE seq (n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 50000)
  SELECT n, 0, REPEAT('x', 100) FROM seq;
DROP PROCEDURE IF EXISTS small_updates;
DELIMITER //
CREATE PROCEDURE small_updates(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 1;
  WHILE i <= n DO
    UPDATE m SET v = v + 1 WHERE id = i;   -- autocommit: mỗi vòng một transaction
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
```

| Session | Thao tác |
| --- | --- |
| S1 (đọc) | `START TRANSACTION WITH CONSISTENT SNAPSHOT` (RR); đo `SUM(v)` một lần; `DO SLEEP(12)`; đo `SUM(v)` ba lần; `COMMIT` |
| S2 (ghi) | `DO SLEEP(0.5)`; 5 lần `UPDATE m SET v = v + 1` (mỗi lần 50.000 hàng, khoảng 0,7 s); `CALL small_updates(1000)` (2,17 s); rồi đo `SUM(v)` ba lần |
| S3 (theo dõi) | đọc `trx_rseg_history_len` trong `information_schema.INNODB_METRICS` lúc t ≈ 0, 11,5, 15,5 và 20,5 s |

Đo thời gian một câu đọc bằng:

```sql
SET @t = NOW(6);
SELECT SUM(v) FROM m;
SELECT TIMESTAMPDIFF(MICROSECOND, @t, NOW(6)) AS us;
```

**Kết quả đã đo** (50.000 hàng; laptop Windows, nhiễu cao).

| Chỉ số | Giá trị |
| --- | --- |
| `SUM(v)` S1 thấy, trước và sau khi S2 ghi | 0 và **0** (snapshot không đổi) |
| `SUM(v)` S2 thấy sau khi ghi | 251.000 |
| Thời gian `SUM(v)` qua snapshot S1, trước khi S2 ghi | 23,5 ms (1 lần) |
| Thời gian `SUM(v)` qua snapshot S1, sau khi S2 ghi | 408, 390, 374 ms → trung vị **390 ms** |
| Thời gian `SUM(v)` qua snapshot mới (S2) | 16,8; 17,3; 17,9 ms → trung vị **17,3 ms** |
| `trx_rseg_history_len` | 10 (đầu) → **1.007** (S1 còn mở) → 1.007 (khoảng 2 s và 7 s sau khi S1 commit) |
| Kiểm tra thêm 13 s sau khi S1 commit | `SHOW ENGINE INNODB STATUS`: "Purge done for trx's n:o < 3928 ... running but idle", History list length 1.007 |
| Sau khi chạy thêm `CALL small_updates(300)` và chờ 3 s | History list length **59** |

**Diễn giải.** S1 vẫn thấy dữ liệu ở thời điểm snapshot (tr. 13), nhưng để làm vậy mỗi hàng phải được dựng lại qua 5–6 undo record (tr. 14): chậm khoảng 20 lần so với đọc bản mới nhất. Đây là "làm thêm việc khi xét từng hàng" và "housekeeping" mà sách nói là cái giá của MVCC (tr. 14). History list length nhảy lên khoảng 1.000 (đúng số transaction nhỏ đã commit trong lúc snapshot mở). Việc nó không giảm ngay sau khi purge báo xong là quan sát chưa được giải thích chắc chắn; giả thuyết (**Bổ sung ngoài sách**) là bộ đếm giảm khi rollback segment được truncate theo lô (`innodb_purge_rseg_truncate_frequency = 128` trên instance này).

**Dọn dẹp.** `DROP TABLE lab.t1, lab.m; DROP PROCEDURE lab.small_updates;`

---

<a id="tn2"></a>

## TN2. Khóa: row lock, gap lock, khóa ở tầng server

**Mục tiêu.** Nhìn thấy khóa mà InnoDB đặt (tr. 5–6, 12, 16), độ mịn khóa, ảnh hưởng của isolation level và access path, khóa ở tầng server (`LOCK TABLES`, tr. 12–13), và hệ quả với ví dụ ngân hàng (tr. 6–7).

**Giả thuyết (Dự đoán).**

- A: `UPDATE` theo primary key chỉ khóa đúng một bản ghi; session sửa hàng khác không bị chặn.
- B, C: `FOR UPDATE` trên một khoảng ở RR khóa cả khoảng trống (next-key), chặn chèn vào khoảng; ở RC thì không.
- D: `UPDATE` trên cột không có index ở RR khóa mọi hàng đã quét.
- E: locking read thấy hàng mới mà consistent read không thấy.
- F: SERIALIZABLE khóa hàng mà `SELECT` thường đọc (khi autocommit tắt).
- G: `LOCK TABLES` không xuất hiện trong `data_locks`.
- H: đọc bằng snapshot rồi ghi giá trị tuyệt đối gây lost update.
- I: job `UPDATE` hàng loạt trên cột không có index khóa gần cả bảng; có index thì chỉ khóa khoảng cần sửa.

**Chuẩn bị** (chạy lại trước mỗi phần A–G):

```sql
USE lab;
DROP TABLE IF EXISTS t2;
CREATE TABLE t2 (id INT PRIMARY KEY, k INT NOT NULL, c INT NOT NULL, KEY idx_k (k)) ENGINE=InnoDB;
INSERT INTO t2 VALUES (10,10,0),(20,20,0),(30,30,0),(40,40,0);
```

Câu theo dõi khóa dùng ở nhiều phần:

```sql
SELECT ENGINE_TRANSACTION_ID AS trx, INDEX_NAME AS idx, LOCK_TYPE,
       LOCK_MODE, LOCK_STATUS, LOCK_DATA
  FROM performance_schema.data_locks
 ORDER BY ENGINE_TRANSACTION_ID, LOCK_TYPE DESC, LOCK_DATA;
```

### Phần A: một row lock và một lần chờ

| Thời điểm | S1 | S2 | S3 | S4 (theo dõi) |
| --- | --- | --- | --- | --- |
| 0 s | `START TRANSACTION; UPDATE t2 SET c = c + 1 WHERE id = 20;` | | | |
| 0,5 s | | | | `data_locks` |
| 1 s | | `UPDATE t2 SET c = c + 10 WHERE id = 20;` | `UPDATE t2 SET c = c + 100 WHERE id = 30;` | |
| 1,5 s | | | | `data_locks`, `data_lock_waits`, `sys.innodb_lock_waits` |
| 3 s | `COMMIT` | | | |

**Kết quả đã đo** (rút gọn).

```text
-- data_locks lúc 0,5 s
1471 | NULL    | TABLE  | IX            | GRANTED | NULL
1471 | PRIMARY | RECORD | X,REC_NOT_GAP | GRANTED | 20
-- data_locks lúc 1,5 s
1471 | NULL    | TABLE  | IX            | GRANTED | NULL
1471 | PRIMARY | RECORD | X,REC_NOT_GAP | GRANTED | 20
1473 | NULL    | TABLE  | IX            | GRANTED | NULL
1473 | PRIMARY | RECORD | X,REC_NOT_GAP | WAITING | 20
-- data_lock_waits: waiting_trx 1473, blocking_trx 1471
-- sys.innodb_lock_waits: locked_index PRIMARY | waiting_query UPDATE t2 SET c = c + 10 WHERE id = 20
--   | blocking_pid 33 | blocking_query DO SLEEP(3) | sql_kill_blocking_connection KILL 33
```

- S2: `UPDATE ... id = 20` mất **2,01 s**. S3: `UPDATE ... id = 30` mất **0,00 s**.
- `Innodb_row_lock_waits` tăng 1, `Innodb_row_lock_time` tăng 2.010 ms.

**Diễn giải.** InnoDB đặt intention lock `IX` ở mức bảng và một khóa bản ghi X trên **bản ghi primary key** (không khóa khoảng trống vì là tìm đúng một giá trị unique). Row-level granularity cho S3 chạy song song (tr. 5). Transaction chặn đang "ngủ" (`DO SLEEP(3)`) mà vẫn giữ khóa: two-phase locking giữ khóa tới commit (tr. 12).

### Phần B: access path quyết định phạm vi khóa

S1 (RR): `START TRANSACTION; SELECT id FROM t2 WHERE id BETWEEN 15 AND 35 FOR UPDATE;` giữ 3 s. S2 lúc 1 s chèn `(25,25,0)`, S3 lúc 1 s chèn `(50,50,0)`, S4 lúc 1,5 s xem `data_locks`.

`EXPLAIN` của câu `FOR UPDATE` (bảng 4 hàng, sau `ANALYZE TABLE`): `type = index`, `key = idx_k`, `Extra = Using where; Using index`; dạng cây: "Filter: (t2.id between 15 and 35) → Index scan on t2 using idx_k".

**Kết quả đã đo.**

| | REPEATABLE READ | READ COMMITTED |
| --- | --- | --- |
| Khóa của S1 | `X,REC_NOT_GAP` trên `PRIMARY` 10, 20, 30, 40; `X` trên `idx_k` (10,10), (20,20), (30,30), (40,40) và supremum | `X,REC_NOT_GAP` trên `PRIMARY` 20, 30 và `idx_k` (20,20), (30,30) |
| Chèn 25 | chờ **2,01 s** | 0,00 s |
| Chèn 50 | chờ **2,01 s** (`X,INSERT_INTENTION` chờ trên supremum) | 0,00 s |

**Diễn giải.** Với 4 hàng, optimizer chọn **quét hết index phụ `idx_k`** vì nó chứa sẵn `id` (covering). Ở RR mọi bản ghi được quét đều bị khóa next-key, nên chèn ở bất cứ đâu cũng bị chặn, kể cả `id = 50` nằm ngoài điều kiện. Ở RC, InnoDB nhả khóa của hàng không khớp và không dùng gap lock. LOCK_DATA "(20, 20)" của `idx_k` là cặp (giá trị `k`, primary key): secondary index chứa primary key (tr. 16–17).

### Phần C: range trên primary key ở RR và RC

Như phần B nhưng S1 chạy `SELECT id, c FROM t2 WHERE id BETWEEN 15 AND 35 FOR UPDATE` (cột `c` không có trong `idx_k`, nên plan đổi thành `type = range`, `key = PRIMARY`, rows = 2). S3 chèn `(50,50,0)` rồi `(5,5,0)`.

**Kết quả đã đo.**

| | REPEATABLE READ | READ COMMITTED |
| --- | --- | --- |
| Khóa của S1 | `X` trên 20 và 30; `X,GAP` trên 40 | `X,REC_NOT_GAP` trên 20 và 30 |
| Chèn 25 | chờ **2,00 s**; khóa chờ `X,GAP,INSERT_INTENTION` trên 30 | 0,00 s |
| Chèn 50, chèn 5 | 0,00 s | 0,00 s |

**Diễn giải.** Đây là next-key locking đúng như tr. 16: khóa `X` (không có hậu tố) trên 20 và 30 là **bản ghi cộng khoảng trống ngay trước nó**, `X,GAP` trên 40 khóa khoảng (30, 40). Khoảng từ 10 tới 40 bị khóa, nên 25 không chèn được, còn 5 và 50 nằm ngoài. Không ai chèn được phantom vào khoảng mà S1 đã đọc có khóa.

### Phần D: `UPDATE` trên cột không có index

S1: `START TRANSACTION; UPDATE t2 SET c = c + 1 WHERE c = 999;` (không có hàng nào khớp; `EXPLAIN` cho `type = index` trên `PRIMARY`, `possible_keys = NULL`), giữ 3 s. S2 lúc 1 s: `UPDATE t2 SET c = c + 1 WHERE id = 40`. S3 lúc 1,5 s xem `data_locks`.

**Kết quả đã đo.**

| | REPEATABLE READ | READ COMMITTED |
| --- | --- | --- |
| Khóa của S1 | `X` trên `PRIMARY` 10, 20, 30, 40 và supremum | chỉ `IX` mức bảng |
| `UPDATE ... id = 40` của S2 | chờ **2,01 s** | 0,00 s |

(`EXPLAIN FORMAT=TREE` không hỗ trợ `UPDATE` trên 8.0.42: trả "not executable by iterator executor"; dùng `EXPLAIN` thường.)

**Diễn giải.** Engine duyệt cả clustered index và khóa mọi bản ghi nó đọc; server lọc `c = 999` sau đó. Ở RR khóa được giữ dù hàng không khớp. Đây là cơ chế đằng sau [tình huống 2](../chapter-01.md#th2) trong ghi chú chương.

### Phần E: consistent read so với locking read trong cùng transaction

S1 (RR): `START TRANSACTION`; `COUNT(*) ... WHERE id >= 0`; `DO SLEEP(2)`; `COUNT(*)` thường; `COUNT(*) ... FOR SHARE`; `COUNT(*)` thường; `UPDATE t2 SET c = c + 1 WHERE id >= 0`; `COUNT(*)` và `SUM(c)`; `ROLLBACK`. S2 lúc 1 s chèn `(50,50,0)` (autocommit).

**Kết quả đã đo.**

| Bước của S1 | Kết quả |
| --- | --- |
| `COUNT(*)` thường, trước khi S2 chèn | 4 |
| `COUNT(*)` thường, sau khi S2 chèn và commit | **4** |
| `COUNT(*) ... FOR SHARE` | **5** |
| `COUNT(*)` thường lần nữa | **4** |
| `UPDATE ... WHERE id >= 0` | 5 hàng bị đổi |
| `COUNT(*)`, `SUM(c)` sau `UPDATE` | 5 và 5 |

**Diễn giải.** Consistent read đọc snapshot (MVCC chặn phantom, tr. 9). Locking read và `UPDATE` đọc bản mới nhất đã commit, nên thấy hàng 50. Sau khi chính S1 sửa hàng 50, hàng đó mang `DB_TRX_ID` của S1 và trở nên nhìn thấy được trong snapshot của S1. Đây là một trong những "điểm tinh tế" của MVCC mà sách khuyên đọc manual (tr. 17). **Bổ sung ngoài sách:** muốn đếm rồi ghi một cách nhất quán, hãy dùng locking read ngay từ đầu.

### Phần F: SERIALIZABLE

S1: `SET SESSION transaction_isolation = 'SERIALIZABLE'; START TRANSACTION; SELECT id, c FROM t2 WHERE id = 20;` giữ 3 s. S2 lúc 1 s: `UPDATE t2 SET c = c + 1 WHERE id = 20`. S3 lúc 1,5 s xem `data_locks`. S4: SERIALIZABLE nhưng `autocommit = 1`, một `SELECT ... WHERE id = 30` rồi đếm khóa của chính nó.

**Kết quả đã đo.**

```text
-- data_locks lúc 1,5 s
1709            | NULL    | TABLE  | IX            | GRANTED | NULL
1709            | PRIMARY | RECORD | X,REC_NOT_GAP | WAITING | 20
284026416991296 | NULL    | TABLE  | IS            | GRANTED | NULL
284026416991296 | PRIMARY | RECORD | S,REC_NOT_GAP | GRANTED | 20
```

- S2 chờ **2,00 s**.
- S4 (autocommit = 1): **0** khóa sau câu `SELECT`.

**Diễn giải.** SERIALIZABLE biến `SELECT` thường thành đọc có khóa S (tr. 9: "khóa mọi hàng nó đọc"), nên người ghi phải chờ. Transaction S1 chỉ đọc nên vẫn mang ID "giả" lớn (xem TN1 phần D). Với autocommit bật, `SELECT` là transaction riêng và không để lại khóa (**Bổ sung ngoài sách**, khớp manual).

### Phần G: `LOCK TABLES` ở tầng server

S1: `LOCK TABLES t2 READ;` giữ 3 s rồi `UNLOCK TABLES`. S2 lúc 1 s: `UPDATE t2 SET c = c + 1 WHERE id = 10`. S3 lúc 1 s: `SELECT COUNT(*) FROM t2`. S4 lúc 1,5 s:

```sql
SELECT OBJECT_TYPE, OBJECT_SCHEMA, OBJECT_NAME, LOCK_TYPE, LOCK_DURATION, LOCK_STATUS
  FROM performance_schema.metadata_locks WHERE OBJECT_SCHEMA = 'lab';
SELECT COUNT(*) FROM performance_schema.data_locks;
SELECT ID, STATE, INFO FROM information_schema.PROCESSLIST WHERE INFO LIKE 'UPDATE t2%';
```

**Kết quả đã đo.**

```text
TABLE | lab | t2 | SHARED_READ_ONLY | TRANSACTION | GRANTED
TABLE | lab | t2 | SHARED_WRITE     | TRANSACTION | PENDING
data_locks: 0 dòng
78 | Waiting for table metadata lock | UPDATE t2 SET c = c + 1 WHERE id = 10
```

- S2 chờ **2,01 s**; S3 đọc ngay (0,00 s).

**Diễn giải.** `LOCK TABLES` cài ở server (tr. 12), InnoDB không biết, nên `data_locks` trống. Ai chẩn đoán khóa chỉ bằng `data_locks` sẽ bỏ sót loại chờ này.

### Phần H: ví dụ ngân hàng và lost update

**Chuẩn bị.**

```sql
USE lab;
DROP TABLE IF EXISTS checking;
CREATE TABLE checking (customer_id INT PRIMARY KEY, balance DECIMAL(12,2) NOT NULL) ENGINE=InnoDB;
INSERT INTO checking VALUES (10233276, 1000.00);
```

Hai session cùng rút $200, S2 bắt đầu sau S1 0,2 s. Ba biến thể (**Ví dụ minh họa do AI xây dựng**, dựa trên kịch bản tr. 6):

```sql
-- (1) ngây thơ: đọc snapshot, ghi giá trị tuyệt đối do "ứng dụng" tính
START TRANSACTION;
SELECT balance INTO @b FROM checking WHERE customer_id = 10233276;
DO SLEEP(1);   -- S2 dùng 1.3
UPDATE checking SET balance = @b - 200.00 WHERE customer_id = 10233276;
COMMIT;

-- (2) như (1) nhưng câu SELECT thêm FOR UPDATE

-- (3) cập nhật có điều kiện, không dùng giá trị đã đọc
START TRANSACTION;
UPDATE checking SET balance = balance - 200.00
 WHERE customer_id = 10233276 AND balance >= 200.00;
SELECT ROW_COUNT();   -- 0 nghĩa là không đủ tiền
COMMIT;
```

**Kết quả đã đo.**

| Biến thể | S1 đọc | S2 đọc | S2 chờ | Số dư cuối |
| --- | --- | --- | --- | --- |
| (1) snapshot + giá trị tuyệt đối | 1000 | 1000 | không | **800** (mất một lần rút) |
| (2) `FOR UPDATE` | 1000 | 800 | 0,80 s | 600 |
| (3) `UPDATE` có điều kiện | 1000 | 1000 (không dùng) | không đáng kể | 600 |

**Diễn giải.** Transaction và RR không tự chặn lost update khi ứng dụng tính giá trị từ một consistent read. `FOR UPDATE` đọc bản mới nhất và khóa nó; `UPDATE ... SET balance = balance - 200` cũng đọc bản mới nhất bên trong engine. Đây là ví dụ cho câu "transaction một mình chưa phải toàn bộ câu chuyện" (tr. 6–7).

### Phần I: job cập nhật hàng loạt, có và không có index (số đo cho tình huống 2)

**Chuẩn bị** (**Ví dụ minh họa do AI xây dựng**; 100.000 hàng, 2% `pending`):

```sql
USE lab;
SET SESSION cte_max_recursion_depth = 200000;
DROP TABLE IF EXISTS orders;
CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer_id INT NOT NULL,
  status VARCHAR(16) NOT NULL,
  updated_at DATETIME NOT NULL
) ENGINE=InnoDB;
INSERT INTO orders (id, customer_id, status, updated_at)
  WITH RECURSIVE seq (n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 100000)
  SELECT n, n MOD 5000, IF(n MOD 50 = 0, 'pending', 'shipped'),
         '2026-09-01' + INTERVAL (n MOD 30) DAY
    FROM seq;
ANALYZE TABLE orders;
-- lần chạy thứ hai: thêm index trước ANALYZE
-- ALTER TABLE orders ADD INDEX idx_status_updated (status, updated_at);
```

| Thời điểm | S1 (job) | S2 (giao dịch OLTP không liên quan) |
| --- | --- | --- |
| 0 s | `EXPLAIN` câu `UPDATE`; `START TRANSACTION`; `UPDATE orders SET status = 'expired' WHERE status = 'pending' AND updated_at < '2026-09-15'` | `SET SESSION innodb_lock_wait_timeout = 1` |
| 0,3 s | đọc `trx_rows_locked`, `trx_rows_modified`, `trx_lock_structs` từ `innodb_trx`; đếm `data_locks` theo index và `LOCK_MODE` | |
| 1 s | | `UPDATE orders SET customer_id = customer_id WHERE id = 99999` (đơn này ở trạng thái `shipped`) |
| khoảng 2,3 s | `ROLLBACK` | |

**Kết quả đã đo.**

| | Không có index | Có `idx_status_updated` |
| --- | --- | --- |
| `EXPLAIN` | `type = index`, `key = PRIMARY`, `possible_keys = NULL`, rows 100.224 | `type = range`, `key = idx_status_updated`, rows 1.333, `Using where; Using temporary` |
| Hàng bị đổi | 1.333 (0,10 s) | 1.333 (0,04 s) |
| `trx_rows_locked` / `trx_lock_structs` | 100.261 / 262 | 4.005 / 277 |
| `data_locks` theo index | `PRIMARY` `X`: 100.261 | `PRIMARY` `X,REC_NOT_GAP`: 1.334; `idx_status_updated` `X`: 1.338; `X,GAP`: 1.333 |
| `UPDATE` của S2 | **`ERROR 1205`** sau khoảng 1 s; câu kế tiếp của S2 chờ 0,76 s tới khi S1 rollback | xong ngay |

**Diễn giải.** Không có index, engine duyệt cả clustered index và ở REPEATABLE READ khóa mọi bản ghi đã duyệt, nên một giao dịch không liên quan cũng bị chặn. Có index, engine chỉ duyệt khoảng `status = 'pending'` và khóa khoảng đó (cộng các bản ghi primary key tương ứng). `Using temporary` xuất hiện vì câu lệnh sửa chính cột thuộc index đang dùng để duyệt (**Bổ sung ngoài sách**: MySQL gom các hàng cần sửa trước để không duyệt lại hàng vừa sửa).

**Dọn dẹp TN2.** `DROP TABLE lab.t2, lab.checking, lab.orders;`

---

<a id="tn3"></a>

## TN3. Deadlock

**Mục tiêu.** Tái hiện deadlock StockPrice (tr. 9–10), đọc báo cáo deadlock, kiểm tra cách InnoDB chọn transaction để rollback, và xem điều gì xảy ra khi chỉ dựa vào lock wait timeout (tr. 10).

**Giả thuyết (Dự đoán).**

- A: một transaction nhận lỗi deadlock gần như ngay khi vòng khép lại; transaction kia chạy tiếp.
- B: InnoDB rollback transaction "nhỏ" hơn, kể cả khi transaction lớn là bên khép vòng.
- C: tắt phát hiện deadlock thì cả hai chờ tới timeout.

**Chuẩn bị.**

```sql
USE lab;
DROP TABLE IF EXISTS StockPrice;
CREATE TABLE StockPrice (
  stock_id INT NOT NULL,
  date DATE NOT NULL,
  open DECIMAL(10,2), close DECIMAL(10,2), high DECIMAL(10,2), low DECIMAL(10,2),
  PRIMARY KEY (stock_id, date)
) ENGINE=InnoDB;
INSERT INTO StockPrice VALUES
 (1,'2020-05-01',10.00,10.50,10.80,9.90),
 (2,'2020-05-01',30.00,30.20,30.90,29.50),
 (3,'2020-05-02',19.00,19.50,20.00,18.90),
 (4,'2020-05-01',45.00,45.20,47.00,44.80);
SET SESSION cte_max_recursion_depth = 5000;
DROP TABLE IF EXISTS filler;
CREATE TABLE filler (id INT PRIMARY KEY, v INT NOT NULL) ENGINE=InnoDB;
INSERT INTO filler (id, v)
  WITH RECURSIVE seq (n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 2000)
  SELECT n, 0 FROM seq;
```

(Lần chạy đầu thiếu dòng `cte_max_recursion_depth` và nhận `ERROR 3636 Recursive query aborted after 1001 iterations`: mặc định chỉ cho 1.000 vòng.)

### Phần A: kịch bản của sách

| Thời điểm | T1 (session 1) | T2 (session 2) |
| --- | --- | --- |
| 0 s | `START TRANSACTION`; sửa `close` của (4, '2020-05-01') | |
| 0,3 s | | `START TRANSACTION`; sửa `high` của (3, '2020-05-02') |
| 1,0 s | sửa `close` của (3, '2020-05-02') → chờ T2 | |
| 1,5 s | | sửa `high` của (4, '2020-05-01') → khép vòng |
| sau đó | `COMMIT` | (script vẫn chạy `COMMIT` nhờ `--force`) |

Một session thứ ba xem `data_locks` lúc 1,25 s. Sau thí nghiệm: `SHOW ENGINE INNODB STATUS\G` và bộ đếm `lock_deadlocks`.

**Kết quả đã đo** (rút gọn).

```text
-- data_locks lúc 1,25 s (LOCK_DATA hiện ngày dưới dạng số nội bộ)
1762 | PRIMARY | RECORD | X,REC_NOT_GAP | WAITING | 3, 1034402
1762 | PRIMARY | RECORD | X,REC_NOT_GAP | GRANTED | 4, 1034401
1763 | PRIMARY | RECORD | X,REC_NOT_GAP | GRANTED | 3, 1034402
-- T2 nhận:
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
-- T1: câu UPDATE thứ hai xong sau 0,52 s chờ
-- SHOW ENGINE INNODB STATUS, mục LATEST DETECTED DEADLOCK:
*** (1) TRANSACTION: TRANSACTION 1762 ... LOCK WAIT 3 lock struct(s), 2 row lock(s), undo log entries 1
UPDATE StockPrice SET close = 19.80 WHERE stock_id = 3 AND date = '2020-05-02'
*** (1) HOLDS THE LOCK(S): ... index PRIMARY ... lock_mode X locks rec but not gap
*** (1) WAITING FOR THIS LOCK TO BE GRANTED: ... lock_mode X locks rec but not gap waiting
*** (2) TRANSACTION: TRANSACTION 1763 ... (đối xứng)
*** WE ROLL BACK TRANSACTION (2)
-- INNODB_METRICS lock_deadlocks: 0 → 1
```

Dữ liệu cuối: hàng 3 có `close = 19.80` (của T1), `high = 20.00` (bản gốc, vì T2 bị rollback); hàng 4 có `close = 45.50` (của T1), `high = 47.00`.

**Đọc báo cáo.** Mỗi transaction có ba khối: thông tin chung và câu đang chạy, khóa đang giữ (`HOLDS THE LOCK(S)`), khóa đang chờ (`WAITING FOR THIS LOCK TO BE GRANTED`); cuối cùng là transaction bị rollback. Phần dump bản ghi cho thấy cột ẩn: trường 2 của bản ghi hàng 4 là `len 6; hex 0000000006e2` = **1762**, đúng `DB_TRX_ID` của T1 đã sửa nó; trường 3 dài 7 byte là rollback pointer (xem MVCC, tr. 13–14). Với `innodb_print_all_deadlocks=ON`, nội dung tương tự được ghi vào error log của instance.

### Phần B: chọn nạn nhân theo kích thước

| Thời điểm | T_lớn | T_nhỏ |
| --- | --- | --- |
| 0 s | `START TRANSACTION`; `UPDATE filler SET v = v + 1` (2.000 hàng); sửa hàng 4 | |
| 0,3 s | | `START TRANSACTION`; sửa hàng 3 |
| 1,0 s | | sửa hàng 4 → chờ T_lớn |
| 1,5 s | sửa hàng 3 → **T_lớn khép vòng** | |

Session theo dõi lúc 1,2 s: `SELECT trx_id, trx_state, trx_weight, trx_rows_locked, trx_rows_modified FROM information_schema.innodb_trx`.

**Kết quả đã đo.**

| Transaction | `trx_weight` | `trx_rows_locked` | `trx_rows_modified` | Kết quả |
| --- | --- | --- | --- | --- |
| T_lớn (1810) | 2008 | 2005 | 2001 | chạy tiếp, commit |
| T_nhỏ (1811) | 4 | 2 | 1 | **`ERROR 1213`, bị rollback** |

Báo cáo deadlock: "WE ROLL BACK TRANSACTION (1)", với (1) là 1811. `lock_deadlocks`: 1 → 2. `SUM(v)` của `filler` sau cùng là 2.000 (thay đổi của T_lớn được commit).

**Diễn giải.** InnoDB không rollback "kẻ đến sau" mà rollback bên **rẻ hơn** để hoàn tác, khớp với ý "dễ rollback nhất" ở tr. 10. Thí nghiệm không tách được "ít exclusive row lock nhất" (sách) với "ít hàng bị sửa nhất" (manual, **Bổ sung ngoài sách**) vì T_lớn hơn ở cả hai.

### Phần C: tắt phát hiện deadlock

```sql
SET GLOBAL innodb_deadlock_detect = OFF;   -- trên instance thử nghiệm
-- mỗi session thêm trước START TRANSACTION:
SET SESSION innodb_lock_wait_timeout = 3;
-- chạy lại đúng kịch bản phần A, rồi:
SET GLOBAL innodb_deadlock_detect = ON;
```

**Kết quả đã đo.**

| | T1 | T2 |
| --- | --- | --- |
| Bắt đầu chờ | 1,0 s | 1,5 s |
| Lỗi | `ERROR 1205 Lock wait timeout exceeded` | `ERROR 1205 Lock wait timeout exceeded` |
| Thời gian chờ | khoảng 3,5 s | khoảng 3,0 s |
| Sau lỗi, script `COMMIT` | commit câu `UPDATE` đầu (hàng 4, `close = 45.50`) | commit câu `UPDATE` đầu (hàng 3, `high = 20.12`) |

Tổng thời gian chạy 4,6 s so với 1,6 s ở phần A. `lock_timeouts`: 0 → 2; `lock_deadlocks` không đổi. Dữ liệu cuối: hàng 3 `close = 19.50` (gốc), `high = 20.12` (T2); hàng 4 `close = 45.50` (T1), `high = 47.00` (gốc). Session theo dõi lúc 2,5 s thấy cả hai transaction ở trạng thái `LOCK WAIT`.

**Diễn giải.** Không có phát hiện deadlock, vòng chờ chỉ bị phá bởi timeout: deadlock "trông như truy vấn rất chậm" (tr. 10). Hai lần timeout xảy ra gần như cùng lúc (bộ kiểm tra timeout chạy theo chu kỳ, nên T1 chờ hơn 3 s). Quan trọng hơn: lock wait timeout chỉ rollback **câu lệnh**, không rollback transaction (`innodb_rollback_on_timeout = OFF`), nên một ứng dụng bỏ qua lỗi rồi `COMMIT` sẽ ghi **nửa transaction**. Đây là "rollback một phần" mà tr. 10 nhắc tới, và lý do ứng dụng phải xử lý lỗi rồi chạy lại **cả** transaction.

**Dọn dẹp TN3.** Kiểm tra `SELECT @@innodb_deadlock_detect;` trả 1. `DROP TABLE lab.StockPrice, lab.filler;`

---

<a id="tn4"></a>

## TN4. Transaction trong MySQL và data dictionary của 8.0

**Mục tiêu.** Kiểm tra commit ngầm của DDL và `AUTOCOMMIT` (tr. 11), trộn engine (tr. 12), atomic DDL (tr. 18) và cách MySQL 8.0 lưu metadata trên đĩa (tr. 16–18).

**Giả thuyết (Dự đoán).** DDL giữa transaction commit luôn phần trước nó; rollback không hoàn tác được MyISAM và có cảnh báo; `DROP TABLE` một bảng có thật cùng một bảng không tồn tại sẽ thất bại trọn vẹn; thư mục dữ liệu không còn `.frm`.

**Chuẩn bị.**

```sql
USE lab;
DROP TABLE IF EXISTS t_inno, t_myi, t4b, d1;
CREATE TABLE t_inno (id INT PRIMARY KEY) ENGINE=InnoDB;
CREATE TABLE t_myi  (id INT PRIMARY KEY) ENGINE=MyISAM;
```

**Các bước** (một session):

```sql
-- A. commit ngầm do DDL
START TRANSACTION;
INSERT INTO t_inno VALUES (1);
CREATE TABLE t4b (id INT) ENGINE=InnoDB;
ROLLBACK;
SELECT COUNT(*) FROM t_inno;

-- B. autocommit = 0
SET SESSION autocommit = 0;
INSERT INTO t_inno VALUES (2);
DO SLEEP(0.2);
SELECT COUNT(*) FROM information_schema.innodb_trx WHERE trx_mysql_thread_id = CONNECTION_ID();
ROLLBACK;
SELECT COUNT(*) FROM t_inno;
SET SESSION autocommit = 1;

-- C. trộn InnoDB và MyISAM
START TRANSACTION;
INSERT INTO t_inno VALUES (3);
INSERT INTO t_myi VALUES (3);
ROLLBACK;
SHOW WARNINGS;
SELECT (SELECT COUNT(*) FROM t_inno WHERE id = 3) AS innodb_con,
       (SELECT COUNT(*) FROM t_myi  WHERE id = 3) AS myisam_con;

-- D. atomic DDL
CREATE TABLE d1 (id INT) ENGINE=InnoDB;
DROP TABLE d1, khong_ton_tai;
SHOW TABLES LIKE 'd1';

-- E. dictionary
SELECT COUNT(*) FROM mysql.tables;
```

Sau đó, ở shell, liệt kê thư mục `lab/` trong datadir và chạy `ibd2sdi lab/t_inno.ibd` (công cụ đi kèm MySQL; chỉ đọc file).

**Kết quả đã đo.**

| Phần | Kết quả |
| --- | --- |
| A | `COUNT(*)` = **1**: `CREATE TABLE` đã commit ngầm `INSERT` trước nó |
| B | Có 1 transaction trong `innodb_trx` sau `INSERT`; sau `ROLLBACK`, `COUNT(*)` vẫn 1 (hàng 2 đã bị hoàn tác) |
| C | `Warning 1196: Some non-transactional changed tables couldn't be rolled back`; `innodb_con = 0`, `myisam_con = 1` |
| D | `ERROR 1051 (42S02): Unknown table 'lab.khong_ton_tai'`; `SHOW TABLES LIKE 'd1'` vẫn trả `d1` |
| E | `ERROR 3554 (HY000): Access to data dictionary table 'mysql.tables' is rejected.` |
| File trong `lab/` | `t_inno.ibd`, `t4b.ibd`, `d1.ibd`, … (một `.ibd` cho mỗi bảng InnoDB); `t_myi.MYD`, `t_myi.MYI`, `t_myi_388.sdi`; **không có `.frm`** |
| File ở gốc datadir | `mysql.ibd` (data dictionary), `ibdata1`, `undo_001`, `undo_002`, thư mục `#innodb_redo`, `#ib_16384_0.dblwr`, `binlog.000001` |
| `ibd2sdi lab/t_inno.ibd` | JSON gồm hai đối tượng: `Table t_inno` (cột `id`, **`DB_TRX_ID`**, **`DB_ROLL_PTR`**) và `Tablespace lab/t_inno`; `mysqld_version_id` 80042 |

**Diễn giải và đối chiếu.**

- A, B khớp tr. 11: DDL commit ngầm; `AUTOCOMMIT=0` nghĩa là luôn ở trong transaction.
- C khớp tr. 12, kể cả câu cảnh báo; và đúng như sách, rollback "thành công" vẫn để lại dữ liệu MyISAM.
- D khớp tr. 18: câu DDL thất bại không để lại thay đổi nào. Kết hợp với A: atomic DDL **không** có nghĩa DDL tham gia transaction.
- File trên đĩa chính xác hóa câu tr. 16: `.frm` biến mất, nhưng `.ibd` **vẫn còn** và chứa SDI bên trong; file `.sdi` riêng chỉ có cho bảng không phải InnoDB (MyISAM ở đây; các bảng CSV `general_log`, `slow_log` trong schema `mysql` cũng có). Dictionary thật nằm trong `mysql.ibd` và không truy vấn trực tiếp được.
- Hai cột ẩn `DB_TRX_ID`, `DB_ROLL_PTR` trong SDI là dữ liệu "lưu thêm cho mỗi hàng" mà MVCC cần (tr. 14).

**Dọn dẹp TN4.** `DROP TABLE lab.t_inno, lab.t_myi, lab.t4b, lab.d1;`

---

<a id="don-dep"></a>

## Dọn dẹp chung

```sql
DROP DATABASE lab;
SET GLOBAL innodb_deadlock_detect = ON;   -- nếu đã tắt ở TN3
```

Rồi tắt instance thử nghiệm: `mysqladmin -h127.0.0.1 -P3311 -uroot shutdown`. Instance của lần chạy này đã được tắt sau khi đo. Thư mục dữ liệu có thể xóa nếu không cần chạy lại.

**Chưa thực thi (ngoài phạm vi môi trường):** thread pool plug-in (cần MySQL Enterprise hoặc Percona Server), replication nhiều node (xem thực hành của [Chương 9](../chapter-09.md)), backup nóng bằng XtraBackup ([Chương 10](../chapter-10.md#s2-6-3)).
