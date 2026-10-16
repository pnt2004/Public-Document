# Thực hành — Chương 8: Tối ưu hiệu năng truy vấn

Quay lại ghi chú: [../chapter-08.md](../chapter-08.md) · Đáp án tự kiểm tra: [../answers/chapter-08.md](../answers/chapter-08.md)

> **Nhãn.** Schema, dữ liệu và SQL trong tệp này là **Ví dụ minh họa do AI xây dựng**. Phần "Giả thuyết" là **Dự đoán** viết dựa trên lời sách trước khi chạy. Phần "Kết quả" là **Kết quả đã đo**: output thật, có rút gọn (ghi rõ chỗ rút gọn). Không có số liệu nào được dựng sẵn.

## Môi trường

| Mục | Giá trị |
| --- | --- |
| MySQL | Community **8.0.42**, `mysqld` tại `C:\Program Files\MySQL\MySQL Server 8.0\bin\` |
| Máy | Laptop Windows 11 Home (10.0.26200); nhiều nhiễu, **không phải benchmark production** |
| Instance | Instance riêng, cổng **3318**, `--no-defaults`, `--bind-address=127.0.0.1`, datadir trong scratchpad; không đụng MySQL của người dùng ở cổng 3306 |
| Khác mặc định | `innodb_buffer_pool_size=512M` (để dữ liệu thử nằm gọn trong bộ nhớ); bật consumer `events_statements_history_long` của Performance Schema. Riêng TN2d khởi động thêm một lần với `--innodb-buffer-pool-load-at-startup=OFF` |
| Mặc định quan trọng (đọc được) | REPEATABLE READ; binary log bật, `binlog_format=ROW`; `sort_buffer_size` 256 KB; `join_buffer_size` 256 KB; `tmp_table_size` 16 MB; `internal_tmp_mem_storage_engine=TempTable`; `optimizer_switch` có `hash_join=on`, `block_nested_loop=on`, `batched_key_access=off`, `derived_condition_pushdown=on`, `hypergraph_optimizer=off` |
| Dữ liệu | Database `hpm8`, tổng hợp bằng recursive CTE: `actor` 200, `film` 1.000, `film_actor` 5.400, `customer` 599 hàng (mô phỏng **kích thước** Sakila, không phải dữ liệu Sakila), `rental` 1.000.000, `messages` 300.000 hàng |
| Cách đo | `EXPLAIN`, `EXPLAIN FORMAT=TREE`, `EXPLAIN ANALYZE`, `FLUSH STATUS` + `SHOW SESSION STATUS`, `performance_schema.events_statements_history(_long)`, `information_schema.OPTIMIZER_TRACE`, `information_schema.innodb_trx`, `sys.innodb_lock_waits`. Thời gian: lặp 3 lần, báo trung vị |

### Khởi tạo, chạy, tắt instance

```bash
# Bash (Git Bash). SP là thư mục scratchpad; BIN là thư mục bin của MySQL 8.0
BIN="/c/Program Files/MySQL/MySQL Server 8.0/bin"
DATADIR="$SP/hpm/mysql/ch08/main/data"
"$BIN/mysqld" --no-defaults --initialize-insecure --datadir="$DATADIR" --console

# chạy nền
"$BIN/mysqld" --no-defaults --datadir="$DATADIR" --port=3318 --mysqlx=OFF \
  --bind-address=127.0.0.1 --socket=ch08.sock --innodb-buffer-pool-size=512M \
  --performance-schema-consumer-events-statements-history-long=ON --console > mysqld.log 2>&1

# kiểm tra đúng instance trước khi làm gì
"$BIN/mysql" -h127.0.0.1 -P3318 -uroot -e "SELECT @@port, @@datadir, @@version"

# tắt khi xong (kể cả khi thí nghiệm lỗi)
"$BIN/mysqladmin" -h127.0.0.1 -P3318 -uroot shutdown
```

Kết quả kiểm tra: `@@port` = 3318, `@@datadir` nằm trong `...\scratchpad\hpm\mysql\ch08\main\data\`, `@@version` = 8.0.42.

### Dữ liệu dùng chung

```sql
-- Ví dụ minh họa do AI xây dựng: dữ liệu tổng hợp
DROP DATABASE IF EXISTS hpm8;
CREATE DATABASE hpm8;
USE hpm8;
SET SESSION cte_max_recursion_depth = 2000000;

CREATE TABLE actor (
  actor_id    SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  first_name  VARCHAR(45) NOT NULL,
  last_name   VARCHAR(45) NOT NULL,
  last_update TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (actor_id),
  KEY idx_actor_last_name (last_name)
) ENGINE=InnoDB;

CREATE TABLE film (
  film_id      SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  title        VARCHAR(128) NOT NULL,
  description  TEXT,
  release_year YEAR,
  length       SMALLINT UNSIGNED,
  rating       ENUM('G','PG','PG-13','R','NC-17') DEFAULT 'G',
  last_update  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (film_id),
  KEY idx_title (title)
) ENGINE=InnoDB;

CREATE TABLE film_actor (
  actor_id    SMALLINT UNSIGNED NOT NULL,
  film_id     SMALLINT UNSIGNED NOT NULL,
  last_update TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (actor_id, film_id),
  KEY idx_fk_film_id (film_id)
) ENGINE=InnoDB;

CREATE TABLE customer (
  customer_id SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  first_name  VARCHAR(45) NOT NULL,
  last_name   VARCHAR(45) NOT NULL,
  email       VARCHAR(50),
  PRIMARY KEY (customer_id),
  KEY idx_last_name (last_name)
) ENGINE=InnoDB;

INSERT INTO actor (actor_id, first_name, last_name)
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n < 200)
SELECT n, CONCAT('First', n), CONCAT('Last', LPAD((n*7919) % 1000, 4, '0')) FROM s;

INSERT INTO film (film_id, title, description, release_year, length, rating)
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n < 1000)
SELECT n, CONCAT('TITLE ', LPAD((n*104729) % 100000, 5, '0')),
       REPEAT(CONCAT('desc', n, ' '), 20), 2006, 46 + (n % 140),
       ELT(1 + (n % 5), 'G','PG','PG-13','R','NC-17')
FROM s;

-- mỗi actor 27 phim; công thức chỉ chạm 729 phim nên 271 phim không có actor
INSERT IGNORE INTO film_actor (actor_id, film_id)
WITH RECURSIVE a(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM a WHERE n < 200),
               k(m) AS (SELECT 0 UNION ALL SELECT m+1 FROM k WHERE m < 26)
SELECT a.n, ((a.n*37 + k.m*101) % 999) + 1 FROM a CROSS JOIN k;

INSERT INTO customer (customer_id, first_name, last_name, email)
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n < 599)
SELECT n, CONCAT('Cust', n), CONCAT('CLast', LPAD((n*6007) % 1000, 4, '0')),
       CONCAT('c', n, '@example.com') FROM s;

CREATE TABLE rental (
  rental_id    INT UNSIGNED NOT NULL AUTO_INCREMENT,
  rental_date  DATETIME NOT NULL,
  customer_id  SMALLINT UNSIGNED NOT NULL,
  inventory_id MEDIUMINT UNSIGNED NOT NULL,
  return_date  DATETIME NULL,
  note         VARCHAR(100) NOT NULL,
  PRIMARY KEY (rental_id),
  KEY idx_customer (customer_id)
) ENGINE=InnoDB;

INSERT INTO rental (rental_id, rental_date, customer_id, inventory_id, return_date, note)
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n < 1000000)
SELECT n, '2020-01-01' + INTERVAL n MINUTE, 1 + (n*7) % 599, 1 + (n*13) % 4581,
       IF(n % 10 = 0, NULL, '2020-01-03' + INTERVAL n MINUTE),
       RPAD(CONCAT('note-', n), 80, 'x')
FROM s;

