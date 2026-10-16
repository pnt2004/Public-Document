# Thực hành — Chương 9: Replication

Về ghi chú chương: [../chapter-09.md](../chapter-09.md) · Đáp án tự kiểm tra: [../answers/chapter-09.md](../answers/chapter-09.md)

> **Quy ước.** "Kết quả đã đo" là output thật của lần chạy ngày 2026-10-16, đã rút gọn khi ghi "(đã rút gọn)". "Dự đoán" là điều kỳ vọng viết **trước** khi chạy. Dữ liệu, tình huống và mọi SQL ở đây là **Ví dụ minh họa do AI xây dựng**, trừ chỗ ghi trang sách.

<a id="moi-truong"></a>

## Môi trường

| Mục | Giá trị |
| --- | --- |
| MySQL | Community Server **8.0.42** for Win64 (`mysqld --version`) |
| Máy | Laptop Windows 11, SSD; **nhiều nhiễu, không phải benchmark production** |
| Instance | Ba instance riêng, mỗi cái một datadir, khởi tạo bằng `--initialize-insecure`, chạy `--no-defaults`, `--bind-address=127.0.0.1`. Không đụng MySQL của người dùng ở cổng 3306 |
| Vai trò ban đầu | **3319** source (`server_id=1`), **3333** replica 1 (`server_id=2`), **3334** replica 2 (`server_id=3`). Vai trò đổi ở TN5 |
| Tùy chọn thêm | `--gtid-mode=ON --enforce-gtid-consistency=ON` cho cả ba; còn lại là **mặc định** của 8.0.42 |
| Dữ liệu | Tổng hợp bằng recursive CTE và stored procedure: 10.000 hàng (`misc.fmt`), 200.000 hàng (`misc.big`), vài nghìn transaction nhỏ (`misc.small`) |

### Dựng ba instance (dùng lại cho mọi thí nghiệm)

```bash
BIN="C:/Program Files/MySQL/MySQL Server 8.0/bin"
LAB="<thư mục thí nghiệm riêng>/ch09"     # ví dụ trong scratchpad, KHÔNG dùng datadir của server thật

for n in source replica1 replica2; do
  mkdir -p "$LAB/$n"
  "$BIN/mysqld" --no-defaults --initialize-insecure --datadir="$LAB/$n/data" --console > "$LAB/$n/init.log" 2>&1
done

# Mỗi lệnh dưới chạy nền trong một cửa sổ hay tiến trình riêng
"$BIN/mysqld" --no-defaults --datadir="$LAB/source/data"   --port=3319 --mysqlx=OFF --bind-address=127.0.0.1 --socket=ch09src.sock --server-id=1 --gtid-mode=ON --enforce-gtid-consistency=ON --console > "$LAB/source/run1.log" 2>&1
"$BIN/mysqld" --no-defaults --datadir="$LAB/replica1/data" --port=3333 --mysqlx=OFF --bind-address=127.0.0.1 --socket=ch09r1.sock  --server-id=2 --gtid-mode=ON --enforce-gtid-consistency=ON --console > "$LAB/replica1/run1.log" 2>&1
"$BIN/mysqld" --no-defaults --datadir="$LAB/replica2/data" --port=3334 --mysqlx=OFF --bind-address=127.0.0.1 --socket=ch09r2.sock  --server-id=3 --gtid-mode=ON --enforce-gtid-consistency=ON --console > "$LAB/replica2/run1.log" 2>&1

# Kiểm tra đúng instance trước khi làm gì
for p in 3319 3333 3334; do "$BIN/mysql" -h127.0.0.1 -P$p -uroot -N -e "SELECT @@port, @@server_id, @@server_uuid, @@gtid_mode, @@datadir"; done
```

Hai script phụ (đặt trong `$LAB`, không nằm trong datadir):

```bash
# q.sh PORT "SQL" [tùy chọn mysql]
P=$1; shift; S=$1; shift
"C:/Program Files/MySQL/MySQL Server 8.0/bin/mysql" -h127.0.0.1 -P$P -uroot "$@" -e "$S"
```

```bash
# lagmon.sh PORT N KHOANG_GIAY: mỗi lần lấy mẫu in heartbeat lag, lag theo Performance Schema,
# các GTID đã nhận chưa thực thi, trạng thái hai thread và Seconds_Behind_Source (cột 11, 12, 33 của SHOW REPLICA STATUS)
P=$1; N=$2; I=$3; F=$(mktemp)
for i in $(seq 1 $N); do cat >> $F <<SQL
SELECT 'T', DATE_FORMAT(NOW(6),'%H:%i:%s.%f'), ROUND(TIMESTAMPDIFF(MICROSECOND, h.ts, NOW(6))/1e6,2),
 IFNULL((SELECT ROUND(MAX(TIMESTAMPDIFF(MICROSECOND, APPLYING_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP, NOW(6)))/1e6,2)
         FROM performance_schema.replication_applier_status_by_worker WHERE APPLYING_TRANSACTION <> ''),'-'),
 IFNULL(NULLIF(GTID_SUBTRACT((SELECT RECEIVED_TRANSACTION_SET FROM performance_schema.replication_connection_status LIMIT 1),
                             @@GLOBAL.gtid_executed),''),'-')
FROM misc.heartbeat h WHERE h.id=1;
SHOW REPLICA STATUS;
DO SLEEP($I);
SQL
done
"C:/Program Files/MySQL/MySQL Server 8.0/bin/mysql" -h127.0.0.1 -P$P -uroot -N -B < $F |
  awk -F'\t' '$1=="T"{t=$2" hb="$3" ps="$4" backlog="$5; next} NF>40{print t" io="$11" sql="$12" SBS="$33}'
rm -f $F
```

> **Lưu ý đo thời gian.** `NOW()` trả về thời điểm **bắt đầu câu lệnh**, nên `SELECT WAIT_FOR_EXECUTED_GTID_SET(...), NOW(6)` cho thời gian sai (lần chạy đầu của ghi chú đã mắc lỗi này và cho "0,02 giây"). Dùng `SYSDATE(6)` ở một câu lệnh **riêng** sau khi chờ xong.

---

<a id="tn1"></a>

## TN1 — Dựng replication GTID với auto-position, đọc trạng thái, tự nối lại, `super_read_only`

**Mục tiêu.** Kiểm các khẳng định ở tr. 229–233 và 250: replication gồm ba bước qua relay log; GTID ghép `server_uuid` với số tăng dần; replica tự biết bắt đầu từ đâu sau khi khởi động lại; `read_only` không chặn được người có quyền cao, `super_read_only` thì chặn. Đồng thời ghi lại mặc định của các biến "crash safe" trên 8.0.42.

**Giả thuyết (Dự đoán).**
- `CREATE DATABASE misc` sinh một GTID `<uuid của source>:N`; replica có cùng GTID sau khi áp dụng.
- Tắt replica, ghi thêm trên source, bật lại: replica tự lấy phần còn thiếu, không cần file hay vị trí.
- Với `read_only=ON`, root vẫn ghi được trên replica; với `super_read_only=ON` thì không.

