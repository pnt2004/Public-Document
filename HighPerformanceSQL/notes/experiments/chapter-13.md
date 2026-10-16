# Thực hành — Chương 13: Compliance với MySQL

Liên kết: [về ghi chú chương 13](../chapter-13.md) · [đáp án tự kiểm tra](../answers/chapter-13.md)

Năm thí nghiệm dưới đây kiểm chứng những khẳng định của chương 13 có thể kiểm trên một instance MySQL: user riêng và role với quyền tối thiểu, xoay vòng credential (đổi username và dual password), cách biết một user còn được dùng không bằng Performance Schema, khóa tài khoản, nơi mật khẩu có thể lộ trong log, và chi phí của trigger dùng để audit. Phần cần Percona Server, rsyslog hay cloud được viết thành quy trình ở [mục cuối](#chua-thuc-thi).

> **Mọi mật khẩu trong bài là giá trị giả** (ví dụ `Fake-Pass-v1-NOT-REAL`). Không dùng chúng ở đâu khác, và không bao giờ chạy các bài này trên server có dữ liệu thật.

<a id="moi-truong"></a>

## Môi trường

| Mục | Giá trị |
| --- | --- |
| MySQL | **8.0.42 Community Server - GPL** (`@@version_comment`) |
| Máy | Laptop Windows 11, Git Bash; nhiều tiến trình nền, **không phải benchmark production** |
| Instance | Riêng, datadir trong scratchpad, cổng **3323**, `--no-defaults`, `--bind-address=127.0.0.1`, `--mysqlx=OFF` |
| Khởi tạo | `--initialize-insecure` (root không mật khẩu, chỉ nghe ở 127.0.0.1) |
| Cấu hình | Mặc định của 8.0.42: `binlog_format=ROW`, `log_bin=ON`, `sync_binlog=1`, `innodb_flush_log_at_trx_commit=1`, `performance_schema=ON`, `default_authentication_plugin=caching_sha2_password` |
| Dữ liệu | Tổng hợp bằng recursive CTE: `shop.orders` 200.000 hàng, `shop.cards` 100 hàng (token giả `tok_fake_0001`…) |

**Khởi tạo, chạy và tắt** (Git Bash; `BIN` là thư mục `bin` của MySQL 8.0, `DD` là thư mục dữ liệu riêng):

```bash
BIN="/c/Program Files/MySQL/MySQL Server 8.0/bin"
DD="<thư mục scratchpad>/hpm/mysql/ch13/main/data"
"$BIN/mysqld" --no-defaults --initialize-insecure --datadir="$DD" --console
# chạy nền
"$BIN/mysqld" --no-defaults --datadir="$DD" --port=3323 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=ch13.sock --console > run.log 2>&1 &
# chờ server sẵn sàng rồi kiểm tra đúng instance
until "$BIN/mysql" -h127.0.0.1 -P3323 -uroot -e "SELECT 1" >/dev/null 2>&1; do sleep 1; done
"$BIN/mysql" -h127.0.0.1 -P3323 -uroot -e "SELECT @@port, @@datadir, @@version"
# ... chạy thí nghiệm ...
"$BIN/mysqladmin" -h127.0.0.1 -P3323 -uroot shutdown
```

Lưu ý khi đọc output: kết nối TCP tới `127.0.0.1` được server nhận là host `localhost`, nên thông báo lỗi ghi `'svc_…'@'localhost'` dù tài khoản tạo với host `'%'`.

---

<a id="tn1"></a>

## Thí nghiệm 1 — User riêng, role và quyền tối thiểu

**Mục tiêu.** Kiểm chứng cách cài "separate database users" và least privilege (tr. 330, 332) bằng role của MySQL 8.0, và tìm bẫy khi gán role.

**Giả thuyết (Dự đoán).** User được gán role `r_orders_rw` chỉ đọc, thêm, sửa được `shop.orders`, bị từ chối mọi thứ khác. Chưa rõ: gán role xong user có dùng được quyền ngay không.

**Chuẩn bị.**

```sql
CREATE DATABASE shop;
CREATE TABLE shop.orders (
  id          INT PRIMARY KEY AUTO_INCREMENT,
  customer_id INT NOT NULL,
  amount      DECIMAL(10,2) NOT NULL,
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE shop.cards (
  id          INT PRIMARY KEY AUTO_INCREMENT,
  customer_id INT NOT NULL,
  card_token  VARCHAR(64) NOT NULL
);
SET SESSION cte_max_recursion_depth = 300000;
INSERT INTO shop.orders (customer_id, amount)
WITH RECURSIVE seq(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 200000)
SELECT n % 5000, (n % 997) + 0.99 FROM seq;
INSERT INTO shop.cards (customer_id, card_token)
WITH RECURSIVE seq(n) AS (SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 100)
SELECT n, CONCAT('tok_fake_', LPAD(n, 4, '0')) FROM seq;

CREATE ROLE 'r_orders_rw', 'r_cards_ro';
GRANT SELECT, INSERT, UPDATE ON shop.orders TO 'r_orders_rw';
GRANT SELECT ON shop.cards TO 'r_cards_ro';
CREATE USER 'svc_orders_v1'@'%' IDENTIFIED BY 'Fake-Pass-v1-NOT-REAL';
GRANT 'r_orders_rw' TO 'svc_orders_v1'@'%';
```

**Các bước và lệnh đo.**

1. Xem `@@activate_all_roles_on_login`, `SHOW GRANTS` (có và không có `USING`), `mysql.default_roles`.
2. Đăng nhập bằng `svc_orders_v1`, chạy `SELECT CURRENT_ROLE()` và `SELECT COUNT(*) FROM shop.orders`; thử `SET ROLE 'r_orders_rw'` trong phiên.
3. Root chạy `SET DEFAULT ROLE ALL TO 'svc_orders_v1'@'%'`; đăng nhập lại, thử đọc `orders`, `INSERT` một hàng, đọc `cards`, `DELETE`, `DROP TABLE`, `SHOW DATABASES`.

Để client chạy tiếp sau lỗi, đưa câu lệnh qua stdin với `--force` (với `-e`, client dừng ở lỗi đầu tiên).

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định, dữ liệu tổng hợp 200.000 hàng; output đã rút gọn):