CREATE TABLE messages (
  id      INT UNSIGNED NOT NULL AUTO_INCREMENT,
  created DATETIME NOT NULL,
  user_id INT UNSIGNED NOT NULL,
  body    VARCHAR(200) NOT NULL,
  PRIMARY KEY (id),
  KEY idx_created (created)
) ENGINE=InnoDB;

INSERT INTO messages (id, created, user_id, body)
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n < 300000)
SELECT n, '2024-01-01' + INTERVAL n MINUTE, 1 + n % 5000, RPAD(CONCAT('m', n), 150, 'y') FROM s;

ANALYZE TABLE actor, film, film_actor, customer, rental, messages;
```

Nạp xong trong khoảng 29 giây. Số hàng: actor 200, film 1.000, film_actor 5.400, customer 599, rental 1.000.000, messages 300.000.

---

<a id="tn1"></a>

## TN1 — Rows examined, access type và `SELECT *`

**Mục tiêu.** Kiểm chứng ý của tr. 194–198: rows examined so với rows returned, access type, và việc `SELECT *` làm mất covering index (tr. 193).

**Giả thuyết (Dự đoán, dựa trên sách).**

- Có index `idx_fk_film_id` thì `film_id = 1` dùng `ref`, đọc đúng số hàng trả về; bỏ index thì `ALL`, đọc cả bảng (tr. 196).
- GROUP BY không WHERE đọc cả bảng dù trả ít hàng (tr. 197–198).
- `SELECT *` qua index phụ tốn hơn nhiều so với truy vấn chỉ lấy cột của index.

**Các bước và lệnh đo.**

```sql
USE hpm8;
-- a) ref so với full scan (dùng IGNORE INDEX thay cho việc xóa index như sách)
EXPLAIN SELECT * FROM film_actor WHERE film_id = 1;
EXPLAIN SELECT * FROM film_actor IGNORE INDEX (idx_fk_film_id) WHERE film_id = 1;
TRUNCATE performance_schema.events_statements_history;
FLUSH STATUS;
SELECT * FROM film_actor WHERE film_id = 1;
SHOW SESSION STATUS LIKE 'Handler_read%';
FLUSH STATUS;
SELECT * FROM film_actor IGNORE INDEX (idx_fk_film_id) WHERE film_id = 1;
SHOW SESSION STATUS LIKE 'Handler_read%';
-- b) GROUP BY trả ít hàng
EXPLAIN SELECT actor_id, COUNT(*) FROM film_actor GROUP BY actor_id;
FLUSH STATUS;
SELECT actor_id, COUNT(*) FROM film_actor GROUP BY actor_id;
SHOW SESSION STATUS LIKE 'Handler_read%';
SELECT EVENT_ID, LEFT(SQL_TEXT, 70) AS sql_text, ROWS_EXAMINED, ROWS_SENT
FROM performance_schema.events_statements_history
WHERE THREAD_ID = PS_CURRENT_THREAD_ID() AND SQL_TEXT LIKE 'SELECT%film_actor%'
ORDER BY EVENT_ID;

-- c) SELECT * so với chỉ cột của index (chạy một lần trước cho nóng buffer pool)
EXPLAIN SELECT * FROM rental WHERE customer_id BETWEEN 1 AND 30;
EXPLAIN SELECT rental_id, customer_id FROM rental WHERE customer_id BETWEEN 1 AND 30;
SELECT VARIABLE_VALUE INTO @r0 FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests';
SELECT VARIABLE_VALUE INTO @b0 FROM performance_schema.session_status WHERE VARIABLE_NAME='Bytes_sent';
SELECT * FROM rental WHERE customer_id BETWEEN 1 AND 30;
SELECT VARIABLE_VALUE INTO @r1 FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests';
SELECT VARIABLE_VALUE INTO @b1 FROM performance_schema.session_status WHERE VARIABLE_NAME='Bytes_sent';
SELECT rental_id, customer_id FROM rental WHERE customer_id BETWEEN 1 AND 30;
SELECT VARIABLE_VALUE INTO @r2 FROM performance_schema.global_status WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests';
SELECT VARIABLE_VALUE INTO @b2 FROM performance_schema.session_status WHERE VARIABLE_NAME='Bytes_sent';
SELECT @r1-@r0 AS bp_req_star, @r2-@r1 AS bp_req_covering, @b1-@b0 AS bytes_star, @b2-@b1 AS bytes_covering;
```

**Kết quả** (Kết quả đã đo, MySQL 8.0.42, Windows 11, instance riêng, dữ liệu tổng hợp; output rút gọn cột).

a) `EXPLAIN`:

```text
type=ref  key=idx_fk_film_id key_len=2 ref=const rows=7    filtered=100.00 Extra=NULL
type=ALL  key=NULL                               rows=5400 filtered=0.14   Extra=Using where
```

`Handler_read_*`: có index `read_key=1, read_next=7`; `IGNORE INDEX` thì `read_first=1, read_key=1, read_rnd_next=5401`.

b) `EXPLAIN`: `type=index key=PRIMARY key_len=4 rows=5400 Extra=Using index`; `Handler_read_first=1, read_key=1, read_next=5400`.

Performance Schema:

```text
| sql_text                                                     | ROWS_EXAMINED | ROWS_SENT |
| SELECT * FROM film_actor WHERE film_id = 1                   |             7 |         7 |
| SELECT * FROM film_actor IGNORE INDEX (idx_fk_film_id) WH... |          5400 |         7 |
| SELECT actor_id, COUNT(*) FROM film_actor GROUP BY actor_id  |          5400 |       200 |
```

c) Cả hai truy vấn trả **50.086 hàng**.

```text
SELECT *                    : type=range key=idx_customer rows=109436 Extra=Using index condition
SELECT rental_id,customer_id: type=range key=idx_customer rows=109436 Extra=Using where; Using index