**Chuẩn bị.**

```sql
-- Trên source (3319): user replication
CREATE USER 'repl'@'127.0.0.1' IDENTIFIED BY 'replpass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'127.0.0.1';

-- Trên mỗi replica (3333, 3334)
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='127.0.0.1', SOURCE_PORT=3319,
  SOURCE_USER='repl', SOURCE_PASSWORD='replpass',
  SOURCE_AUTO_POSITION=1,
  GET_SOURCE_PUBLIC_KEY=1;     -- user dùng caching_sha2_password, kết nối không TLS
START REPLICA;
```

**Các bước và lệnh đo.**
1. Ghi lại mặc định: `SELECT @@binlog_format, @@sync_binlog, @@innodb_flush_log_at_trx_commit, @@relay_log_info_repository, @@relay_log_recovery, @@replica_parallel_workers, @@replica_parallel_type, @@replica_preserve_commit_order, @@log_replica_updates;`
2. `SHOW REPLICA STATUS\G` trên 3333.
3. Trên source: `CREATE DATABASE misc;` rồi `SHOW BINLOG EVENTS IN 'binlog.000001' FROM 708;`
4. `mysqladmin -h127.0.0.1 -P3333 -uroot shutdown`; trên source tạo `misc.t` và chèn 4 hàng (mỗi lệnh một transaction); khởi động lại 3333; xem trạng thái và log lỗi.
5. Trên 3333: `SET GLOBAL read_only=ON;` rồi `INSERT` bằng root; sau đó `SET GLOBAL super_read_only=ON;` rồi `INSERT` lại.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, ba instance riêng cấu hình mặc định trừ GTID).

Mặc định (giống nhau trên cả ba server):

| Biến | Giá trị | Ghi chú |
| --- | --- | --- |
| `binlog_format` | ROW | |
| `sync_binlog` | 1 | |
| `innodb_flush_log_at_trx_commit` | 1 | |
| `relay_log_info_repository` | TABLE | đọc biến phát Warning 1287 (deprecated) |
| `relay_log_recovery` | **0 (OFF)** | sách khuyên ON (tr. 233) |
| `replica_parallel_workers` | 4 | |
| `replica_parallel_type` | LOGICAL_CLOCK | deprecated |
| `replica_preserve_commit_order` | 1 | |
| `log_replica_updates` | 1 | |
| `binlog_transaction_dependency_tracking` | COMMIT_ORDER | deprecated |
| `binlog_row_image` | FULL | |

`SHOW REPLICA STATUS` trên 3333 ngay sau khi chạy (đã rút gọn):

```text
             Replica_IO_State: Waiting for source to send event
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
        Seconds_Behind_Source: 0
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Retrieved_Gtid_Set: 88389ddb-c932-11f1-9b76-00090ffe0001:1-2
            Executed_Gtid_Set: 88389ddb-c932-11f1-9b76-00090ffe0001:1-2
                Auto_Position: 1
```

GTID 1–2 là `CREATE USER` và `GRANT`. Sự kiện trong binlog của source cho `CREATE DATABASE misc`:

```text
binlog.000001  708  Gtid   1  785  SET @@SESSION.GTID_NEXT= '88389ddb-c932-11f1-9b76-00090ffe0001:3'
binlog.000001  785  Query  1  893  CREATE DATABASE misc /* xid=34 */
```

Sau khi tắt 3333, source ghi thêm đến GTID 8. Bật lại 3333:

```text
[System] [MY-014002] [Repl] Replica receiver thread for channel '': connected to source 'repl@127.0.0.1:3319'
  with server_uuid=88389ddb-..., server_id=1. Starting GTID-based replication.
           Retrieved_Gtid_Set: 88389ddb-c932-11f1-9b76-00090ffe0001:1-8
            Executed_Gtid_Set: 88389ddb-c932-11f1-9b76-00090ffe0001:1-8
```

Bảng `misc.t` trên 3333 có đủ 4 hàng. Thử ghi trên replica:

| Thiết lập trên 3333 | `INSERT` bằng root | `gtid_executed` của 3333 sau đó |
| --- | --- | --- |
| `read_only=1, super_read_only=0` | **Thành công** | `88389ddb-...:1-8, 8abfb139-c932-11f1-9f56-00090ffe0001:1` |
| `read_only=1, super_read_only=1` | `ERROR 1290 (HY000): The MySQL server is running with the --super-read-only option` | không đổi; GTID 9 từ source vẫn được áp dụng bình thường |