```text
@@activate_all_roles_on_login = 0, @@mandatory_roles = ''

SHOW GRANTS FOR 'svc_orders_v1'@'%':
  GRANT USAGE ON *.* TO `svc_orders_v1`@`%`
  GRANT `r_orders_rw`@`%` TO `svc_orders_v1`@`%`
SHOW GRANTS FOR 'svc_orders_v1'@'%' USING 'r_orders_rw':
  GRANT USAGE ON *.* TO `svc_orders_v1`@`%`
  GRANT SELECT, INSERT, UPDATE ON `shop`.`orders` TO `svc_orders_v1`@`%`
  GRANT `r_orders_rw`@`%` TO `svc_orders_v1`@`%`
mysql.default_roles: (rỗng)

-- Lượt 1: đăng nhập svc_orders_v1, chưa có default role
CURRENT_USER() = svc_orders_v1@%, CURRENT_ROLE() = NONE
ERROR 1142 (42000) at line 1: SELECT command denied to user 'svc_orders_v1'@'localhost' for table 'orders'

-- root: SET DEFAULT ROLE ALL TO 'svc_orders_v1'@'%'
mysql.default_roles: HOST %, USER svc_orders_v1, DEFAULT_ROLE_HOST %, DEFAULT_ROLE_USER r_orders_rw

-- Lượt 2: đăng nhập lại
CURRENT_ROLE() = `r_orders_rw`@`%`
SELECT COUNT(*) FROM shop.orders                 -> 200000
INSERT INTO shop.orders(...) VALUES (1, 9.99)    -> ROW_COUNT() = 1

-- Lượt 3: root đặt lại SET DEFAULT ROLE NONE; ứng dụng tự SET ROLE trong phiên
CURRENT_ROLE() = NONE
ERROR 1142 (42000) at line 2: SELECT command denied to user 'svc_orders_v1'@'localhost' for table 'orders'
SET ROLE 'r_orders_rw';  -> CURRENT_ROLE() = `r_orders_rw`@`%`, COUNT(*) = 200001

-- Lượt 4: root SET DEFAULT ROLE ALL lần nữa; thử các thao tác vượt quyền (stdin, --force)
ERROR 1142 (42000) at line 1: SELECT command denied to user 'svc_orders_v1'@'localhost' for table 'cards'
ERROR 1142 (42000) at line 2: DELETE command denied to user 'svc_orders_v1'@'localhost' for table 'orders'
ERROR 1142 (42000) at line 3: DROP command denied to user 'svc_orders_v1'@'localhost' for table 'orders'
SHOW DATABASES -> information_schema, performance_schema, shop
```

(200001 ở lượt 3 là do hàng `INSERT` ở lượt 2.)

**Diễn giải và đối chiếu với sách.**

- Least privilege (tr. 332) cài được gọn bằng role: quyền thừa (đọc bảng thẻ, xóa, drop) đều bị từ chối với `ERROR 1142`. `SHOW DATABASES` chỉ hiện schema user có quyền.
- **Bẫy:** `GRANT role` chưa làm role có hiệu lực. Vì `activate_all_roles_on_login = OFF`, phải có default role (hoặc ứng dụng tự `SET ROLE`). Một user mới "đã gán đúng role" vẫn có thể gây sự cố khi deploy. Đây là điều sách không nói, và quan trọng cho bước 2 của quy trình xoay vòng (tr. 337).
- `CREATE USER … DEFAULT ROLE …` vừa cấp vừa kích hoạt role (đã thấy ở thí nghiệm 2 với `svc_report`, user này đọc được `orders` mà không có `GRANT` riêng).

**Dọn dẹp.** Giữ schema và role cho thí nghiệm sau. Cuối buổi: `DROP DATABASE shop; DROP ROLE 'r_orders_rw', 'r_cards_ro', 'r_orders_ro';` và drop các user `svc_%`, hoặc xóa cả datadir riêng.

---

<a id="tn2"></a>

## Thí nghiệm 2 — Xoay vòng credential: đổi username và dual password

**Mục tiêu.** Chạy thử quy trình 5 bước của sách (tr. 337) và dual password của MySQL 8.0.14+ (hộp mẹo tr. 337); tìm điều gì xảy ra với phiên đang mở.

**Giả thuyết (Dự đoán).**

- So `SHOW GRANTS` đủ để biết user mới có cùng quyền (theo bước 2 của sách).
- Drop user cũ khi còn phiên mở thì phiên đó vẫn chạy tiếp tới khi ngắt.
- Dual password cho phép hai mật khẩu cùng lúc; discard thì kết nối mới bằng mật khẩu cũ bị từ chối.

### 2a. Cách của sách: user mới

**Các bước.**

```bash
# Bước 1: tạo user mới, cố ý QUÊN default role
mysql -h127.0.0.1 -P3323 -uroot -e "CREATE USER 'svc_orders_v2'@'%' IDENTIFIED BY 'Fake-Pass-v2-NOT-REAL'; GRANT 'r_orders_rw' TO 'svc_orders_v2'@'%';"
# Bước 2: so SHOW GRANTS sau khi thay tên user bằng một chuỗi chung
mysql -h127.0.0.1 -P3323 -uroot -N -e "SHOW GRANTS FOR 'svc_orders_v1'@'%'" | sed 's/svc_orders_v1/SVC/g' > g_v1.txt
mysql -h127.0.0.1 -P3323 -uroot -N -e "SHOW GRANTS FOR 'svc_orders_v2'@'%'" | sed 's/svc_orders_v2/SVC/g' > g_v2.txt
diff g_v1.txt g_v2.txt && echo "SHOW GRANTS: GIONG NHAU"
```

