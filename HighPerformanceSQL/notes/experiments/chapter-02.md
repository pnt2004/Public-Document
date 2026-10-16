# Thực hành — Chương 2: Giám sát MySQL theo tư duy reliability engineering

Về ghi chú chương: [../chapter-02.md](../chapter-02.md) · Đáp án: [../answers/chapter-02.md](../answers/chapter-02.md)

Chương 2 là chương vận hành. Năm thí nghiệm dưới đây không nhằm benchmark, mà để **nhìn tận mắt** các tín hiệu sách nhắc tới: hai trạng thái tăng kết nối (tr. 31), lỗi "too many connections" (tr. 28, 30), lỗi theo tốc độ và lỗi một lần (tr. 28), probe availability (tr. 26), trung bình so với percentile (tr. 38) và không gian auto-increment (tr. 33–34). Mọi SQL, script và dữ liệu là **Ví dụ minh họa do AI xây dựng**; output là **Kết quả đã đo**.

## Môi trường

| Mục | Giá trị |
| --- | --- |
| MySQL | Community **8.0.42** (`mysqld --no-defaults`, cấu hình mặc định trừ chỗ ghi khác) |
| Máy | Laptop Windows 11 Home, 16 CPU logic; chạy bằng Git Bash |
| Instance | Riêng cho chương này: cổng **3312**, `--bind-address=127.0.0.1`, `--mysqlx=OFF`, datadir trong thư mục scratchpad |
| Biến mặc định liên quan (đã đọc) | `max_connections` = 151, `innodb_lock_wait_timeout` = 50, `innodb_autoinc_lock_mode` = 2, `information_schema_stats_expiry` = 86400, `performance_schema` = ON |
| Dữ liệu | Tổng hợp, rất nhỏ (vài bảng, nhiều nhất 10.080 hàng) |

> **Cảnh báo an toàn.** Các thí nghiệm đổi biến toàn cục (`max_connections`, `event_scheduler`, `super_read_only`) và mở hàng chục kết nối. **Chỉ chạy trên instance thử nghiệm riêng**, không chạy trên MySQL đang dùng thật. Mỗi thí nghiệm có bước khôi phục.

**Khởi tạo và chạy instance** (thay `<datadir>` bằng một thư mục trống):

```bash
BIN="C:/Program Files/MySQL/MySQL Server 8.0/bin"
"$BIN/mysqld" --no-defaults --initialize-insecure --datadir="<datadir>" --console
"$BIN/mysqld" --no-defaults --datadir="<datadir>" --port=3312 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=ch02.sock --console > mysqld.log 2>&1 &
# chờ tới khi kết nối được, rồi kiểm tra đúng instance
"$BIN/mysql" -h127.0.0.1 -P3312 -uroot -e "SELECT @@port, @@datadir, @@version; CREATE DATABASE IF NOT EXISTS ch02;"
```

Mọi script dưới đây dùng hàm `M() { "$BIN/mysql" -h127.0.0.1 -P3312 -uroot "$@"; }`.

## Mục lục