| bp_req_star | bp_req_covering | bytes_star | bytes_covering |
|      200447 |             106 |    6884597 |         680770 |
```

**Diễn giải và đối chiếu với sách.**

- Khớp tr. 196: có index thì đọc đúng 7 hàng cho 7 hàng trả về; không có thì 5.400. Sách in `rows: 5462, filtered: 10.00` vì Sakila có 5.462 hàng và index đã bị **xóa hẳn**; ở đây `filtered` = 0,14 vì index chỉ bị bỏ qua, optimizer có vẻ vẫn dùng statistics của nó để ước lượng độ chọn lọc (diễn giải, chưa kiểm chứng sâu).
- Khớp tr. 197–198: GROUP BY đọc 5.400 hàng để trả 200, kể cả khi dùng covering index (`Using index`).
- `SELECT *` tốn khoảng **1.900 lần** lượt đọc page so với covering (≈ 4 lượt mỗi hàng do tra clustered index), và gấp 10 lần byte gửi: minh họa tr. 193. `rows=109436` là ước lượng; thực tế 50.086.
- Ước lượng `rows` (109.436) lệch thực tế hơn 2 lần: một ví dụ về statistics không chính xác (tr. 206).

**Dọn dẹp.** Không thay đổi dữ liệu.

---

<a id="tn2"></a>

## TN2 — Optimizer: thứ tự join, hash join, cost model, các quy tắc tối ưu

**Mục tiêu.** Kiểm chứng tr. 205–218: join optimizer đổi thứ tự join; hash join (8.0.20+); `Last_query_cost` và statistics; các tối ưu như hằng số hóa, kết thúc sớm, equality propagation; filesort khi join; trạng thái thread.

**Giả thuyết (Dự đoán, dựa trên sách).**

- Optimizer bắt đầu từ `actor` thay vì `film`, chi phí thấp hơn plan `STRAIGHT_JOIN` (tr. 214–216).
- Join không có index dùng được sẽ thành hash join (tr. 212) và đọc ít hàng hơn nhiều so với nested-loop.
- Theo sách, chi phí không phụ thuộc dữ liệu có trong cache hay không (tr. 206).
- "Range checked for each record" làm tăng `Select_full_range_join` (chú thích 6, tr. 207).

### TN2a — Thứ tự join và STRAIGHT_JOIN

```sql
USE hpm8;
EXPLAIN SELECT film.film_id, film.title, film.release_year, actor.actor_id, actor.first_name, actor.last_name
FROM film INNER JOIN film_actor USING(film_id) INNER JOIN actor USING(actor_id);
SHOW WARNINGS;
FLUSH STATUS;
EXPLAIN ANALYZE SELECT film.film_id, film.title, film.release_year, actor.actor_id, actor.first_name, actor.last_name
FROM film INNER JOIN film_actor USING(film_id) INNER JOIN actor USING(actor_id);
SHOW SESSION STATUS LIKE 'Handler_read%';
-- lặp lại với SELECT STRAIGHT_JOIN ...
-- rồi chạy thật từng câu (không LIMIT 0) và đọc SHOW SESSION STATUS LIKE 'Last_query_cost'
```

**Kết quả** (rút gọn):

```text
-- optimizer tự chọn
actor       ALL     rows=200
film_actor  ref     key=PRIMARY        ref=hpm8.actor.actor_id      rows=27 Using index
film        eq_ref  key=PRIMARY        ref=hpm8.film_actor.film_id  rows=1
-> Nested loop inner join  (cost=2502 rows=5400) (actual time=0.0624..13.4 rows=5400 loops=1)
    -> Nested loop inner join  (cost=612 rows=5400) (actual time=0.0508..3.58 rows=5400 loops=1)
        -> Table scan on actor  (cost=20.2 rows=200) (actual time=0.0286..0.146 rows=200 loops=1)
        -> Covering index lookup on film_actor using PRIMARY (actor_id=actor.actor_id)  (cost=0.27 rows=27) (actual ... rows=27 loops=200)
    -> Single-row index lookup on film using PRIMARY (film_id=film_actor.film_id)  (cost=0.25 rows=1) (actual ... rows=1 loops=5400)
Handler_read_first=1 read_key=5601 read_next=5400 read_rnd_next=201
Last_query_cost = 2501.517293

-- STRAIGHT_JOIN
film        ALL     rows=1000
film_actor  ref     key=idx_fk_film_id ref=hpm8.film.film_id        rows=7 Using index
actor       eq_ref  key=PRIMARY        ref=hpm8.film_actor.actor_id rows=1
-> Nested loop inner join  (cost=3688 rows=7407) (actual time=0.112..17.2 rows=5400 loops=1)
    -> Nested loop inner join  (cost=1096 rows=7407) (actual time=0.102..7.25 rows=5400 loops=1)
        -> Table scan on film  (cost=104 rows=1000) (actual ... rows=1000 loops=1)
        -> Covering index lookup on film_actor using idx_fk_film_id (film_id=film.film_id)  (cost=0.252 rows=7.41) (actual ... rows=5.4 loops=1000)
    -> Single-row index lookup on actor using PRIMARY (actor_id=film_actor.actor_id)  (cost=0.25 rows=1) (actual ... rows=1 loops=5400)
Handler_read_first=1 read_key=6401 read_next=5400 read_rnd_next=1001
Last_query_cost = 3688.254936
```

`SHOW WARNINGS` sau `EXPLAIN` in Note 1003 với câu viết lại: các `USING` thành `where ((film.film_id = film_actor.film_id) and (film_actor.actor_id = actor.actor_id))`.

**Lưu ý phương pháp.** Lần thử đầu tiên bọc truy vấn trong `SELECT COUNT(*) FROM (...) t` để khỏi in kết quả: cả hai biến thể cho cùng bộ đếm, vì derived table bị gộp vào truy vấn ngoài và `STRAIGHT_JOIN` bên trong mất tác dụng. Lần thử với `LIMIT 0` cho `Last_query_cost = 0`. Số liệu ở trên lấy từ `EXPLAIN ANALYZE` và từ lần chạy thật không bọc.

### TN2b — Hash join và nested-loop không index

```sql
EXPLAIN SELECT a.actor_id, c.customer_id FROM actor a JOIN customer c ON c.first_name = CONCAT('Cust', a.actor_id);
FLUSH STATUS;
EXPLAIN ANALYZE SELECT a.actor_id, c.customer_id FROM actor a JOIN customer c ON c.first_name = CONCAT('Cust', a.actor_id);
SHOW SESSION STATUS LIKE 'Handler_read_rnd_next';
FLUSH STATUS;
EXPLAIN ANALYZE SELECT /*+ NO_BNL(a, c) */ a.actor_id, c.customer_id FROM actor a JOIN customer c ON c.first_name = CONCAT('Cust', a.actor_id);
SHOW SESSION STATUS LIKE 'Handler_read_rnd_next';
EXPLAIN FORMAT=TREE SELECT COUNT(*) FROM actor a JOIN customer c ON c.first_name < CONCAT('Cust', a.actor_id);
EXPLAIN FORMAT=TREE SELECT a.actor_id, c.customer_id FROM actor a LEFT JOIN customer c ON c.first_name = CONCAT('Cust', a.actor_id);
```

**Kết quả** (rút gọn):

```text
c  ALL rows=599 filtered=10.00 Extra=Using where; Using join buffer (hash join)
-> Inner hash join (c.first_name = concat('Cust',a.actor_id))  (cost=12001 rows=11980) (actual time=0.327..0.775 rows=200 loops=1)
    -> Table scan on c  (cost=0.035 rows=599) (actual ... rows=599 loops=1)
    -> Hash
        -> Covering index scan on a using idx_actor_last_name  (cost=20.2 rows=200) (actual ... rows=200 loops=1)
Handler_read_rnd_next = 600

-- NO_BNL
-> Nested loop inner join  (cost=12200 rows=11980) (actual time=0.0845..102 rows=200 loops=1)
    -> Covering index scan on a using idx_actor_last_name  (... rows=200 loops=1)
    -> Filter: (c.first_name = concat('Cust',a.actor_id))  (... rows=1 loops=200)
        -> Table scan on c  (... rows=599 loops=200)
Handler_read_rnd_next = 120000

-- join không bằng
-> Aggregate: count(0)
    -> Filter: (c.first_name < concat('Cust',a.actor_id))
        -> Inner hash join (no condition)
-- LEFT JOIN
-> Left hash join (c.first_name = concat('Cust',a.actor_id))
```

### TN2c — Cost model, biến của optimizer, các lệnh cũ

```sql
SELECT cost_name, cost_value, default_value FROM mysql.server_cost;
SELECT engine_name, cost_name, cost_value, default_value FROM mysql.engine_cost;
SELECT @@optimizer_search_depth, @@optimizer_prune_level, @@eq_range_index_dive_limit,
       @@sort_buffer_size, @@max_length_for_sort_data, @@join_buffer_size, @@tmp_table_size;
SELECT SQL_NO_CACHE COUNT(*) FROM film_actor;
SHOW WARNINGS;
SHOW SESSION STATUS LIKE 'Last_query_cost';
EXPLAIN EXTENDED SELECT * FROM actor WHERE actor_id = 1;
SET SESSION optimizer_switch = 'hypergraph_optimizer=on';
```

**Kết quả:**

```text
server_cost (cost_value NULL = dùng default_value):
disk_temptable_create_cost 20 | disk_temptable_row_cost 0.5 | key_compare_cost 0.05
memory_temptable_create_cost 1 | memory_temptable_row_cost 0.1 | row_evaluate_cost 0.1
engine_cost: io_block_read_cost 1 | memory_block_read_cost 0.25
optimizer_search_depth=62 optimizer_prune_level=1 eq_range_index_dive_limit=200
sort_buffer_size=262144 max_length_for_sort_data=4096 join_buffer_size=262144 tmp_table_size=16777216

