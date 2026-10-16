# Thực hành — Chương 5: Optimizing Server Settings

[← Về ghi chú chương 5](../chapter-05.md) · [Đáp án tự kiểm tra](../answers/chapter-05.md)

> **Nhãn.** Mọi số liệu trong tệp này là **Kết quả đã đo** trên một instance thử nghiệm, trừ chỗ ghi rõ **Dự đoán** hay **Chưa thực thi**. Lệnh SQL, thủ tục và dữ liệu tổng hợp là **Ví dụ minh họa do AI xây dựng**. Output được chép thật, có chỗ rút gọn (ghi "đã rút gọn").
>
> **Không phải benchmark production.** Đây là laptop Windows, có nhiều tiến trình khác chạy song song (kể cả các instance MySQL thử nghiệm khác). Thời gian và thông lượng chỉ dùng để so sánh **tương đối** giữa các cấu hình chạy xen kẽ nhau trên cùng máy, trong cùng vài phút. Các bộ đếm như số fsync, số trang đọc từ đĩa thì ổn định hơn nhiều so với thời gian.

## Mục lục

- [Môi trường](#moi-truong)
- [TN1. Scope, dynamic, `DEFAULT` và `SET PERSIST`](#tn1)
- [TN2. File cấu hình mẫu của sách so với MySQL 8.0.42](#tn2)
- [TN3. Buffer pool: hit ratio, resize online, `innodb_old_blocks_time`, thread cache](#tn3)
- [TN4. Durability và thông lượng commit](#tn4)
- [TN5. Transaction dài, history list và purge](#tn5)
- [Dọn dẹp chung](#don-dep)

---

<a id="moi-truong"></a>

## Môi trường

| Mục | Giá trị |
| --- | --- |
| Phần mềm | MySQL Community Server **8.0.42** (`@@version_compile_os` = Win64), cài sẵn tại `C:\Program Files\MySQL\MySQL Server 8.0\bin\` |
| Máy | Windows 11 Home, 16 CPU logic, 32 GB RAM. Loại ổ đĩa không được xác minh. |
| Instance | Instance riêng, cổng **3315**, datadir `<SP>\hpm\mysql\ch05\main\data` (SP là thư mục scratchpad của phiên làm việc). Không đụng tới MySQL của người dùng ở cổng 3306. |
| Khởi tạo | `mysqld --no-defaults --initialize-insecure --datadir=<datadir> --console` |
| Lần chạy 1 và 2 | `mysqld --no-defaults --datadir=<datadir> --port=3315 --mysqlx=OFF --bind-address=127.0.0.1 --socket=ch05.sock --sort_buffer_size=1048576 --console`. Ngoài `sort_buffer_size` (đặt để TN1 có giá trị "từ dòng lệnh"), mọi biến để **mặc định**. |
| Lần chạy 3 | Như trên nhưng thay `--sort_buffer_size` bằng `--innodb_log_file_size=256M` (chỉ dùng trong TN2). |
| Kết nối | `mysql -h127.0.0.1 -P3315 -uroot`; trước mỗi thí nghiệm kiểm `SELECT @@port, @@datadir` (đã kiểm: 3315 và datadir ở trên). |
| Dữ liệu | Schema `lab05`: bảng `big` 1.000.000 hàng (DATA_LENGTH 301,8 MB), bảng `hot` 80.000 hàng (19,5 MB), bảng `c` cho thí nghiệm commit. Sinh bằng recursive CTE, không tải gì từ internet. |
| Mặc định quan trọng đo được | `innodb_buffer_pool_size` = 128 MiB (8192 trang 16 KiB), `innodb_buffer_pool_instances` = 1, `innodb_redo_log_capacity` = 100 MiB, `innodb_flush_method` = `unbuffered`, `log_bin` = ON, `sync_binlog` = 1, `innodb_flush_log_at_trx_commit` = 1 |

**Thứ tự thực tế đã chạy** (khác thứ tự trình bày): phần scope của TN1 → TN3 → phần `SET PERSIST` và khởi động lại của TN1 (quan sát luôn buffer pool dump/load cho TN3) → TN4 → TN5 → tắt server → TN2 (`--validate-config`, rồi lần chạy 3). Mọi lần chạy kết thúc bằng `mysqladmin -h127.0.0.1 -P3315 -uroot shutdown`; đã kiểm tra cổng 3315 không còn nhận kết nối (ERROR 2003).

**Tạo dữ liệu** (dùng chung cho TN3, TN5):

```sql
CREATE DATABASE lab05;
USE lab05;
CREATE TABLE big (id INT PRIMARY KEY, grp INT NOT NULL, pad CHAR(200) NOT NULL) ENGINE=InnoDB;
CREATE TABLE hot (id INT PRIMARY KEY, grp INT NOT NULL, pad CHAR(200) NOT NULL) ENGINE=InnoDB;
-- 1000 x 1000 = 1.000.000 hàng, không cần tăng cte_max_recursion_depth
INSERT INTO big (id, grp, pad)
WITH RECURSIVE k(n) AS (SELECT 0 UNION ALL SELECT n + 1 FROM k WHERE n < 999)
SELECT a.n * 1000 + b.n + 1, (a.n * 1000 + b.n) % 100, RPAD(CONCAT('row-', a.n * 1000 + b.n), 200, 'x')
FROM k AS a CROSS JOIN k AS b;
INSERT INTO hot (id, grp, pad) SELECT id, grp, pad FROM big WHERE id <= 80000;
ANALYZE TABLE big, hot;
```

Kết quả đã đo: hai lệnh INSERT xong trong khoảng 20 giây. Ngay sau đó buffer pool đầy (`Innodb_buffer_pool_pages_free` = 0) với 3815 trang bẩn.

---

<a id="tn1"></a>

## TN1. Scope, dynamic, `DEFAULT` và `SET PERSIST`

**Mục tiêu.** Kiểm chứng các khẳng định ở tr. 101–103: biến session và global; `SET GLOBAL` không ảnh hưởng session đang mở; `SET ... = DEFAULT` cho session và cho global khác nhau; `SET PERSIST` của 8.0 ghi xuống đĩa. Xem [mục 2.1 của ghi chú](../chapter-05.md#s2-1).

**Giả thuyết (Dự đoán).**

1. Đổi `sort_buffer_size` ở session chỉ đổi session đó.
2. `SET GLOBAL` không đổi giá trị session của kết nối đang mở, kể cả kết nối vừa gõ lệnh; kết nối mới nhận giá trị global mới.
3. `SET SESSION x = DEFAULT` lấy giá trị global hiện tại; `SET GLOBAL x = DEFAULT` lấy **giá trị biên dịch sẵn** (262144 cho `sort_buffer_size`), không phải 1 MiB đặt ở dòng lệnh.
4. `SET PERSIST` ghi tệp `mysqld-auto.cnf` trong datadir; biến read-only chỉ ghi được bằng `SET PERSIST_ONLY`.

**Chuẩn bị.** Server chạy với `--sort_buffer_size=1048576`.

**Các bước và lệnh đo.**

```sql
-- nguồn gốc giá trị của biến (Performance Schema, xem chương 3 tr. 69)
SELECT VARIABLE_NAME, VARIABLE_SOURCE, MIN_VALUE, MAX_VALUE
FROM performance_schema.variables_info
WHERE VARIABLE_NAME IN ('sort_buffer_size','max_connections','innodb_buffer_pool_size',
                        'innodb_buffer_pool_instances','innodb_flush_log_at_trx_commit','port');
SET sort_buffer_size = 2097152;                  -- session
SELECT @@session.sort_buffer_size, @@global.sort_buffer_size;
SET SESSION max_connections = 500;               -- biến chỉ có global
SET GLOBAL innodb_buffer_pool_instances = 2;     -- biến read-only
SET GLOBAL sysdate_is_now = 1;                   -- tùy chọn dòng lệnh, không phải biến
```

Phần "session đang mở": kết nối B mở trước và `DO SLEEP(3)`; trong lúc đó kết nối A chạy `SET GLOBAL sort_buffer_size = 4194304`; B đọc lại giá trị của nó; rồi mở kết nối C mới. Cuối cùng thử `DEFAULT`:

```sql
-- kết nối C
SET sort_buffer_size = 65536;
SET SESSION sort_buffer_size = DEFAULT;   -- về giá trị global
-- kết nối D
SET GLOBAL sort_buffer_size = DEFAULT;    -- về giá trị biên dịch sẵn?
SELECT VARIABLE_NAME, VARIABLE_SOURCE, SET_TIME, SET_USER
FROM performance_schema.variables_info WHERE VARIABLE_NAME = 'sort_buffer_size';
SET GLOBAL sort_buffer_size = 1048576;    -- khôi phục
```

Phần persist:

```sql
SET PERSIST max_connections = 300;
SET PERSIST innodb_read_io_threads = 8;        -- read-only: dự đoán lỗi
SET PERSIST_ONLY innodb_read_io_threads = 8;   -- chỉ ghi tệp, không đổi runtime
SELECT @@global.max_connections, @@global.innodb_read_io_threads;
SELECT * FROM performance_schema.persisted_variables;
```

Sau đó đọc `mysqld-auto.cnf`, tắt server, khởi động lại **với cùng lệnh `--no-defaults`**, đọc lại các biến; cuối cùng `RESET PERSIST`.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng, cấu hình mặc định trừ `sort_buffer_size`).

`variables_info` lúc đầu: `sort_buffer_size` và `port` có nguồn `COMMAND_LINE`; bốn biến còn lại `COMPILED`. Các lỗi:

| Lệnh | Kết quả |
| --- | --- |
| `SET SESSION max_connections = 500` | `ERROR 1229 (HY000): Variable 'max_connections' is a GLOBAL variable and should be set with SET GLOBAL` |
| `SET GLOBAL innodb_buffer_pool_instances = 2` | `ERROR 1238 (HY000): Variable 'innodb_buffer_pool_instances' is a read only variable` |
| `SET GLOBAL sysdate_is_now = 1` | `ERROR 1193 (HY000): Unknown system variable 'sysdate_is_now'` |
| `SET PERSIST innodb_read_io_threads = 8` | `ERROR 1238 ... is a read only variable` |
| `SET PERSIST_ONLY innodb_read_io_threads = 8` | OK; runtime vẫn là 4 |

Giá trị `sort_buffer_size` qua các bước (byte):

| Bước | session | global |
| --- | --- | --- |
| Session A sau `SET sort_buffer_size = 2097152` | 2097152 | 1048576 |
| Kết nối B lúc vừa kết nối | 1048576 | 1048576 |
| A chạy `SET GLOBAL ... = 4194304`, đọc lại chính A | 1048576 | 4194304 |
| B sau 3 giây (A đã `SET GLOBAL`) | **1048576** | 4194304 |
| Kết nối C mới | **4194304** | 4194304 |
| C sau `SET ... = 65536` rồi `SET SESSION ... = DEFAULT` | 4194304 | 4194304 |
| D sau `SET GLOBAL ... = DEFAULT` | — | **262144** |

Sau `SET GLOBAL ... = DEFAULT`, `variables_info` ghi `VARIABLE_SOURCE = DYNAMIC`, `SET_USER = root`. Đã khôi phục global về 1048576.

Persist: `@@global.max_connections` = 300 ngay lập tức, `@@global.innodb_read_io_threads` = 4. `performance_schema.persisted_variables` có hai dòng (300 và 8). Nội dung `mysqld-auto.cnf` (đã rút gọn metadata):

```json
{
  "Version": 2,
  "mysql_static_variables": {
    "innodb_read_io_threads": { "Value": "8", "Metadata": { "Host": "localhost", "User": "root", "Timestamp": 1792135895700127 } }
  },
  "mysql_dynamic_parse_early_variables": {
    "max_connections": { "Value": "300", "Metadata": { "Host": "localhost", "User": "root", "Timestamp": 1792135895556034 } }
  }
}
```

Sau khi khởi động lại **với `--no-defaults`**:

| Biến | Giá trị | `VARIABLE_SOURCE` |
| --- | --- | --- |
| `max_connections` | **151** (không phải 300) | COMPILED |
| `innodb_read_io_threads` | **4** (không phải 8) | COMPILED |
| `sort_buffer_size` | 1048576 | COMMAND_LINE |

`performance_schema.persisted_variables` vẫn liệt kê hai giá trị đã persist, tệp `mysqld-auto.cnf` vẫn còn, nhưng **không được áp dụng**. `mysqld --verbose --help` của bản này mô tả `--no-defaults` là "Don't read default options from any option file, except for login file". Sau `RESET PERSIST`, bảng `persisted_variables` rỗng và tệp chỉ còn `{"Version": 2}`.

**Diễn giải và đối chiếu với sách.**

- Khớp tr. 102: thay đổi session chỉ ảnh hưởng kết nối đó; hộp Mẹo tr. 102 đúng cả với **chính kết nối** gõ `SET GLOBAL` (A vẫn giữ 1 MiB). Ứng dụng giữ kết nối lâu (connection pool) sẽ không thấy thay đổi cho tới khi kết nối được mở lại.
- Khớp tr. 102–103: `DEFAULT` ở global trả về giá trị biên dịch sẵn (262144), **không** phải 1 MiB đã truyền ở dòng lệnh lúc khởi động.
- Khớp tr. 103 về `SET PERSIST`. **Bổ sung ngoài sách (đã đo):** `--no-defaults` khiến server bỏ qua cả `mysqld-auto.cnf`. Vì quy tắc môi trường bắt buộc `--no-defaults`, thí nghiệm **không** kiểm được trường hợp khởi động bình thường; theo tài liệu MySQL 8.0 thì khi `persisted_globals_load` = ON (mặc định, đo được) và server đọc option file như thường, giá trị persist được áp dụng lúc khởi động. Phần đó là **Chưa thực thi**.
- Hệ quả vận hành (**Bổ sung ngoài sách**): ai đó khởi động server "sạch" bằng `--no-defaults` để gỡ lỗi sẽ âm thầm mất mọi giá trị đã `SET PERSIST`.

**Dọn dẹp và khôi phục.** `SET GLOBAL sort_buffer_size = 1048576`; `RESET PERSIST` (đã chạy, tệp chỉ còn `{"Version": 2}`).

---

<a id="tn2"></a>

## TN2. File cấu hình mẫu của sách so với MySQL 8.0.42

**Mục tiêu.** Sách nói file mẫu "built around MySQL 8.0" (tr. 107) và chú thích 3 cảnh báo tùy chọn có thể bị đổi hay bỏ. Kiểm tra từng tùy chọn của file mẫu và vài giá trị mặc định mà sách nêu, trên đúng bản 8.0.42. Xem [mục 2.3.1](../chapter-05.md#s2-3-1) và [bảng phụ thuộc phiên bản](../chapter-05.md#phien-ban).

**Giả thuyết (Dự đoán).** `log_slow_queries` đã bị bỏ từ lâu nên server từ chối; `O_DIRECT` là giá trị chỉ có trên Unix; `innodb_log_file_size` vẫn được chấp nhận nhưng đã deprecated; nhiều mặc định trong sách là của bản cũ.

**Chuẩn bị.** Server **đã tắt**. Dùng `--validate-config` (có từ 8.0.16, **Bổ sung ngoài sách**): server chỉ kiểm cấu hình rồi thoát, không mở dữ liệu. Giá trị `<value>` của sách được điền giả định (buffer pool 1G, log file 256M, `max_connections` 500, `thread_cache_size` 50, `table_open_cache` 4000); đường dẫn Unix đổi sang thư mục thử nghiệm.

**Các bước và lệnh đo.**

```text
mysqld --no-defaults --datadir=<datadir> --port=3315 ... <tùy chọn cần thử> --validate-config
```

Thử: V1 toàn bộ tùy chọn của file mẫu; V2 chỉ `--log_slow_queries=...`; V3 chỉ `--innodb_flush_method=O_DIRECT`; V4 chỉ `--innodb_log_file_size=256M`; V5 chỉ `--user=mysql`; V6 bộ thay thế cho 8.0.30+ (`--slow_query_log=ON --slow_query_log_file=... --innodb_redo_log_capacity=512M --innodb_flush_method=unbuffered`); V7 `--sysdate-is-now`; V8 đối chứng với giá trị sai kiểu `--max_connections=abc`.

Sau đó khởi động thật (lần chạy 3) với `--innodb_log_file_size=256M`, đọc biến, trạng thái redo, thử `SET GLOBAL innodb_redo_log_capacity`. Ngoài ra, khi server chạy: đọc mặc định bằng `performance_schema.global_variables`, liệt kê datadir, và gõ thử cú pháp cũ `SHOW INNODB STATUS`.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng; `--validate-config` chạy khi server đã tắt; lần chạy 3 cấu hình mặc định trừ `innodb_log_file_size`).

| Thử | Exit code | Thông báo chính (đã rút gọn timestamp) |
| --- | --- | --- |
| V1 toàn bộ file mẫu | 1 | `Error while setting value 'O_DIRECT' to 'innodb-flush-method'`; `Parsing options for plugin 'InnoDB' failed`; kéo theo `unknown variable 'innodb_buffer_pool_size=1G'`; `Aborting` |
| V2 `log_slow_queries` | 1 | `unknown variable 'log_slow_queries=...'`; `Aborting` |
| V3 `innodb_flush_method=O_DIRECT` | 1 | như dòng đầu của V1 |
| V4 `innodb_log_file_size=256M` | 0 | không có thông báo khi chỉ validate |
| V5 `user=mysql` | 0 | không có thông báo |
| V6 bộ thay thế 8.0.30+ | 0 | không có thông báo |
| V7 `--sysdate-is-now` | 0 | không có thông báo |
| V8 `max_connections=abc` | 1 | `Unknown suffix 'a' used for variable 'max_connections'` |

Lần chạy 3 với `--innodb_log_file_size=256M`, error log có:

```text
[Warning] [MY-013907] [InnoDB] Deprecated configuration parameters innodb_log_file_size and/or
innodb_log_files_in_group have been used to compute innodb_redo_log_capacity=536870912.
Please use innodb_redo_log_capacity instead.
```

| Quan sát | Giá trị |
| --- | --- |
| `@@innodb_log_file_size`, `@@innodb_log_files_in_group` | 268435456, 2 |
| `@@innodb_redo_log_capacity` (biến) | 104857600 (vẫn là mặc định, nguồn COMPILED) |
| `Innodb_redo_log_capacity_resized` (status) | **536870912** |
| Tổng kích thước `#innodb_redo` | 523.370.496 byte, 32 tệp |
| `SET GLOBAL innodb_redo_log_capacity = 209715200` | OK; status `Innodb_redo_log_capacity_resized` = 209715200, `Innodb_redo_log_resize_status` = OK; thư mục còn 206.438.400 byte |
| `SET GLOBAL innodb_log_file_size = 100000000` | `ERROR 1238 ... read only variable` |

`SHOW INNODB STATUS` (cú pháp sách dùng ở tr. 110 và 118): `ERROR 1064 (42000) ... near 'INNODB STATUS'`. `SHOW ENGINE INNODB STATUS` chạy bình thường.

Một số mặc định đo được so với sách (đầy đủ ở [bảng phụ thuộc phiên bản](../chapter-05.md#phien-ban)):

| Biến | Sách nói | 8.0.42 đo được |
| --- | --- | --- |
| `innodb_log_buffer_size` | 1 MB (tr. 113) | 16777216 (16 MiB) |
| `innodb_data_file_path` | một tệp 10 MB tự giãn (tr. 116) | `ibdata1:12M:autoextend` |
| `innodb_old_blocks_time` | 0 (tr. 124) | 1000 |
| `thread_cache_size` | -1, tự tính (tr. 111) | 9 (giá trị đã tính) |
| `max_connect_errors` | 100 (tr. 120) | 100 |
| `max_connections` | 151 (tr. 121) | 151 |
| `sync_binlog` | 1 (tr. 118) | 1 |
| `innodb_thread_sleep_delay` | 10000 µs (tr. 119) | 10000 |
| `innodb_read_io_threads`, `innodb_write_io_threads` | 4 và 4 (tr. 123) | 4 và 4 |
| `tmp_table_size`, `max_heap_table_size` | sách đặt 32M (tr. 107) | mặc định 16777216 mỗi biến |
| `open_files_limit` | sách đặt 65535 (tr. 107) | 10209 (Windows) |
| `thread_handling` | — | `one-thread-per-connection`; `--help` liệt kê thêm `no-threads`, `loaded-dynamically` |

Datadir sau khi khởi tạo và sau khi nạp khoảng 360 MB dữ liệu:

| Tệp hay thư mục | Sau khởi tạo | Sau thí nghiệm |
| --- | --- | --- |
| `ibdata1` (system tablespace) | 12.582.912 byte | 12.582.912 byte (không đổi) |
| `undo_001`, `undo_002` | 16 MiB mỗi tệp | 32 MiB mỗi tệp |
| `#ib_16384_0.dblwr`, `#ib_16384_1.dblwr` (doublewrite) | 196.608 và 8.585.216 byte | không đổi |
| `#innodb_redo` | 32 tệp, tổng 104.857.600 byte | như cũ (trước lần chạy 3) |
| `lab05\big.ibd`, `lab05\hot.ibd` | — | 327.155.712 và 28.311.552 byte |
| `binlog.000001`, `binlog.000002` | — | 228.922.365 và 165.040.628 byte |
| `ib_buffer_pool` | 6.677 byte | 14.479 byte (1894 dòng) |

`information_schema.FILES` xác nhận `undo_001`, `undo_002` có `FILE_TYPE = 'UNDO LOG'`; mỗi bảng của `lab05` là một tablespace riêng.

**Diễn giải và đối chiếu với sách.**

- File mẫu của sách **không khởi động được nguyên trạng** trên 8.0.42: `log_slow_queries` không còn tồn tại (thay bằng `slow_query_log` và `slow_query_log_file`), và trên Windows `O_DIRECT` không phải giá trị hợp lệ. Đây đúng là điều chú thích 3 (tr. 107) cảnh báo; nó cũng cho thấy vì sao nên kiểm file cấu hình bằng `--validate-config` trước khi khởi động lại production (**Bổ sung ngoài sách**).
- `innodb_log_file_size` (tr. 107, 113, 124) vẫn được chấp nhận ở 8.0.42 nhưng chỉ để **tính** `innodb_redo_log_capacity`. Bẫy đo được: biến `innodb_redo_log_capacity` vẫn hiện 100 MiB trong khi dung lượng thật là 512 MiB; phải đọc status `Innodb_redo_log_capacity_resized`. Từ 8.0.30, capacity đổi được **online**, còn `innodb_log_file_size` là read-only.
- Sách viết undo log nằm trong tablespace chính (tr. 117). Ở 8.0.42, undo nằm trong hai undo tablespace riêng, và `ibdata1` không lớn lên dù có 20.000 transaction cập nhật trong TN5.
- Binary log bật mặc định ở 8.0 và chiếm khoảng 394 MB sau các thí nghiệm. File mẫu của sách không nhắc binary log; sách nói sẽ bàn sau (tr. 107).

**Dọn dẹp và khôi phục.** `--validate-config` không thay đổi gì. Lần chạy 3 để lại redo capacity 200 MiB trong datadir thử nghiệm; không ảnh hưởng gì ngoài instance này. Đã tắt server.

---

<a id="tn3"></a>

## TN3. Buffer pool: hit ratio, resize online, `innodb_old_blocks_time`, thread cache

**Mục tiêu.**

1. Kiểm chứng lập luận "đừng tune theo tỉ lệ" (tr. 105–106): hit ratio của buffer pool phụ thuộc workload và có thể bị thổi phồng mà chi phí I/O thật không đổi.
2. Quan sát trang bẩn, buffer pool dump và load khi khởi động lại (tr. 110–111), resize online.
3. Kiểm chứng `innodb_old_blocks_time` (tr. 124): một lần quét lớn có đẩy "hot set" ra khỏi buffer pool không, khi giá trị là 1000 so với 0.
4. Đo thread cache bằng `Threads_created` (tr. 111) và kiểm mô tả về việc thu nhỏ cache (tr. 103–104).

Xem [mục 2.2](../chapter-05.md#s2-2), [2.4](../chapter-05.md#s2-4), [2.8](../chapter-05.md#s2-8).

**Giả thuyết (Dự đoán).**

- Quét toàn bảng `big` (khoảng 2,3 lần buffer pool) đọc gần như mọi trang từ đĩa, nhưng hit ratio vẫn không thấp tới mức phản ánh điều đó.
- Thêm nhiều lookup rẻ vào cùng khoảng đo sẽ làm hit ratio tăng vọt, trong khi số trang đọc từ đĩa không đổi.
- Với `innodb_old_blocks_time = 1000`, hot set sống sót sau lần quét; với 0 thì bị đẩy ra.
- `thread_cache_size = 0` làm mỗi kết nối tạo một thread mới.

**Chuẩn bị.** Bảng `big`, `hot` như ở [Môi trường](#moi-truong). Hai thủ tục đo (đã rút gọn phần khai báo), lưu vào bảng `bp_result`:

```sql
-- m_start(): chụp các bộ đếm vào bảng bp_snap
--   Innodb_buffer_pool_read_requests  (đọc logic: số lần lấy trang trong buffer pool)
--   Innodb_buffer_pool_reads          (đọc logic không có sẵn trong pool, phải đọc đĩa)
--   Innodb_buffer_pool_read_ahead     (trang đọc trước bởi read-ahead)
--   Innodb_pages_read                 (tổng số trang đọc từ đĩa)
--   SUM(PAGES_MADE_YOUNG), SUM(PAGES_NOT_MADE_YOUNG) từ information_schema.INNODB_BUFFER_POOL_STATS
-- m_end(lbl): tính hiệu số và hit_ratio = 100 * (1 - reads / read_requests)

DELIMITER //
CREATE PROCEDURE lookups(IN n INT, IN maxid INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  DECLARE k INT;
  DECLARE v INT;
  WHILE i < n DO
    SET k = 1 + FLOOR(RAND() * maxid);   -- tính khóa trước, để WHERE là so sánh hằng
    SELECT grp INTO v FROM big WHERE id = k;
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
```

`EXPLAIN` xác nhận lookup dùng `type = const` trên PRIMARY, còn `SELECT SUM(LENGTH(pad)) FROM big` là `Table scan on big`.

**Các bước và lệnh đo.**

1. Buffer pool 128 MiB: W1a và W1b quét `big` hai lần; W2 là 20.000 lookup với id trong 1..1000; W3 là 20.000 lookup với id trong 1..1.000.000; W4 là một lần quét `big` **cộng** 200.000 lookup id 1..1000 trong cùng một khoảng đo.
2. `SET GLOBAL innodb_buffer_pool_size = 536870912`; chờ `Innodb_buffer_pool_resize_status`; lặp quét (W5a, W5b, W5c) và W6 (như W3).
3. Thu lại 128 MiB. Thủ tục `lru_round(obt)`: đặt `innodb_old_blocks_time = obt`; quét `hot` ba lần, mỗi lần cách 1,5 giây (để trang được đưa lên phần "new"); đo một lần quét `big`; rồi đo một lần quét `hot`. Chạy xen kẽ 1000, 0, 1000, 0.
4. Thread cache: đọc `Threads_created`, `Threads_cached`, `Connections`; mở 30 kết nối tuần tự (`mysql -e "DO 1"`) với cache mặc định, rồi với `thread_cache_size = 0`, rồi trả về 9. Thử riêng: làm ấm cache bằng 6 kết nối song song, rồi trong **một** session đọc `Threads_cached`, `SET GLOBAL thread_cache_size = 0`, đọc lại ngay.
5. Trang bẩn: đọc `Innodb_buffer_pool_pages_dirty` mỗi 4 giây sau khi nạp dữ liệu. Dump/load: đếm dòng `ib_buffer_pool` sau khi tắt, đọc `Innodb_buffer_pool_load_status` sau khi khởi động lại (trong TN1).

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng, buffer pool 128 MiB trừ khi ghi khác, dữ liệu tổng hợp 1.000.000 + 80.000 hàng).

*Hit ratio và số trang đọc từ đĩa:*

| Lần đo | Đọc logic | Đọc đĩa (`reads`) | Trang đọc trước | Tổng trang từ đĩa | Hit ratio |
| --- | --- | --- | --- | --- | --- |
| W1a quét `big` | 35.654 | 12.468 | 0 | 12.468 | 65,03% |
| W1b quét `big` lần 2 | 34.889 | 11.818 | 0 | 11.818 | 66,13% |
| W2 20.000 lookup id 1..1000 | 40.450 | 11 | 0 | 11 | 99,97% |
| W3 20.000 lookup id 1..1.000.000 | 72.060 | 11.394 | 0 | 11.394 | 84,19% |
| W4 quét `big` + 200.000 lookup id 1..1000 | 437.672 | 11.443 | 0 | 11.443 | **97,39%** |
| W5a quét `big` ngay sau khi tăng lên 512 MiB | 32.872 | 9.805 | 0 | 9.805 | 70,17% |
| W5b quét lần 2 (512 MiB) | 23.088 | 3 | 0 | 3 | 99,99% |
| W5c quét lần 3 (512 MiB) | 23.070 | 3 | 0 | 3 | 99,99% |
| W6 20.000 lookup id 1..1.000.000 (512 MiB) | 60.672 | 3 | 0 | 3 | 99,995% |

Một lần chạy thăm dò trước đó (cùng workload, trước khi sửa cách đọc bộ đếm made young) cho 63,2%, 63,2%, 99,97%, 83,0% với W1a, W1b, W2, W3. Sau resize lên 512 MiB, `Innodb_buffer_pool_pages_total` = 32768, `pages_data` = 16935 (cả bảng `big` đã nằm trong pool), `Innodb_buffer_pool_resize_status` = "Completed resizing buffer pool".

*`innodb_old_blocks_time` (buffer pool 128 MiB, bảng `hot` khoảng 1250 trang):*

| Vòng | `old_blocks_time` | Quét `big`: made young / not young | Quét `hot` sau đó: trang từ đĩa (trong đó read-ahead) | Hit ratio của lần quét `hot` |
| --- | --- | --- | --- | --- |
| R1 | 1000 | 1.384 / 31.396 | **3** (0) | 99,83% |
| R2 | 0 | 16.858 / 0 | **1.645** (1.152) | 85,88% |
| R3 | 1000 | 0 / 19.055 | **0** (0) | 100% |
| R4 | 0 | 16.863 / 0 | **1.263** (1.152) | 96,30% |

Đã trả `innodb_old_blocks_time` về 1000. Lưu ý: lần thu nhỏ buffer pool về 128 MiB hoàn tất lúc 14:30:14; vòng R1 bắt đầu ngay sau đó, kết quả R1 khớp R3.

*Thread cache* (mỗi lần đọc trạng thái cũng là một kết nối, nên mỗi nhóm có 31 kết nối):

| Bước | `thread_cache_size` | `Threads_created` | `Threads_cached` | `Connections` |
| --- | --- | --- | --- | --- |
| Trước | 9 | 2 | 1 | 36 |
| Sau 30 kết nối | 9 | 2 | 1 | 67 |
| Ngay sau `SET GLOBAL thread_cache_size = 0` | 0 | 3 | 0 | 69 |
| Sau 30 kết nối | 0 | **34** | 0 | 100 |
| Sau `SET GLOBAL thread_cache_size = 9` | 9 | 35 | 0 | 102 |
| Sau 30 kết nối | 9 | 35 | 0 | 133 |

Thử riêng việc thu nhỏ: trong cùng một session, `Threads_cached` = 5 → chạy `SET GLOBAL thread_cache_size = 0` → `Threads_cached` = **0** ngay lập tức, và vẫn 0 sau 1 giây; `Threads_created` không đổi (40).

*Trang bẩn và dump/load:* sau khi nạp dữ liệu, số trang bẩn là 3815, rồi 1142, 342, 0, 0 (mẫu cách nhau khoảng 4 giây, server rảnh). Khi tắt server, tệp `ib_buffer_pool` có 1894 dòng (mỗi dòng là cặp space id, page number; `innodb_buffer_pool_dump_pct` = 25). Server khởi động lúc 14:31:51; `Innodb_buffer_pool_load_status` = "Buffer pool(s) load completed at 261016 14:31:52"; `Innodb_buffer_pool_pages_data` = 2944 ngay sau đó.

**Diễn giải và đối chiếu với sách.**

- **Hit ratio phụ thuộc workload** (tr. 105): W2 đạt 99,97% còn W3 chỉ 84% trên **cùng** buffer pool, cùng bảng.
- **Hit ratio không nói gì về chi phí thật**: W4 đọc từ đĩa đúng bằng W1 (khoảng 11–12 nghìn trang) nhưng hit ratio nhảy từ 65% lên 97%, chỉ vì thêm 200.000 lần đọc rẻ vào mẫu số. Đây chính là ý của chú thích 2 (tr. 106) về công cụ tạo ra hit ratio tùy ý.
- **Tương quan ngẫu nhiên** (tr. 105): khi pool đủ chứa cả bảng (512 MiB), hit ratio lên gần 100% và số trang đọc từ đĩa gần 0. Nhưng thứ cho biết cần 512 MiB là **kích thước working set** so với pool và **số trang đọc từ đĩa mỗi giây**, không phải con số phần trăm (W2 đã 99,97% ngay ở 128 MiB).
- **Thêm một bằng chứng (Bổ sung ngoài sách):** ở R2 và R4, 1.152 trang được nạp bằng read-ahead; `Innodb_buffer_pool_reads` không đếm các trang này, nên hit ratio của R4 là 96% dù gần như cả bảng `hot` được đọc lại từ đĩa.
- **`innodb_old_blocks_time`**: kết quả khớp **khuyến nghị** của sách (đặt khoảng 1000 rất hiệu quả, tr. 124) nhưng **không khớp mô tả cơ chế và giá trị mặc định** của sách. Ở 8.0.42 mặc định đã là 1000. Bản `--help` mô tả: trang được chuyển lên đầu "new" của buffer pool nếu lần truy cập đầu tiên đã cách ít nhất số mili giây này. Tức là trang mới vào nằm ở phần **old** và chỉ lên phần **new (young)** khi đủ điều kiện; sách viết ngược hai phần. Xem [mục 2.8](../chapter-05.md#old-blocks-time).
- **Thread cache**: với cache mặc định, 31 kết nối không tạo thread nào; với cache 0, 31 kết nối tạo 31 thread. Khớp tr. 111. **Không khớp** tr. 103–104: sách nói thu nhỏ `thread_cache_size` không làm giảm số thread trong cache ngay; ở 8.0.42, `Threads_cached` về 0 ngay sau `SET GLOBAL`. Mô tả của sách có thể đúng với bản cũ; điều này chưa được kiểm chứng.
- Dump/load khớp tr. 110–111: mặc định bật cả hai, và ở quy mô nhỏ này việc nạp xong trong khoảng 1 giây.
- Read-ahead bằng 0 trong mọi lần quét `big`. Nguyên nhân **chưa được điều tra**; không suy ra kết luận chung từ quan sát này.

**Dọn dẹp và khôi phục.** `innodb_buffer_pool_size` đã trả về 134217728, `innodb_old_blocks_time` về 1000, `thread_cache_size` về 9. Bảng giữ lại cho TN5.

---

<a id="tn4"></a>

## TN4. Durability và thông lượng commit

**Mục tiêu.** Định lượng đánh đổi ở tr. 114–115 và tr. 118: `innodb_flush_log_at_trx_commit` (0, 1, 2) và `sync_binlog` (0, 1) thay đổi số lần fsync và thông lượng commit ra sao; quan sát group commit (tr. 114). Xem [mục 2.5.2](../chapter-05.md#s2-5-2) và [2.5.4](../chapter-05.md#s2-5-4).

**Giả thuyết (Dự đoán).**

- Mặc định (1, 1): mỗi commit tốn ít nhất một fsync cho redo và một cho binary log.
- Đặt chỉ một trong hai về giá trị "lỏng" thì fsync của cái còn lại vẫn là nút thắt.
- (2, 0) và (0, 0) gần như không fsync khi commit, nhanh hơn nhiều lần.
- Nhiều client đồng thời thì số fsync trên mỗi commit giảm nhờ group commit.

**Chuẩn bị.**

```sql
USE lab05;
CREATE TABLE c (id BIGINT AUTO_INCREMENT PRIMARY KEY, v INT NOT NULL) ENGINE=InnoDB;
DELIMITER //
CREATE PROCEDURE ins_loop(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < n DO
    INSERT INTO c (v) VALUES (i);   -- autocommit = 1: mỗi INSERT là một transaction
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
```

Thủ tục `dur_run(rnd, f, sb, n)` (đã rút gọn): `SET GLOBAL innodb_flush_log_at_trx_commit = f`, `SET GLOBAL sync_binlog = sb`; chụp `Innodb_os_log_fsyncs`, `Innodb_data_fsyncs` và `COUNT_MISC` của `wait/io/file/sql/binlog` trong `performance_schema.file_summary_by_event_name` (fsync của binary log được tính vào nhóm MISC); lấy `SYSDATE(6)`; `CALL ins_loop(n)`; lấy lại thời gian và bộ đếm; ghi vào bảng `dur_result`.

**Các bước và lệnh đo.**

1. Năm cấu hình (f, sb) = (1,1), (2,1), (1,0), (2,0), (0,0); mỗi lần 3000 commit; **5 vòng**, mỗi vòng chạy lần lượt cả năm cấu hình (xen kẽ để chia đều nhiễu).
2. Group commit: (1,1); 8000 commit bằng 1 client (`ins_loop(8000)`) so với 8 client song song (`ins_loop(1000)` mỗi client); 3 lần lặp; thời gian đo từ shell, nên gồm cả chi phí khởi động tiến trình `mysql`.
3. Trả về (1,1).

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định, binary log bật, một client chạy thủ tục ngay trong server nên không có round trip mạng cho từng commit).

*Ma trận durability, 3000 commit mỗi lần, 5 lần mỗi cấu hình:*

| `innodb_flush_log_at_trx_commit` | `sync_binlog` | Trung vị commit/giây | Khoảng (min–max) | fsync redo / commit (trung vị) | fsync binlog / commit |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 (mặc định) | **645** | 593–682 | 1,335 | 1,0 |
| 2 | 1 | **1.234** | 1.047–1.333 | 0,002 | 1,0 |
| 1 | 0 | **954** | 939–1.129 | 1,470 | 0 |
| 2 | 0 | **8.214** | 6.414–11.161 | 0,001 | 0 |
| 0 | 0 | **23.208** | 20.484–25.205 | 0 | 0 |

Số liệu thô của từng lần (5 vòng × 5 cấu hình, đã rút gọn cột): (1,1) 592,6 · 645,0 · 669,8 · 681,6 · 598,6; (2,1) 1046,5 · 1234,2 · 1333,3 · 1306,9 · 1201,3; (1,0) 953,8 · 938,5 · 1129,0 · 949,4 · 1006,8; (2,0) 9026,6 · 6414,3 · 8213,6 · 8086,0 · 11161,2; (0,0) 23241,4 · 23029,8 · 23207,8 · 20483,5 · 25205,0.

*Group commit, cấu hình (1,1), 8000 commit mỗi lần:*

| Số client | Commit/giây (3 lần) | Trung vị | fsync redo / commit | fsync binlog / commit |
| --- | --- | --- | --- | --- |
| 1 | 678,8 · 618,6 · 587,7 | **619** | 1,28–1,34 | 1,000 |
| 8 | 3174,6 · 3047,9 · 2918,4 | **3.048** | 0,46–0,49 | 0,285–0,297 |

**Diễn giải và đối chiếu với sách.**

- **Nhiễu.** Khoảng min–max của cùng một cấu hình rộng tới khoảng 20% (hơn nữa với (2,0)). Chênh lệch giữa các cấu hình lớn hơn nhiều, nên thứ hạng là đáng tin; con số tuyệt đối thì không. Số fsync trên mỗi commit ổn định giữa các lần và là bằng chứng chắc hơn thời gian.
- **Khớp tr. 114**: với giá trị 1, redo được fsync ở mỗi commit; với 2 và 0, gần như không có fsync redo khi commit (chỉ khoảng một lần mỗi giây).
- **Khớp tr. 118 và cho thấy vì sao hai biến đi cùng nhau**: chỉ nới `innodb_flush_log_at_trx_commit` (2,1) hay chỉ nới `sync_binlog` (1,0) mới tăng thông lượng khoảng 1,5–1,9 lần, vì fsync còn lại vẫn nằm trên đường commit. Nới cả hai thì tăng khoảng 13 lần trong phép đo này.
- **Bổ sung ngoài sách**: số fsync redo trên mỗi commit lớn hơn 1 (khoảng 1,3–1,5). MySQL 8.0 ghi và flush redo bằng các luồng nền riêng; số fsync không nhất thiết bằng số commit. Phép đo này không tách được từng nguồn fsync.
- **Group commit** (tr. 114): với 8 client, mỗi fsync binlog phục vụ trung bình khoảng 3,4 commit và mỗi fsync redo khoảng 2 commit; thông lượng tăng khoảng 4,9 lần mà không giảm durability.
- **Giới hạn của phép đo (Dự đoán).** Ứng dụng thật có round trip mạng cho từng commit, nên tỉ lệ chênh giữa các cấu hình sẽ nhỏ hơn ở đây. Trên RAID có write cache dùng pin hay ổ có tụ bảo vệ, fsync rẻ hơn nhiều, khoảng cách cũng thu hẹp (tr. 115, 118–119).
- **Không đo được ở đây**: dữ liệu bị mất thật khi crash hay mất điện. Thí nghiệm chỉ đo chi phí, không đo rủi ro; rủi ro theo sách là mất tối đa khoảng một giây với 0 và 2 (có thể hơn khi flush bị nghẽn), và binlog lệch dữ liệu khi `sync_binlog` khác 1 (tr. 114, 118).

**Dọn dẹp và khôi phục.** Đã chạy `SET GLOBAL innodb_flush_log_at_trx_commit = 1; SET GLOBAL sync_binlog = 1;` và kiểm lại giá trị (1, 1).

---

<a id="tn5"></a>

## TN5. Transaction dài, history list và purge

**Mục tiêu.** Kiểm chứng tr. 117–118: một transaction để mở lâu ở REPEATABLE READ ngăn purge dọn phiên bản cũ; history list length tăng; đóng transaction thì purge bắt kịp. Xem [mục 2.5.3](../chapter-05.md#old-row-versions).

**Giả thuyết (Dự đoán).** Trong khi một snapshot còn mở, history list tăng theo số transaction ghi; sau khi snapshot commit, nó giảm về gần 0 trong vài giây.

**Chuẩn bị.**

```sql
DELIMITER //
CREATE PROCEDURE upd_loop(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < n DO
    UPDATE hot SET grp = grp + 1 WHERE id = 1 + (i MOD 80000);   -- mỗi UPDATE là một transaction
    SET i = i + 1;
  END WHILE;
END//
DELIMITER ;
```

Đọc history list bằng `SELECT COUNT FROM information_schema.INNODB_METRICS WHERE NAME = 'trx_rseg_history_len'` (metric này bật sẵn), đối chiếu với dòng "History list length" của `SHOW ENGINE INNODB STATUS`.

**Các bước và lệnh đo.**

- Lần A: session X chạy `START TRANSACTION WITH CONSISTENT SNAPSHOT; SELECT COUNT(*) FROM hot; DO SLEEP(25); SELECT SUM(grp) FROM hot; COMMIT;`. Trong lúc đó session Y chạy **ba** lệnh `UPDATE hot SET grp = grp + 1` (mỗi lệnh 80.000 hàng, một transaction).
- Lần B: X mở snapshot như trên (ngủ 30 giây); Y chạy `CALL upd_loop(20000)`, tức **20.000** transaction nhỏ. Để Y chạy nhanh hơn, tạm đặt `innodb_flush_log_at_trx_commit = 2` rồi trả về 1 sau thí nghiệm.

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng cấu hình mặc định trừ ghi chú về `innodb_flush_log_at_trx_commit` ở lần B, bảng `hot` 80.000 hàng).

Lần A: history list 2 → 3 → 4 → 5 sau mỗi lệnh UPDATE 80.000 hàng. Session X (`@@transaction_isolation` = REPEATABLE-READ) vẫn đọc `SUM(grp)` = 3.960.000 (giá trị cũ) sau khi Y đã cập nhật; sau đó Y đọc 4.200.000. Sau khi X commit, history list vẫn là 5 trong 10 giây quan sát (số quá nhỏ, purge chưa chạy lượt nào trong khoảng đó).

Lần B:

| Thời điểm | Trạng thái | History list |
| --- | --- | --- |
| 14:38:10 | trước | 0 |
| 14:38:35 | sau 20.000 UPDATE, X đang mở | **20.000** |
| 14:38:40 | X vẫn mở thêm 5 giây | 20.000 |
| 14:38:40 | ngay sau khi X commit | 19.701 |
| 14:38:42 | sau 2 giây | **0** |
| 14:38:44–14:38:53 | các lần đọc tiếp | 0 |

Tệp `undo_001`, `undo_002` lớn từ 16 MiB lên 32 MiB mỗi tệp trong quá trình thí nghiệm; `ibdata1` giữ nguyên 12 MiB.

**Diễn giải và đối chiếu với sách.**

- Khớp tr. 117: snapshot của X giữ phiên bản cũ (X vẫn thấy tổng cũ), và phiên bản cũ không được purge khi X còn mở.
- History list đếm **số transaction** (nhóm undo log) chờ purge, không phải số hàng: một UPDATE 80.000 hàng chỉ làm nó tăng 1, còn 20.000 UPDATE một hàng làm nó tăng 20.000. Điều này khớp cách sách giải thích `innodb_max_purge_lag` là số transaction chờ purge (tr. 118); câu "cho thấy kích thước undo log" (tr. 118) nên hiểu theo nghĩa đó.
- Khi X commit, purge dọn 20.000 transaction trong khoảng 2 giây ở quy mô này. Sách không nói tốc độ purge; con số này chỉ đúng cho thí nghiệm nhỏ trên máy này.
- `innodb_max_purge_lag` (mặc định đo được là 0, tức không hãm) **không được thử**: muốn thấy nó hãm ghi cần history list lớn hơn ngưỡng trong thời gian dài. Phần này **Chưa thực thi**.

**Dọn dẹp và khôi phục.** `innodb_flush_log_at_trx_commit` đã trả về 1 (kiểm lại bằng `SELECT`).

---

<a id="don-dep"></a>

## Dọn dẹp chung

- Instance cổng 3315 đã tắt bằng `mysqladmin -h127.0.0.1 -P3315 -uroot shutdown`; lần kết nối thử sau đó trả về `ERROR 2003 (HY000): Can't connect to MySQL server on '127.0.0.1:3315'`.
- Mọi thay đổi chỉ nằm trong datadir thử nghiệm `<SP>\hpm\mysql\ch05\main\data`. Muốn làm lại từ đầu: xóa thư mục đó rồi khởi tạo lại bằng `--initialize-insecure`.
- Muốn chạy lại trên máy của bạn: dùng cổng và datadir riêng, luôn `--bind-address=127.0.0.1`, **không** chạy trên server có dữ liệu thật. Các thí nghiệm TN3 và TN4 đổi biến global (buffer pool, durability); hãy khôi phục như mô tả ở cuối mỗi thí nghiệm.