```sql
-- So thêm default role
SELECT u.User, u.Host, COUNT(d.DEFAULT_ROLE_USER) AS n_default_roles
FROM mysql.user u
LEFT JOIN mysql.default_roles d ON d.USER = u.User AND d.HOST = u.Host
WHERE u.User LIKE 'svc_orders_v%'
GROUP BY u.User, u.Host;
```

Sau đó: smoke test đăng nhập bằng v2; sửa bằng `SET DEFAULT ROLE ALL`. Cuối cùng mở một phiên v1 chạy `SELECT SLEEP(6)` rồi `SELECT COUNT(*)` và `INSERT`; trong lúc phiên đang ngủ, root xem `performance_schema.threads` và `DROP USER 'svc_orders_v1'@'%'`.

```bash
( printf "SELECT NOW(6), CURRENT_USER();\nSELECT SLEEP(6);\nSELECT COUNT(*) FROM shop.orders;\nINSERT INTO shop.orders(customer_id, amount) VALUES (2, 1.00);\n" \
  | mysql -h127.0.0.1 -P3323 -usvc_orders_v1 --password='Fake-Pass-v1-NOT-REAL' -t --force > open_session_v1.txt 2>&1 ) &
sleep 2
mysql -h127.0.0.1 -P3323 -uroot -e "SELECT PROCESSLIST_ID, PROCESSLIST_USER, PROCESSLIST_STATE FROM performance_schema.threads WHERE PROCESSLIST_USER='svc_orders_v1'; DROP USER 'svc_orders_v1'@'%'; SHOW WARNINGS;"
wait; cat open_session_v1.txt
```

**Kết quả đã đo** (rút gọn):

```text
GRANT USAGE ON *.* TO `SVC`@`%`
GRANT `r_orders_rw`@`%` TO `SVC`@`%`
SHOW GRANTS: GIONG NHAU

User           Host  n_default_roles
svc_orders_v1  %     1
svc_orders_v2  %     0

-- smoke test v2 trước khi sửa
ERROR 1142 (42000) at line 1: SELECT command denied to user 'svc_orders_v2'@'localhost' for table 'orders'
-- sau SET DEFAULT ROLE ALL TO v2
CURRENT_USER() = svc_orders_v2@%, COUNT(*) = 200001

-- phiên v1 đang ngủ lúc DROP
PROCESSLIST_ID 24, svc_orders_v1, localhost, Query, User sleep
DROP USER: thành công, SHOW WARNINGS rỗng; mysql.user không còn svc_orders_v1
-- output của chính phiên v1
t_start = 14:26:05.75, CURRENT_USER() = svc_orders_v1@%
SLEEP(6) -> 0
ERROR 1142 (42000) at line 3: SELECT command denied to user 'svc_orders_v1'@'localhost' for table 'orders'
ERROR 1142 (42000) at line 4: INSERT command denied to user 'svc_orders_v1'@'localhost' for table 'orders'
-- kết nối mới bằng v1
ERROR 1045 (28000): Access denied for user 'svc_orders_v1'@'localhost' (using password: YES)
```

### 2b. Dual password (MySQL 8.0.14+)

**Các bước.**

```sql
CREATE ROLE 'r_orders_ro';
GRANT SELECT ON shop.orders TO 'r_orders_ro';
CREATE USER 'svc_report'@'%' IDENTIFIED BY 'Fake-Pass-A-111' DEFAULT ROLE 'r_orders_ro';
ALTER USER 'svc_report'@'%' IDENTIFIED BY 'Fake-Pass-B-222' RETAIN CURRENT PASSWORD;
SELECT User, JSON_KEYS(User_attributes) AS attr_keys FROM mysql.user WHERE User = 'svc_report';
ALTER USER 'svc_report'@'%' IDENTIFIED BY 'Fake-Pass-C-333' RETAIN CURRENT PASSWORD;
-- (mở một phiên bằng mật khẩu B đang SLEEP(5), rồi:)
ALTER USER 'svc_report'@'%' DISCARD OLD PASSWORD;
-- thử giữ mật khẩu rỗng làm mật khẩu phụ
CREATE USER 'svc_nopass'@'%';
ALTER USER 'svc_nopass'@'%' IDENTIFIED BY 'Fake-Pass-X-999' RETAIN CURRENT PASSWORD;
```

Sau mỗi bước, thử đăng nhập bằng từng mật khẩu với `mysql -usvc_report --password=… -e "SELECT 'LOGIN_OK'"`.

**Kết quả đã đo:**

| Thời điểm | A (`…-111`) | B (`…-222`) | C (`…-333`) | `User_attributes` |
| --- | --- | --- | --- | --- |
| Sau `CREATE USER` với A | đăng nhập được | — | — | — |
| Sau đổi sang B, `RETAIN` | đăng nhập được | đăng nhập được | — | `["additional_password"]` |
| Sau đổi sang C, `RETAIN` | `ERROR 1045` | đăng nhập được | đăng nhập được | |
| Sau `DISCARD OLD PASSWORD` | `ERROR 1045` | `ERROR 1045` (kết nối mới) | đăng nhập được | `NULL` |

```text
-- phiên đăng nhập bằng B, đang ngủ khi DISCARD chạy
slept = 0
s = still_works, COUNT(*) = 200001
-- giữ mật khẩu rỗng
ERROR 3878 (HY000): Empty password can not be retained as second password for user 'svc_nopass'@'%'.
```

### 2c. Hết hạn mật khẩu

**Các bước.**

