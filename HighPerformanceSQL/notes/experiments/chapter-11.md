# Thực hành — Chương 11: Scaling MySQL

Quay lại ghi chú: [../chapter-11.md](../chapter-11.md) · Đáp án tự kiểm tra: [../answers/chapter-11.md](../answers/chapter-11.md)

> **Phạm vi.** Chương 11 bàn kiến trúc nhiều node: read pool, load balancer, service discovery, ProxySQL, Vitess. Môi trường thực hành chỉ cho phép **một** instance MySQL và không được tải phần mềm, nên các thí nghiệm dưới đây chọn những phần **đo được trên một instance** mà vẫn kiểm chứng trực tiếp lập luận của sách:
>
> 1. [Phân loại workload đọc hay ghi bằng Performance Schema](#tn1) (tr. 289–290)
> 2. [`read_only`, `super_read_only` và logic health check](#tn2) (tr. 295–296)
> 3. [Mô phỏng hai cách chọn partitioning key](#tn3) (tr. 300–303)
> 4. [Chi phí kết nối và `max_connections`](#tn4) (tr. 307, 310)
>
> Phần [Chưa thực thi](#chua-thuc-thi) ghi quy trình cho những gì cần HAProxy, ProxySQL, Vitess hay nhiều instance. Phần [Bài tập thiết kế trên giấy](#bai-tap-giay) có hai bài kèm gợi ý.
>
> **Nhãn:** "Kết quả đã đo" là output thật, có thể đã rút gọn (ghi rõ). Các bước, schema, dữ liệu, SQL là **Ví dụ minh họa do AI xây dựng**.

## Môi trường

| Mục | Giá trị |
| --- | --- |
| MySQL | Community **8.0.42**, `C:\Program Files\MySQL\MySQL Server 8.0\bin\` |
| Máy | Laptop Windows 11 Home, chạy từ Git Bash |
| Instance | Riêng, `--no-defaults`, cổng **3321**, `--bind-address=127.0.0.1`, datadir trong thư mục tạm của phiên làm việc; **không** đụng MySQL của người dùng ở cổng 3306 |
| Cấu hình | Mặc định của 8.0.42: `innodb_buffer_pool_size` = 128 MiB, `innodb_flush_log_at_trx_commit` = 1, `sync_binlog` = 1, binary log bật, `max_connections` = 151, `performance_schema` = ON |
| Dữ liệu | Tổng hợp bằng recursive CTE: bảng 200.000 hàng (TN1), 100.000 bài viết của 2.000 user (TN3) |
| Đo thời gian | Lặp 3 lần, báo trung vị. Laptop nhiều nhiễu, **không phải benchmark production**. Ưu tiên chỉ số tất định: số câu, số hàng, bộ đếm `Handler_*`, `Innodb_*` |

**Khởi tạo và chạy instance** (đặt `BIN` là thư mục bin ở trên, `DD` là datadir riêng):

```bash
"$BIN/mysqld" --no-defaults --initialize-insecure --datadir="$DD" --console
# chạy nền (Bash với run_in_background)
"$BIN/mysqld" --no-defaults --datadir="$DD" --port=3321 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=ch11.sock --console > mysqld.log 2>&1
# chờ sẵn sàng rồi kiểm tra đúng instance
"$BIN/mysql" -h127.0.0.1 -P3321 -uroot -e "SELECT @@port, @@datadir, @@version"
```

Kết quả kiểm tra: `3321`, datadir trong thư mục tạm, `8.0.42`.

---

<a id="tn1"></a>

## Thí nghiệm 1 — Phân loại workload đọc hay ghi bằng Performance Schema

### Mục tiêu

Kiểm chứng ý "không phải truy vấn nào cũng như nhau" (tr. 289–290): tỷ lệ **số câu** không nói lên tỷ lệ **chi phí**. Đồng thời kiểm tra cách đo latency đọc so với ghi mà sách trỏ về Chương 3 (tr. 62).

### Giả thuyết (Dự đoán)

- Hai workload có cùng 5.000 `SELECT` theo primary key và 50 `UPDATE`. Nếu `UPDATE` sửa nhiều hàng (workload A), tỷ trọng **thời gian** của `UPDATE` sẽ lớn, có thể vượt `SELECT`; nếu `UPDATE` sửa một hàng (workload B), `SELECT` sẽ chiếm phần lớn thời gian.
- Lượng redo (`Innodb_os_log_written`) của A lớn hơn B nhiều lần.
- Cột `LOCK_TIME` không phản ánh tổng latency.

### Chuẩn bị

```sql
CREATE DATABASE IF NOT EXISTS ch11;
USE ch11;
CREATE TABLE orders (
  id          INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  customer_id INT UNSIGNED NOT NULL,
  status      TINYINT UNSIGNED NOT NULL,
  amount      DECIMAL(10,2) NOT NULL,
  note        VARCHAR(100) NOT NULL,
  created_at  DATETIME NOT NULL,
  KEY idx_customer (customer_id)
) ENGINE=InnoDB;
SET SESSION cte_max_recursion_depth = 300000;
INSERT INTO orders (customer_id, status, amount, note, created_at)
WITH RECURSIVE seq(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 200000)
SELECT 1 + (CRC32(n) MOD 20000), n MOD 5, (CRC32(n * 7) MOD 100000) / 100,
       REPEAT('x', 40 + (n MOD 50)), '2026-01-01' + INTERVAL (n MOD 260) DAY
FROM seq;
ANALYZE TABLE orders;
```

Sinh hai tệp workload bằng `awk` (mỗi 100 `SELECT` xen một `UPDATE`; id tất định, không ngẫu nhiên):

```bash
# Workload A: UPDATE sửa 2.000 hàng mỗi câu
awk 'BEGIN{print "USE ch11;"; for(i=1;i<=5000;i++){
  printf "SELECT amount FROM orders WHERE id = %d;\n", (i*7919)%200000+1;
  if(i%100==0){a=((i/100)*3989)%198000+1;
    printf "UPDATE orders SET amount = amount + 1, note = CONCAT(\"a\", %d) WHERE id BETWEEN %d AND %d;\n", i, a, a+1999} } }' > w_a.sql
# Workload B: giống A nhưng UPDATE chỉ sửa một hàng (WHERE id = a)
awk 'BEGIN{print "USE ch11;"; for(i=1;i<=5000;i++){
  printf "SELECT amount FROM orders WHERE id = %d;\n", (i*7919)%200000+1;
  if(i%100==0){a=((i/100)*3989)%198000+1;
    printf "UPDATE orders SET amount = amount + 1, note = CONCAT(\"b\", %d) WHERE id = %d;\n", i, a} } }' > w_b.sql
```

Trước mỗi lần chạy, xóa số liệu tổng hợp và chụp bộ đếm toàn cục (`e1_reset.sql`):

```sql
TRUNCATE TABLE performance_schema.events_statements_summary_global_by_event_name;
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
DROP TABLE IF EXISTS ch11.gs_before;
CREATE TABLE ch11.gs_before AS
  SELECT VARIABLE_NAME, CAST(VARIABLE_VALUE AS UNSIGNED) AS v
  FROM performance_schema.global_status
  WHERE VARIABLE_NAME LIKE 'Handler\_%'
     OR VARIABLE_NAME IN ('Innodb_rows_read','Innodb_rows_updated',
        'Innodb_buffer_pool_read_requests','Innodb_buffer_pool_write_requests',
        'Innodb_os_log_written','Innodb_data_fsyncs','Innodb_data_writes',
        'Innodb_data_written','Innodb_buffer_pool_pages_flushed',
        'Com_select','Com_update','Questions');
```

### Các bước và lệnh đo

1. Chạy `e1_reset.sql`, rồi `mysql -h127.0.0.1 -P3321 -uroot < w_a.sql > /dev/null`, rồi báo cáo (`e1_report.sql`):

   ```sql
   SELECT EVENT_NAME, COUNT_STAR AS cnt,
          ROUND(SUM_TIMER_WAIT/1e9,1) AS total_ms,   -- picosecond -> ms
          ROUND(SUM_LOCK_TIME/1e9,1)  AS lock_ms,
          SUM_ROWS_EXAMINED AS rows_examined, SUM_ROWS_AFFECTED AS rows_affected,
          SUM_ROWS_SENT AS rows_sent
   FROM performance_schema.events_statements_summary_global_by_event_name
   WHERE EVENT_NAME IN ('statement/sql/select','statement/sql/update')
   ORDER BY EVENT_NAME;

   SELECT g.VARIABLE_NAME, CAST(g.VARIABLE_VALUE AS UNSIGNED) - b.v AS delta
   FROM performance_schema.global_status g JOIN ch11.gs_before b USING (VARIABLE_NAME)
   WHERE CAST(g.VARIABLE_VALUE AS UNSIGNED) - b.v <> 0
   ORDER BY g.VARIABLE_NAME;
   ```

2. Lặp 3 lần cho A, 3 lần cho B.
3. Thêm một lần chạy A với consumer `events_statements_history_long` được bật tạm thời, để chạy truy vấn **theo kiểu Chương 3** (cộng `LOCK_TIME/1000000`) cạnh cột `TIMER_WAIT`, và xem top digest:

   ```sql
   UPDATE performance_schema.setup_consumers SET ENABLED = 'YES'
   WHERE NAME = 'events_statements_history_long';
   -- ... reset, chạy w_a.sql ...
   SELECT LEFT(DIGEST_TEXT, 70) AS digest, COUNT_STAR AS cnt,
          ROUND(SUM_TIMER_WAIT/1e9,1) AS total_ms,
          ROUND(SUM_TIMER_WAIT/COUNT_STAR/1e6,1) AS avg_us,
          SUM_ROWS_EXAMINED AS rows_exam
   FROM performance_schema.events_statements_summary_by_digest
   WHERE SCHEMA_NAME = 'ch11'
   ORDER BY SUM_TIMER_WAIT DESC LIMIT 3;

   SELECT EVENT_NAME, COUNT(*) AS cnt,
          SUM(LOCK_TIME/1000000)       AS lock_div_1e6,
          ROUND(SUM(TIMER_WAIT)/1e9,1) AS timer_wait_ms
   FROM performance_schema.events_statements_history_long
   WHERE EVENT_NAME IN ('statement/sql/select','statement/sql/update')
   GROUP BY EVENT_NAME ORDER BY timer_wait_ms DESC;
   -- sau đó tắt lại consumer
   ```

### Kết quả đã đo

MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định, dữ liệu tổng hợp 200.000 hàng.

**Theo loại câu** (mỗi lần chạy; thời gian trong ngoặc là ba lần chạy):

| Workload | Loại | cnt | total_ms (lần 1, 2, 3) | Trung vị | lock_ms | rows_examined | rows_affected |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A | select | 5.003 | 860,6 · 871,2 · 958,1 | **871,2** | 7,4–7,7 | 5.003 | 0 |
| A | update | 50 | 2.336,1 · 1.451,2 · 1.431,6 | **1.451,2** | 0,1 | 100.000 | 100.000 |
| B | select | 5.003 | 979,7 · 1.078,0 · 1.020,2 | **1.020,2** | 8,0–9,7 | 5.003 | 0 |
| B | update | 50 | 110,2 · 114,0 · 113,0 | **113,0** | 0,1 | 50 | 50 |

Thời gian tường của cả lần chạy client: A 3.570 · 2.705 · 2.747 ms (trung vị 2.747); B 1.467 · 1.710 · 1.626 ms (trung vị 1.626).

**Hiệu bộ đếm toàn cục** (đã rút gọn; số tất định giống nhau ở cả ba lần):

| Bộ đếm | A | B |
| --- | --- | --- |
| `Handler_read_key` | 5.097 | 5.097 |
| `Handler_read_next` | 100.011 | 11 |
| `Handler_update` | 100.000 | 50 |
| `Innodb_rows_read` | 105.001 | 5.051 |
| `Innodb_rows_updated` | 100.000 | 50 |
| `Questions` | 5.058 | 5.058 |
| `Innodb_os_log_written` (byte, lần 1 · 2 · 3) | 24.530.944 · 9.611.776 · 9.574.400 | 129.536 · 123.904 · 131.072 |
| `Innodb_buffer_pool_write_requests` (lần 1 · 2 · 3) | 284.702 · 204.702 · 203.507 | 2.011 · 890 · 1.070 |
| `Innodb_data_fsyncs` (lần 1 · 2 · 3) | 605 · 583 · 614 | 250 · 232 · 206 |

**Top digest và truy vấn kiểu Chương 3** (một lần chạy A riêng):

```text
digest                                                               cnt   total_ms  avg_us   rows_exam
UPDATE `orders` SET `amount` = `amount` + ? , `note` = `CONCAT` (...)  50    1494.8  29895.7     100000
SELECT `amount` FROM `orders` WHERE `id` = ?                         5000    850.5    170.1       5000

EVENT_NAME            cnt   lock_div_1e6  timer_wait_ms
statement/sql/update    50       83.0000         1494.8
statement/sql/select  5006     7470.0000          854.1
```

*(Đã rút gọn khung bảng. `performance_schema_events_statements_history_long_size` = 10000, đủ chứa cả workload.)*

### Diễn giải và đối chiếu với sách

- **Số câu khác chi phí** (tr. 289–290). Cả hai workload có cùng tỷ lệ 5.003 : 50. Ở A, `UPDATE` chiếm 1.451 / (1.451 + 871) ≈ **62%** thời gian câu lệnh; ở B chỉ ≈ **10%** (113 / 1.133). Trung bình một `UPDATE` của A mất khoảng 29 ms, gấp khoảng 170 lần một `SELECT` theo primary key (khoảng 0,17 ms). Nếu chỉ nhìn tỷ lệ số câu, cả hai đều "99% đọc".
- **Đường ghi đắt.** `Innodb_os_log_written` của A khoảng 9,6 MB ở lần 2 và 3, so với khoảng 0,13 MB của B, phù hợp với việc A sửa 100.000 hàng còn B sửa 50 hàng. Lần chạy A đầu tiên sinh nhiều redo hơn (24,5 MB). **Giải thích khả dĩ, chưa kiểm chứng:** ở lần đầu cột `note` đổi từ chuỗi 40–89 ký tự sang chuỗi ngắn nên bản ghi đổi kích thước; các lần sau `note` được gán đúng giá trị cũ, chỉ `amount` đổi.
- **`LOCK_TIME` không phải latency.** Truy vấn kiểu tr. 62 (cộng `LOCK_TIME/1000000`) xếp `SELECT` lên đầu với 7.470, trong khi `TIMER_WAIT` xếp `UPDATE` lên đầu (1.494,8 ms so với 854,1 ms). `LOCK_TIME` tính bằng picosecond, nên chia 10⁶ ra **micro giây**: 7.470 µs ≈ 7,5 ms, khớp với `lock_ms` 7,4–7,7 ở bảng tổng hợp. Hơn nữa nó chỉ là thời gian chờ lock; trong workload không tranh chấp này nó chỉ khoảng 1% tổng thời gian của `SELECT`. Để phân loại workload theo tinh thần tr. 290, dùng `SUM_TIMER_WAIT`.
- **Số đếm phụ.** 5.003 thay vì 5.000 `SELECT` vì client `mysql` tự gửi vài câu như `select @@version_comment` khi kết nối và `SELECT DATABASE()` khi xử lý `USE`. Trong bảng `history_long` có 5.006 vì gồm thêm câu của các phiên reset và báo cáo.
- **Giới hạn.** Một client, không tranh chấp, thời gian bị ảnh hưởng bởi fsync trên đĩa laptop. Kết luận định tính (tỷ trọng thời gian khác tỷ trọng số câu) không phụ thuộc các con số tuyệt đối.

### Dọn dẹp

```sql
UPDATE performance_schema.setup_consumers SET ENABLED = 'NO'
WHERE NAME = 'events_statements_history_long';
DROP DATABASE IF EXISTS ch11;   -- chạy ở cuối, sau TN2 và TN3 vì chúng dùng chung schema
```

---

<a id="tn2"></a>

## Thí nghiệm 2 — `read_only`, `super_read_only` và logic health check

### Mục tiêu

Kiểm chứng vì sao mẹo ở tr. 295 bảo kiểm tra **cả** `read_only` lẫn `super_read_only`, và thử logic của một script `/check-lag` (tr. 296) trên các trạng thái giả lập.

### Giả thuyết (Dự đoán)

- `read_only=ON` chặn user thường nhưng không chặn root; `super_read_only=ON` chặn cả root.
- Một health check chỉ nhìn heartbeat sẽ coi **source** là khỏe (heartbeat trên source luôn mới).

### Chuẩn bị

```sql
CREATE USER IF NOT EXISTS 'app_rw'@'127.0.0.1' IDENTIFIED BY 'App_rw_pw1';
GRANT SELECT, INSERT, UPDATE, DELETE ON ch11.* TO 'app_rw'@'127.0.0.1';
CREATE TABLE IF NOT EXISTS ch11.ro_test (id INT AUTO_INCREMENT PRIMARY KEY, who VARCHAR(20));
CREATE TABLE IF NOT EXISTS ch11.heartbeat (id TINYINT PRIMARY KEY, ts DATETIME(6) NOT NULL);
REPLACE INTO ch11.heartbeat VALUES (1, NOW(6));
```

### Các bước và lệnh đo

**Phần A: hai biến chỉ đọc.**

1. `SET GLOBAL read_only = ON;` rồi `INSERT INTO ch11.ro_test (who) VALUES (...)` lần lượt bằng `app_rw` và `root`.
2. `SET GLOBAL super_read_only = ON;` rồi thử `INSERT` bằng `root`.
3. `SET GLOBAL read_only = OFF;` và xem cả hai biến.
4. `SHOW REPLICA STATUS` và đếm hàng trong `performance_schema.replication_connection_status`.

**Phần B: hai kiểu health check** (`e2_check.sql`, ngưỡng 30 giây lấy từ ví dụ ở tr. 295):

```sql
-- Check "ngây thơ": chỉ nhìn độ trễ heartbeat
SELECT 'naive' AS kieu_check,
       TIMESTAMPDIFF(SECOND, (SELECT MAX(ts) FROM ch11.heartbeat), NOW(6)) AS lag_s,
       IF(TIMESTAMPDIFF(SECOND, (SELECT MAX(ts) FROM ch11.heartbeat), NOW(6)) <= 30,
          'HEALTHY (200)', 'UNHEALTHY (503)') AS ket_luan;

-- Check "chặt": phải là replica, thread replication chạy, lag <= 30 s
SELECT 'strict' AS kieu_check,
       @@super_read_only AS super_ro,
       (SELECT COUNT(*) FROM performance_schema.replication_connection_status
         WHERE SERVICE_STATE = 'ON') AS io_on,
       (SELECT COUNT(*) FROM performance_schema.replication_applier_status
         WHERE SERVICE_STATE = 'ON') AS sql_on,
       TIMESTAMPDIFF(SECOND, (SELECT MAX(ts) FROM ch11.heartbeat), NOW(6)) AS lag_s,
       CASE
         WHEN @@super_read_only = 0 THEN 'UNHEALTHY (503): khong phai replica'
         WHEN (SELECT COUNT(*) FROM performance_schema.replication_connection_status
                WHERE SERVICE_STATE = 'ON') = 0
           OR (SELECT COUNT(*) FROM performance_schema.replication_applier_status
                WHERE SERVICE_STATE = 'ON') = 0
              THEN 'UNHEALTHY (503): replication khong chay'
         WHEN TIMESTAMPDIFF(SECOND, (SELECT MAX(ts) FROM ch11.heartbeat), NOW(6)) > 30
              THEN 'UNHEALTHY (503): lag qua nguong'
         ELSE 'HEALTHY (200)'
       END AS ket_luan;
```

Chạy ở ba trạng thái giả lập: (A) source với heartbeat mới; (B) đặt heartbeat lùi 45 giây (`UPDATE ch11.heartbeat SET ts = NOW(6) - INTERVAL 45 SECOND`); (C) heartbeat mới và `super_read_only=ON`, giả lập một replica có replication đã dừng.

### Kết quả đã đo

**Phần A** (đã rút gọn):

```text
== read_only=ON
@@read_only  @@super_read_only
1            0
-- app_rw insert:
ERROR 1290 (HY000) at line 1: The MySQL server is running with the --read-only option so it cannot execute this statement
-- root insert: thành công (rows_in_ro_test = 1)
== super_read_only=ON
@@read_only  @@super_read_only
1            1
-- root insert:
ERROR 1290 (HY000) at line 1: The MySQL server is running with the --super-read-only option so it cannot execute this statement
== read_only=OFF
@@read_only  @@super_read_only
0            0
== SHOW REPLICA STATUS: không trả hàng nào; replication_connection_status: n_channels = 0
```

**Phần B:**

| Trạng thái | Check ngây thơ | Check chặt |
| --- | --- | --- |
| A. Source, heartbeat mới | lag 0, **HEALTHY (200)** | super_ro 0, io_on 0, sql_on 0, lag 0, **UNHEALTHY: khong phai replica** |
| B. Heartbeat cũ 45 giây | lag 45, UNHEALTHY (503) | super_ro 0, lag 45, UNHEALTHY: khong phai replica |
| C. `super_read_only=ON`, replication không chạy | lag 0, **HEALTHY (200)** | super_ro 1, io_on 0, sql_on 0, **UNHEALTHY: replication khong chay** |

### Diễn giải và đối chiếu với sách

- `read_only` một mình **không** chặn tài khoản quản trị; chỉ `super_read_only` chặn. Bật `super_read_only` tự bật `read_only`; tắt `read_only` tự tắt `super_read_only`. Đây là lý do mẹo ở tr. 295 nhắc cả hai biến: một replica đúng chuẩn nên bật cả hai, và một node có cả hai tắt gần như chắc chắn là source.
- Check chỉ nhìn heartbeat nhận nhầm hai trường hợp nguy hiểm (A và C). Đúng như cảnh báo ở tr. 296, health check có logic mà phải thiết kế cẩn thận; và đúng như Chương 9 (tr. 248), không nên dựa vào một con số lag duy nhất.
- Trên máy không phải replica, `SHOW REPLICA STATUS` trả **tập rỗng**, không phải "lag = 0". Script phải coi tập rỗng là "không phải replica".
- **Giới hạn:** không có replica thật nên không đo được lag thật; trạng thái B và C là giả lập. Script thật còn phải xử lý trường hợp **mọi** replica cùng trượt check bằng pool fallback (tr. 296), phần này nằm ở load balancer, không ở SQL.

### Dọn dẹp

```sql
SET GLOBAL super_read_only = OFF;
SET GLOBAL read_only = OFF;
DROP USER IF EXISTS 'app_rw'@'127.0.0.1';
```

---

<a id="tn3"></a>

## Thí nghiệm 3 — Mô phỏng hai cách chọn partitioning key

### Mục tiêu

Kiểm chứng lập luận ở tr. 300–303: partitioning key quyết định truy vấn nào khu trú trong một shard, truy vấn nào phải hỏi mọi shard, và phép gộp xuyên shard có tách được theo shard hay không.

### Giả thuyết (Dự đoán)

- Shard theo `post_id` (giống băm primary key của NDB, tr. 300): "bài của một user" phải hỏi mọi shard; "bài theo ID" hỏi một shard. Shard theo `user_id`: ngược lại.
- Lấy top 10 của mỗi shard rồi gộp cho kết quả **đúng** khi shard theo `user_id`, **có thể sai** khi shard theo `post_id`.
- Shard theo `user_id` bị lệch kích thước khi có user rất hoạt động (tr. 289, 301).

### Chuẩn bị

Bốn "shard" là bốn schema trên **cùng một instance**: `p_0`…`p_3` (layout P, `post_id MOD 4`) và `u_0`…`u_3` (layout U, `user_id MOD 4`). Phân bố user lệch theo lũy thừa 3 để có vài user rất hoạt động; ngày đăng lấy từ một hàm băm khác để không tương quan với `post_id MOD 4`.

```sql
SET SESSION cte_max_recursion_depth = 200000;
CREATE TABLE ch11.posts_all (
  post_id    INT UNSIGNED PRIMARY KEY,
  user_id    INT UNSIGNED NOT NULL,
  created_at DATETIME NOT NULL,
  title      VARCHAR(100) NOT NULL,
  KEY idx_user (user_id)
) ENGINE=InnoDB;
INSERT INTO ch11.posts_all
WITH RECURSIVE seq(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 100000)
SELECT n,
       1 + FLOOR(POW((CRC32(n) MOD 1000000) / 1000000, 3) * 2000),
       '2026-01-01' + INTERVAL (CRC32(n * 13) MOD 300) DAY,
       CONCAT('post ', n)
FROM seq;
-- với k = 0..3:
CREATE DATABASE p_0;  CREATE TABLE p_0.posts LIKE ch11.posts_all;
INSERT INTO p_0.posts SELECT * FROM ch11.posts_all WHERE post_id MOD 4 = 0;
CREATE DATABASE u_0;  CREATE TABLE u_0.posts LIKE ch11.posts_all;
INSERT INTO u_0.posts SELECT * FROM ch11.posts_all WHERE user_id MOD 4 = 0;
-- ... tương tự cho 1, 2, 3; rồi ANALYZE TABLE từng bảng
```

> **Ghi chú về một lần chạy hỏng.** Lần sinh dữ liệu đầu tiên dùng `created_at = '2026-01-01' + INTERVAL (n MOD 300) DAY`. Vì 300 chia hết cho 4, mỗi shard của layout P chỉ chứa những ngày có cùng số dư theo 4, tức layout P vô tình thành "chia theo thời gian". Lần chạy đó bị loại; mọi kết quả dưới đây là của dữ liệu đã sửa.

### Các bước và lệnh đo

1. Đếm hàng mỗi shard và phân bố bài theo user.
2. Q1 "bài của user 3", Q2 "bài có `post_id = 12345`" trên từng layout. Đo bằng lịch sử câu lệnh của chính phiên (`PS_CURRENT_THREAD_ID()` có từ 8.0.16):

   ```sql
   SELECT COUNT(*) FROM p_0.posts WHERE user_id = 3;   -- lặp cho p_1, p_2, p_3
   SELECT LEFT(SQL_TEXT, 60) AS sql_text, ROWS_SENT, ROWS_EXAMINED
   FROM performance_schema.events_statements_history
   WHERE THREAD_ID = PS_CURRENT_THREAD_ID() AND SQL_TEXT LIKE 'SELECT%posts%'
   ORDER BY EVENT_ID;
   ```

3. Q3 "top 10 user nhiều bài nhất trong 7 ngày (2026-03-01 tới 2026-03-07)". So kết quả thật (tính trên bảng gốc) với gộp ngây thơ: top 10 của mỗi shard, `UNION ALL`, cộng theo user, lấy top 10. Câu cho layout P (layout U thay `p_` bằng `u_`):

   ```sql
   WITH truth AS (
          SELECT user_id, COUNT(*) AS c FROM ch11.posts_all
          WHERE created_at >= '2026-03-01' AND created_at < '2026-03-08'
          GROUP BY user_id),
        merged AS (
          SELECT user_id, SUM(c) AS c FROM (
            (SELECT user_id, COUNT(*) AS c FROM p_0.posts
             WHERE created_at >= '2026-03-01' AND created_at < '2026-03-08'
             GROUP BY user_id ORDER BY c DESC, user_id LIMIT 10)
            UNION ALL
            (SELECT user_id, COUNT(*) AS c FROM p_1.posts
             WHERE created_at >= '2026-03-01' AND created_at < '2026-03-08'
             GROUP BY user_id ORDER BY c DESC, user_id LIMIT 10)
            UNION ALL
            (SELECT user_id, COUNT(*) AS c FROM p_2.posts
             WHERE created_at >= '2026-03-01' AND created_at < '2026-03-08'
             GROUP BY user_id ORDER BY c DESC, user_id LIMIT 10)
            UNION ALL
            (SELECT user_id, COUNT(*) AS c FROM p_3.posts
             WHERE created_at >= '2026-03-01' AND created_at < '2026-03-08'
             GROUP BY user_id ORDER BY c DESC, user_id LIMIT 10)
          ) x GROUP BY user_id),
        t AS (SELECT ROW_NUMBER() OVER (ORDER BY c DESC, user_id) rk, user_id, c FROM truth),
        m AS (SELECT ROW_NUMBER() OVER (ORDER BY c DESC, user_id) rk, user_id, c FROM merged)
   SELECT t.rk, t.user_id AS true_user, t.c AS true_posts,
          m.user_id AS merged_user, m.c AS merged_posts,
          IF(t.user_id = m.user_id AND t.c = m.c, 'dung', 'SAI') AS ket_qua
   FROM t JOIN m USING (rk) WHERE t.rk <= 10 ORDER BY t.rk;
   ```

4. Đếm số nhóm (user) mỗi shard có trong 7 ngày, tức số hàng phải gửi về nếu gộp **đúng** ở layout P, và kiểm tra gộp đầy đủ (bỏ `LIMIT`) có khớp với kết quả thật.

### Kết quả đã đo

MySQL 8.0.42, instance riêng, 100.000 bài, 2.000 user.

**Phân bố:**

```text
layout              s0      s1      s2      s3
P (post_id MOD 4)   25000   25000   25000   25000
U (user_id MOD 4)   22449   30325   23904   23322

users  max_posts_per_user  min_posts  avg_posts
2000   7948                7          50.0

top user: 1 -> 7948 bài, 2 -> 2086, 3 -> 1397
trong 7 ngày: 2317 bài, 1039 user
```

**Q1 và Q2** (ROWS_SENT, ROWS_EXAMINED):

| Truy vấn | Layout P | Layout U |
| --- | --- | --- |
| Q1 bài của user 3 | 4 câu: p_0 xét 345, p_1 365, p_2 340, p_3 347 hàng | 1 câu: u_3 xét 1.397 hàng |
| Q2 `post_id = 12345` | 1 câu: p_1 xét 1 hàng | 4 câu: u_0 trả 1 hàng (user 1212), u_1, u_2, u_3 trả 0 hàng, xét 0 |

**Q3 top 10 trong 7 ngày:**

```text
layout P (post_id MOD 4)                         layout U (user_id MOD 4)
rk true_user true_posts merged_user merged_posts | ket_qua (U: mọi hàng khớp)
1  1         191        1           191   dung    | dung
2  2         53         2           53    dung    | dung
3  3         38         3           38    dung    | dung
4  4         23         4           23    dung    | dung
5  10        23         8           19    SAI     | dung
6  8         19         10          19    SAI     | dung
7  5         18         5           15    SAI     | dung
8  11        17         11          12    SAI     | dung
9  6         15         6           11    SAI     | dung
10 9         15         7           10    SAI     | dung
```

*(Đã ghép hai bảng output thành một để dễ so; layout U trả đúng cùng user và số bài như cột true ở mọi hạng.)*

**Số nhóm mỗi shard trong 7 ngày:** P: 386, 407, 389, 398 (tổng 1.580); U: 257, 263, 253, 266. Gộp đầy đủ ở P (không `LIMIT`): `users_compared` = 1.039, `counts_equal` = 1.039.

### Diễn giải và đối chiếu với sách

- **Fan-out, không phải số hàng.** Q1 xét tổng 1.397 hàng ở cả hai layout; khác biệt là phải hỏi 4 shard hay 1. Trên hệ thật, mỗi shard thêm một vòng mạng và latency bằng shard chậm nhất. Đây là "tránh crosstalk giữa các node" và "băm primary key dễ lưu, khó lấy" ở tr. 300.
- **Mỗi khóa phục vụ một chiều truy vấn.** Q2 cho kết quả ngược lại: layout U phải hỏi mọi shard để tìm một bài theo ID, trừ khi có bảng tra cứu hay mã hóa shard vào ID. Đây là lý do sách bàn nhiều partitioning key (tr. 302).
- **Gộp xuyên shard.** Ở layout U, top 10 mỗi shard gộp lại cho kết quả đúng vì mỗi user nằm trọn một shard. Ở layout P, cách đó sai 6/10 hạng; muốn đúng phải gửi về 1.580 hàng thay vì 40. Đây là ví dụ cụ thể cho câu "truy vấn xuyên shard chậm và đắt hơn nhiều, thường phải cache mạnh tay" (tr. 302) và cho việc dùng summary table (tr. 303).
- **Lệch kích thước.** User 1 chiếm khoảng 8% số bài; shard chứa user đó (u_1) có 30.325 hàng, lớn hơn trung bình 25.000 khoảng 21%. Sách khuyên shard đủ nhỏ để cân bằng được bằng cách gộp (tr. 301), và lưu ý có user hoạt động hơn hẳn trung bình (tr. 289).
- **Giới hạn.** Bốn schema trên một server không mô phỏng được mạng, độ trễ đuôi hay lỗi một phần. Kết quả về số shard phải hỏi, số hàng xét và tính đúng của phép gộp không phụ thuộc điều đó.

### Dọn dẹp

```sql
DROP DATABASE IF EXISTS p_0;  -- và p_1, p_2, p_3, u_0, u_1, u_2, u_3
DROP DATABASE IF EXISTS ch11;
```

---

<a id="tn4"></a>

## Thí nghiệm 4 — Chi phí kết nối và `max_connections`

### Mục tiêu

Đặt số liệu cho lập luận ở tr. 307 và 310: mỗi kết nối tới database tốn tài nguyên, và "mở thêm kết nối" khi hệ thống chậm có thể dẫn tới từ chối kết nối cùng thử lại dồn dập.

### Giả thuyết (Dự đoán)

- 100 kết nối đồng thời làm tăng bộ nhớ server đo được qua Performance Schema.
- Vượt `max_connections` (151 mặc định), client bị từ chối với lỗi 1040; nếu client tự thử lại, số lần kết nối hỏng lớn hơn số client.

### Chuẩn bị

Dùng `mysqlslap` có sẵn trong thư mục bin. Câu truy vấn `SELECT SLEEP(4)` giữ mỗi kết nối khoảng 4 giây. Câu đo (`$Q`):

```sql
SELECT VARIABLE_NAME, VARIABLE_VALUE FROM performance_schema.global_status
WHERE VARIABLE_NAME IN ('Threads_connected','Threads_running','Threads_created',
  'Threads_cached','Max_used_connections','Connection_errors_max_connections');
SELECT ROUND(SUM(CURRENT_NUMBER_OF_BYTES_USED)/1048576,1) AS ps_tracked_mem_mib
FROM performance_schema.memory_summary_global_by_event_name;
SELECT @@max_connections, @@thread_cache_size;
```

### Các bước và lệnh đo

```bash
# 1. trước
mysql -h127.0.0.1 -P3321 -uroot -t -e "$Q"
# 2. 100 client, đo sau 2 giây khi chúng còn đang SLEEP
mysqlslap -h127.0.0.1 -P3321 -uroot --concurrency=100 --iterations=1 \
  --number-of-queries=100 --query="SELECT SLEEP(4)" --create-schema=ch11 &
sleep 2; mysql -h127.0.0.1 -P3321 -uroot -t -e "$Q"; wait
# 3. sau
mysql -h127.0.0.1 -P3321 -uroot -t -e "$Q"
# 4. 200 client bằng user thường app_rw (không có CONNECTION_ADMIN)
mysqlslap -h127.0.0.1 -P3321 -uapp_rw -p... --concurrency=200 --iterations=1 \
  --number-of-queries=200 --query="SELECT SLEEP(4)" --create-schema=ch11 > slap200.txt 2>&1 &
sleep 2; mysql -h127.0.0.1 -P3321 -uroot -t -e "<các bộ đếm kết nối>"; wait
grep -c "Too many" slap200.txt
```

### Kết quả đã đo

MySQL 8.0.42 cấu hình mặc định (`max_connections` = 151, `thread_cache_size` = 9, `thread_stack` = 1.048.576).

| Thời điểm | Threads_connected | Threads_created | Max_used_connections | Bộ nhớ P_S theo dõi |
| --- | --- | --- | --- | --- |
| Trước | 1 | 1 | 1 | 450,2 MiB |
| Đang có 100 client | 102 | 102 | 102 | 462,6 MiB |
| Sau khi 100 client ngắt | 1 | 102 | 102 | 450,3 MiB (Threads_cached = 8) |

Lần 200 client:

```text
Aborted_connects                   47
Connection_errors_max_connections  500
Max_used_connections               152
Threads_connected                  152   (trong lúc chạy)
mysqlslap: ... Error when connecting to server: 1040 Too many connections   (50 dòng)
```

### Diễn giải và đối chiếu với sách

- `Threads_connected` = 102 khi có 100 client: 100 client, cộng một kết nối mà `mysqlslap` tự giữ (khả năng là kết nối điều khiển của nó; chưa kiểm chứng), cộng phiên `root` dùng để đo. Bộ nhớ Performance Schema theo dõi tăng khoảng 12,4 MiB, tức khoảng 125 KiB cho mỗi kết nối **đang chạy một câu đơn giản** (12,4 MiB / 101). Con số này chưa gồm stack của thread và phần hệ điều hành không đo ở đây; truy vấn có sort, join buffer, bảng tạm sẽ tốn hơn nhiều. Bộ nhớ trở về gần mức cũ sau khi ngắt, và 8 thread được giữ lại trong thread cache.
- Với 200 client, 50 client nhận lỗi 1040 (151 chỗ của `app_rw` gồm 150 client cộng kết nối riêng của `mysqlslap`, theo cách giải thích ở trên). `Connection_errors_max_connections` tăng 500, gấp 10 lần số client bị từ chối. **Diễn giải, chưa kiểm chứng trong mã nguồn:** `mysqlslap` tự thử kết nối lại nhiều lần cho mỗi client. Đây là hình ảnh thu nhỏ của vòng xoáy mà sách mô tả: hệ thống quá tải, client mở thêm hay thử lại kết nối, làm tải tăng thêm (tr. 310, chú thích 7 về thundering herd).
- `Threads_connected` = 152 = 151 kết nối của `app_rw` cộng phiên `root` dùng để đo: MySQL dành thêm một kết nối cho tài khoản có `CONNECTION_ADMIN` hay `SUPER`, nên quản trị viên vẫn vào được khi server đầy. Đây là lý do không nên cho tài khoản ứng dụng quyền quản trị.
- Proxy có connection pooling như ProxySQL giữ số kết nối tới database ở mức "làm được việc" và tái sử dụng chúng (tr. 307, 310), nên số kết nối phía database không tăng theo số node ứng dụng. Thí nghiệm này **không** chạy ProxySQL; nó chỉ đo chi phí mà ProxySQL giúp tránh.
- `Aborted_connects` = 47 chưa được giải thích đầy đủ trong thí nghiệm này; không dùng nó để kết luận.

### Dọn dẹp

Không có trạng thái cần khôi phục ngoài việc xóa user `app_rw` (đã gồm trong dọn dẹp TN2).

---

<a id="chua-thuc-thi"></a>

## Chưa thực thi

| Thí nghiệm | Lý do chưa chạy | Quy trình đề xuất |
| --- | --- | --- |
| Read pool thật với HAProxy và lag thật | Cần ít nhất ba instance (source và hai replica) và HAProxy; môi trường chỉ cho một cổng, không tải phần mềm | Dựng source và 2 replica dùng GTID; HAProxy với cấu hình như tr. 294 cộng `httpchk`; chạy ghi nặng trên source, dừng SQL thread ở một replica; quan sát replica bị loại và được đưa lại; dừng cả hai để thử pool fallback (tr. 296) |
| `leastconn` và cache lạnh | Cần read pool thật | Khởi động lại một replica với `innodb_buffer_pool_load_at_startup=OFF`, đưa vào pool, so p99 latency với lần có nạp buffer pool (tr. 297) |
| ProxySQL: định tuyến theo user và schema | Không có ProxySQL | Chạy listing tr. 309 với cú pháp đã sửa (`LOAD MYSQL USERS TO RUNTIME`), khai báo hostgroup trong `mysql_servers`, kiểm định tuyến bằng `SELECT @@port` qua cổng 6033; kiểm `stats_mysql_connection_pool` để thấy multiplexing |
| Vitess | Cần Kubernetes hoặc bộ binary của Vitess | Theo ba bước tr. 306: đo latency thêm vào, vttablet "externally managed", rồi mới shard |
| Service discovery (Consul) | Không có Consul | Trả lời năm câu hỏi ở tr. 295 bằng cách tắt agent, tắt server Consul và quan sát load balancer |

---

<a id="bai-tap-giay"></a>

## Bài tập thiết kế trên giấy

> **Ví dụ minh họa do AI xây dựng.** Làm trước, rồi mới đọc gợi ý.

### Bài 1 — Thiết kế read pool

Một ứng dụng có 1 source và 6 replica. Có ba loại đọc: (a) trang sản phẩm, cũ 5 phút không sao; (b) bảng điều khiển của người bán, cũ tối đa 10 giây; (c) xuất báo cáo CSV hàng ngày, quét bảng lớn trong 40 phút. Backup chạy mỗi đêm trên một replica.

1. Chia 6 replica vào các pool thế nào? Pool nào cần check lag, ngưỡng bao nhiêu?
2. Chọn thuật toán cân bằng tải và giải thích cách tránh dội tải vào replica mới.
3. Nếu cả cluster lag 2 phút vì migration, mỗi pool cư xử ra sao?

**Gợi ý.** Backup và báo cáo CSV có thể dùng chung 1 replica ngoài pool vì chạy khác giờ, hoặc mỗi việc 1 replica nếu chồng giờ (tr. 293). Còn 4 replica: không đủ cho hai pool tối thiểu ba node (tr. 294–295), nên dùng một pool chung cho (a) và (b) với ngưỡng lag theo loại khắt khe hơn (10 giây), hoặc 5 replica nếu báo cáo và backup chung máy. Health check: port, `read_only`/`super_read_only`, replication chạy, heartbeat (tr. 295–296). `leastconn` kèm tăng trọng số dần và nạp buffer pool (tr. 294, 297). Khi cả cluster lag 2 phút: pool trượt check lag, cần fallback; trang sản phẩm chấp nhận được dữ liệu 2 phút, bảng điều khiển người bán thì cần hỏi đội sản phẩm có chấp nhận hay nên chuyển tạm về source (tr. 296).

### Bài 2 — Chọn partitioning key cho SaaS nhiều khách hàng

Một SaaS quản lý dự án phục vụ 20.000 công ty. Bảng chính: `companies`, `users` (thuộc một công ty), `projects`, `tasks`, `comments`. 95% truy vấn nằm trong phạm vi một công ty. Có 5 công ty rất lớn, mỗi công ty chiếm khoảng 4% dữ liệu. Đội sản phẩm muốn thêm trang "thống kê toàn hệ thống" cho nội bộ.

1. Chọn unit of sharding và giải thích bằng tiêu chí ở tr. 300.
2. Những bảng nào phải shard cùng nhau (tr. 299)?
3. Làm sao để 5 công ty lớn không làm lệch shard (tr. 301)?
4. Trang thống kê toàn hệ thống nên làm thế nào (tr. 302–303)?

**Gợi ý.** Unit of sharding là công ty (`company_id`), primary key của thực thể quan trọng nhất, và 95% truy vấn chỉ chạm một shard (tr. 300). `users`, `projects`, `tasks`, `comments` đều mang `company_id` và được shard cùng để tránh `JOIN` xuyên database (tr. 299). Dùng nhiều shard logic hơn số server rồi gộp: 5 công ty lớn mỗi công ty có thể chiếm riêng một server, các công ty nhỏ gộp nhiều vào một server (tr. 301); nếu một công ty lớn tới mức vượt một server thì phải có khóa phụ, và đó là bài toán nhiều partitioning key (tr. 302). Trang thống kê dùng summary table do job nền tổng hợp từ mọi shard, đặt ở kho riêng hay global node, có cache (tr. 303). `companies` có thể nằm ở global node vì nhỏ và được đọc nhiều (tr. 303).

---

## Trạng thái và dọn dẹp chung

- Bốn thí nghiệm **đã chạy** trên MySQL 8.0.42, instance riêng cổng 3321.
- Đã chạy dọn dẹp: tắt `super_read_only` và `read_only`, xóa các database `ch11`, `p_0`…`p_3`, `u_0`…`u_3`, xóa user `app_rw`, tắt lại consumer `events_statements_history_long`.
- Instance đã được tắt bằng `mysqladmin -h127.0.0.1 -P3321 -uroot shutdown`; log ghi "Shutdown complete (mysqld 8.0.42)".