**Diễn giải và đối chiếu với sách.**
- GTID ghép `server_uuid` của source với số tăng dần, đúng tr. 231. Replica chạy lại tự lấy GTID 4–8, đúng ví dụ tr. 232 và chú thích 3 về `SOURCE_AUTO_POSITION=1`.
- Ba trong bốn khuyến nghị crash-safe của sách (tr. 232–233) đã là mặc định của 8.0.42; `relay_log_recovery` thì không.
- `read_only` không chặn root, đúng tr. 250. Lần ghi nhầm để lại một GTID mang `server_uuid` của replica (**Bổ sung ngoài sách:** *errant transaction*). GTID này còn theo server 3333 tới cuối buổi và được dùng lại ở [TN5](#tn5).

**Dọn dẹp và khôi phục.** Giữ topology cho các thí nghiệm sau. Bật `super_read_only` cho cả hai replica (`SET GLOBAL super_read_only=ON`). Hàng id 100 ghi nhầm được giữ lại có chủ đích.

---

<a id="tn2"></a>

## TN2 — ROW và STATEMENT: kích thước binlog, `mysqlbinlog`, lệch thầm lặng, filter

**Mục tiêu.** Kiểm tr. 230–231 (row-based tốn binlog hơn; statement-based gặp vấn đề với truy vấn không tất định), tr. 250 (`DELETE ... LIMIT` không `ORDER BY`), tr. 238 và 252 (`*_do_db` lọc theo database mặc định).

**Giả thuyết (Dự đoán).**
- Cùng câu `UPDATE` 10.000 hàng: STATEMENT vài trăm byte, ROW vài trăm KB.
- Nếu replica có access path khác (thiếu một index), `DELETE ... LIMIT 10` ở STATEMENT xóa hàng khác trên replica **mà không báo lỗi**; ở ROW thì xóa đúng cùng hàng.
- Với `REPLICATE_DO_DB=(misc)`, câu `INSERT INTO misc.t` chạy khi đang `USE other` bị lọc ở STATEMENT.

**Chuẩn bị.**

```sql
-- Trên source (3319)
SET SESSION cte_max_recursion_depth = 20000;
CREATE TABLE misc.fmt (
  id INT PRIMARY KEY,
  k  INT NOT NULL,          -- k = 10001 - id: thứ tự theo k ngược với thứ tự theo id
  v  VARCHAR(20),
  INDEX idx_k (k)
);
INSERT INTO misc.fmt (id, k, v)
WITH RECURSIVE seq(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 10000)
SELECT n, 10001 - n, 'init' FROM seq;

-- Trên replica 2 (3334): cố ý tạo lệch schema, KHÔNG ghi binlog để không sinh GTID lạ
SET GLOBAL super_read_only = OFF;
SET SESSION sql_log_bin = 0;
ALTER TABLE misc.fmt DROP INDEX idx_k;
SET SESSION sql_log_bin = 1;
SET GLOBAL super_read_only = ON;
```

**Các bước và lệnh đo.**

*Phần A — kích thước binlog.* Trên source, ghi vị trí binlog trước và sau mỗi câu:

```sql
SHOW MASTER STATUS;
SET SESSION binlog_format = 'STATEMENT';  UPDATE misc.fmt SET v = CONCAT('s', id);  SHOW MASTER STATUS;
SET SESSION binlog_format = 'ROW';        UPDATE misc.fmt SET v = CONCAT('r', id);  SHOW MASTER STATUS;
SET SESSION binlog_row_image = 'MINIMAL'; UPDATE misc.fmt SET v = CONCAT('m', id);  SHOW MASTER STATUS;
```

Giải mã bằng `mysqlbinlog --start-position=... --stop-position=... -v --base64-output=DECODE-ROWS "$LAB/source/data/binlog.000001"`.

*Phần B — lệch thầm lặng.* Xem access path, rồi xóa ở STATEMENT và so sánh:

```sql
EXPLAIN DELETE FROM misc.fmt WHERE k BETWEEN 1 AND 500 LIMIT 10;          -- trên source
EXPLAIN SELECT * FROM misc.fmt WHERE k BETWEEN 1 AND 500 LIMIT 10;        -- trên replica (super_read_only chặn EXPLAIN DELETE)

SET SESSION binlog_format = 'STATEMENT';
DELETE FROM misc.fmt WHERE k BETWEEN 1 AND 500 LIMIT 10;                  -- trên source, kèm --show-warnings

-- Trên cả ba server
SELECT COUNT(*), SUM(id), BIT_XOR(CRC32(CONCAT_WS('#', id, k, v))) FROM misc.fmt;
CHECKSUM TABLE misc.fmt;
```

Lặp lại câu `DELETE` với `binlog_format='ROW'`. Sau đó `UPDATE misc.fmt SET v='x' WHERE id=9505;` (hàng mà replica 2 đã xóa nhầm).

*Phần D — filter* (chạy ở **cuối buổi**, sau TN5, trên server 3333 lúc đó là replica của 3319):

```sql
-- Trên replica 3333
STOP REPLICA;
CHANGE REPLICATION FILTER REPLICATE_DO_DB = (misc);
START REPLICA;

-- Trên source 3319
CREATE DATABASE other;
USE other; SET SESSION binlog_format='STATEMENT'; INSERT INTO misc.t VALUES (20, 'a) STATEMENT, USE other');
           SET SESSION binlog_format='ROW';       INSERT INTO misc.t VALUES (21, 'b) ROW, USE other');
USE misc;  SET SESSION binlog_format='STATEMENT'; INSERT INTO misc.t VALUES (22, 'c) STATEMENT, USE misc');
           SET SESSION binlog_format='ROW';
USE other; CREATE TABLE misc.ddl_from_other (id INT PRIMARY KEY);
USE misc;  CREATE TABLE other.ddl_from_misc (id INT PRIMARY KEY);
```

**Kết quả đã đo** (MySQL 8.0.42, dữ liệu tổng hợp 10.000 hàng).

*Phần A:*

| Định dạng | Vị trí binlog trước → sau | Byte cho một `UPDATE` 10.000 hàng |
| --- | --- | --- |
| STATEMENT | 143.625 → 143.943 | **318** |
| ROW, `binlog_row_image=FULL` | 143.943 → 443.312 | **299.369** |
| ROW, `binlog_row_image=MINIMAL` | 443.312 → 562.995 | **119.683** |

Mỗi lệnh `SET SESSION binlog_format` phát `Warning 1287: '@@binlog_format' is deprecated`. Giải mã (đã rút gọn):

```text
# STATEMENT: một event Query chứa nguyên câu lệnh
#261016 14:25:46 server id 1 ... GTID last_committed=11 sequence_number=12 rbr_only=no ... transaction_length=318
SET @@SESSION.GTID_NEXT= '88389ddb-...:12'
UPDATE misc.fmt SET v = CONCAT('s', id)

# ROW FULL: Table_map rồi nhiều event Update_rows (mỗi event khoảng 8 KB); 10.000 khối như sau
### UPDATE `misc`.`fmt`
### WHERE
###   @1=1
###   @2=10000
###   @3='s1'
### SET
###   @1=1
###   @2=10000
###   @3='r1'

# ROW MINIMAL: chỉ khóa chính ở ảnh trước, chỉ cột đổi ở ảnh sau
### UPDATE `misc`.`fmt`
### WHERE
###   @1=1
### SET
###   @3='m1'
```

*Phần B:*

```text
-- EXPLAIN trên source (3319) và replica 1 (3333): type=range, key=idx_k, rows=500
-- EXPLAIN trên replica 2 (3334):                  type=ALL,   key=NULL,  rows=9948

Note (Code 1592): Unsafe statement written to the binary log using statement format since
BINLOG_FORMAT = STATEMENT. The statement is unsafe because it uses a LIMIT clause. ...
```

| Server | COUNT | SUM(id) | id bị xóa | `CHECKSUM TABLE` | Trạng thái replication |
| --- | --- | --- | --- | --- | --- |
| 3319 (source) | 9.990 | 49.905.045 | 9991–10000 | 3768461462 | — |
| 3333 (cùng index) | 9.990 | 49.905.045 | 9991–10000 | 3768461462 | Yes / Yes, errno 0 |
| 3334 (thiếu index) | 9.990 | **49.909.945** | **9501–9510** | **4081622650** | **Yes / Yes, errno 0** |

Cùng câu `DELETE` ở ROW: cả ba server xóa **id 9981–9990**, kể cả 3334. Sau đó `UPDATE ... WHERE id=9505` ở ROW làm 3334 dừng:

```text
Replica_SQL_Running: No
Last_SQL_Errno: 1032
Last_SQL_Error: Coordinator stopped because there were error(s) in the worker(s). ... Worker 1 failed executing
  transaction '88389ddb-...:17' ... See error log and/or performance_schema.replication_applier_status_by_worker ...
-- trong replication_applier_status_by_worker:
Could not execute Update_rows event on table misc.fmt; Can't find record in 'fmt', Error_code: 1032 ...
Seconds_Behind_Source: NULL
```

Sửa tay trên 3334 (với `sql_log_bin=0`): chèn lại 10 hàng 9501–9510 lấy giá trị từ source, tạo lại `idx_k`, `START REPLICA`. Checksum vẫn lệch (9.990 hàng so với 9.980) vì 3334 **còn thừa** id 9991–10000 mà source đã xóa. Xóa nốt 10 hàng đó thì cả ba server có `CHECKSUM TABLE` = 3404726626.

*Phần D* (replica 3333 với `REPLICATE_DO_DB = (misc)`):

| Câu trên source | Database mặc định | Định dạng | Trên 3333 |
| --- | --- | --- | --- |
| `INSERT ... (20, ...)` vào `misc.t` | other | STATEMENT | **không có** |
| `INSERT ... (21, ...)` vào `misc.t` | other | ROW | có |
| `INSERT ... (22, ...)` vào `misc.t` | misc | STATEMENT | có |
| `CREATE TABLE misc.ddl_from_other` | other | DDL | **không có** |
| `CREATE TABLE other.ddl_from_misc` | misc | DDL | **SQL thread dừng** |

```text
LAST_ERROR_NUMBER: 1049
Worker 1 failed executing transaction '88389ddb-...:32102' ...; Error 'Unknown database 'other'' on query.
Default database: 'misc'. Query: 'CREATE TABLE other.ddl_from_misc (id INT PRIMARY KEY)'
```

(`CREATE DATABASE other` chạy khi không có database mặc định nên đã bị lọc.)

**Diễn giải và đối chiếu với sách.**
- Tỉ lệ khoảng 940 lần giữa ROW FULL và STATEMENT minh họa nhận xét "binlog có thể phình rất lớn" (tr. 231). Con số phụ thuộc độ rộng hàng và số hàng; đây là bảng hẹp.
- Phần B tái hiện đúng kịch bản tr. 230 và tr. 250: thứ tự hàng mà **physical execution** gặp phụ thuộc access path (range trên `idx_k` so với full scan theo khóa chính), nên `LIMIT` không `ORDER BY` chọn hàng khác nhau. Replica không có cách nào biết mình lệch; số hàng còn bằng nhau. Lệch schema ở đây là do cố ý tạo ra; ngoài thực tế nó có thể đến từ ghi nhầm vào replica hay migration chạy không đều.
- ROW không làm lệch ở câu `DELETE` thứ hai, và khi gặp hàng đã lệch thì **dừng với lỗi** (1032) thay vì im lặng. Đó là một ưu điểm vận hành: lệch bị lộ ra.
- Sửa lệch phải xử lý hai phía (thiếu và thừa). Sách khuyên **dựng lại replica** (tr. 250); sửa tay chỉ hợp lý trong phòng thí nghiệm.
- Phần D xác nhận tr. 238 cho câu lệnh ở định dạng STATEMENT và cho DDL. Với DML ở ROW, bộ lọc xét database của bảng (**Bổ sung ngoài sách**, khớp tài liệu MySQL). Lỗi 1049 minh họa câu "có loại câu lệnh còn làm replication dừng với lỗi vì phụ thuộc không được replicate" (tr. 252).

**Dọn dẹp và khôi phục.** Phần B: đã sửa 3334 như trên và kiểm checksum bằng nhau. Phần D: không khôi phục (cuối buổi, các instance được tắt ngay sau đó). Muốn bỏ filter: `STOP REPLICA; CHANGE REPLICATION FILTER REPLICATE_DO_DB = (); START REPLICA;` rồi dựng lại replica vì nó đã thiếu dữ liệu.

---

<a id="tn3"></a>

## TN3 — Tạo lag và đo bằng bốn cách; transaction dài; song song và durability

**Mục tiêu.** Kiểm tr. 248–249 (các điểm yếu của `Seconds_Behind_Source`, heartbeat là cách đo tốt nhất), tr. 234–236 (song song phụ thuộc group commit; đo mức bận từng worker), tr. 253 (tạm giảm durability để bắt kịp).

**Giả thuyết (Dự đoán).**
- Dừng SQL thread hay I/O thread: `Seconds_Behind_Source` = `NULL`; heartbeat vẫn tăng đều theo thời gian.
- Transaction mở lâu trên source làm `Seconds_Behind_Source` nhảy lên cỡ thời gian mở transaction (tr. 249).
- Workload tuần tự không có song song cho `LOGICAL_CLOCK`; workload nhiều session thì có.
- Giảm durability trên replica rút ngắn thời gian bắt kịp đáng kể với transaction nhỏ.

**Chuẩn bị.**

```sql
-- Trên source: heartbeat kiểu pt-heartbeat, cập nhật mỗi giây bằng event scheduler (mặc định ON trên 8.0.42)
CREATE TABLE misc.heartbeat (id INT PRIMARY KEY, ts DATETIME(6) NOT NULL, server_id INT UNSIGNED NOT NULL);
INSERT INTO misc.heartbeat VALUES (1, NOW(6), @@server_id);
CREATE EVENT misc.hb ON SCHEDULE EVERY 1 SECOND
  DO UPDATE misc.heartbeat SET ts = NOW(6), server_id = @@server_id WHERE id = 1;

-- Bảng lớn cho ghi nặng
SET SESSION cte_max_recursion_depth = 300000;
CREATE TABLE misc.big (id INT PRIMARY KEY, c INT NOT NULL, pad CHAR(100) NOT NULL);
INSERT INTO misc.big
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM s WHERE n < 200000)
SELECT n, 0, REPEAT('x', 100) FROM s;

-- Sinh nhiều transaction nhỏ (autocommit: mỗi INSERT là một transaction)
CREATE TABLE misc.small (id INT AUTO_INCREMENT PRIMARY KEY, run INT NOT NULL, v INT NOT NULL);
DELIMITER //
CREATE PROCEDURE misc.gen_small(IN p_run INT, IN p_n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < p_n DO
    INSERT INTO misc.small (run, v) VALUES (p_run, i);
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
```

Event được replicate sang replica với trạng thái `SLAVESIDE_DISABLED` (đã kiểm trong `information_schema.EVENTS`), nên chỉ source cập nhật heartbeat.

**Các bước và lệnh đo.**
- **A.** Trên 3333: `STOP REPLICA SQL_THREAD`; ghi 1 transaction trên source; `lagmon.sh 3333 6 2`; `START REPLICA SQL_THREAD`; `lagmon.sh 3333 4 0.5`. Lặp lại với `STOP REPLICA IO_THREAD`.
- **B.** Chạy `lagmon.sh 3333 30 0.3` nền; trên source `UPDATE misc.big SET c = c + 1;` (một transaction 200.000 hàng).
- **C.** Chạy `lagmon.sh 3333 75 0.3` nền; trên source trong một session: `BEGIN; UPDATE misc.big SET c = c + 1; DO SLEEP(15); COMMIT;`. Làm hai lần: lần 1 với `replica_parallel_workers=4` (mặc định), lần 2 với `replica_parallel_workers=0` (đổi bằng `STOP REPLICA SQL_THREAD; SET GLOBAL replica_parallel_workers=0; START REPLICA SQL_THREAD;`).
- **D.** Song song: `CALL misc.gen_small(0, 20)` một session, rồi bốn session đồng thời mỗi session `CALL misc.gen_small(r, 300)`; đếm `last_committed` trong `mysqlbinlog`. Trên 3333, đếm transaction mỗi worker (bản đã sửa của listing tr. 236):

```sql
SELECT w.WORKER_ID, s.COUNT_STAR
FROM performance_schema.events_transactions_summary_by_thread_by_event_name AS s
JOIN performance_schema.replication_applier_status_by_worker AS w ON w.THREAD_ID = s.THREAD_ID
ORDER BY w.WORKER_ID;
```

- **E.** Durability: với mỗi cấu hình (`sync_binlog`, `innodb_flush_log_at_trx_commit`) = (1, 1) và (0, 0), làm 3 lần xen kẽ: `STOP REPLICA SQL_THREAD`; đặt cấu hình trên 3333; source `CALL misc.gen_small(run, 2000)`; lấy `@@gtid_executed` của source vào `$G`; trên 3333:

```sql
SET @t0 = SYSDATE(6);
START REPLICA SQL_THREAD;
SELECT WAIT_FOR_EXECUTED_GTID_SET('$G', 300);
SELECT ROUND(TIMESTAMPDIFF(MICROSECOND, @t0, SYSDATE(6)) / 1e6, 3);
```

**Kết quả đã đo** (MySQL 8.0.42; hb = heartbeat lag, ps = lag theo Performance Schema, backlog = GTID đã nhận chưa thực thi, SBS = `Seconds_Behind_Source`; đã rút gọn).

*A — dừng SQL thread:*

```text
14:28:03.96 hb=0.51  ps=- backlog=...:55       io=Yes sql=No  SBS=NULL
14:28:07.98 hb=4.53  ps=- backlog=...:55-59    io=Yes sql=No  SBS=NULL
14:28:13.99 hb=10.54 ps=- backlog=...:55-65    io=Yes sql=No  SBS=NULL
-- START REPLICA SQL_THREAD
14:28:16.66 hb=0.07  ps=- backlog=-            io=Yes sql=Yes SBS=13
14:28:17.17 hb=0.58  ps=- backlog=-            io=Yes sql=Yes SBS=0
```

*A — dừng I/O thread:*

```text
14:30:33.80 hb=0.88 ps=- backlog=- io=No sql=Yes SBS=NULL
14:30:37.83 hb=4.91 ps=- backlog=- io=No sql=Yes SBS=NULL
-- START REPLICA IO_THREAD
14:30:40.30 hb=2.32 ps=1.31 backlog=...:212-213 io=Yes sql=Yes SBS=7
14:30:40.80 hb=0.8  ps=-    backlog=-           io=Yes sql=Yes SBS=0
```

*B — `UPDATE` 200.000 hàng.* Source: bắt đầu 14:28:31.51, xong 14:28:33.80 (2,3 giây).

```text
14:28:33.73 hb=0.97 ps=-    backlog=-          SBS=0
14:28:34.04 hb=1.28 ps=0.4  backlog=-          SBS=3
14:28:35.29 hb=2.53 ps=1.65 backlog=...:85-87  SBS=4
14:28:37.16 hb=4.4  ps=3.52 backlog=...:85-89  SBS=6
14:28:37.47 hb=4.71 ps=3.83 backlog=...:85-89  SBS=6
14:28:37.77 hb=0.98 ps=-    backlog=-          SBS=3
14:28:38.08 hb=0.28 ps=-    backlog=-          SBS=4
14:28:38.39 hb=0.59 ps=-    backlog=-          SBS=0
```

*C — transaction mở lâu:*

| Applier | `UPDATE` xong trên source | `COMMIT` trên source | SBS lớn nhất | hb lớn nhất | ps lớn nhất |
| --- | --- | --- | --- | --- | --- |
| 4 worker (mặc định) | 14:28:53.69 | 14:29:08.87 | **5** | 4,92 | 4,3 |
| 1 luồng (`replica_parallel_workers=0`) | 14:29:53.10 | 14:30:08.31 | **21** | 4,64 | 4,16 |

```text
-- 1 luồng (đã rút gọn)
14:30:08.52 hb=0.89 ps=0.42 SBS=18
14:30:10.09 hb=2.46 ps=1.98 SBS=20
14:30:11.97 hb=4.34 ps=3.86 SBS=21
14:30:12.27 hb=4.64 ps=4.16 SBS=4
14:30:14.14 hb=0.45 ps=-    SBS=0
```

*D — song song:*

```text
-- một session, 20 transaction (mysqlbinlog)
GTID last_committed=224 sequence_number=225
GTID last_committed=225 sequence_number=226
GTID last_committed=226 sequence_number=227      ... (mỗi transaction chờ transaction trước)

-- bốn session đồng thời, 1.201 transaction (gồm heartbeat)
transactions=1201 distinct_last_committed=668 max_txn_per_last_committed=3 avg=1.7979
```

| Workload trên source | Worker 1 | Worker 2 | Worker 3 | Worker 4 |
| --- | --- | --- | --- | --- |
| 4 session × 300 transaction | 356 | 325 | 270 | 255 |
| 1 session × 1.200 transaction | **1.200** | 0 | 0 | 0 |

Instrument `transaction` (ENABLED, TIMED) và consumer `events_transactions_current`, `events_transactions_history` đã bật sẵn trên 8.0.42.

*E — thời gian bắt kịp 2.000 transaction một hàng:*

| Cấu hình replica | Lần 1 | Lần 2 | Lần 3 | Trung vị |
| --- | --- | --- | --- | --- |
| `sync_binlog=1`, `innodb_flush_log_at_trx_commit=1` | 4,204 s | 4,299 s | 4,105 s | **4,204 s** |
| `sync_binlog=0`, `innodb_flush_log_at_trx_commit=0` | 0,658 s | 0,596 s | 0,786 s | **0,658 s** |

Sau phần E, replica được đặt lại `sync_binlog=1`, `innodb_flush_log_at_trx_commit=1`.

**Diễn giải và đối chiếu với sách.**
- `NULL` khi thread không chạy: đúng tr. 248. Khi I/O thread dừng, replica **không biết** mình thiếu gì (backlog trống) và `Seconds_Behind_Source` không nói gì; chỉ heartbeat (hoặc so với `gtid_executed` của source) cho thấy độ trễ 5 giây.
- Ngay sau khi bắt kịp, `Seconds_Behind_Source` còn báo 13 rồi 3, 4 giây dù heartbeat dưới 1 giây: số liệu có thể **cũ** (gần với ý "không phải lúc nào cũng chính xác", tr. 248).
- Khẳng định về transaction dài (tr. 249) **đúng với applier một luồng**: SBS 18–21 giây trong khi dữ liệu chỉ trễ 1–4,6 giây. Với applier 4 luồng mặc định, không thấy hiện tượng này. Giả thuyết (**Bổ sung ngoài sách**, chưa đối chiếu mã nguồn): applier một luồng lấy mốc từ timestamp của từng event (lúc câu `UPDATE` bắt đầu), applier đa luồng lấy mốc theo nhóm transaction ở checkpoint (gần lúc `COMMIT`). Ở phần B (câu tự commit), hai mốc trùng nhau nên SBS lớn hơn ps khoảng 2,3 giây, đúng bằng thời gian câu chạy trên source.
- Heartbeat có độ phân giải 1 giây (cập nhật mỗi giây), nên giá trị dao động 0–1 giây khi không lag. Ngưỡng cảnh báo nên đặt lớn hơn hẳn 1 giây.
- Phần D minh họa hộp group commit (tr. 234–235) và cách đo của tr. 236: song song trên replica bị giới hạn bởi song song trên source. Con số 1.200 / 0 / 0 / 0 là lý do thêm worker không giúp workload tuần tự.
- Phần E: giảm durability nhanh khoảng 6 lần khi nút thắt là `fsync` mỗi commit (tr. 253). Kết quả này **không** cho phép kết luận gì về workload có transaction lớn hay bị giới hạn bởi CPU.

**Dọn dẹp và khôi phục.** Đặt lại `replica_parallel_workers=4`, `sync_binlog=1`, `innodb_flush_log_at_trx_commit=1` trên 3333 (đã làm và kiểm). Event `misc.hb` được giữ tới TN4 rồi `DROP EVENT misc.hb` ở đầu TN5.

---

<a id="tn4"></a>

## TN4 — Delayed replica cứu một `DROP TABLE` nhầm

**Mục tiêu.** Tái hiện kịch bản tr. 233: tìm GTID của `DROP TABLE` và cho delayed replica chạy tới ngay trước đó; kiểm các cột giám sát delayed replica (tr. 248).

**Giả thuyết (Dự đoán).** Với `SOURCE_DELAY=45`, bảng bị xóa trên source vẫn còn trên replica trong 45 giây; `START REPLICA SQL_THREAD UNTIL SQL_BEFORE_GTIDS` dừng đúng trước `DROP`, giữ cả các hàng ghi sau khi bật delay.

**Chuẩn bị.**

```sql
-- Trên source: bảng "quan trọng" có trước khi bật delay
CREATE TABLE misc.important (id INT PRIMARY KEY, note VARCHAR(50));
INSERT INTO misc.important VALUES (1, 'du lieu truoc khi tre');

-- Trên replica 2 (3334)
STOP REPLICA SQL_THREAD;
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 45;
START REPLICA SQL_THREAD;
```

**Các bước và lệnh đo.**

```sql
-- Source: tách binlog mới cho dễ tìm, ghi thêm, rồi "lỡ tay" xóa bảng
FLUSH BINARY LOGS;
INSERT INTO misc.important VALUES (2, 'ghi sau khi bat delay');
-- ... khoảng 5 giây sau
DROP TABLE misc.important;
SHOW BINLOG EVENTS IN 'binlog.000002';           -- tìm GTID của DROP

-- Replica 2: kiểm trạng thái delay
SHOW REPLICA STATUS\G                            -- SQL_Delay, SQL_Remaining_Delay, Replica_SQL_Running_State

-- Replica 2: chạy tới ngay trước DROP
STOP REPLICA SQL_THREAD;
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 0;
START REPLICA SQL_THREAD UNTIL SQL_BEFORE_GTIDS = '88389ddb-c932-11f1-9b76-00090ffe0001:29629';
```

Rồi `mysqldump --set-gtid-purged=OFF misc important` từ 3334, nạp vào source, và `START REPLICA SQL_THREAD` trên 3334.

**Kết quả đã đo** (MySQL 8.0.42).

```text
-- SHOW BINLOG EVENTS trên source (đã rút gọn)
binlog.000002  2173  Gtid   1  2250  SET @@SESSION.GTID_NEXT= '88389ddb-c932-11f1-9b76-00090ffe0001:29629'
binlog.000002  2250  Query  1  2387  DROP TABLE `misc`.`important` /* generated by server */ /* xid=88393 */

-- Replica 2 khoảng 9 giây sau khi bật delay
          Replica_SQL_Running: Yes
        Seconds_Behind_Source: 8
                    SQL_Delay: 45
          SQL_Remaining_Delay: 38
    Replica_SQL_Running_State: Waiting until SOURCE_DELAY seconds after source executed event
-- heartbeat lag lúc đó: 9,3 giây; misc.important trên replica: chỉ hàng 1

-- Sau START ... UNTIL
          Replica_SQL_Running: No
              Until_Condition: SQL_BEFORE_GTIDS
                    SQL_Delay: 0
            Executed_Gtid_Set: 88389ddb-c932-11f1-9b76-00090ffe0001:1-29628
-- misc.important trên replica: hàng 1 và 2
-- mysqldump: INSERT INTO `important` VALUES (1,'du lieu truoc khi tre'),(2,'ghi sau khi bat delay');
```

Sau khi nạp dump vào source và cho replica 2 chạy tiếp, cả ba server có `misc.important` với id 1, 2; replica 2 chạy bình thường (`SQL_Delay: 0`, không lỗi).

**Diễn giải và đối chiếu với sách.**
- Đúng kịch bản tr. 233: dữ liệu còn trên delayed replica, và GTID cho phép dừng **chính xác** trước câu lệnh hỏng.
- `SQL_Remaining_Delay`, `SQL_Delay` và trạng thái "Waiting until SOURCE_DELAY…" là những thứ cần giám sát để biết delayed replica trễ **đúng** mức (tr. 248). `Seconds_Behind_Source` và heartbeat của delayed replica luôn lớn; hệ cảnh báo lag phải có ngoại lệ cho nó (tr. 233–234).
- **Bổ sung ngoài sách.** Thời gian 45 giây ở đây chỉ để thí nghiệm ngắn. Delay thực tế (giờ, ngày) phải dài hơn thời gian **phát hiện** lỗi; nếu lỗi được phát hiện sau khi hết delay thì delayed replica cũng đã áp dụng lỗi.

**Dọn dẹp và khôi phục.** Đặt `SOURCE_DELAY=0` (đã làm), bảng đã khôi phục trên source và lan ra mọi server.

---

<a id="tn5"></a>

## TN5 — Failover có kế hoạch, semisync và failover ngoài kế hoạch

**Mục tiêu.** Làm đúng bảy bước planned promotion (tr. 240); kiểm semisync thêm latency và rơi về bất đồng bộ khi hết timeout (tr. 237); làm unplanned promotion (tr. 240) và đo phần dữ liệu thiếu (tr. 240–241); thử các lỗi server ID (tr. 251–252).

**Giả thuyết (Dự đoán).**
- Planned: không mất transaction nào; cửa sổ chặn ghi ngắn.
- Semisync: latency mỗi commit tăng; khi mọi replica ngừng ACK, commit chờ đúng `rpl_semi_sync_source_timeout` rồi **thành công**.
- Unplanned sau khi semisync đã rơi về async: source cũ có transaction mà source mới không có; trỏ source cũ về sẽ lỗi nếu khóa trùng.

**Chuẩn bị.** Topology sau TN4: 3319 source; 3333, 3334 replica. Bỏ event heartbeat và tạo bảng có `AUTO_INCREMENT`:

```sql
DROP EVENT misc.hb;
CREATE TABLE misc.orders (id INT AUTO_INCREMENT PRIMARY KEY, src_port INT NOT NULL, note VARCHAR(60) NOT NULL);
INSERT INTO misc.orders (src_port, note) VALUES (@@port, 'truoc switchover');
```

### Phần A — planned promotion từ 3319 sang 3334

```bash
# B0: tìm GTID lạ trên các ứng viên
S=$(q.sh 3319 "SELECT @@gtid_executed" -N | sed 's/\\n//g')
q.sh 3333 "SELECT GTID_SUBTRACT(@@gtid_executed, '$S') errant, GTID_SUBTRACT('$S', @@gtid_executed) missing"
q.sh 3334 "SELECT GTID_SUBTRACT(@@gtid_executed, '$S') errant, GTID_SUBTRACT('$S', @@gtid_executed) missing"
```

```sql
-- B2: lag trên đích (3334):            SHOW REPLICA STATUS\G
-- B3: trên source cũ (3319):           SET GLOBAL super_read_only = ON;
-- B4: trên đích, với $S mới lấy lại:   SELECT WAIT_FOR_EXECUTED_GTID_SET('$S', 10); SELECT GTID_SUBSET('$S', @@gtid_executed);
-- B5: trên đích:                       STOP REPLICA; RESET REPLICA ALL; SET GLOBAL read_only = OFF;
-- B6: "ứng dụng" ghi vào đích:         INSERT INTO misc.orders (src_port, note) VALUES (@@port, 'sau switchover');
-- B7: trên 3333 và 3319:
STOP REPLICA;
CHANGE REPLICATION SOURCE TO SOURCE_HOST='127.0.0.1', SOURCE_PORT=3334, SOURCE_USER='repl',
  SOURCE_PASSWORD='replpass', SOURCE_AUTO_POSITION=1, GET_SOURCE_PUBLIC_KEY=1;
START REPLICA;
```

**Kết quả đã đo.**

```text
B0  3333: errant = 8abfb139-c932-11f1-9f56-00090ffe0001:1   missing = (rỗng)
    3334: errant = (rỗng)                                   missing = (rỗng)      → chọn 3334
B2  3334: Running Yes/Yes, Seconds_Behind_Source 0
B3  t3 = 14:35:05.558855, read_only=1, super_read_only=1
    INSERT của ứng dụng vào 3319: ERROR 1290 ... --super-read-only
B4  WAIT_FOR_EXECUTED_GTID_SET = 0, GTID_SUBSET = 1
B5  t5 = 14:35:06.104517, read_only=0, super_read_only=0
B6  gtid_executed của 3334: 88389ddb-...:1-29693, 8d9e0610-c932-11f1-9d46-00090ffe0001:1
B7  3333 và 3319: Source_Port 3334, Running Yes/Yes, mọi Errno 0, Auto_Position 1
    misc.orders trên cả hai: 1:3319:truoc switchover | 2:3334:sau switchover
```

Cửa sổ từ B3 tới B5: **khoảng 0,55 giây** (chạy bằng script trên một máy).

### Phần B — semisync, rơi về async, rồi failover ngoài kế hoạch

Source mới là 3334; replica là 3319 và 3333.

```sql
-- Trên 3334 (source)
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.dll';
SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_source_timeout = 3000;     -- ms

-- Trên 3319 và 3333 (replica): INSTALL PLUGIN ghi vào mysql.plugin nên phải tạm tắt super_read_only
SET GLOBAL super_read_only = OFF;
SET SESSION sql_log_bin = 0;
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.dll';
SET SESSION sql_log_bin = 1;
SET GLOBAL super_read_only = ON;
SET GLOBAL rpl_semi_sync_replica_enabled = 1;
STOP REPLICA IO_THREAD; START REPLICA IO_THREAD;
```

Các bước đo:
1. Latency: trên 3334, xen kẽ `rpl_semi_sync_source_enabled` = 1 và 0, mỗi lần `SET @t0=SYSDATE(6); CALL misc.gen_small(run, 500);` rồi tính micro giây mỗi commit; 3 lần mỗi chế độ.
2. Cắt ACK: `STOP REPLICA IO_THREAD` trên **cả hai** replica; trên 3334 đo thời gian một `INSERT`, xem `Rpl_semi_sync_source_status`; chèn thêm hai hàng.
3. "Source chết": `mysqladmin -P3334 shutdown`. So `gtid_executed` của 3319 và 3333.
4. Unplanned promotion sang 3319: `STOP REPLICA; RESET REPLICA ALL; SET GLOBAL read_only=OFF;`; ứng dụng chèn 3 hàng; trỏ 3333 về 3319.
5. Source cũ quay lại với `--super-read-only=ON` trên dòng lệnh; thử ghi; `GTID_SUBTRACT` hai chiều với `gtid_executed` của 3319; trỏ nó về 3319 bằng auto-position.

**Kết quả đã đo.**

```text
-- Mặc định khi cài: rpl_semi_sync_source_wait_point=AFTER_SYNC, wait_for_replica_count=1
-- Lần đầu cài plugin trên replica khi super_read_only=ON: ERROR 1290 (phải tạm tắt như trên)
Rpl_semi_sync_source_clients 2, Rpl_semi_sync_source_status ON, Rpl_semi_sync_replica_status ON (cả hai replica)
```

| Lần | semisync = 1 (µs/commit) | semisync = 0 (µs/commit) |
| --- | --- | --- |
| 1 | 2.743 | 2.117 |
| 2 | 2.230 | 2.027 |
| 3 | 2.395 | 2.025 |
| **Trung vị** | **2.395** | **2.027** |

`Rpl_semi_sync_source_tx_avg_wait_time` = 264 µs, `Rpl_semi_sync_source_yes_tx` = 1.502 sau phần đo latency.

```text
-- Cắt ACK
commit_s = 3.015
Rpl_semi_sync_source_status OFF, Rpl_semi_sync_source_no_tx 3
two_commits_s = 0.005                      (hai INSERT tiếp theo, đã là async)
misc.orders trên 3334: 7 'commit khi mat ACK', 8 'async 1', 9 'async 2'
gtid_executed 3334: 88389ddb-...:1-29693, 8d9e0610-...:1-3008
log lỗi 3334: [Warning] [MY-011153] [Repl] Timeout waiting for reply of binlog ...

-- Sau khi tắt 3334
3319: 88389ddb-...:1-29693, 8d9e0610-...:1-3005
3333: 88389ddb-...:1-29693, 8abfb139-...:1, 8d9e0610-...:1-3005      → chọn 3319 (không có GTID lạ)
3319 Replica_IO_Running: Connecting, Last_IO_Errno 2003 (không nối được 3334)

-- Unplanned promotion 3319; ứng dụng ghi 3 hàng
misc.orders trên 3319: 6 '(3334) semisync on 3', 7 '(3319) sau failover A', 8 '... B', 9 '... C'
3333 trỏ về 3319: Running Yes/Yes, không lỗi

-- Source cũ 3334 quay lại với --super-read-only=ON
INSERT: ERROR 1290 ... --super-read-only
only_on_old_source = 8d9e0610-c932-11f1-9d46-00090ffe0001:3006-3008
only_on_new_source = 88389ddb-c932-11f1-9b76-00090ffe0001:29694-29695
misc.orders trên 3334: 7 'commit khi mat ACK', 8 'async 1', 9 'async 2'

-- Trỏ 3334 về 3319
Replica_SQL_Running: No, Last_SQL_Errno: 1062
Worker 1 failed executing transaction '88389ddb-...:29694' ...; Could not execute Write_rows event on table
misc.orders; Duplicate entry '7' for key 'orders.PRIMARY', Error_code: 1062
```

### Phần C — server ID (chạy sau phần B, trên 3334 đang lỗi và 3333)

```sql
-- Trên 3334
STOP REPLICA; SET GLOBAL server_id = 0; START REPLICA;      -- (1)
SET GLOBAL server_id = 1; START REPLICA IO_THREAD;          -- (2) trùng với source 3319
-- (3) 3334 về server_id=3 và chạy IO thread; 3333 đổi sang server_id=3 rồi chạy lại IO thread
-- Quan sát trong 2,5 phút: SHOW REPLICAS và PROCESSLIST trên 3319, trạng thái và log lỗi của hai replica
```

**Kết quả đã đo.**

```text
(1) ERROR 1200 (HY000): The server is not configured as replica; fix in config file or with CHANGE REPLICATION SOURCE TO
    (log: [ERROR] [MY-010546] [Repl] Server id not set, will not start replica for channel '')
(2) Last_IO_Errno: 13117  Fatal error: The replica I/O thread stops because source and replica have equal MySQL server ids
    mysqld --verbose --help: server-id mặc định = 1
(3) SHOW REPLICAS trên 3319 (mọi lần lấy mẫu trong 2 phút):   3   3333   1   8abfb139-...     ← chỉ một dòng
    PROCESSLIST trên 3319: hai thread 'Binlog Dump GTID', mỗi replica một thread
    Cả hai replica: Replica_IO_Running Yes, Last_IO_Errno 0, RECEIVED_TRANSACTION_SET giống nhau
    Không có thông báo ngắt hay nối lại trong log lỗi của hai replica
```

**Diễn giải và đối chiếu với sách.**
- **Phần A** đi đúng bảy bước tr. 240. Bước kiểm GTID lạ (B0) không có trong sách (**Bổ sung ngoài sách**) nhưng đã loại một ứng viên mà nếu chọn sẽ phát tán hàng ghi nhầm từ TN1 ra mọi server. Bước 4 dùng tập GTID làm bằng chứng đồng bộ, đúng "Compare GTIDs to be sure".
- **Phần B** xác nhận tr. 237: semisync tăng latency mỗi commit (khoảng 18% ở localhost; qua mạng thật sẽ phụ thuộc RTT, **Dự đoán**) và khi hết timeout thì **không làm transaction thất bại**. Ba transaction sinh ra trong lúc mất ACK chỉ tồn tại trên source cũ. Đây chính là điều tr. 240–241 cảnh báo: sau unplanned promotion, khó biết đích thiếu bao nhiêu. `GTID_SUBTRACT` đo được con số đó **chỉ khi** source cũ quay lại. Lỗi 1062 cho thấy `AUTO_INCREMENT` của source mới cấp lại id đã dùng; ba hàng của source cũ phải đối soát thủ công rồi dựng lại source cũ thành replica (tr. 250).
- **Phần C**: lỗi 1200 đúng tr. 252. Trên 8.0, mặc định `server_id=1` nên "quên đặt" thường biểu hiện thành lỗi 13117 khi source cũng dùng mặc định. Với hai replica trùng ID, `SHOW REPLICAS` chỉ thấy một (đúng tr. 251), nhưng trong 2,5 phút không thấy hiện tượng "đánh nhau"; sách đã nói hậu quả **tùy phiên bản**. Ghi chú không thử các hậu quả khác (bỏ sót event khi một replica trùng ID được promote).

**Dọn dẹp và khôi phục.** Đặt lại `server_id=3` cho 3334 và `server_id=2` cho 3333. Không khôi phục ba hàng lạc (ngoài mục đích thí nghiệm). Cuối buổi tắt cả ba instance:

```bash
for p in 3333 3334 3319; do "$BIN/mysqladmin" -h127.0.0.1 -P$p -uroot shutdown; done
netstat -ano | grep -E ":(3319|3333|3334) .*LISTEN"     # không còn dòng nào
```

**Kết quả đã đo:** cả ba lệnh `shutdown` trả về 0, không còn cổng nào lắng nghe. Datadir được giữ trong thư mục thí nghiệm riêng; muốn làm lại từ đầu thì xóa ba thư mục `data` và khởi tạo lại.

---

## Chưa thực thi

| Nội dung | Lý do | Quy trình đề xuất |
| --- | --- | --- |
| `pt-heartbeat`, `pt-table-checksum`, `pt-table-sync` (tr. 249) | Percona Toolkit cần tải từ internet, không được phép trong môi trường này | Thay bằng heartbeat tự làm (TN3) và `CHECKSUM TABLE` (TN2) |
| `max_allowed_packet` của source lớn hơn replica (tr. 248, 254) | Giới hạn thời gian; cần sinh event lớn hơn `replica_max_allowed_packet` (mặc định 1 GB trên 8.0.42) | Giảm `replica_max_allowed_packet` trên replica, ghi một hàng `LONGBLOB` lớn trên source, xem lỗi I/O và giá trị `Seconds_Behind_Source` |
| Hết đĩa trên source và replica (tr. 248, 254) | Không an toàn để làm đầy đĩa laptop | Dùng phân vùng nhỏ riêng hoặc đặt `relay_log_space_limit` thấp và quan sát replica tạm dừng |
| Multisource với hai channel (tr. 246–247), ring (tr. 246) | Ngoài phạm vi thời gian; sách khuyên tránh ring | `CHANGE REPLICATION SOURCE TO ... FOR CHANNEL 'video'` và `FOR CHANNEL 'user'` trên một replica thứ ba |
| `binlog_transaction_dependency_tracking=WRITESET` | Không có trong sách; để dành | Lặp phần D và E của TN3 với WRITESET trên source |