```sql
SELECT @@default_password_lifetime, @@disconnect_on_expired_password;
ALTER USER 'svc_report'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;
SELECT User, password_lifetime, password_expired, password_last_changed
FROM mysql.user WHERE User = 'svc_report';
ALTER USER 'svc_report'@'%' PASSWORD EXPIRE;   -- buộc hết hạn ngay
```

Rồi đăng nhập bằng `svc_report` theo ba cách: `mysql -e`, `mysql` đọc stdin, `mysql --connect-expired-password`; cuối cùng root đặt mật khẩu mới.

**Kết quả đã đo:**

```text
@@default_password_lifetime = 0, @@disconnect_on_expired_password = 1
svc_report: password_lifetime = 90, password_expired = N

-- sau PASSWORD EXPIRE
mysql -e        : Please use --connect-expired-password option or invoke mysql in interactive mode.
mysql (stdin)   : Please use --connect-expired-password option or invoke mysql in interactive mode.
mysql --connect-expired-password:
  ERROR 1820 (HY000) at line 1: You must reset your password using ALTER USER statement before executing this statement.
-- root: ALTER USER ... IDENTIFIED BY 'Fake-Pass-D-444'
password_expired = N, đăng nhập lại được
```

**Diễn giải và đối chiếu với sách.**

- Bước 2 của sách (so `SHOW GRANTS`, tr. 337) **cần nhưng chưa đủ** khi dùng role: hai user có `SHOW GRANTS` giống hệt nhưng một user không dùng được quyền. Kiểm tra tự động nên gồm `mysql.default_roles` và **một lần đăng nhập thật** với truy vấn đại diện.
- Bước 4 trước bước 5 (restart mọi instance trước khi drop) là **bắt buộc**, không chỉ là cẩn thận: `DROP USER` không chờ phiên cũ, và phiên còn sót mất quyền ngay ở câu kế tiếp. Ngược với giả thuyết ban đầu. Tài liệu MySQL nói `DROP USER` không tự đóng phiên đang mở; điều đó khớp (phiên vẫn kết nối), nhưng phiên không còn dùng được quyền trên bảng. Hãy kiểm tra `performance_schema.threads` trước khi drop.
- Dual password đúng như hộp mẹo của sách (tr. 337): giữ username, không cần tạo lại quyền. Chỉ giữ tối đa **hai** mật khẩu. Phiên đang mở không bị cắt khi discard, nên connection pool sống lâu vẫn chạy bằng mật khẩu cũ cho tới khi tái kết nối; đó là điều tốt cho availability nhưng cũng nghĩa là phải tái tạo pool để thật sự "chuyển" sang mật khẩu mới.
- Chính sách hết hạn (tr. 337 nhắc "password expiration policy") áp cho service account: khi hết hạn, client không xử lý được chế độ hạn chế sẽ không kết nối, client xử lý được thì mọi câu lệnh lỗi 1820. Với `default_password_lifetime = 0` (mặc định 8.0.42) mật khẩu không tự hết hạn.

**Dọn dẹp.** `DROP USER 'svc_orders_v2'@'%', 'svc_report'@'%';` (`svc_orders_v1` đã bị drop, `svc_nopass` cần drop nếu còn). Xóa tệp `g_v1.txt`, `g_v2.txt`, `open_session_*.txt`.

---

<a id="tn3"></a>

## Thí nghiệm 3 — User còn được dùng không? Performance Schema và khóa tài khoản

**Mục tiêu.** Kiểm chứng cách sách đề xuất để biết một user còn được dùng (tr. 338): bật instrument `memory/sql/user_conn` rồi xem `performance_schema.users`. Thử thêm `ACCOUNT LOCK` và `FAILED_LOGIN_ATTEMPTS` như bước "nghỉ hưu" tài khoản.

**Giả thuyết (Dự đoán), theo sách.** Phải bật `memory/sql/user_conn` thì bảng `users` mới có dữ liệu; dữ liệu chỉ kéo dài trong đời tiến trình server.

**Các bước và lệnh đo.**

```sql
-- 1) Trạng thái mặc định
SELECT NAME, ENABLED, TIMED FROM performance_schema.setup_instruments
WHERE NAME = 'memory/sql/user_conn';
SHOW VARIABLES LIKE 'performance_schema_%s_size';   -- xem users_size, accounts_size

-- 2) TẮT instrument mà sách bảo bật, rồi tạo hai user mới
UPDATE performance_schema.setup_instruments SET ENABLED = 'NO'
WHERE NAME = 'memory/sql/user_conn';
CREATE USER 'svc_probe'@'%'  IDENTIFIED BY 'Fake-Pass-P-555';
CREATE USER 'svc_unused'@'%' IDENTIFIED BY 'Fake-Pass-U-666';
```

Kết nối 3 lần bằng `svc_probe` (`DO 1`), 2 lần bằng `svc_unused` với **mật khẩu sai**, rồi:

```sql
SELECT USER, CURRENT_CONNECTIONS, TOTAL_CONNECTIONS FROM performance_schema.users ORDER BY USER;
SELECT USER, HOST, CURRENT_CONNECTIONS, TOTAL_CONNECTIONS FROM performance_schema.accounts ORDER BY USER, HOST;
SHOW GLOBAL STATUS LIKE 'Performance_schema_users_lost';

-- 3) Tài khoản đăng nhập được mà chưa kết nối lần nào từ khi server khởi động
SELECT u.User, u.Host, u.password_last_changed,
       COALESCE(pu.TOTAL_CONNECTIONS, 0) AS conns_since_startup
FROM mysql.user AS u
LEFT JOIN performance_schema.users AS pu ON pu.USER = u.User
WHERE u.account_locked = 'N'
ORDER BY conns_since_startup, u.User;
```