Warning 1681: 'SQL_NO_CACHE' is deprecated and will be removed in a future release.
Last_query_cost = 543.749000
ERROR 1064 (42000): ... near 'SELECT * FROM actor WHERE actor_id = 1'      -- EXPLAIN EXTENDED
ERROR 3999 (42000): The hypergraph optimizer does not yet support 'use in non-debug builds'
```

### TN2d — Chi phí khi buffer pool lạnh và nóng

Khởi động lại instance với `--innodb-buffer-pool-load-at-startup=OFF` (để buffer pool rỗng), rồi:

```sql
USE hpm8;
SELECT table_name, n_rows, clustered_index_size, sum_of_other_index_sizes
FROM mysql.innodb_table_stats WHERE database_name='hpm8' AND table_name IN ('actor','film','rental');
SELECT COUNT(*) FROM information_schema.INNODB_BUFFER_PAGE
WHERE TABLE_NAME = '`hpm8`.`rental`' AND INDEX_NAME = 'PRIMARY';
EXPLAIN FORMAT=TREE SELECT * FROM rental;
EXPLAIN FORMAT=TREE SELECT * FROM actor;
SELECT SUM(LENGTH(note)) FROM rental IGNORE INDEX (idx_customer);   -- làm nóng
SELECT * FROM actor LIMIT 1;
-- lặp lại ba câu đếm page và EXPLAIN
```

**Kết quả:**

```text
actor  n_rows=200    clustered_index_size=1    sum_of_other_index_sizes=1
film   n_rows=1000   clustered_index_size=15   sum_of_other_index_sizes=3
rental n_rows=992250 clustered_index_size=7913 sum_of_other_index_sizes=1955

Lạnh: rental_pages_in_bp=0     | Table scan on rental (cost=107138 rows=992250) | Table scan on actor (cost=21 rows=200)
Nóng: rental_pages_in_bp=7877  | Table scan on rental (cost=101203 rows=992250) | Table scan on actor (cost=20.2 rows=200)
```

Phép tính kiểm tra (**Bổ sung ngoài sách**): 992.250 × 0,1 (`row_evaluate_cost`) = 99.225. Cộng 7.913 page × 1,0 (`io_block_read_cost`) = 107.138; cộng 7.913 × 0,25 (`memory_block_read_cost`) = 101.203,25. Với `actor`: 200 × 0,1 + 1 × 1,0 = 21 và 20 + 0,25 = 20,25.

### TN2e — Các quy tắc tối ưu, sort khi join, range check

```sql
EXPLAIN SELECT film.film_id, film_actor.actor_id FROM film INNER JOIN film_actor USING(film_id) WHERE film.film_id = 1;
EXPLAIN SELECT film.film_id FROM film WHERE film_id = -1;
EXPLAIN SELECT film.film_id FROM film WHERE film_id = 5000;
EXPLAIN SELECT MIN(actor_id), MAX(actor_id) FROM film_actor;
EXPLAIN SELECT film.film_id FROM film LEFT OUTER JOIN film_actor USING(film_id) WHERE film_actor.film_id IS NULL;
EXPLAIN FORMAT=TREE SELECT film.film_id FROM film LEFT OUTER JOIN film_actor USING(film_id) WHERE film_actor.film_id IS NULL;
SELECT COUNT(*) FROM film WHERE NOT EXISTS (SELECT 1 FROM film_actor fa WHERE fa.film_id = film.film_id);
EXPLAIN FORMAT=TREE SELECT STRAIGHT_JOIN film.film_id FROM film_actor INNER JOIN film USING(film_id) WHERE film.film_id > 990;
EXPLAIN SELECT a.actor_id, fa.film_id FROM actor a JOIN film_actor fa USING(actor_id) ORDER BY a.first_name;
EXPLAIN SELECT a.actor_id, fa.film_id, f.title FROM actor a JOIN film_actor fa USING(actor_id) JOIN film f USING(film_id) ORDER BY f.length, a.first_name;
FLUSH STATUS;
SELECT COUNT(*) FROM customer c JOIN rental r ON r.customer_id BETWEEN c.customer_id AND c.customer_id + 1 WHERE c.customer_id <= 3;
SHOW SESSION STATUS WHERE Variable_name IN ('Select_range_check','Select_full_range_join','Select_full_join','Select_range','Select_scan');
```

**Kết quả** (rút gọn):

```text
film       const  key=PRIMARY        ref=const rows=1  Extra=Using index
film_actor ref    key=idx_fk_film_id ref=const rows=7  Extra=Using index
film_id = -1    -> Extra: Impossible WHERE
film_id = 5000  -> Extra: no matching row in const table
MIN/MAX         -> Extra: Select tables optimized away
LEFT JOIN ... IS NULL:
  film       index idx_title rows=1000 Using index
  film_actor ref   idx_fk_film_id rows=7 Extra=Using where; Not exists; Using index
  -> Filter: (film_actor.film_id is null)
      -> Nested loop antijoin
          -> Index scan on film using idx_title
          -> Covering index lookup on film_actor using idx_fk_film_id (film_id=film.film_id)
films_without_actor = 271
STRAIGHT_JOIN, điều kiện viết trên film:
  -> Covering index range scan on film_actor using idx_fk_film_id over (990 < film_id)
ORDER BY a.first_name          -> actor: Using filesort
ORDER BY f.length, a.first_name -> actor: Using temporary; Using filesort
r ALL possible_keys=idx_customer Extra=Range checked for each record (index map: 0x2)
  -> Index range scan on r (re-planned for each iteration)