- [TN1 — Hai trạng thái tăng kết nối](#tn1)
- [TN2 — Chạm max_connections](#tn2)
- [TN3 — Lỗi theo tốc độ, lỗi một lần, và probe availability](#tn3)
- [TN4 — Trung bình so với percentile](#tn4)
- [TN5 — Giám sát không gian auto-increment](#tn5)
- [Dọn dẹp cuối cùng](#cleanup)

---

<a id="tn1"></a>

## TN1 — Hai trạng thái tăng kết nối

**Mục tiêu.** Tái hiện hai trạng thái ở tr. 31: ứng dụng mở nhiều kết nối **không dùng** (`Threads_connected` cao, `Threads_running` thấp) và ứng dụng **thực sự dùng** nhiều kết nối (cả hai cùng cao). Tính tỉ lệ `Threads_connected` / `max_connections` như sách gợi ý.

**Giả thuyết (Dự đoán).** 40 kết nối rảnh làm `Threads_connected` tăng thêm 40, `Threads_running` gần như không đổi. Thêm 40 kết nối chạy `SELECT SLEEP(20)` làm cả hai tăng thêm khoảng 40. Ở trạng thái nền, `Threads_running` bằng 1 (chính phiên đo).

**Chuẩn bị.** Không cần bảng. Kết nối rảnh được tạo bằng cách cho client `mysql` đọc từ một pipe không có dữ liệu (`sleep 40 | mysql ...`): client kết nối rồi đứng chờ, nên phiên ở trạng thái `Sleep`.

**Các bước và lệnh đo.**

```bash
SNAP="SELECT g1.VARIABLE_VALUE AS threads_connected, g2.VARIABLE_VALUE AS threads_running,
  @@max_connections AS max_conn, ROUND(100*g1.VARIABLE_VALUE/@@max_connections,1) AS pct_conn_used
  FROM performance_schema.global_status g1 JOIN performance_schema.global_status g2
  WHERE g1.VARIABLE_NAME='Threads_connected' AND g2.VARIABLE_NAME='Threads_running';"
# Pha 0: nền (FLUSH STATUS đưa Max_used_connections về số kết nối hiện tại)
M -e "FLUSH STATUS; $SNAP SHOW GLOBAL STATUS LIKE 'Max_used_connections';"
# Pha 0b: event scheduler có tính vào Threads_running không?
M -e "SET GLOBAL event_scheduler=OFF;"; sleep 1; M -e "$SNAP"
M -e "SET GLOBAL event_scheduler=ON;"
# Pha 1: 40 kết nối rảnh
for i in $(seq 1 40); do (sleep 40 | "$BIN/mysql" -h127.0.0.1 -P3312 -uroot >/dev/null 2>&1 &); done
sleep 4; M -e "$SNAP SELECT COMMAND, COUNT(*) AS n FROM information_schema.PROCESSLIST GROUP BY COMMAND ORDER BY COMMAND;"
# Pha 2: thêm 40 kết nối đang chạy
for i in $(seq 1 40); do ("$BIN/mysql" -h127.0.0.1 -P3312 -uroot -e "SELECT SLEEP(20)" >/dev/null 2>&1 &); done
sleep 4; M -e "$SNAP SELECT COMMAND, COUNT(*) AS n FROM information_schema.PROCESSLIST GROUP BY COMMAND ORDER BY COMMAND;"
# Pha 3: chờ mọi kết nối đóng
sleep 40; M -e "$SNAP SHOW GLOBAL STATUS WHERE Variable_name IN ('Max_used_connections','Max_used_connections_time');"
```

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định, không có dữ liệu). Output đã rút gọn và gộp thành bảng:

| Pha | `Threads_connected` | `Threads_running` | Phần trăm `max_connections` (151) | `PROCESSLIST` theo `COMMAND` |
| --- | --- | --- | --- | --- |
| 0. Nền | 1 | **2** | 0,7 | |
| 0b. Tắt `event_scheduler` | 1 | **1** | 0,7 | |
| 1. Thêm 40 kết nối rảnh | 41 | 2 | 27,2 | Daemon 1, Query 1, Sleep 40 |
| 2. Thêm 40 kết nối `SLEEP(20)` | 81 | 42 | 53,6 | Daemon 1, Query 41, Sleep 40 |
| 3. Sau khi đóng hết | 1 | 2 | 0,7 | |

`Max_used_connections` = 81 sau thí nghiệm (đã là 1 ngay sau `FLUSH STATUS` ở pha 0).

**Diễn giải và đối chiếu với sách.**

- Pha 1 đúng trạng thái thứ nhất ở tr. 31: kết nối tăng mà không có việc. Tỉ lệ 27,2% là chỉ số sách gợi ý để thấy tầng ứng dụng đang đẩy bạn tới giới hạn (tr. 31).
- Pha 2 đúng trạng thái thứ hai: mỗi kết nối đang chạy query làm `Threads_running` tăng đúng một. `SLEEP` không tốn CPU, nên đây là ví dụ cho thấy `Threads_running` đếm thread **không ngủ**, không phải thread **đang dùng CPU**: 42 thread "đang chạy" trên máy 16 CPU mà CPU gần như rảnh. Ngưỡng "vượt số core" (tr. 27) vì vậy cần đọc cùng chỉ số CPU và thời gian chờ.
- Dự đoán "nền bằng 1" **sai**: nền là 2, vì thread daemon của event scheduler (bật mặc định trong 8.0) cũng được đếm. Khi đặt ngưỡng tuyệt đối thấp, cần biết phần nền này.
- `FLUSH STATUS` có đưa `Max_used_connections` về số kết nối hiện tại; đây là "mức nước cao nhất" mà chương 5 khuyên theo dõi (tr. 121).

**Dọn dẹp và khôi phục.** Các kết nối tự đóng sau 40 giây. `event_scheduler` đã được bật lại trong script; kiểm tra bằng `SELECT @@event_scheduler;` (phải là `ON`).

---

<a id="tn2"></a>

## TN2 — Chạm max_connections

**Mục tiêu.** Xem điều gì xảy ra khi tầng ứng dụng mở nhiều kết nối hơn `max_connections` (tr. 28, 30): mã lỗi, bộ đếm `Connection_errors_max_connections`, và kết nối dự phòng cho quản trị.

**Giả thuyết (Dự đoán).** Với `max_connections` = 10, kết nối thứ 11 của user thường nhận lỗi 1040. Tài khoản có `CONNECTION_ADMIN` (ở đây là `root`) vẫn vào được thêm **một** kết nối (**Bổ sung ngoài sách**, theo tài liệu MySQL). Mỗi lần bị từ chối làm `Connection_errors_max_connections` tăng 1.

**Chuẩn bị.**

```sql
CREATE USER IF NOT EXISTS 'app'@'127.0.0.1' IDENTIFIED BY 'app_pw';
GRANT SELECT, INSERT, UPDATE, DELETE ON ch02.* TO 'app'@'127.0.0.1';
FLUSH STATUS;
SET GLOBAL max_connections = 10;
```

**Các bước và lệnh đo.**

```bash
M -e "SELECT @@max_connections; SHOW GLOBAL STATUS WHERE Variable_name IN
      ('Threads_connected','Connection_errors_max_connections','Max_used_connections');"
# 10 kết nối rảnh bằng user app (giữ 30 giây)
for i in $(seq 1 10); do (sleep 30 | "$BIN/mysql" -h127.0.0.1 -P3312 -uapp -papp_pw ch02 >/dev/null 2>&1 &); done
sleep 4
# kết nối thứ 11 bằng user app
"$BIN/mysql" -h127.0.0.1 -P3312 -uapp -papp_pw -e "SELECT 'app connected'"
# một kết nối root giữ mở 20 giây
(sleep 20 | "$BIN/mysql" -h127.0.0.1 -P3312 -uroot -e "SELECT SLEEP(15)" >/dev/null 2>&1 &)
sleep 3
# thêm một kết nối root nữa
M -e "SELECT 'second root connected'"
sleep 35
M -e "SHOW GLOBAL STATUS WHERE Variable_name IN
      ('Threads_connected','Connection_errors_max_connections','Max_used_connections');"
```

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng, `max_connections` = 10 trong thí nghiệm). Script chạy hai lần; output lần hai, đã rút gọn. Phần sau dấu `<-` là chú thích của người viết, không phải output:

```text
== Truoc khi mo ket noi
@@max_connections  10
Connection_errors_max_connections  2      <- còn lại từ lần chạy 1, FLUSH STATUS không xóa
Max_used_connections               1
Threads_connected                  1
== Ket noi thu 11 bang user app (khong co CONNECTION_ADMIN)
ERROR 1040 (08004): Too many connections
== Them mot ket noi root nua               <- root đầu tiên đã vào được và đang giữ kết nối
ERROR 1040 (HY000): Too many connections
== Sau khi tat ca ket noi dong
Connection_errors_max_connections  4
Max_used_connections               11
Threads_connected                  1
```

**Diễn giải và đối chiếu với sách.**

- Lỗi 1040 là lỗi "too many connections" mà sách xếp vào loại **một lần là đủ** để xử lý (tr. 28). Với ứng dụng, nó thành lỗi 5xx ngay.
- `Max_used_connections` = 11 = `max_connections` + 1: kết nối thứ 11 chính là `root`, đi vào chỗ dự phòng cho quản trị. Chỗ này chỉ có **một**: `root` thứ hai bị từ chối. Nếu agent giám sát chạy bằng tài khoản quản trị, nó có thể chiếm mất chỗ này (**Bổ sung ngoài sách**).
- SQLSTATE khác nhau (08004 cho user thường, HY000 cho `root`) nhưng cùng mã lỗi 1040; khi đếm lỗi, nên đếm theo mã lỗi.
- `Connection_errors_max_connections` tăng 2 mỗi lần chạy và **không** bị `FLUSH STATUS` xóa, trong khi `Max_used_connections` thì bị. Vì vậy sách nói theo dõi **sự tăng đột ngột** của các bộ đếm `Connection_errors_xxx` (tr. 28): cần lấy hiệu giữa hai lần đọc, không đọc giá trị tuyệt đối.

**Dọn dẹp và khôi phục.** `SET GLOBAL max_connections = 151;` (script đã làm và in lại 151). User `app` giữ lại cho TN3.

---

<a id="tn3"></a>

## TN3 — Lỗi theo tốc độ, lỗi một lần, và probe availability

**Mục tiêu.**

1. Tạo một chuỗi **lock wait timeout** và xem các bộ đếm phía server ghi nhận nó thế nào (tr. 28).
2. Mô phỏng một replica vừa được promote mà vẫn **read-only** (tr. 28), và so hai probe availability: `SELECT 1` và ghi rồi đọc (tr. 26).

**Giả thuyết (Dự đoán).** Năm lần cập nhật cùng một hàng đang bị khóa, mỗi lần chờ 2 giây, cho 5 lỗi 1205; `Innodb_row_lock_waits` tăng 5 và `Innodb_row_lock_time` tăng khoảng 10.000 ms. Cập nhật hàng khác không bị chặn. Khi `super_read_only` bật, `SELECT 1` vẫn thành công còn lệnh ghi bị lỗi 1290.

**Chuẩn bị.**

```sql
USE ch02;
DROP TABLE IF EXISTS inventory, probe;
CREATE TABLE inventory (id INT PRIMARY KEY, qty INT NOT NULL);
INSERT INTO inventory VALUES (1,100),(2,100),(3,100);
CREATE TABLE probe (id INT AUTO_INCREMENT PRIMARY KEY, ts DATETIME(6) NOT NULL);
TRUNCATE TABLE performance_schema.events_errors_summary_global_by_error;
```

**Các bước và lệnh đo.**

```bash
A() { "$BIN/mysql" -h127.0.0.1 -P3312 -uapp -papp_pw ch02 "$@"; }
ERRQ="SELECT ERROR_NUMBER, ERROR_NAME, SQL_STATE, SUM_ERROR_RAISED
      FROM performance_schema.events_errors_summary_global_by_error
      WHERE ERROR_NAME IN ('ER_LOCK_WAIT_TIMEOUT','ER_OPTION_PREVENTS_STATEMENT','ER_CON_COUNT_ERROR');"
LOCKQ="SHOW GLOBAL STATUS WHERE Variable_name IN
       ('Innodb_row_lock_waits','Innodb_row_lock_time','Innodb_row_lock_current_waits');"
M -e "$ERRQ $LOCKQ"
# Phiên A giữ khóa hàng id=1 trong 20 giây
("$BIN/mysql" -h127.0.0.1 -P3312 -uroot ch02 -e "START TRANSACTION;
   UPDATE inventory SET qty=qty-1 WHERE id=1; SELECT SLEEP(20); ROLLBACK;" >/dev/null 2>&1 &)
sleep 2
# Phiên B (user app) cập nhật cùng hàng 5 lần, mỗi lần chờ tối đa 2 giây
for i in 1 2 3 4 5; do A -e "SET SESSION innodb_lock_wait_timeout=2;
   UPDATE inventory SET qty=qty-1 WHERE id=1;"; done
# Cùng lúc, cập nhật hàng khác
A -e "SET SESSION innodb_lock_wait_timeout=2; UPDATE inventory SET qty=qty-1 WHERE id=2; SELECT ROW_COUNT() AS updated;"
M -e "$ERRQ $LOCKQ"
sleep 10
# Mô phỏng source mới quên tắt read-only
M -e "SET GLOBAL super_read_only = ON; SELECT @@read_only, @@super_read_only;"
A -e "SELECT 1 AS probe_select1;"                                       # probe 1
A -e "INSERT INTO probe (ts) VALUES (NOW(6)); SELECT MAX(id) FROM probe;" # probe 2
M -e "$ERRQ"
M -e "SET GLOBAL super_read_only = OFF; SET GLOBAL read_only = OFF;"
A -e "INSERT INTO probe (ts) VALUES (NOW(6)); SELECT MAX(id) AS last_probe_id FROM probe;"
```

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng, bảng 3 hàng). Output đã rút gọn (bỏ cảnh báo về mật khẩu trên dòng lệnh):

```text
== Truoc
1205  ER_LOCK_WAIT_TIMEOUT          HY000  0
1290  ER_OPTION_PREVENTS_STATEMENT  HY000  0
Innodb_row_lock_time 0 | Innodb_row_lock_waits 0
== 5 lan cap nhat hang id=1
ERROR 1205 (HY000) at line 1: Lock wait timeout exceeded; try restarting transaction   (x5)
== Cap nhat hang id=2 cung luc
updated 1
== Sau chuoi lock wait timeout
1205  ER_LOCK_WAIT_TIMEOUT          HY000  5
1290  ER_OPTION_PREVENTS_STATEMENT  HY000  0
Innodb_row_lock_current_waits 0 | Innodb_row_lock_time 10056 | Innodb_row_lock_waits 5
== super_read_only bat
@@read_only 1 | @@super_read_only 1
-- Probe 1: SELECT 1
probe_select1 1
-- Probe 2: ghi roi doc
ERROR 1290 (HY000) at line 1: The MySQL server is running with the --read-only option so it cannot execute this statement
1290  ER_OPTION_PREVENTS_STATEMENT  HY000  1
== Tat read-only, probe 2 lan nua
last_probe_id 1
```

**Diễn giải và đối chiếu với sách.**

- **Lock wait timeout** đếm được ở hai nơi: theo mã lỗi (`SUM_ERROR_RAISED` = 5) và theo InnoDB (`Innodb_row_lock_waits` = 5, tổng thời gian chờ 10.056 ms, tức khoảng 2 giây mỗi lần). Một vài lỗi như vậy là "nhiễu"; sách nói hãy cảnh báo khi **tốc độ** của chúng tăng vọt, vì đó là tranh chấp row lock leo thang, có thể dẫn tới mất ghi (tr. 28). Cập nhật hàng `id=2` thành công cho thấy khóa ở mức hàng.
- **Read-only**: chỉ một lần lỗi 1290 đã đủ nghĩa "cluster đang không ghi được" (tr. 28). Thông báo nói `--read-only` dù ta bật `super_read_only`, vì user `app` không có quyền quản trị nên bị chặn ngay ở tầng `read_only`.
- **Probe**: `SELECT 1` thành công trong khi ghi đã chết. Sách nói `SELECT 1` chỉ xác nhận MySQL nhận và parse được query, không chạm tầng lưu trữ (tr. 26); thí nghiệm cho thấy thêm một điểm mù: nó không biết server có nhận ghi không. Probe "ghi rồi đọc" (tr. 26) bắt được lỗi này.
- Bộ đếm phía server ở đây giúp **chẩn đoán**; theo sách, tỉ lệ lỗi nên được đo **ở phía client** trên toàn fleet (tr. 28).

**Dọn dẹp và khôi phục.** Script đã tắt `super_read_only` và `read_only` (kiểm tra `SELECT @@read_only, @@super_read_only;` phải là `0 0`). Phiên A tự rollback sau 20 giây. Bảng `inventory`, `probe` nằm trong schema `ch02`, xóa ở bước dọn dẹp cuối.

---

<a id="tn4"></a>

## TN4 — Trung bình so với percentile

**Mục tiêu.** Kiểm hai lời khuyên ở tr. 38 bằng số liệu thật: (A) percentile mô tả latency tốt hơn trung bình; (B) gộp chuỗi thời gian thành trung bình khi lưu dài hạn sẽ san phẳng đỉnh.

### Phần A — Phân phối latency của một digest

**Giả thuyết (Dự đoán).** 96 câu `SELECT SLEEP(0.01)` và 4 câu `SELECT SLEEP(0.5)` có cùng digest. Trung bình sẽ vào khoảng (96 × 10 + 4 × 500) / 100 ≈ 30 ms, một giá trị không câu nào có; p95 rơi vào cụm nhanh, p99 rơi vào cụm chậm.

**Chuẩn bị.** Tạo file workload 100 dòng (câu thứ 25, 50, 75, 100 là câu chậm):

```bash
: > tn4_workload.sql
for i in $(seq 1 100); do
  if [ $((i % 25)) -eq 0 ]; then echo "SELECT SLEEP(0.5);" >> tn4_workload.sql
  else echo "SELECT SLEEP(0.01);" >> tn4_workload.sql; fi
done
```

Truy vấn đo (`tn4_digest.sql`; digest text trong 8.0 có dạng ``SELECT `SLEEP` (?)``):

```sql
SELECT DIGEST_TEXT, COUNT_STAR,
       ROUND(MIN_TIMER_WAIT/1e9,1) AS min_ms,
       ROUND(AVG_TIMER_WAIT/1e9,1) AS avg_ms,
       ROUND(QUANTILE_95/1e9,1)    AS p95_ms,
       ROUND(QUANTILE_99/1e9,1)    AS p99_ms,
       ROUND(MAX_TIMER_WAIT/1e9,1) AS max_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT = 'SELECT `SLEEP` (?)';

SELECT h.BUCKET_NUMBER,
       ROUND(h.BUCKET_TIMER_LOW/1e9,2)  AS low_ms,
       ROUND(h.BUCKET_TIMER_HIGH/1e9,2) AS high_ms,
       h.COUNT_BUCKET, h.COUNT_BUCKET_AND_LOWER, h.BUCKET_QUANTILE
FROM performance_schema.events_statements_histogram_by_digest h
JOIN performance_schema.events_statements_summary_by_digest d
  ON d.SCHEMA_NAME <=> h.SCHEMA_NAME AND d.DIGEST = h.DIGEST
WHERE d.DIGEST_TEXT = 'SELECT `SLEEP` (?)' AND h.COUNT_BUCKET > 0
ORDER BY h.BUCKET_NUMBER;
```

**Các bước.** Lặp ba lần:

```bash
for run in 1 2 3; do
  M -e "TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
        TRUNCATE TABLE performance_schema.events_statements_histogram_by_digest;"
  M ch02 < tn4_workload.sql > /dev/null
  M < tn4_digest.sql
done
```

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng, 100 câu mỗi lần, ba lần chạy):

| Lần | min (ms) | trung bình (ms) | p95 (ms) | p99 (ms) | max (ms) |
| --- | --- | --- | --- | --- | --- |
| 1 | 10,1 | 35,7 | 25,1 | 524,8 | 514,6 |
| 2 | 10,7 | 35,3 | 20,9 | 524,8 | 514,3 |
| 3 | 12,3 | 35,4 | 22,9 | 524,8 | 514,7 |
| **Trung vị** | 10,7 | **35,4** | **22,9** | **524,8** | 514,6 |

Histogram lần 2 (đã rút gọn, chỉ các bucket có câu):

| Bucket (ms) | Số câu | Lũy kế |
| --- | --- | --- |
| 10,47–10,96 | 1 | 1 |
| 12,02–12,59 | 1 | 2 |
| 13,80–14,45 | 2 | 4 |
| 14,45–15,14 | 26 | 30 |
| 15,14–15,85 | 50 | 80 |
| 15,85–16,60 | 10 | 90 |
| 17,38–21,88 (5 bucket) | 6 | 96 |
| 501,19–524,81 | 4 | 100 |

**Diễn giải và đối chiếu với sách.**

- **Trung bình 35,4 ms không ứng với câu nào**: ở cả ba lần, không có câu nào nằm giữa 26,3 ms (cận trên bucket nhanh cao nhất, lần 1) và 501 ms. Đây là lý do sách khuyên dùng percentile để đặt cạnh SLO (tr. 38).
- p95 (khoảng 21–25 ms) mô tả trải nghiệm của "gần như mọi người"; p99 (524,8 ms) lộ ra đuôi chậm. Một SLO kiểu "99% câu dưới 100 ms" sẽ **trượt**, điều mà trung bình 35 ms không cho thấy.
- **p99 lớn hơn max** (524,8 so với 514,6): `QUANTILE_xx` trong Performance Schema là **cận trên của bucket** histogram chứa percentile đó, không phải giá trị chính xác. Độ phân giải bucket khoảng 4–5% mỗi bậc.
- Dự đoán "trung bình khoảng 30 ms" lệch: câu `SLEEP(0.01)` thực tế mất khoảng 15 ms trên máy này. **Giả thuyết** (chưa kiểm): độ phân giải bộ định thời của Windows khoảng 15,6 ms. Đây là ví dụ nhỏ cho thấy phải đo, không suy ra từ cú pháp.
- **Bổ sung ngoài sách:** cột `QUANTILE_95/99/999` và bảng histogram chỉ có từ MySQL 8.0; trên 5.7 phải tự tính percentile từ dữ liệu thô hoặc dùng công cụ ngoài.

### Phần B — Gộp chuỗi thời gian: trung bình so với đỉnh

**Giả thuyết (Dự đoán).** Một đỉnh 20 phút mỗi tối sẽ gần như biến mất khi gộp thành trung bình ngày; p95 theo ngày cũng không thấy nó (20/1.440 ≈ 1,4% số điểm), chỉ p99 và max thấy.

**Chuẩn bị và lệnh** (**Ví dụ minh họa do AI xây dựng**: dữ liệu IOutil giả lập theo phút trong 7 ngày):

```sql
SET SESSION cte_max_recursion_depth = 20000;
DROP TABLE IF EXISTS ch02.io_util_minute;
CREATE TABLE ch02.io_util_minute (
  ts   DATETIME NOT NULL PRIMARY KEY,
  util DECIMAL(5,1) NOT NULL
);
INSERT INTO ch02.io_util_minute (ts, util)
WITH RECURSIVE m (n) AS (SELECT 0 UNION ALL SELECT n + 1 FROM m WHERE n < 7*1440 - 1)
SELECT '2026-11-23 00:00:00' + INTERVAL n MINUTE,
       CASE WHEN MOD(n, 1440) BETWEEN 20*60 AND 20*60 + 19 THEN 92  -- 20 phút cao điểm mỗi tối
            ELSE 35 + MOD(n, 7) - 3                                  -- nền 32..38
       END
FROM m;

-- gộp theo giờ (ba giờ quanh đỉnh của ngày đầu)
SELECT DATE_FORMAT(ts, '%Y-%m-%d %H:00') AS gio, ROUND(AVG(util),1) AS avg_gio, MAX(util) AS max_gio
FROM ch02.io_util_minute
WHERE ts >= '2026-11-23 19:00:00' AND ts < '2026-11-23 22:00:00'
GROUP BY gio ORDER BY gio;

-- gộp theo ngày: trung bình, p95, p99, max (percentile bằng ROW_NUMBER vì MySQL không có PERCENTILE_CONT)
WITH r AS (
  SELECT DATE(ts) AS ngay, util,
         ROW_NUMBER() OVER (PARTITION BY DATE(ts) ORDER BY util) AS rn,
         COUNT(*)     OVER (PARTITION BY DATE(ts))               AS cnt
  FROM ch02.io_util_minute
)
SELECT ngay, ROUND(AVG(util),1) AS avg_ngay,
       MAX(CASE WHEN rn = CEIL(0.95*cnt) THEN util END) AS p95_ngay,
       MAX(CASE WHEN rn = CEIL(0.99*cnt) THEN util END) AS p99_ngay,
       MAX(util) AS max_ngay
FROM r GROUP BY ngay ORDER BY ngay;
```

**Kết quả đã đo** (MySQL 8.0.42, dữ liệu tổng hợp 10.080 hàng). Output đã rút gọn:

```text
so_diem 10080 | min 32.0 | max 92.0 | avg_7_ngay 35.8
gio               avg_gio  max_gio
2026-11-23 19:00  35.0     38.0
2026-11-23 20:00  54.1     92.0
2026-11-23 21:00  34.9     38.0
ngay        avg_ngay  p95_ngay  p99_ngay  max_ngay
2026-11-23  35.8      38.0      92.0      92.0
... (6 ngày còn lại giống hệt)
```

**Diễn giải và đối chiếu với sách.** Nếu hệ thống chỉ số chỉ giữ trung bình ngày, đồ thị 6 tháng sẽ phẳng ở 35,8%, và câu hỏi của sách "disk I/O utilization có tăng gấp đôi được trong năm tới không" (tr. 38) sẽ nhận câu trả lời sai: 2 × 35,8 ≈ 72%, trong khi đỉnh thật đã 92%. Ngay cả trung bình theo giờ cũng chỉ còn 54,1%. Đúng như sách: khi xem dữ liệu nhiều tháng, hãy xem **đỉnh** (tr. 38). Kết quả cũng cho thấy p95 theo ngày (38%) che mất đỉnh ngắn; percentile hợp với phân phối latency của request, còn câu hỏi năng lực cần max.

**Dọn dẹp và khôi phục.** Bảng `ch02.io_util_minute` xóa ở bước dọn dẹp cuối. Các bảng summary của Performance Schema chỉ nằm trong bộ nhớ; `TRUNCATE` chỉ xóa số liệu thống kê, không ảnh hưởng dữ liệu.

---

<a id="tn5"></a>

## TN5 — Giám sát không gian auto-increment

**Mục tiêu.** Dựng một query giám sát tương tự query của sách (tr. 33–34) và kiểm ba điều trên MySQL 8.0: giá trị đọc từ `information_schema` có mới không; bộ đếm auto-increment có tăng đúng bằng số hàng không; khi cạn thì lỗi gì.

**Giả thuyết (Dự đoán).** Query cho phần trăm đã dùng đúng ngay sau khi chèn. 100 hàng chèn vào bảng `TINYINT` đưa bộ đếm lên 101. Khi cạn, insert nhận lỗi duplicate key.

**Chuẩn bị** (`tn5_setup.sql`):

```sql
SET SESSION cte_max_recursion_depth = 1000;
DROP TABLE IF EXISTS ch02.t_tiny, ch02.t_tiny_u, ch02.t_int, ch02.t_big_u;
CREATE TABLE ch02.t_tiny   (id TINYINT AUTO_INCREMENT PRIMARY KEY, v INT);
CREATE TABLE ch02.t_tiny_u (id TINYINT UNSIGNED AUTO_INCREMENT PRIMARY KEY, v INT);
CREATE TABLE ch02.t_int    (id INT AUTO_INCREMENT PRIMARY KEY, v INT) AUTO_INCREMENT = 2000000000;
CREATE TABLE ch02.t_big_u  (id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY, v INT);
INSERT INTO ch02.t_tiny (v)   WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n < 100) SELECT n FROM s;
INSERT INTO ch02.t_tiny_u (v) WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n < 100) SELECT n FROM s;
INSERT INTO ch02.t_int (v) VALUES (1),(2),(3);
INSERT INTO ch02.t_big_u (v) VALUES (1);
```

Query giám sát tự viết (`tn5_check.sql`). Khác query của sách: lọc theo `COLUMNS.EXTRA` thay vì `COLUMN_KEY = 'PRI'`, tính `(AUTO_INCREMENT − 1)` là giá trị lớn nhất đã cấp, và bỏ schema hệ thống:

```sql
SELECT t.TABLE_SCHEMA, t.TABLE_NAME, c.COLUMN_NAME, c.COLUMN_TYPE,
       t.AUTO_INCREMENT AS next_value,
       ROUND(100 * (t.AUTO_INCREMENT - 1) /
         CASE c.DATA_TYPE
           WHEN 'tinyint'   THEN IF(c.COLUMN_TYPE LIKE '%unsigned%', 255, 127)
           WHEN 'smallint'  THEN IF(c.COLUMN_TYPE LIKE '%unsigned%', 65535, 32767)
           WHEN 'mediumint' THEN IF(c.COLUMN_TYPE LIKE '%unsigned%', 16777215, 8388607)
           WHEN 'int'       THEN IF(c.COLUMN_TYPE LIKE '%unsigned%', 4294967295, 2147483647)
           WHEN 'bigint'    THEN IF(c.COLUMN_TYPE LIKE '%unsigned%', 18446744073709551615, 9223372036854775807)
         END, 2) AS pct_used
FROM information_schema.TABLES t
JOIN information_schema.COLUMNS c
  ON c.TABLE_SCHEMA = t.TABLE_SCHEMA AND c.TABLE_NAME = t.TABLE_NAME
WHERE c.EXTRA LIKE '%auto_increment%'
  AND t.TABLE_SCHEMA NOT IN ('mysql','sys','performance_schema','information_schema')
ORDER BY pct_used DESC;
```

`tn5_check_fresh.sql` là cùng query, có thêm dòng đầu `SET SESSION information_schema_stats_expiry = 0;`.

**Các bước và lệnh đo.**

```bash
M < tn5_setup.sql
# B1: số hàng thật so với bộ đếm
M -e "SELECT 't_tiny' AS tbl, COUNT(*) AS so_hang, MAX(id) AS max_id FROM ch02.t_tiny
      UNION ALL SELECT 't_tiny_u', COUNT(*), MAX(id) FROM ch02.t_tiny_u
      UNION ALL SELECT 't_int', COUNT(*), MAX(id) FROM ch02.t_int;"
M < tn5_check.sql                       # B2: đọc lần 1
M -e "INSERT INTO ch02.t_int (v) VALUES (1),(2),(3),(4),(5),(6),(7),(8),(9),(10),
      (11),(12),(13),(14),(15),(16),(17),(18),(19),(20);"   # B3: thêm 20 hàng
M < tn5_check.sql                       # B4: đọc lần 2, expiry mặc định
M < tn5_check_fresh.sql                 # B5: đọc lần 3, expiry = 0
# B6: làm cạn khóa của t_tiny
M -e "INSERT INTO ch02.t_tiny (v) WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n < 20) SELECT n FROM s;"
M -e "INSERT INTO ch02.t_tiny (v) VALUES (500); SELECT COUNT(*), MAX(id) FROM ch02.t_tiny;"
M -e "INSERT INTO ch02.t_tiny (v) VALUES (501);"
```

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng, `innodb_autoinc_lock_mode` = 2, `information_schema_stats_expiry` = 86400, bảng nhỏ). Output đã rút gọn (bỏ các dòng của bảng `probe` và `t_big_u` có `pct_used` = 0, ở B4 và B5 chỉ giữ dòng `t_int`); phần sau dấu `<-` là chú thích của người viết:

```text
== B1: so hang that su so voi bo dem
t_tiny    so_hang 100  max_id 100
t_tiny_u  so_hang 100  max_id 100
t_int     so_hang 3    max_id 2000000002
== B2: doc lan 1 (mac dinh)
t_tiny    tinyint           next_value 127         pct_used 99.21
t_int     int               next_value 2000000003  pct_used 93.13
t_tiny_u  tinyint unsigned  next_value 128         pct_used 49.80
== B3: chen 20 hang vao t_int -> so_hang 23, max_id 2000000022
== B4: doc lan 2 (mac dinh 86400 giay)
t_int     int               next_value 2000000003  pct_used 93.13     <- giá trị cũ
== B5: doc lan 3 (information_schema_stats_expiry = 0)
t_int     int               next_value 2000000023  pct_used 93.13     <- giá trị mới
== B6: can key space cua t_tiny
-- INSERT ... SELECT 20 hang
ERROR 1467 (HY000) at line 1: Failed to read auto-increment value from storage engine
-- INSERT 1 hang
COUNT(*) 101  MAX(id) 127
-- INSERT 1 hang nua
ERROR 1062 (23000) at line 1: Duplicate entry '127' for key 't_tiny.PRIMARY'
```

Thêm một lần chạy query **theo đúng logic của sách** (gõ lại để kiểm, không chép vào đây) sau khi tạo bảng `t_composite` có primary key `(tenant_id, id)`: bảng `t_tiny` hiện cột `max_value` = 100.0000, bảng `t_tiny_u` = 50.1961, và `t_composite` xuất hiện **hai lần** (một lần cho mỗi cột số nguyên trong primary key). Kết quả còn có hai bảng của schema hệ thống `mysql` (`component`, `time_zone`).

**Diễn giải và đối chiếu với sách.**

- **Dự đoán sai, theo cách đáng chú ý:** 100 hàng chèn bằng `INSERT ... SELECT` đẩy bộ đếm lên 127 và 128, không phải 101. Với `innodb_autoinc_lock_mode` = 2 (mặc định của 8.0), câu insert không biết trước số hàng được cấp giá trị theo lô lớn dần, phần thừa bị bỏ (**Bổ sung ngoài sách**, theo tài liệu InnoDB về AUTO_INCREMENT). Bảng `TINYINT` có dấu đã 99% "đầy" dù mới có 100 hàng. Vì vậy sách đúng khi bảo theo dõi **không gian khóa còn lại** (tr. 33), không phải số hàng.
- **Giá trị cũ trong `information_schema`:** ở B4, sau khi thêm 20 hàng, query vẫn báo `next_value` 2000000003; chỉ khi đặt `information_schema_stats_expiry` = 0 mới thấy 2000000023. Query của sách viết cho `information_schema` (tr. 33) vẫn dùng được trên 8.0, nhưng job giám sát nên tắt cache trong phiên của nó. Với ngưỡng cảnh báo tính theo tuần thì trễ một ngày thường chấp nhận được; với bảng tăng rất nhanh thì không.
- **Khi cạn khóa**, mọi insert vào bảng đều lỗi, dưới hai mã khác nhau (1467 và 1062). Đây là sự cố "mất ghi" cho đúng tính năng dùng bảng đó, điều sách muốn tránh bằng cách dự đoán trước (tr. 33).
- Cột `max_value` trong query của sách thực chất là **phần trăm đã dùng**; với primary key nhiều cột, query trả trùng dòng. Không ảnh hưởng ý chính của sách, nhưng cần biết khi dùng làm cảnh báo.

**Dọn dẹp và khôi phục.** Các bảng `t_*` nằm trong schema `ch02`, xóa ở bước cuối. Không có biến toàn cục nào bị đổi (`information_schema_stats_expiry` chỉ đổi ở mức phiên).

---

<a id="cleanup"></a>

## Dọn dẹp cuối cùng

```sql
SET GLOBAL max_connections = 151;
SET GLOBAL super_read_only = OFF; SET GLOBAL read_only = OFF;
SET GLOBAL event_scheduler = ON;
DROP USER IF EXISTS 'app'@'127.0.0.1';
DROP DATABASE IF EXISTS ch02;
```

```bash
"$BIN/mysqladmin" -h127.0.0.1 -P3312 -uroot shutdown
```

Trong lần chạy của ghi chú này, các biến toàn cục đã được khôi phục ngay trong từng script (`max_connections` = 151, `read_only` và `super_read_only` = 0, `event_scheduler` = ON), và instance đã được tắt bằng `mysqladmin shutdown` (kết nối lại cổng 3312 báo lỗi 2003, xác nhận đã tắt). Hai lệnh `DROP` ở trên **chưa** chạy: schema `ch02` và user `app` còn nằm trong datadir riêng của thí nghiệm, không ảnh hưởng instance nào khác. Có thể xóa cả datadir nếu không cần chạy lại.

## Những phần của chương chưa thực thi

| Nội dung sách | Lý do chưa chạy | Cách làm nếu có môi trường |
| --- | --- | --- |
| Latency phía client (tr. 27) | Cần một ứng dụng thật có instrument | Ghi thời gian quanh mỗi lời gọi database trong ứng dụng, gửi histogram về hệ thống chỉ số, so với `QUANTILE_99` phía server |
| PMM và cờ `-collect.auto_increment.columns` (tr. 24, 33) | Không cài phần mềm ngoài | Cài PMM client và server trên môi trường thử, bật cờ của exporter, so với query ở TN5 |
| `pt-query-digest`, `pt-kill` (tr. 24, 31) | Percona Toolkit không có sẵn, không tải từ internet | Bật slow log với `long_query_time` thấp, chạy workload, phân tích bằng `pt-query-digest` |
| `iostat`, `IOwait`, `IOutil` (tr. 32–33) | Công cụ Linux | Trên Linux: `iostat -x 1` trong lúc chạy một query quét bảng lớn, xem `%util`, `r_await`, `aqu-sz` |
| Thời gian restore (tr. 34–35) | Thuộc chương 10 | Xem [chương 10](../chapter-10.md) |