4) `mysqladmin shutdown`, khởi động lại, chạy lại truy vấn ở bước 3 và xem lại `setup_instruments`.

5) Khóa và mở khóa; khóa tạm do đăng nhập sai (MySQL 8.0.19+):

```sql
ALTER USER 'svc_unused'@'%' ACCOUNT LOCK;     -- rồi thử đăng nhập
ALTER USER 'svc_unused'@'%' ACCOUNT UNLOCK;   -- rồi thử lại
ALTER USER 'svc_probe'@'%' FAILED_LOGIN_ATTEMPTS 3 PASSWORD_LOCK_TIME 1;
-- thử 4 lần mật khẩu sai, rồi 1 lần mật khẩu đúng
SELECT User, account_locked, User_attributes FROM mysql.user WHERE User = 'svc_probe';
ALTER USER 'svc_probe'@'%' ACCOUNT UNLOCK;   -- rồi thử mật khẩu đúng
```

6) Chạy đúng câu in trong sách, với **nháy cong** quanh `YES` như bản in, và với nháy thẳng.

**Kết quả đã đo** (rút gọn):

```text
-- 1) mặc định
memory/sql/user_conn  ENABLED = YES  TIMED = NULL
performance_schema_users_size = -1, accounts_size = -1, hosts_size = -1

-- 2) sau khi TẮT instrument (ENABLED = NO), 3 lần kết nối svc_probe, 2 lần svc_unused sai mật khẩu
performance_schema.users:
USER            CURRENT_CONNECTIONS  TOTAL_CONNECTIONS
NULL            33                   71
event_scheduler 1                    1
root            1                    31
svc_orders_v1   0                    5      <- user đã bị DROP ở TN2 vẫn còn hàng
svc_orders_v2   0                    2
svc_probe       0                    3
svc_report      0                    9
(svc_unused: không có hàng)
performance_schema.accounts: cùng các user, HOST = localhost
Performance_schema_users_lost = 0, Performance_schema_accounts_lost = 0

-- 3) trước restart (uptime 168 s)
User           Host       password_last_changed  conns_since_startup
svc_unused     %          2026-10-16 14:26:57    0
svc_orders_v2  %          2026-10-16 14:25:56    2
svc_probe      %          2026-10-16 14:26:57    3
svc_report     %          2026-10-16 14:26:46    9
root           localhost  2026-10-16 14:24:14    34

-- 4) sau restart (uptime 1 s)
svc_orders_v2 0 | svc_probe 0 | svc_report 0 | svc_unused 0 | root 3
memory/sql/user_conn  ENABLED = YES   (thay đổi lúc chạy đã mất)

-- 5) khóa
ERROR 3118 (HY000): Access denied for user 'svc_unused'@'localhost'. Account is locked.
(sau ACCOUNT UNLOCK) LOGIN_OK_AFTER_UNLOCK
lần sai 1: ERROR 1045 (28000): Access denied for user 'svc_probe'@'localhost' (using password: YES)
lần sai 2: ERROR 1045 (28000): ...
lần sai 3: ERROR 3955 (HY000): Access denied for user 'svc_probe'@'localhost'. Account is blocked for 1 day(s) (1 day(s) remaining) due to 3 consecutive failed logins.
lần sai 4: ERROR 3955 ...
mật khẩu ĐÚNG ngay sau đó: ERROR 3955 ...
account_locked = N, User_attributes = {"Password_locking": {"failed_login_attempts": 3, "password_lock_time_days": 1}}
(sau ACCOUNT UNLOCK) LOGIN_OK_AFTER_ADMIN_UNLOCK

-- 6) câu của sách
nháy cong : ERROR 1054 (42S22) at line 1: Unknown column '‘YES’' in 'field list'
nháy thẳng: ROW_COUNT() = 0
```

**Diễn giải và đối chiếu với sách.**

- Trên 8.0.42, bước bật `memory/sql/user_conn` (tr. 338) **không cần**: instrument đã bật sẵn, và kể cả khi tắt, bảng `users` vẫn ghi. Output "1 row affected" của sách có lẽ đến từ môi trường mà instrument này đang tắt (**Dự đoán**, chưa kiểm trên bản cũ hơn).
- Đúng như sách (tr. 338), bảng chỉ ghi user **đã** kết nối: `svc_unused` không có hàng, nên phải đối chiếu với danh sách user đã biết. Truy vấn ở bước 3 làm việc đó và tự loại role, tài khoản hệ thống (`account_locked = 'Y'`).
- Đăng nhập **sai** không tạo hàng; một user chỉ bị dò mật khẩu sẽ vẫn trông như "không dùng".
- Bảng gom theo **tên user**, không theo host, và còn giữ hàng của user đã bị drop.
- Restart xóa sạch bộ đếm và cả thay đổi `setup_instruments` lúc chạy. Chính sách "sáu tháng" (tr. 338) cần một job lưu định kỳ ra nơi bền, hoặc audit log tập trung.
- `ACCOUNT LOCK` là bước "nghỉ hưu" đảo ngược được, hợp làm bước đệm trước `DROP USER`.
- `FAILED_LOGIN_ATTEMPTS` chặn cả mật khẩu đúng trong thời gian khóa: hữu ích chống dò mật khẩu, nhưng nguy hiểm cho service account trong lúc xoay vòng (instance cũ thử mật khẩu đã discard có thể khóa cả fleet). `account_locked` vẫn là `N`, nên truy vấn chỉ nhìn cột này sẽ không thấy khóa tạm.

**Dọn dẹp.** `DROP USER 'svc_probe'@'%', 'svc_unused'@'%';`. Không cần khôi phục `setup_instruments` vì restart đã đưa về mặc định; nếu không restart, chạy lại câu `UPDATE` với `ENABLED = 'YES'`.

---

<a id="tn4"></a>