COUNT(*) = 10017
Select_full_join=0 Select_full_range_join=0 Select_range=1 Select_range_check=1 Select_scan=0
```

### TN2f — Trạng thái thread trong lúc chạy

Một procedure lấy mẫu `performance_schema.processlist` mỗi 50 ms (140 mẫu), chạy song song với một session thực hiện ba truy vấn dài (chạy `mysql --comments` để giữ chú thích đánh dấu):

```sql
CREATE TABLE state_log (ts DATETIME(6), cmd VARCHAR(32), state VARCHAR(64), info VARCHAR(60));
DELIMITER //
CREATE PROCEDURE sample_states(IN p_n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < p_n DO
    INSERT INTO state_log
      SELECT SYSDATE(6), COMMAND, STATE, LEFT(INFO, 60) FROM performance_schema.processlist
      WHERE INFO LIKE 'SELECT /* e6%';
    DO SLEEP(0.05);
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
-- session làm việc:
-- SELECT /* e6_sort */ rental_id, note FROM rental ORDER BY note DESC LIMIT 1;
-- SELECT /* e6_group */ note, COUNT(*) FROM rental GROUP BY note ORDER BY NULL LIMIT 1;
```

**Kết quả:**

```text
| q            | cmd   | state                     | samples |
| e6_sort      | Query | executing                 |       5 |
| e6_group     | Query | executing                 |       3 |
| e6_group     | Query | converting HEAP to ondisk |       3 |
| e6_group     | Query | (chuỗi rỗng)              |     123 |
```

Truy vấn thứ ba (UNION) chạy sau khi procedure lấy mẫu đã kết thúc nên không có mẫu. Lần thử đầu không có mẫu nào vì client `mysql` mặc định bỏ chú thích khỏi câu lệnh.

**Diễn giải và đối chiếu với sách (TN2).**

- **Khớp sách:** đảo thứ tự join (tr. 214–216); không index thì hash join (tr. 212); hằng số hóa, kết thúc sớm, not-exists, equality propagation (tr. 208–210); `Using filesort` so với `Using temporary; Using filesort` (tr. 217).
- **Khác sách:** cost model có tính page nóng hay lạnh (sách tr. 206 nói không); `EXPLAIN EXTENDED` không còn (tr. 212); `SQL_NO_CACHE` deprecated (tr. 205); biến tăng là `Select_range_check` (chú thích 6, tr. 207); không thấy trạng thái "Sending data" hay "Sorting result" (tr. 203–204).
- **Chi tiết đáng chú ý:** với `film_id = 1`, `film_actor` có `type: ref` và `ref: const`; sách viết "ref type is const" (tr. 209) trong khi output sách in là `type: index` vì index đã bị xóa ở ví dụ trước.
- `Impossible WHERE` với `-1` xuất hiện vì `film_id` là `SMALLINT UNSIGNED`, giá trị âm không thể tồn tại; với một id dương không có trong bảng, MySQL tra primary key lúc tối ưu và báo `no matching row in const table`.

**Dọn dẹp.** `DROP TABLE hpm8.state_log; DROP PROCEDURE hpm8.sample_states;` (hoặc xóa cả database ở cuối). Khởi động lại instance theo cấu hình thường sau TN2d nếu cần.

---

<a id="tn3"></a>

## TN3 — Phân trang: OFFSET, deferred join, keyset; `SQL_CALC_FOUND_ROWS`

**Mục tiêu.** Kiểm chứng tr. 223–225.

**Giả thuyết (Dự đoán, dựa trên sách).** OFFSET lớn đọc OFFSET + N hàng; deferred join rẻ hơn vì làm OFFSET trên covering index; keyset chỉ đọc N hàng bất kể độ sâu; `SQL_CALC_FOUND_ROWS` đọc hết kết quả.

**Các bước và lệnh đo.** Mỗi câu chạy 3 lần qua `mysql --comments` (kết quả bỏ vào `/dev/null`), rồi đọc số liệu từ `events_statements_history_long`:

```sql
USE hpm8;
SELECT /* e3_q1_offset_pk */ * FROM rental ORDER BY rental_id DESC LIMIT 500000, 20;
SELECT /* e3_q2_keyset_pk */ * FROM rental WHERE rental_id < 500001 ORDER BY rental_id DESC LIMIT 20;
SELECT /* e3_q3_offset_sec */ * FROM rental ORDER BY customer_id, rental_id LIMIT 300000, 20;
SELECT /* e3_q4_deferred */ r.* FROM rental r
  JOIN (SELECT rental_id FROM rental ORDER BY customer_id, rental_id LIMIT 300000, 20) AS lim USING(rental_id);
SELECT /* e3_q5_keyset_rowcons */ * FROM rental WHERE (customer_id, rental_id) > (180, 300000)
  ORDER BY customer_id, rental_id LIMIT 20;
SELECT /* e3_q6_keyset_or */ * FROM rental WHERE customer_id > 180 OR (customer_id = 180 AND rental_id > 300000)
  ORDER BY customer_id, rental_id LIMIT 20;
SELECT /* e3_q7_calc_found */ SQL_CALC_FOUND_ROWS * FROM rental WHERE customer_id BETWEEN 100 AND 200
  ORDER BY customer_id, rental_id LIMIT 20;
SELECT /* e3_q8_found_rows */ FOUND_ROWS();
SELECT /* e3_q9_limit21 */ * FROM rental WHERE customer_id BETWEEN 100 AND 200 ORDER BY customer_id, rental_id LIMIT 21;
SELECT /* e3_q10_count */ COUNT(*) FROM rental WHERE customer_id BETWEEN 100 AND 200;

-- đọc số liệu
SELECT SUBSTRING_INDEX(SUBSTRING_INDEX(SQL_TEXT,'/* ',-1),' */',1) AS q, COUNT(*) AS runs,
       GROUP_CONCAT(ROUND(TIMER_WAIT/1e9,1) ORDER BY EVENT_ID SEPARATOR ' / ') AS ms_each_run,
       MAX(ROWS_EXAMINED) AS rows_examined, MAX(ROWS_SENT) AS rows_sent,
       MAX(SORT_ROWS) AS sort_rows, MAX(CREATED_TMP_TABLES) AS tmp_tables
FROM performance_schema.events_statements_history_long
WHERE SQL_TEXT LIKE 'SELECT /* e3_%' GROUP BY q ORDER BY q;
```

Thêm: `EXPLAIN FORMAT=TREE` cho từng biến thể; bộ đếm `Handler_read_*` và `Sort_*` cho q3, q4 (bọc trong `SELECT COUNT(*) FROM (...) t`, vì có LIMIT nên derived table không bị gộp); optimizer trace cho filesort.

**Kết quả.**

Plan (`FORMAT=TREE`, rút gọn):

```text
q1: Limit/Offset: 20/500000 row(s) -> Index scan on rental using PRIMARY (reverse)
q2: Limit: 20 row(s) -> Index range scan on rental using PRIMARY over (rental_id < 500001) (reverse)
q3: Limit/Offset: 20/300000 row(s) -> Sort: rental.customer_id, rental.rental_id, limit input to 300020 row(s) per chunk -> Table scan on rental
q4: Nested loop inner join -> Table scan on lim -> Materialize -> Limit/Offset: 20/300000 row(s) -> Index scan on rental using idx_customer
                           -> Single-row index lookup on r using PRIMARY (rental_id=lim.rental_id)
q5: Limit: 20 row(s) -> Filter: ((rental.customer_id,rental.rental_id) > (180,300000)) -> Index scan on rental using idx_customer
q6: Limit: 20 row(s) -> Index range scan on rental using idx_customer over (customer_id = 180 AND 300000 < rental_id) OR (180 < customer_id)
```

Số liệu 3 lần chạy:

```text
| q                    | ms_each_run              | rows_examined | rows_sent | sort_rows | tmp_tables |
| e3_q1_offset_pk      | 311 / 323.8 / 332.5      |        500020 |        20 |         0 |          0 |
| e3_q2_keyset_pk      | 0.7 / 0.5 / 0.7          |            20 |        20 |         0 |          0 |
| e3_q3_offset_sec     | 1631.3 / 1638.5 / 1597.7 |       1300020 |        20 |    300020 |          0 |
| e3_q4_deferred       | 81.6 / 83.3 / 84.2       |            40 |        20 |         0 |          1 |
| e3_q5_keyset_rowcons | 1075.8 / 1069.6 / 1641   |        299360 |        20 |         0 |          0 |
| e3_q6_keyset_or      | 1.9 / 1.8 / 1.6          |            20 |        20 |         0 |          0 |
| e3_q7_calc_found     | 832.7 / 787 / 801.9      |       1168612 |        20 |    168612 |          0 |
| e3_q8_found_rows     | 0.2 / 0.2 / 0.2          |             1 |         1 |         0 |          0 |
| e3_q9_limit21        | 1.6 / 1.4 / 1.5          |            21 |        21 |         0 |          0 |
| e3_q10_count         | 67 / 57 / 57.5           |        168612 |         1 |         0 |          0 |
```

Bộ đếm handler:

```text
q3 (OFFSET, SELECT *): Handler_read_rnd_next=1000022 Sort_rows=300020 Sort_merge_passes=83 Sort_scan=1
q4 (deferred join):    Handler_read_first=1 read_key=21 read_next=300019 read_rnd_next=21 Sort_rows=0
```

`SQL_CALC_FOUND_ROWS`: `Warning 1287: SQL_CALC_FOUND_ROWS is deprecated and will be removed in a future release. Consider using two separate queries instead.`

Optimizer trace (rút gọn):

```text
ORDER BY customer_id, rental_id LIMIT 300000, 1:
  filesort_priority_queue_optimization: {"limit": 300001}           -- không chọn
  sort_mode: <fixed_sort_key, packed_additional_fields>, row_size: 428, max_rows_per_buffer: 612,
  memory_available: 262144, num_rows_found: 1000000, num_initial_chunks_spilled_to_disk: 445
ORDER BY inventory_id LIMIT 20:
  filesort_priority_queue_optimization: {"limit": 20, "chosen": true}
  sort_mode: <fixed_sort_key, additional_fields>, peak_memory_used: 8925,
  num_rows_found: 1000000, num_initial_chunks_spilled_to_disk: 0
```

**Diễn giải và đối chiếu với sách.**

- OFFSET tốn tuyến tính theo OFFSET (tr. 224): q1 đọc 500.020 hàng, trung vị 323,8 ms; keyset q2 đọc 20 hàng, 0,7 ms (tr. 224–225).
- Với thứ tự theo index phụ và `SELECT *`, optimizer chọn quét bảng + filesort (q3): đúng điều sách nói về việc cần index hỗ trợ thứ tự, nếu không phải filesort nhiều (tr. 223).
- Deferred join (q4) nhanh hơn q3 khoảng 20 lần (83,3 so với 1.631 ms) dù vẫn đi qua 300.019 mục index (tr. 224). `ROWS_EXAMINED` = 40 của Performance Schema **đếm thiếu** phần duyệt index trong subquery; bộ đếm handler mới cho thấy công việc thật.
- Keyset viết bằng row constructor (q5) không được dựng thành range trên 8.0.42 nên quét index từ đầu (299.360 hàng); dạng OR mở rộng (q6) chỉ đọc 20 hàng. Đây là phát hiện ngoài sách.
- `SQL_CALC_FOUND_ROWS` (q7) đọc 1.168.612 hàng (quét bảng + sort 168.612 hàng); `LIMIT 21` (q9) đọc 21 hàng; `COUNT(*)` riêng (q10) đọc 168.612 mục covering index trong 57,5 ms: đúng thứ tự ưu tiên của sách (tr. 225).
- Trace xác nhận ý "bản ghi kích thước cố định theo bộ lớn nhất" (tr. 217) ở chỗ `max_rows_per_buffer` = 262.144 / 428, và cho thấy priority queue khi LIMIT nhỏ (ngoài sách).

**Dọn dẹp.** Không thay đổi dữ liệu.

---

<a id="tn4"></a>

## TN4 — COUNT, UNION và UNION ALL, ROLLUP, lỗi 1093

**Mục tiêu.** Kiểm chứng tr. 219–226 (trừ phần LIMIT đã làm ở TN3).

**Giả thuyết (Dự đoán, dựa trên sách).**

- `COUNT(cột nullable)` khác `COUNT(*)` về kết quả và có thể chậm hơn (tr. 221).
- `COUNT(a OR b)` không tách được số đếm (tr. 222).
- UNION luôn tạo bảng tạm, kể cả UNION ALL (tr. 225–226); ví dụ UNION ALL + LIMIT 20 của sách ghi 799 hàng vào bảng tạm (tr. 219).
- UPDATE với subquery trên cùng bảng báo lỗi 1093; derived table lách được (tr. 220–221).
- `WITH ROLLUP` có thể đổi cách gộp nhóm (tr. 223).

### TN4a — COUNT

```sql
USE hpm8;
SELECT @@innodb_parallel_read_threads;
EXPLAIN SELECT COUNT(*) FROM rental;
EXPLAIN FORMAT=TREE SELECT COUNT(*) FROM rental;
EXPLAIN FORMAT=TREE SELECT COUNT(customer_id) FROM rental;
EXPLAIN FORMAT=TREE SELECT COUNT(return_date) FROM rental;
SELECT COUNT(*), COUNT(rental_id), COUNT(customer_id), COUNT(return_date) FROM rental;
SELECT SUM(IF(customer_id = 1, 1, 0)) AS c1, SUM(IF(customer_id = 2, 1, 0)) AS c2,
       COUNT(customer_id = 1 OR NULL) AS c1b, COUNT(customer_id = 2 OR NULL) AS c2b,
       COUNT(customer_id = 1 OR customer_id = 2) AS wrong_or FROM rental;
-- đo thời gian: 3 lần mỗi câu, đánh dấu /* e4_c1_star */ ... /* e4_c5_star_where */
SELECT /* e4_c1_star */ COUNT(*) FROM rental;
SELECT /* e4_c2_pk */ COUNT(rental_id) FROM rental;
SELECT /* e4_c3_notnull_idx */ COUNT(customer_id) FROM rental;
SELECT /* e4_c4_nullable_noidx */ COUNT(return_date) FROM rental;
SELECT /* e4_c5_star_where */ COUNT(*) FROM rental WHERE customer_id BETWEEN 100 AND 200;
```

**Kết quả:**

```text
innodb_parallel_read_threads = 4
EXPLAIN COUNT(*): type=index key=idx_customer rows=992250 Extra=Using index
FORMAT=TREE COUNT(*), COUNT(rental_id), COUNT(customer_id): -> Count rows in rental
FORMAT=TREE COUNT(return_date): -> Aggregate: count(rental.return_date) -> Table scan on rental
COUNT(*)=1000000 COUNT(rental_id)=1000000 COUNT(customer_id)=1000000 COUNT(return_date)=900000
c1=1669 c2=1669 c1b=1669 c2b=1669 wrong_or=1000000

| q                    | ms_each_run           | rows_examined |
| e4_c1_star           | 264.1 / 259.8 / 242.8 |             0 |
| e4_c2_pk             | 292.2 / 270.1 / 303.3 |             0 |
| e4_c3_notnull_idx    | 276.8 / 271 / 295.9   |             0 |
| e4_c4_nullable_noidx | 286.8 / 282.9 / 292   |       1000000 |
| e4_c5_star_where     | 56.5 / 50.6 / 54.4    |        168612 |
```

### TN4b — UNION và UNION ALL

```sql
EXPLAIN SELECT first_name, last_name FROM actor UNION SELECT first_name, last_name FROM customer;
EXPLAIN SELECT first_name, last_name FROM actor UNION ALL SELECT first_name, last_name FROM customer;
EXPLAIN FORMAT=TREE SELECT first_name, last_name FROM actor UNION SELECT first_name, last_name FROM customer;
EXPLAIN FORMAT=TREE SELECT first_name, last_name FROM actor UNION ALL SELECT first_name, last_name FROM customer;
-- bốn biến thể, mỗi biến thể: FLUSH STATUS; <câu>; SHOW SESSION STATUS WHERE Variable_name IN
--   ('Created_tmp_tables','Created_tmp_disk_tables','Handler_read_rnd_next','Handler_write');
(SELECT first_name, last_name FROM actor ORDER BY last_name)
UNION ALL (SELECT first_name, last_name FROM customer ORDER BY last_name) LIMIT 20;
(SELECT first_name, last_name FROM actor ORDER BY last_name LIMIT 20)
UNION ALL (SELECT first_name, last_name FROM customer ORDER BY last_name LIMIT 20) ORDER BY last_name LIMIT 20;
(SELECT first_name, last_name FROM actor)
UNION ALL (SELECT first_name, last_name FROM customer) ORDER BY last_name LIMIT 20;
-- đẩy điều kiện vào derived table chứa UNION
EXPLAIN FORMAT=TREE SELECT * FROM (SELECT first_name, last_name FROM actor
  UNION ALL SELECT first_name, last_name FROM customer) AS t WHERE t.last_name = 'Last0500';
-- thời gian với 200.000 hàng, 3 lần mỗi câu
SELECT /* e4_u1_union */ rental_id, note FROM rental WHERE rental_id <= 100000
UNION SELECT rental_id, note FROM rental WHERE rental_id > 900000;
SELECT /* e4_u2_unionall */ rental_id, note FROM rental WHERE rental_id <= 100000
UNION ALL SELECT rental_id, note FROM rental WHERE rental_id > 900000;
```

**Kết quả** (rút gọn):

```text
UNION:     PRIMARY actor ALL 200 | UNION customer ALL 599 | UNION RESULT <union1,2> Extra=Using temporary
UNION ALL: PRIMARY actor ALL 200 | UNION customer ALL 599        (không có dòng UNION RESULT)
TREE UNION:     -> Table scan on <union temporary> -> Union materialize with deduplication -> ...
TREE UNION ALL: -> Append -> Stream results -> Table scan on actor ; -> Stream results -> Table scan on customer

| Biến thể                                   | Created_tmp_tables | Handler_write | Handler_read_rnd_next |
| actor UNION customer                       |                  1 |           799 |                  1601 |
| actor UNION ALL customer                   |                  0 |             0 |                   801 |
| câu đầu của sách (LIMIT ngoài, ORDER trong) |                  0 |             0 |                    20 |
| LIMIT 20 trong nhánh + ORDER BY ngoài      |                  1 |            40 |                   842 |
| ORDER BY ngoài, nhánh không LIMIT          |                  1 |           799 |                  1601 |

derived + UNION ALL + WHERE:
-> Table scan on t -> Union all materialize
    -> Index lookup on actor using idx_actor_last_name (last_name='Last0500')
    -> Index lookup on customer using idx_last_name (last_name='Last0500')

| q              | ms_each_run           | rows_examined | rows_sent | tmp_tables | tmp_disk |
| e4_u1_union    | 1030 / 1059 / 1079.3  |        400000 |    200000 |          1 |        1 |
| e4_u2_unionall | 293.9 / 298.4 / 304.8 |        200000 |    200000 |          0 |        0 |
```

### TN4c — SELECT và UPDATE cùng bảng

```sql
DROP TABLE IF EXISTS tbl;
CREATE TABLE tbl (id INT PRIMARY KEY AUTO_INCREMENT, type CHAR(1) NOT NULL, c INT NOT NULL DEFAULT 0);
INSERT INTO tbl (type) VALUES ('a'),('a'),('b'),('c'),('c'),('c');
UPDATE tbl AS outer_tbl SET c = (SELECT count(*) FROM tbl AS inner_tbl WHERE inner_tbl.type = outer_tbl.type);
UPDATE tbl INNER JOIN (SELECT type, count(*) AS c FROM tbl GROUP BY type) AS der USING(type) SET tbl.c = der.c;
SELECT * FROM tbl;
UPDATE tbl SET c = 0 WHERE id IN (SELECT id FROM (SELECT id FROM tbl WHERE type = 'c') AS d);
SELECT ROW_COUNT();
EXPLAIN UPDATE tbl SET c = 0 WHERE id IN (SELECT id FROM (SELECT id FROM tbl WHERE type = 'c') AS d);
EXPLAIN UPDATE tbl SET c = 0 WHERE id IN (SELECT /*+ MERGE(d) */ id FROM (SELECT id FROM tbl WHERE type = 'c') AS d);
```

**Kết quả:**

```text
ERROR 1093 (HY000): You can't specify target table 'outer_tbl' for update in FROM clause
SELECT * FROM tbl (sau bản derived table có GROUP BY): (1,a,2) (2,a,2) (3,b,1) (4,c,3) (5,c,3) (6,c,3)
Derived table không GROUP BY: ROW_COUNT() = 3
EXPLAIN: UPDATE tbl ALL | PRIMARY <derived3> ALL Extra=Using where; FirstMatch(tbl) | DERIVED tbl ALL
Với hint MERGE(d): ERROR 1093 (HY000): You can't specify target table 'tbl' for update in FROM clause
```

### TN4d — GROUP BY và WITH ROLLUP

```sql
EXPLAIN SELECT rating, COUNT(*) FROM film GROUP BY rating;
EXPLAIN SELECT rating, COUNT(*) FROM film GROUP BY rating WITH ROLLUP;
EXPLAIN FORMAT=TREE SELECT rating, COUNT(*) FROM film GROUP BY rating WITH ROLLUP;
SELECT rating, GROUPING(rating) AS g, COUNT(*) FROM film GROUP BY rating WITH ROLLUP;
```

**Kết quả:**

```text
GROUP BY rating              : film ALL rows=1000 Extra=Using temporary
GROUP BY rating WITH ROLLUP  : film ALL rows=1000 Extra=Using filesort
-> Group aggregate with rollup: count(0) -> Sort: rating -> Table scan on film
G 0 200 | PG 0 200 | PG-13 0 200 | R 0 200 | NC-17 0 200 | NULL 1 1000
```

**Diễn giải và đối chiếu với sách (TN4).**

- `COUNT(*)` và `COUNT(cột NOT NULL)` cùng plan `Count rows` (InnoDB tự đếm, có đọc song song từ 8.0.14); `COUNT(cột nullable)` cho 900.000 thay vì 1.000.000 và phải quét bảng (tr. 221). Thời gian gần nhau vì dữ liệu nằm trong bộ nhớ; `ROWS_EXAMINED = 0` của đường `Count rows` là bẫy đo lường.
- `COUNT(... OR ...)` đếm cả 1.000.000 hàng; `SUM(IF())` và `COUNT(... OR NULL)` cho đúng 1.669 (tr. 222).
- **UNION ALL không tạo bảng tạm** trên 8.0.42 khi không có ORDER BY chung, trái với tr. 226; câu đầu của sách ở tr. 219 chỉ đọc 20 hàng. Khi viết đúng ngữ nghĩa (ORDER BY chung), bảng tạm 799 hàng; đẩy LIMIT vào nhánh còn 40 hàng, khớp sách.
- UNION có loại trùng chậm hơn UNION ALL khoảng 3,5 lần ở 200.000 hàng, bảng tạm tràn ra đĩa (tr. 226).
- Lỗi 1093 và cách lách khớp tr. 220–221; điều kiện thật là materialization (hint `MERGE` làm lỗi quay lại).
- ROLLUP đổi cách gộp từ bảng tạm sang filesort: đúng loại khác biệt sách bảo kiểm tra (tr. 223).

**Dọn dẹp.** `DROP TABLE hpm8.tbl;`

---

<a id="tn5"></a>

## TN5 — Chia nhỏ một DELETE lớn

**Mục tiêu.** Kiểm chứng tr. 199–200: một DELETE lớn khóa nhiều hàng lâu và chặn truy vấn nhỏ; chia lô giảm tác động.

**Giả thuyết (Dự đoán, dựa trên sách).** Một câu DELETE xóa 131.039 hàng khóa ít nhất 131.039 hàng tới khi commit, chặn truy vấn khác chạm vào các hàng đó; chia lô 10.000 hàng giảm số hàng khóa mỗi transaction và cho truy vấn khác xen vào; tổng thời gian có thể tương đương.

**Chuẩn bị.**

```sql
USE hpm8;
DROP TABLE IF EXISTS messages_a, messages_b, purge_log;
CREATE TABLE messages_a LIKE messages;  INSERT INTO messages_a SELECT * FROM messages;
CREATE TABLE messages_b LIKE messages;  INSERT INTO messages_b SELECT * FROM messages;
ANALYZE TABLE messages_a, messages_b;
CREATE TABLE purge_log (chunk_no INT PRIMARY KEY, rows_deleted INT, rows_locked BIGINT,
                        lock_structs BIGINT, elapsed_ms DECIMAL(10,1));
DELIMITER //
CREATE PROCEDURE purge_chunked(IN p_limit INT, IN p_sleep DECIMAL(5,2))
BEGIN
  DECLARE v_n INT DEFAULT 1;
  DECLARE v_chunk INT DEFAULT 0;
  DECLARE v_t0 DATETIME(6);
  DECLARE v_locked BIGINT;
  DECLARE v_structs BIGINT;
  WHILE v_n > 0 DO
    SET v_chunk = v_chunk + 1;
    SET v_t0 = SYSDATE(6);
    START TRANSACTION;
    DELETE FROM messages_b WHERE created < '2024-04-01' LIMIT p_limit;
    SET v_n = ROW_COUNT();
    SELECT trx_rows_locked, trx_lock_structs INTO v_locked, v_structs
      FROM information_schema.innodb_trx WHERE trx_mysql_thread_id = CONNECTION_ID();
    COMMIT;
    INSERT INTO purge_log VALUES (v_chunk, v_n, v_locked, v_structs,
      TIMESTAMPDIFF(MICROSECOND, v_t0, SYSDATE(6)) / 1000);
    IF v_n > 0 AND p_sleep > 0 THEN DO SLEEP(p_sleep); END IF;
  END WHILE;
END//
DELIMITER ;
EXPLAIN DELETE FROM messages WHERE created < '2024-04-01';
EXPLAIN DELETE FROM messages WHERE created < '2024-04-01' LIMIT 10000;
```

**Các bước.** Mỗi biến thể chạy **ba session song song** (ba tiến trình `mysql` chạy nền, `wait` chờ cả ba):

```sql
-- Session A (một câu DELETE, giữ transaction mở 8 giây)
START TRANSACTION;
SET @t0 = SYSDATE(6);
DELETE FROM messages_a WHERE created < '2024-04-01';
SELECT ROW_COUNT() AS rows_deleted, TIMESTAMPDIFF(MICROSECOND, @t0, SYSDATE(6))/1000 AS delete_ms;
SELECT trx_rows_locked, trx_rows_modified, trx_lock_structs, trx_lock_memory_bytes
FROM information_schema.innodb_trx WHERE trx_mysql_thread_id = CONNECTION_ID();
DO SLEEP(8);
COMMIT;

-- Probe (sau 2 giây): cập nhật một tin nhắn MỚI, không thuộc phạm vi xóa
DO SLEEP(2);
SET SESSION innodb_lock_wait_timeout = 3;
SET @t0 = SYSDATE(6);
UPDATE messages_a SET body = 'probe' WHERE id = 250000;
SELECT TIMESTAMPDIFF(MICROSECOND, @t0, SYSDATE(6))/1000 AS waited_ms;

-- Watch (sau 3 giây)
SELECT waiting_query, blocking_trx_rows_locked, blocking_trx_rows_modified, wait_age FROM sys.innodb_lock_waits;

-- Biến thể chia lô: thay session A bằng CALL purge_chunked(10000, 0.3) trên messages_b,
-- probe chạy cùng kịch bản trên messages_b.
```

Sau đó lặp 3 lần một kịch bản không có probe và không nghỉ giữa lô (tạo lại `messages_a`, `messages_b`, chạy DELETE một lần với autocommit rồi `CALL purge_chunked(10000, 0)`) để lấy trung vị thời gian.

**Kết quả.**

```text
EXPLAIN DELETE (không LIMIT):  type=ALL   key=NULL        rows=296460 Extra=Using where
EXPLAIN DELETE ... LIMIT 10000: type=range key=idx_created rows=148230 Extra=Using where

-- Một câu DELETE
rows_deleted=131039 delete_ms=1350.7
trx_rows_locked=303660 trx_rows_modified=131039 trx_lock_structs=3661 trx_lock_memory_bytes=417912
probe: ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction  (waited_ms=3021.2)
sys.innodb_lock_waits: UPDATE messages_a SET body = 'probe' WHERE id = 250000 | 303660 | 131039 | 00:00:02

-- Chia lô 10000, nghỉ 0,3 giây
| chunk_no | rows_deleted | rows_locked | lock_structs | elapsed_ms |
|        1 |        10000 |       20009 |          134 |       82.6 |
|      ... |          ... |         ... |          ... |        ... |    (lô 2–13: 20008–20009 hàng khóa, 89–136 ms)
|       14 |         1039 |        2081 |           17 |       11.1 |
|       15 |            0 |           2 |            3 |        0.5 |
chunks=14 deleted=131039 max_rows_locked_per_trx=20009 max_chunk_ms=136.1 total_ms=5626.6 (gồm nghỉ)
probe: waited_ms=2.9 (thành công)

-- Lặp 3 lần, không nghỉ
| mono_rows | mono_ms | chunk_rows | chunk_sum_ms | chunk_wall_ms | max_chunk_ms |
|    131039 |  1154.6 |     131039 |       1652.6 |        1689.6 |        151.9 |
|    131039 |   998.2 |     131039 |       2226.5 |        2268.5 |        262.1 |
|    131039 |   933.1 |     131039 |       1603.8 |        1663.5 |        151.5 |
Trung vị: một câu 998,2 ms; chia lô 1.652,6 ms (tổng các lô); lô dài nhất 151,9 ms
```

**Diễn giải và đối chiếu với sách.**

- **Khóa nhiều hơn số hàng xóa.** Optimizer chọn quét toàn bảng vì 44% hàng khớp; DELETE ở REPEATABLE READ khóa mọi hàng quét qua, nên 303.660 hàng bị khóa và một UPDATE vào hàng **không bị xóa** vẫn bị chặn tới timeout. Đây là minh họa trực tiếp cho câu "khóa nhiều hàng trong thời gian dài... chặn truy vấn nhỏ" (tr. 199). Cơ chế khóa hàng quét qua là **Bổ sung ngoài sách** (tài liệu InnoDB).
- **Chia lô** đổi plan sang `range` trên `idx_created`, mỗi transaction khóa tối đa 20.009 hàng (mục index phụ + hàng clustered) trong khoảng 0,1–0,15 giây; probe chen vào giữa các lô và xong sau 2,9 ms. Với lần đo có probe, probe rơi đúng vào lúc nào (trong một lô hay trong lúc nghỉ) là ngẫu nhiên; con số 2,9 ms chỉ cho thấy nó không bị chặn lâu.
- **Tổng thời gian không giảm.** Trung vị 1,65 giây cho chia lô so với 1,0 giây cho một câu. Sách nói chia lô "cải thiện hiệu năng" theo nghĩa giảm tác động lên server và replication lag (tr. 199–200), không phải job nhanh hơn. Giả thuyết cho phần tăng thêm (mỗi lô quét lại các mục đã bị đánh dấu xóa ở đầu khoảng index) là **Dự đoán**, chưa kiểm chứng.
- Replication lag không đo được vì không có replica trong thí nghiệm này (**Chưa thực thi** phần đó).

**Dọn dẹp và khôi phục.**

```sql
DROP TABLE IF EXISTS hpm8.messages_a, hpm8.messages_b, hpm8.purge_log;
DROP PROCEDURE IF EXISTS hpm8.purge_chunked;
-- Xóa toàn bộ dữ liệu thí nghiệm của chương:
DROP DATABASE IF EXISTS hpm8;
```

Sau cùng: `mysqladmin -h127.0.0.1 -P3318 -uroot shutdown`. Instance đã được tắt sau khi đo; datadir nằm trong scratchpad, không ảnh hưởng MySQL của người dùng.

---

## Những gì chưa thực thi

| Nội dung | Lý do |
| --- | --- |
| So sánh `IN()` với chuỗi OR (tr. 211) | Không đủ thời gian trong phạm vi chương; khó đo khác biệt có ý nghĩa ở danh sách nhỏ |
| Chuyển OUTER JOIN thành INNER JOIN (tr. 207) | Chưa dựng ví dụ; có thể kiểm bằng `EXPLAIN` + `SHOW WARNINGS` |
| `COUNT(*)` so với `COUNT(cột nullable)` khi buffer pool lạnh | Chưa đo |
| Hash join tràn đĩa khi `join_buffer_size` nhỏ | Chưa đo |
| Replication lag khi DELETE lớn so với chia lô (tr. 199) | Cần replica; thuộc phạm vi chương 9 |
| Đường thực thi của prepared statement (tr. 202, 207) | Sách không bàn chi tiết; ngoài phạm vi |