## Thí nghiệm 4 — Mật khẩu có xuất hiện dạng thô trong log của server không?

**Mục tiêu.** Kiểm chứng phần phía server của lời khuyên "mật khẩu luôn được lưu mã hóa, không bao giờ ở dạng văn bản thô" (tr. 341): general log, Performance Schema, binary log và `mysql.user`.

**Giả thuyết (Dự đoán).** Câu quản lý tài khoản được viết lại trước khi ghi log; binary log chứa hash thay cho mật khẩu. Secret trong câu SQL thông thường thì không được che.

**Các bước và lệnh đo.**

```sql
FLUSH BINARY LOGS;                     -- binlog mới cho dễ đọc
SET GLOBAL log_output = 'TABLE';
SET GLOBAL general_log = ON;
CREATE USER 'svc_tmp'@'%' IDENTIFIED BY 'Fake-Secret-ZZZ-777';
ALTER USER 'svc_tmp'@'%' IDENTIFIED BY 'Fake-Secret-YYY-888' RETAIN CURRENT PASSWORD;
SET PASSWORD FOR 'svc_tmp'@'%' = 'Fake-Secret-XXX-999';
SELECT 'Fake-Secret-in-plain-SELECT' AS oops;
SET GLOBAL general_log = OFF;
SELECT command_type, argument FROM mysql.general_log
WHERE argument LIKE '%svc_tmp%' OR argument LIKE '%Fake-Secret%';

-- Performance Schema: đọc lịch sử của chính phiên (history_long tắt mặc định)
CREATE USER 'svc_tmp2'@'%' IDENTIFIED BY 'Fake-Secret-WWW-123';
SELECT SQL_TEXT, DIGEST_TEXT FROM performance_schema.events_statements_history
WHERE THREAD_ID = PS_CURRENT_THREAD_ID() AND SQL_TEXT LIKE '%svc_tmp2%IDENTIFIED%';

SELECT User, plugin, LEFT(authentication_string, 7) AS prefix,
       LENGTH(authentication_string) AS len
FROM mysql.user WHERE User LIKE 'svc_tmp%';
```

```bash
mysqlbinlog "$DD/binlog.000003" > binlog3.txt
grep -c 'Fake-Secret-[ZYXW]' binlog3.txt          # trong bản giải mã
grep -a -c 'Fake-Secret-[ZYXW]' "$DD/binlog.000003" # trong tệp nhị phân
grep -a -E "USER|PASSWORD" binlog3.txt            # xem dạng câu (không in hash ra tài liệu)
```

**Kết quả đã đo** (hash bị che khi chép vào đây):

```text
mysql.general_log:
Query | CREATE USER 'svc_tmp'@'%' IDENTIFIED BY <secret>
Query | ALTER USER 'svc_tmp'@'%' IDENTIFIED BY <secret> RETAIN CURRENT PASSWORD
Query | SET PASSWORD FOR `svc_tmp`@`%`=<secret>
Query | SELECT 'Fake-Secret-in-plain-SELECT' AS oops

events_statements_history:
SQL_TEXT    = CREATE USER 'svc_tmp2'@'%' IDENTIFIED BY <secret>
DIGEST_TEXT = CREATE SYSTEM_USER ? @? IDENTIFIED BY ?

setup_consumers: events_statements_history = YES, events_statements_history_long = NO

binlog.000003: 'Fake-Secret' trong bản giải mã = 0, trong tệp nhị phân = 0
CREATE USER 'svc_tmp'@'%' IDENTIFIED WITH 'caching_sha2_password' AS '<hash>'
ALTER USER 'svc_tmp'@'%' IDENTIFIED WITH 'caching_sha2_password' AS '<hash>' RETAIN ...
ALTER USER 'svc_tmp'@'%' IDENTIFIED WITH 'caching_sha2_password' AS '<hash>'   <- từ SET PASSWORD
CREATE USER 'svc_tmp2'@'%' IDENTIFIED WITH 'caching_sha2_password' AS '<hash>'

mysql.user: plugin = caching_sha2_password, prefix = $A$005$, len = 70
```

**Diễn giải và đối chiếu với sách.**

- Phía server đáp ứng phần "không lưu dạng thô" của Summary (tr. 341): `mysql.user` giữ hash, log và Performance Schema thấy `<secret>`, binary log ghi lại câu dưới dạng `IDENTIFIED WITH … AS '<hash>'` để replica tái tạo đúng tài khoản.
- **Binary log và backup chứa hash mật khẩu.** Hash không phải mật khẩu, nhưng vẫn là dữ liệu nhạy cảm (dùng để dò offline). Đây thêm một lý do cho lời khuyên kiểm soát truy cập backup và bucket (tr. 340).
- Câu `SELECT` chứa chuỗi bí mật xuất hiện **nguyên văn** trong general log. Rewrite chỉ áp cho cú pháp quản lý tài khoản. Token, API key ứng dụng ghi vào bảng sẽ có mặt trong general log (nếu bật), trong binlog (row event) và trong backup.
- `DIGEST_TEXT` ghi `CREATE SYSTEM_USER ? @? …` là một kỳ quặc của bộ chuẩn hóa digest trên 8.0.42; không ảnh hưởng kết luận.

**Dọn dẹp.**

```sql
TRUNCATE TABLE mysql.general_log;
SET GLOBAL log_output = 'FILE';
DROP USER 'svc_tmp'@'%', 'svc_tmp2'@'%';
```

Xóa `binlog3.txt`. Binary log trong datadir riêng sẽ mất khi xóa datadir.

---

<a id="tn5"></a>

## Thí nghiệm 5 — Trigger để audit: chi phí, quy kết người thay đổi, và audit plug-in trên Community

**Mục tiêu.** Kiểm chứng lý do sách không khuyên dùng trigger để theo dõi thay đổi (tr. 334): chi phí ghi, chỉ thấy ghi. Thử thêm cách trigger quy kết người thay đổi, và xem MySQL Community có cài được audit plug-in không.

**Giả thuyết (Dự đoán).** Trigger `AFTER UPDATE` ghi bảng audit làm `Handler_write` tăng đúng bằng số hàng bị cập nhật, binlog lớn hơn, thời gian tăng. Mức tăng thời gian chưa đoán được.

**Chuẩn bị.**

```sql
CREATE TABLE shop.orders_audit (
  audit_id     BIGINT AUTO_INCREMENT PRIMARY KEY,
  order_id     INT NOT NULL,
  old_amount   DECIMAL(10,2),
  new_amount   DECIMAL(10,2),
  session_user VARCHAR(288),
  definer_user VARCHAR(288),
  changed_at   DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
);
```

Tệp đo `e5_run.sql` (chạy bằng root, mỗi lần một phiên mới):

```sql
SHOW MASTER STATUS;                          -- vị trí binlog trước
FLUSH STATUS;
UPDATE shop.orders SET amount = amount + 0.01;   -- 200.001 hàng
SELECT ROUND(TIMER_WAIT/1e12, 3) AS update_seconds, ROWS_AFFECTED
FROM performance_schema.events_statements_history
WHERE THREAD_ID = PS_CURRENT_THREAD_ID() AND SQL_TEXT LIKE 'UPDATE shop.orders %'
ORDER BY EVENT_ID DESC LIMIT 1;
SHOW SESSION STATUS WHERE Variable_name IN
  ('Handler_update','Handler_write','Handler_read_rnd_next');
SHOW MASTER STATUS;                          -- vị trí binlog sau
```

Chạy 3 lần không trigger; tạo trigger; chạy 3 lần có trigger:

```sql
CREATE TRIGGER shop.trg_orders_au AFTER UPDATE ON shop.orders FOR EACH ROW
  INSERT INTO shop.orders_audit (order_id, old_amount, new_amount, session_user, definer_user)
  VALUES (OLD.id, OLD.amount, NEW.amount, USER(), CURRENT_USER());
```

Workload thứ hai, các cập nhật nhỏ commit từng câu:

```sql
DELIMITER //
CREATE PROCEDURE shop.p_small_updates(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 1;
  WHILE i <= n DO
    UPDATE shop.orders SET amount = amount + 0.01 WHERE id = i;
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
-- đo: FLUSH STATUS; CALL shop.p_small_updates(2000); rồi TIMER_WAIT của câu CALL và Handler_*
```

Cuối cùng: `svc_orders_v2` (không có quyền trên `orders_audit`) cập nhật một hàng; `svc_report` đọc 1.000 hàng; root xem bảng audit; thử `INSTALL PLUGIN audit_log SONAME 'audit_log.dll'`.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, laptop nhiều nhiễu; `autocommit=1`, `sync_binlog=1`, `innodb_flush_log_at_trx_commit=1`):

Cập nhật một lần 200.001 hàng:

| Lần | Trigger | Thời gian (s) | `Handler_update` | `Handler_write` | `Handler_read_rnd_next` | Byte binlog sinh ra |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | không | 2,457 | 200.001 | 0 | 200.002 | 7.633.958 |
| 2 | không | 2,299 | 200.001 | 0 | 200.002 | 7.633.958 |
| 3 | không | 2,278 | 200.001 | 0 | 200.002 | 7.633.958 |
| 1 | có | 22,080 | 200.001 | 200.001 | 200.002 | 34.400.659 |
| 2 | có | 21,942 | 200.001 | 200.001 | 200.002 | 34.400.659 |
| 3 | có | 26,020 | 200.001 | 200.001 | 200.002 | 34.400.659 |

Trung vị: **2,299 s** không trigger, **22,080 s** có trigger. Byte binlog là hiệu hai vị trí `SHOW MASTER STATUS`. Sau 3 lần có trigger: `orders_audit` có 600.003 hàng, `DATA_LENGTH` khoảng 44,6 MB so với 8,5 MB của `orders`.

2.000 câu `UPDATE` một hàng, mỗi câu tự commit:

| Lần | Trigger | Thời gian `CALL` (s) | `Handler_update` | `Handler_write` | `Handler_commit` |
| --- | --- | --- | --- | --- | --- |
| 1 | có | 3,871 | 2.000 | 2.000 | 4.001 |
| 2 | có | 3,917 | 2.000 | 2.000 | 4.000 |
| 3 | có | 3,903 | 2.000 | 2.000 | 4.000 |
| 1 | không | 3,819 | 2.000 | 0 | 4.005 |
| 2 | không | 4,131 | 2.000 | 0 | 4.000 |
| 3 | không | 4,457 | 2.000 | 0 | 4.000 |

Trung vị: **3,903 s** có trigger, **4,131 s** không trigger (chạy có trigger trước; khác biệt nằm trong nhiễu).

Quy kết và đọc:

```text
information_schema.TRIGGERS: trg_orders_au, DEFINER = root@localhost, AFTER UPDATE
-- svc_orders_v2: UPDATE shop.orders SET amount = 1.23 WHERE id = 42   -> thành công
-- svc_orders_v2: SELECT COUNT(*) FROM shop.orders_audit
ERROR 1142 (42000): SELECT command denied to user 'svc_orders_v2'@'localhost' for table 'orders_audit'
-- svc_report đọc 1000 hàng -> trigger không ghi gì
orders_audit:
audit_id order_id old_amount new_amount session_user             definer_user
1        42       43.11      1.23       svc_orders_v2@localhost  root@localhost

INSTALL PLUGIN audit_log SONAME 'audit_log.dll':
ERROR 1126 (HY000): Can't open shared library 'C:\Program Files\MySQL\MySQL Server 8.0\lib\plugin\audit_log.dll' (errno: 126 The specified module could not be found.)
information_schema.PLUGINS (PLUGIN_TYPE = 'AUDIT'): sha2_cache_cleaner, mysqlx_cache_cleaner
@@version_comment = MySQL Community Server - GPL
```

**Diễn giải và đối chiếu với sách.**

- **Chi phí ghi là thật và tất định** (tr. 334): mỗi hàng bị cập nhật sinh thêm một lần ghi (`Handler_write` = số hàng), binlog lớn khoảng 4,5 lần, bảng audit phình nhanh hơn bảng gốc.
- **Độ lớn về thời gian phụ thuộc workload.** Lô lớn chậm khoảng 9,6 lần trên máy này; câu nhỏ commit từng câu thì thời gian bị chi phối bởi fsync lúc commit nên không thấy khác biệt. Điều đó hợp với cụm "lúc tệ nhất" của sách: chi phí lộ rõ trong các job ghi hàng loạt và giờ cao điểm. Nguyên nhân chi tiết của mức 9,6 lần (thực thi trigger theo hàng, binlog cache tràn ra đĩa, hay cả hai) **chưa được đo riêng**; có thể kiểm thêm `Binlog_cache_disk_use`.
- **Chỉ thấy ghi** (tr. 334): đọc 1.000 hàng không để lại dấu vết nào.
- **Quy kết dễ sai** (**Bổ sung ngoài sách**): trigger chạy với quyền definer, nên `CURRENT_USER()` cho `root@localhost`; chỉ `USER()` cho người thực sự thay đổi. App user không cần (và không có) quyền trên bảng audit, điều này tốt, nhưng ai có quyền `TRIGGER` trên bảng có thể gỡ trigger.
- **Community không có audit plug-in** của Oracle: khác với câu ở tr. 334. Muốn audit ở tầng database với MySQL Community, cần đổi sang Percona Server hoặc Enterprise Edition, hoặc ghi ở proxy hay dịch vụ managed (chương 11 tr. 310, chương 12 tr. 314, 317).

**Dọn dẹp.**

```sql
DROP TRIGGER shop.trg_orders_au;
DROP PROCEDURE shop.p_small_updates;
DROP TABLE shop.orders_audit;
```

Cuối buổi: `DROP DATABASE shop;`, drop các role và user còn lại, rồi `mysqladmin -h127.0.0.1 -P3323 -uroot shutdown`. Có thể xóa cả datadir riêng.

---

<a id="chua-thuc-thi"></a>

## Chưa thực thi: quy trình cho phần cần Percona Server, Linux hoặc cloud

Các bài dưới đây **chưa chạy** trong môi trường này. Lý do: máy chỉ có MySQL 8.0.42 Community trên Windows; không có Percona Server, rsyslog hay dịch vụ cloud, và quy tắc của dự án không cho tải phần mềm.

### A. Audit log plug-in của Percona: bật, kiểm tra trạng thái thật, tắt không restart

Môi trường cần: Percona Server for MySQL 8.0 trên Linux, instance thử nghiệm.

1. `INSTALL PLUGIN audit_log SONAME 'audit_log.so';` rồi `SHOW PLUGINS;` (tr. 334).
2. Đặt policy chỉ ghi truy vấn, lọc loại lệnh theo tài liệu của đúng phiên bản (tr. 334–335).
3. Kiểm tra **trạng thái đang chạy**, không chỉ tệp cấu hình (tr. 335): truy vấn `information_schema.PLUGINS` và biến policy.
4. Đổi policy lúc chạy, xác nhận lệnh kiểm tra của bạn phát hiện được (đây là lý do sách nói `my.cnf` không đủ).
5. Đo overhead: chạy cùng workload (ví dụ 2.000 câu cập nhật nhỏ như TN5) với plug-in tắt và bật; so trung vị thời gian và byte log sinh ra.

**Dự đoán:** overhead nhỏ hơn trigger ở lô lớn vì plug-in không sinh thêm ghi vào InnoDB hay binlog; overhead phụ thuộc chiến lược ghi buffer và đích log.

### B. Chaos test pipeline log (cảnh báo tr. 336)

1. Cấu hình plug-in gửi qua rsyslog tới một collector.
2. Đo p50, p99 latency của một workload đọc ghi cố định khi pipeline khỏe.
3. Làm chậm hoặc chặn collector (ví dụ chặn cổng mạng của nó), giữ workload, đo lại.
4. Quan sát: latency truy vấn, dung lượng buffer, đĩa nơi lưu tạm.
5. Khôi phục collector; ghi lại thời gian để latency trở lại bình thường.

**Tiêu chí:** nếu latency truy vấn tăng khi chỉ đích log bị chậm, pipeline là một phụ thuộc hiệu năng của database và phải được giám sát như vậy.

### C. Backup tự động có bread crumb và số liệu thời gian (tr. 339–340)

Có thể làm một phần trên máy này bằng `mysqldump` vào instance riêng, nhưng bản chất (lịch chạy, monitoring, ticket, log tập trung) cần hạ tầng thật.

1. Job backup chạy theo lịch; khi xong, ghi timestamp vào một bread crumb ở nơi trung tâm và gửi số liệu thời gian chạy.
2. Job test restore khôi phục backup mới nhất vào instance tạm, chạy kiểm tra (đếm hàng, checksum bảng quan trọng), gửi số liệu thời gian khôi phục.
3. Monitoring kiểm tra bread crumb đủ mới (ví dụ dưới 26 giờ) mỗi vài phút; lỗi thì mở ticket.
4. Dashboard theo cluster: thời gian khôi phục so với MTTR mong muốn.

**Tiêu chí:** có hồ sơ liên tục cho cả thành công và thất bại; mỗi thất bại có ticket; biết cluster nào sắp vượt MTTR.
