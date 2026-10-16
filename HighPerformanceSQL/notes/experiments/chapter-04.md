# Thực hành — Chương 4: Operating System and Hardware Optimization

Quay lại ghi chú: [../chapter-04.md](../chapter-04.md) · Đáp án tự kiểm tra: [../answers/chapter-04.md](../answers/chapter-04.md)

Chương 4 chủ yếu bàn phần cứng và Linux. Máy thực hành là Windows, nên các công cụ `vmstat`, `iostat`, `mpstat` không chạy được; phần đó chỉ có quy trình ([thí nghiệm 5](#exp5), **Chưa thực thi**). Bốn thí nghiệm còn lại dùng chính MySQL để quan sát những cơ chế mà chương mô tả: cache đọc, working set, cách cache xử lý ghi, và "mỗi truy vấn một CPU". Chỉ số chính là các biến trạng thái tất định của InnoDB, không phải thời gian.

## Môi trường

> **Kết quả đã đo** trong tệp này đều chạy trên:
>
> - **MySQL Community 8.0.42**, Windows 11, laptop AMD Ryzen 7 PRO 8840U (8 nhân, 16 luồng), 32 GB RAM, SSD NVMe.
> - **Instance riêng** ở cổng 3314, khởi động với `--no-defaults`, datadir trong thư mục scratchpad; không đụng tới MySQL của người dùng ở cổng 3306.
> - Cấu hình mặc định của 8.0.42, **trừ**: `innodb_buffer_pool_size` (256 MB hoặc 16 MB tùy thí nghiệm), `innodb_buffer_pool_dump_at_shutdown=OFF` và `innodb_buffer_pool_load_at_startup=OFF` để mỗi lần khởi động là cache lạnh thật.
> - Giá trị mặc định liên quan, đã đọc bằng `SELECT @@...`: `innodb_page_size` 16384; `innodb_flush_method` **unbuffered** (giá trị Windows, I/O không qua cache hệ điều hành); `innodb_buffer_pool_instances` 1; `innodb_read_ahead_threshold` 56; `innodb_random_read_ahead` OFF; `innodb_old_blocks_pct` 37; `innodb_old_blocks_time` 1000; `innodb_adaptive_hash_index` ON; `innodb_doublewrite` ON; `innodb_flush_log_at_trx_commit` 1; `log_bin` ON; `sync_binlog` 1; `innodb_max_dirty_pages_pct` 90; `innodb_max_dirty_pages_pct_lwm` 10; `innodb_io_capacity` 200; `innodb_redo_log_capacity` 100 MB; `innodb_undo_tablespaces` 2; `innodb_rollback_segments` 128.
> - **Dữ liệu tổng hợp:** 300.000 hàng, `DATA_LENGTH` 75.071.488 byte = **4.582 trang** (71,6 MB), `n_leaf_pages` 4.546, file `t.ibd` 80 MB.
> - **Nhiễu:** trong lúc đo, nhiều instance MySQL khác (của các phiên làm việc song song) chạy trên cùng máy; CPU nền khoảng 20%. Đây là laptop, không phải benchmark production. Thời gian chỉ dùng để thấy xu hướng; luôn lặp ít nhất 3 lần và báo trung vị.

## Chuẩn bị chung

**Khởi tạo và khởi động** (thay `<bin>`, `<datadir>`, `<log>`; chạy nền):

```text
"<bin>/mysqld" --no-defaults --initialize-insecure --datadir="<datadir>" --console
"<bin>/mysqld" --no-defaults --datadir="<datadir>" --port=3314 --mysqlx=OFF --bind-address=127.0.0.1 --socket=ch04.sock --innodb-buffer-pool-size=256M --innodb-buffer-pool-dump-at-shutdown=OFF --innodb-buffer-pool-load-at-startup=OFF --console > "<log>" 2>&1
```

Muốn buffer pool 16 MB thì đổi `--innodb-buffer-pool-size=16M` (MySQL tự hạ `innodb_buffer_pool_chunk_size` xuống 16 MB; đã kiểm tra). Mọi lệnh client dùng `mysql -h127.0.0.1 -P3314 -uroot`, và luôn kiểm tra `SELECT @@port, @@datadir` trước khi đo.

**Schema và dữ liệu** (**Ví dụ minh họa do AI xây dựng**; mỗi hàng khoảng 210 byte, khoảng 66 hàng mỗi trang):

```sql
CREATE DATABASE IF NOT EXISTS ch04;
USE ch04;
CREATE TABLE t (
  id  INT NOT NULL PRIMARY KEY,
  grp INT NOT NULL,
  pad VARCHAR(255) NOT NULL
) ENGINE=InnoDB;
SET SESSION cte_max_recursion_depth = 1000000;
INSERT INTO t (id, grp, pad)
WITH RECURSIVE seq(n) AS (
  SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 300000
)
SELECT n, n MOD 100, REPEAT(CHAR(65 + n MOD 26), 200) FROM seq;
ANALYZE TABLE t;
SELECT DATA_LENGTH, DATA_LENGTH/16384 AS data_pages
FROM information_schema.TABLES WHERE TABLE_SCHEMA='ch04' AND TABLE_NAME='t';
```

**Hàm và thủ tục đo.** `snap()` lưu các bộ đếm vào biến phiên; `delta(label)` in phần chênh. Biến người dùng và bảng `performance_schema.global_status` nằm trong bộ nhớ, nên bản thân việc đo không đọc trang InnoDB (đã kiểm tra: một cặp `snap`/`delta` rỗng chỉ tạo 6 logical read request).

```sql
USE ch04;
DELIMITER //
CREATE FUNCTION g(n VARCHAR(64)) RETURNS BIGINT READS SQL DATA
  RETURN (SELECT CAST(VARIABLE_VALUE AS SIGNED)
          FROM performance_schema.global_status WHERE VARIABLE_NAME = n)//
CREATE PROCEDURE snap()
BEGIN
  SET @s_t = NOW(6);
  SET @s_rr = g('Innodb_buffer_pool_read_requests');
  SET @s_r  = g('Innodb_buffer_pool_reads');
  SET @s_ra = g('Innodb_buffer_pool_read_ahead');
  SET @s_rae = g('Innodb_buffer_pool_read_ahead_evicted');
  SET @s_pr = g('Innodb_pages_read');
  SET @s_wr = g('Innodb_buffer_pool_write_requests');
  SET @s_pf = g('Innodb_buffer_pool_pages_flushed');
  SET @s_pw = g('Innodb_pages_written');
  SET @s_dbl = g('Innodb_dblwr_pages_written');
  SET @s_lw = g('Innodb_os_log_written');
  SET @s_lws = g('Innodb_log_writes');
  SET @s_lfs = g('Innodb_os_log_fsyncs');
  SET @s_rows = g('Innodb_rows_read');
END//
CREATE PROCEDURE delta(IN label VARCHAR(64))
BEGIN
  SELECT label AS phase,
    ROUND(TIMESTAMPDIFF(MICROSECOND, @s_t, NOW(6))/1000, 1) AS elapsed_ms,
    g('Innodb_buffer_pool_read_requests') - @s_rr AS bp_read_requests,
    g('Innodb_buffer_pool_reads') - @s_r AS bp_reads_miss,
    g('Innodb_buffer_pool_read_ahead') - @s_ra AS read_ahead,
    g('Innodb_buffer_pool_read_ahead_evicted') - @s_rae AS read_ahead_evicted,
    g('Innodb_pages_read') - @s_pr AS pages_read,
    g('Innodb_rows_read') - @s_rows AS rows_read,
    g('Innodb_buffer_pool_write_requests') - @s_wr AS bp_write_requests,
    g('Innodb_buffer_pool_pages_flushed') - @s_pf AS pages_flushed,
    g('Innodb_pages_written') - @s_pw AS pages_written,
    g('Innodb_dblwr_pages_written') - @s_dbl AS dblwr_pages,
    g('Innodb_os_log_written') - @s_lw AS redo_bytes,
    g('Innodb_log_writes') - @s_lws AS log_writes,
    g('Innodb_os_log_fsyncs') - @s_lfs AS log_fsyncs,
    g('Innodb_buffer_pool_pages_total') AS bp_pages_total,
    g('Innodb_buffer_pool_pages_data') AS bp_pages_data,
    g('Innodb_buffer_pool_pages_free') AS bp_pages_free,
    g('Innodb_buffer_pool_pages_dirty') AS bp_pages_dirty;
END//
CREATE PROCEDURE scan_all()
BEGIN
  SELECT SUM(LENGTH(pad)) INTO @x FROM t;
END//
CREATE PROCEDURE lookups(IN n INT, IN maxid INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  DECLARE k INT;
  DECLARE v INT;
  WHILE i < n DO
    SET k = 1 + ((i * 7919) MOD maxid);
    SELECT LENGTH(pad) INTO v FROM t WHERE id = k;
    SET i = i + 1;
  END WHILE;
END//
CREATE PROCEDURE upd_same(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < n DO
    UPDATE t SET grp = grp + 1 WHERE id = 1;
    SET i = i + 1;
  END WHILE;
END//
CREATE PROCEDURE upd_spread(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < n DO
    UPDATE t SET grp = grp + 1 WHERE id = 1 + ((i * 7919) MOD 300000);
    SET i = i + 1;
  END WHILE;
END//
CREATE PROCEDURE wait_clean(IN max_sec INT)
BEGIN
  DECLARE waited INT DEFAULT 0;
  SET GLOBAL innodb_max_dirty_pages_pct_lwm = 0;
  SET GLOBAL innodb_max_dirty_pages_pct = 0;
  WHILE g('Innodb_buffer_pool_pages_dirty') > 0 AND waited < max_sec * 2 DO
    DO SLEEP(0.5);
    SET waited = waited + 1;
  END WHILE;
  SET GLOBAL innodb_max_dirty_pages_pct = 90;
  SET GLOBAL innodb_max_dirty_pages_pct_lwm = 10;
END//
DELIMITER ;
```

Ghi chú về thiết kế:

- Khóa tra cứu được tính **trước** rồi mới đưa vào `WHERE` (`SET k = ...`). Nếu viết `WHERE id = 1 + FLOOR(RAND()*maxid)`, `RAND()` được tính lại cho từng hàng nên MySQL không dùng được primary key.
- Dãy `1 + ((i * 7919) MOD maxid)` là một hoán vị tất định (7919 là số nguyên tố, không chia hết 20.000 và 300.000), nên chạy lại cho đúng cùng chuỗi khóa.
- `wait_clean` ép page cleaner flush hết trang bẩn rồi trả hai biến về mặc định 90 và 10.

---

<a id="exp1"></a>

## Thí nghiệm 1 — Cache đọc: dữ liệu vừa và không vừa buffer pool

**Mục tiêu.** Kiểm chứng tr. 76: khi dữ liệu vừa bộ nhớ và cache đã ấm, đọc logic vẫn còn nhưng đọc vật lý biến mất; khi không vừa thì không.

**Giả thuyết (Dự đoán).**

- Buffer pool 256 MB (lớn hơn bảng 71,6 MB): lần quét đầu đọc khoảng 4.582 trang từ đĩa; các lần sau đọc 0 trang.
- Buffer pool 16 MB (khoảng 1/4,5 bảng): mọi lần quét đều đọc lại gần hết bảng, vì quét tuần tự đẩy trang cũ ra trước khi được dùng lại.
- Quét tuần tự sẽ kích hoạt **read-ahead** của InnoDB (tr. 84 và 88 nhắc InnoDB tự read-ahead).

**Chuẩn bị.** Khởi động lạnh với buffer pool tương ứng.

**Các bước và lệnh đo.**

```sql
USE ch04;
CALL snap(); CALL delta('baseline');
CALL snap(); CALL scan_all(); CALL delta('scan 1 (lanh)');
CALL snap(); CALL scan_all(); CALL delta('scan 2');
CALL snap(); CALL scan_all(); CALL delta('scan 3');
CALL snap(); CALL scan_all(); CALL delta('scan 4');
```

**Kết quả đã đo** (MySQL 8.0.42, Windows 11, instance riêng, dữ liệu tổng hợp 300.000 hàng; đã rút gọn cột):

*Buffer pool 256 MB (16.384 trang):*

| Lần | elapsed_ms | read_requests | reads (miss) | read_ahead | pages_read |
| --- | --- | --- | --- | --- | --- |
| baseline (không làm gì) | 5,7 | 6 | 0 | 0 | 0 |
| quét 1 (lạnh) | 453,5 | 11.494 | 99 | 4.480 | 4.568 |
| quét 2 | 117,2 | 6.893 | 0 | 0 | 0 |
| quét 3 | 118,2 | 6.893 | 0 | 0 | 0 |
| quét 4 | 118,9 | 6.893 | 0 | 0 | 0 |

Sau quét 1: `bp_pages_data` 5.563, `bp_pages_free` 10.814.

*Buffer pool 16 MB (1.024 trang):*

| Lần | elapsed_ms | read_requests | reads (miss) | read_ahead | read_ahead_evicted | pages_read |
| --- | --- | --- | --- | --- | --- | --- |
| baseline | 8,0 | 6 | 0 | 0 | 0 | 0 |
| quét 1 (lạnh) | 546,7 | 11.499 | 68 | 4.543 | 0 | 4.598 |
| quét 2 | 432,2 | 10.947 | 212 | 3.872 | 30 | 4.084 |
| quét 3 | 289,3 | 10.947 | 212 | 3.872 | 30 | 4.084 |
| quét 4 | 309,3 | 10.947 | 212 | 3.872 | 30 | 4.084 |

Trước quét 1: 308 trang dữ liệu (của hệ thống), 715 trang trống; sau quét 1: 1.023 trang dữ liệu, 0 trang trống.

Trung vị thời gian của các lần quét lặp (2–4): **118,2 ms** khi bảng vừa buffer pool, **309,3 ms** khi không vừa.

**Diễn giải và đối chiếu với sách.**

- Đúng tr. 76: khi bảng vừa buffer pool, quét lại không đọc trang nào từ đĩa, nhưng vẫn có 6.893 logical read request và vẫn tốn khoảng 118 ms CPU. "Cache hit" không miễn phí, nên một truy vấn quét quá nhiều hàng trong bộ nhớ vẫn có thể làm cạn CPU (tr. 75).
- Khi buffer pool nhỏ, mỗi lần quét lặp vẫn đọc 4.084 trên 4.582 trang: dữ liệu đã đọc trước đó bị đẩy ra trước khi được dùng lại. Thêm "một chút" bộ nhớ không giúp được kiểu truy cập này; chỉ khi toàn bộ bảng vừa bộ nhớ thì số đọc mới về 0.
- **Read-ahead của InnoDB** nạp phần lớn trang (4.480 trên 4.568 ở lần lạnh): khi thấy truy cập tuần tự trong một extent, InnoDB đọc trước cả extent kế tiếp (**Bổ sung ngoài sách**: ngưỡng là `innodb_read_ahead_threshold` = 56 trang). Read-ahead vẫn hoạt động dù I/O không qua cache hệ điều hành, nên câu "O_DIRECT tự động tắt read-ahead" ở tr. 88 nên hiểu là read-ahead **của hệ điều hành**.
- `Innodb_pages_read` xấp xỉ `reads + read_ahead` (ví dụ 212 + 3.872 = 4.084). `read_requests` không bằng số trang khác nhau (6.893 so với 4.582): một trang có thể được yêu cầu nhiều lần trong một lần quét. Đừng đọc `read_requests` như "số trang đã chạm".

**Dọn dẹp và khôi phục.** Không thay đổi trạng thái; chỉ tắt server bằng `mysqladmin -h127.0.0.1 -P3314 -uroot shutdown`.

---

<a id="exp2"></a>

## Thí nghiệm 2 — Working set: truy cập nóng so với truy cập rải đều

**Mục tiêu.** Kiểm chứng tr. 77: chỉ working set, không phải cả database, cần vừa bộ nhớ; và minh họa câu hỏi ở Summary (tr. 96): I/O-bound thì cần thêm I/O hay thêm bộ nhớ.

**Giả thuyết (Dự đoán).** Buffer pool 16 MB (1.024 trang):

- Mẫu **nóng** (20.000 tra cứu khóa 1..20.000, khoảng 303 trang lá cộng vài trang trong): sau khi ấm, 0 lần đọc đĩa.
- Mẫu **đều** (20.000 tra cứu khóa 1..300.000): working set là cả 4.546 trang lá, gấp khoảng 4,4 lần buffer pool; dự đoán khoảng 0,8 lần đọc đĩa mỗi tra cứu (xác suất trang đang có trong buffer pool khoảng 1/4,5).
- Mẫu đều sẽ không đẩy được vùng nóng ra, nhờ LRU hai đoạn của InnoDB (**Bổ sung ngoài sách**; chương 5 bàn `innodb_old_blocks_time` ở tr. 124).

**Chuẩn bị.** Khởi động lạnh với `--innodb-buffer-pool-size=16M`. Kiểm tra access path:

```sql
EXPLAIN FORMAT=TREE SELECT LENGTH(pad) FROM t WHERE id = 12345;
```

**Kết quả đã đo:** `-> Rows fetched before execution  (cost=0..0 rows=1)`. Tra cứu bằng hằng số trên primary key được giải ngay lúc tối ưu (truy cập const), mỗi tra cứu là một lần đi xuống B+tree.

**Các bước và lệnh đo** (lần chạy sạch, ngay sau khởi động lạnh):

```sql
USE ch04;
CALL snap(); CALL delta('C0');
CALL snap(); CALL lookups(20000, 20000);  CALL delta('H1');
CALL snap(); CALL lookups(20000, 20000);  CALL delta('H2');
CALL snap(); CALL lookups(20000, 20000);  CALL delta('H3');
SELECT IS_OLD, IS_HASHED, COUNT(*) AS pages
FROM information_schema.INNODB_BUFFER_PAGE_LRU
WHERE TABLE_NAME = '`ch04`.`t`' GROUP BY IS_OLD, IS_HASHED;
CALL snap(); CALL lookups(20000, 300000); CALL delta('U1');
CALL snap(); CALL lookups(20000, 300000); CALL delta('U2');
CALL snap(); CALL lookups(20000, 300000); CALL delta('U3');
-- lặp lại truy vấn INNODB_BUFFER_PAGE_LRU ở trên
CALL snap(); CALL lookups(20000, 20000);  CALL delta('H4');
```

**Kết quả đã đo** (đã rút gọn cột):

| Pha | Mẫu | elapsed_ms | read_requests | reads (miss) | Miss mỗi tra cứu |
| --- | --- | --- | --- | --- | --- |
| C0 | ngay sau khởi động | 8,2 | 6 | 0 | |
| H1 | nóng, lần 1 (lạnh) | 701,0 | 42.464 | 313 | 0,016 |
| H2 | nóng, lần 2 | 556,6 | 40.303 | 0 | 0 |
| H3 | nóng, lần 3 | 543,2 | 40.303 | 0 | 0 |
| U1 | đều, lần 1 | 4.951,5 | 78.281 | 18.632 | 0,93 |
| U2 | đều, lần 2 | 5.011,3 | 77.683 | 18.442 | 0,92 |
| U3 | đều, lần 3 | 4.819,5 | 77.554 | 18.433 | 0,92 |
| H4 | nóng, sau pha đều | 521,8 | 40.304 | 0 | 0 |

Read-ahead bằng 0 ở mọi pha (truy cập không tuần tự). Trang của bảng `t` trong buffer pool, theo `INNODB_BUFFER_PAGE_LRU`:

| Thời điểm | Đoạn young, có trong AHI | Đoạn young, không trong AHI | Đoạn old |
| --- | --- | --- | --- |
| Sau H3 | 304 | 2 | 0 |
| Sau U3 | 501 | 2 | 365 |

Trung vị thời gian: mẫu nóng (H2, H3, H4) **543,2 ms**; mẫu đều (U1–U3) **4.951,5 ms**, tức chậm hơn khoảng **9,1 lần**.

**Thí nghiệm phụ: adaptive hash index.** Ở một lần chạy trước đó (cũng buffer pool 16 MB), sau khi vùng nóng đã nằm trong buffer pool, tắt rồi bật lại AHI:

```sql
SET GLOBAL innodb_adaptive_hash_index = OFF;
CALL snap(); CALL lookups(20000, 20000); CALL delta('AHI OFF');   -- lặp 3 lần
SET GLOBAL innodb_adaptive_hash_index = ON;
CALL snap(); CALL lookups(20000, 20000); CALL delta('AHI ON');    -- lặp 3 lần
```

| AHI | elapsed_ms (3 lần) | read_requests mỗi đợt | read_requests mỗi tra cứu | reads (miss) |
| --- | --- | --- | --- | --- |
| OFF | 704,0 · 671,8 · 696,9 (trung vị 696,9) | 60.606 | 3,03 | 0 |
| ON lại | 584,8 · 604,5 · 622,8 (trung vị 604,5) | 41.575, rồi 40.303 | 2,08 rồi 2,02 | 0 |

**Lần chạy đầu không sạch (ghi lại cho trung thực).** Lần đầu, thí nghiệm 2 chạy ngay sau thí nghiệm 1 trên cùng instance 16 MB, nên H1 chỉ có 49 miss: phần lớn trang của vùng nóng đã nằm sẵn trong buffer pool từ các lần quét trước. Vì vậy thí nghiệm được chạy lại sau một lần khởi động lạnh; bảng trên là lần chạy sạch. Kết quả của pha đều ở lần đầu (18.130 và 18.127 miss) gần như trùng lần sạch.

**Diễn giải và đối chiếu với sách.**

- **Working set, không phải kích thước bảng** (tr. 77): cùng bảng 71,6 MB, cùng buffer pool 16 MB, mẫu nóng cần khoảng 313 trang (khoảng 5 MB) nên chạy hoàn toàn trong bộ nhớ sau lần nạp đầu; mẫu đều có working set là cả bảng nên gần như mọi tra cứu là một lần đọc đĩa.
- **Dự đoán 0,8 miss mỗi tra cứu là hơi lạc quan.** Thực đo 0,92 vì khoảng 300–500 chỗ trong buffer pool đang giữ vùng nóng; mẫu đều chỉ dùng được phần còn lại, cộng 6,7% tra cứu rơi đúng vùng nóng (khóa ≤ 20.000) thì luôn hit.
- **Thêm I/O hay thêm bộ nhớ** (tr. 96): với mẫu nóng, thêm RAM vô ích (đã 0 miss). Với mẫu đều, muốn về 0 miss phải có buffer pool chứa cả bảng; thêm một ít RAM chỉ hạ tỷ lệ miss theo tỷ lệ. Mỗi lần đọc trên SSD NVMe ở đây chỉ khoảng 0,24 ms (phần chênh khoảng 4,4 giây chia cho khoảng 18.400 lần đọc; **Bổ sung ngoài sách**, phép tính) nên thiết bị không hề bão hòa, nhưng mẫu đều vẫn chậm khoảng 9 lần. Khớp tr. 75: SSD làm truy cập đĩa "tạm ổn", không làm nó bằng RAM.
- **LRU hai đoạn bảo vệ vùng nóng** (**Bổ sung ngoài sách**): sau H3, cả 306 trang nóng nằm ở đoạn young (được truy cập lại sau hơn `innodb_old_blocks_time` = 1000 ms). Pha đều đọc khoảng 3.700 trang mỗi giây; mỗi trang mới chỉ sống trong đoạn old một phần giây, chưa kịp được thăng hạng thì đã bị đẩy ra. Vì vậy H4 vẫn 0 miss. Lưu ý: chương 5 (tr. 124) mô tả cơ chế này với tên hai đoạn bị đảo và nói mặc định của `innodb_old_blocks_time` là 0; trên 8.0.42 giá trị mặc định đo được là **1000**.
- **Logical read mỗi tra cứu cho biết chiều cao B+tree** (**Bổ sung ngoài sách**): khi tắt AHI, mỗi tra cứu cần đúng khoảng 3 lần lấy trang (root, trang trong, trang lá), khớp với cây 3 tầng của 4.546 trang lá. Bật AHI, con số giảm còn khoảng 2 và thời gian giảm khoảng 13%. Đây là bằng chứng đo được cho adaptive hash index, tính năng mà tr. 17 nói sẽ bàn ở chương 4 nhưng chương này không bàn (xem [kết quả kiểm chứng trong ghi chú chương 4](../chapter-04.md#kiem-chung-roadmap)). Tôi không tách được phần 1 lần lấy trang còn lại khi có AHI đến từ đâu.

**Dọn dẹp và khôi phục.** Thí nghiệm phụ đã bật lại `innodb_adaptive_hash_index = ON`. Tắt server.

---

<a id="exp3"></a>

## Thí nghiệm 3 — Cache trì hoãn ghi nhưng không khử được ghi

**Mục tiêu.** Kiểm chứng tr. 76–77 (many writes, one flush; I/O merging; WAL) và tr. 85 (chi phí ghi đồng bộ ở mỗi commit).

**Giả thuyết (Dự đoán).**

- W1, 5.000 `UPDATE` **cùng một hàng**, mỗi cái một commit: bảng chỉ có 1 trang bẩn, phải flush khoảng 1 trang. Redo log vẫn tăng theo số thay đổi.
- W2, 5.000 `UPDATE` vào **5.000 hàng rải rác**, mỗi cái một commit: hàng nghìn trang bẩn, phải flush hàng nghìn trang.
- W3, giống W2 nhưng trong **một** transaction: số trang bẩn như W2, nhưng số lần `fsync` và thời gian giảm mạnh.

**Chuẩn bị.** Khởi động với buffer pool 256 MB, chạy `CALL scan_all()` để nạp bảng vào bộ nhớ (để `UPDATE` không phải đọc đĩa), `CALL wait_clean(60)`.

**Các bước và lệnh đo** (mỗi kịch bản lặp 3 vòng, theo thứ tự W1, W2, W3):

```sql
CALL wait_clean(60);
SELECT SUM(COUNT_WRITE), SUM(COUNT_MISC) INTO @bw0, @bm0
FROM performance_schema.file_summary_by_event_name
WHERE EVENT_NAME = 'wait/io/file/sql/binlog';
CALL snap();
CALL upd_same(5000);                                  -- W1
-- hoặc: CALL upd_spread(5000);                        -- W2
-- hoặc: START TRANSACTION; CALL upd_spread(5000); COMMIT;   -- W3
CALL delta('ngay sau workload');
SELECT SUM(COUNT_WRITE) - @bw0 AS binlog_writes, SUM(COUNT_MISC) - @bm0 AS binlog_misc
FROM performance_schema.file_summary_by_event_name
WHERE EVENT_NAME = 'wait/io/file/sql/binlog';
CALL wait_clean(60);
CALL delta('sau khi flush het');   -- cộng dồn từ snap()
```

Để biết trang bẩn thuộc về đâu, chạy thêm một vòng W1 và một vòng W3 rồi ngay sau workload:

```sql
SELECT CASE WHEN p.TABLE_NAME IS NOT NULL THEN p.TABLE_NAME
            WHEN ts.NAME IS NOT NULL THEN ts.NAME
            ELSE CONCAT('space ', p.SPACE) END AS owner,
       p.PAGE_TYPE, COUNT(*) AS dirty_pages
FROM information_schema.INNODB_BUFFER_PAGE p
LEFT JOIN information_schema.INNODB_TABLESPACES ts ON ts.SPACE = p.SPACE
WHERE p.OLDEST_MODIFICATION > 0
GROUP BY owner, p.PAGE_TYPE ORDER BY dirty_pages DESC;
```

**Kết quả đã đo** (đã rút gọn; "ngay sau" là lúc workload vừa xong, "tổng" là cộng dồn tới khi không còn trang bẩn):

| Kịch bản, vòng | Thời gian workload (ms) | Trang bẩn ngay sau | Trang flush tổng | Redo ghi tổng (byte) | `Innodb_os_log_fsyncs` tổng | binlog write / misc (ngay sau) |
| --- | --- | --- | --- | --- | --- | --- |
| W1 cùng hàng, 5.000 commit, vòng 1 | 10.858,8 | 524 | 776 | 5.076.992 | 8.191 | 5.000 / 5.000 |
| W1 vòng 2 | 10.638,4 | 588 | 1.200 | 5.161.984 | 8.130 | 5.000 / 5.000 |
| W1 vòng 3 | 10.292,9 | 659 | 1.257 | 5.195.264 | 8.186 | 5.000 / 5.000 |
| W2 rải rác, 5.000 commit, vòng 1 | 11.453,6 | 4.380 | 5.225 | 5.007.360 | 8.206 | 5.000 / 5.000 |
| W2 vòng 2 | 11.078,1 | 4.196 | 5.349 | 4.978.688 | 8.102 | 5.000 / 5.000 |
| W2 vòng 3 | 10.414,2 | 4.205 | 5.350 | 4.959.744 | 8.154 | 5.000 / 5.000 |
| W3 rải rác, 1 transaction, vòng 1 | 431,7 | 4.121 | 4.121 | 439.808 | 56 | 315 / 1 |
| W3 vòng 2 | 329,9 | 4.121 | 4.121 | 434.176 | 67 | 315 / 1 |
| W3 vòng 3 | 302,3 | 4.121 | 4.121 | 432.640 | 68 | 315 / 1 |

Ở mọi dòng, `Innodb_dblwr_pages_written` bằng đúng số trang flush (mỗi trang được ghi thêm một lần vào vùng doublewrite). Cột "tổng" của redo và `fsync` gồm cả phần phát sinh trong lúc chờ flush hết trang; với W3, số `fsync` redo là 28–52 ngay sau workload và 56–68 khi đã sạch. Với W1 và W2, gần như toàn bộ `fsync` xảy ra trong lúc chạy workload (ví dụ W1 vòng 1: 8.187 ngay sau, 8.191 tổng).

Trung vị thời gian workload: W1 **10.638,4 ms**, W2 **11.078,1 ms**, W3 **329,9 ms**.

Trang bẩn theo chủ sở hữu (vòng phụ):

| Sau W1 (cùng hàng) | Trang bẩn | | Sau W3 (rải rác, 1 transaction) | Trang bẩn |
| --- | --- | --- | --- | --- |
| `innodb_undo_001`, `innodb_undo_002`: UNDO_LOG | 192 + 192 | | bảng `` `ch04`.`t` ``: INDEX | **4.105** |
| `innodb_undo_001`, `innodb_undo_002`: SYSTEM | 128 + 128 | | `innodb_undo_002`: UNDO_LOG | 11 |
| undo tablespace: INODE, FILE_SPACE_HEADER | 14 | | undo và system: các trang khác | 4 |
| system tablespace: TRX_SYSTEM | 1 | | | |
| bảng `` `ch04`.`t` ``: INDEX | **1** | | | |

**Diễn giải và đối chiếu với sách.**

- **Many writes, one flush** (tr. 77): 5.000 lần sửa cùng một hàng chỉ làm **1** trang của bảng bẩn. Đúng ví dụ bộ đếm của sách.
- **Gom ghi có giới hạn**: khi thay đổi rải trên 4.105 trang, phải flush 4.105 trang. Cache chỉ trì hoãn, không khử được ghi (tr. 76).
- **Ghi đồng bộ là chi phí chính của commit nhỏ**: W2 và W3 làm cùng 5.000 thay đổi trên cùng các trang, nhưng W3 nhanh hơn khoảng **34 lần** (11.078 so với 330 ms trung vị) vì chỉ có một commit: vài chục `fsync` redo so với khoảng 8.100, và 1 thao tác đồng bộ binlog so với 5.000. Đây là loại ghi mà sách nói hưởng lợi lớn từ write-back cache có pin (tr. 77, 85).
- **Redo log ghi nhiều hơn mức "cần"** khi commit dày (**Bổ sung ngoài sách**): W2 ghi khoảng 5,0 MB redo, W3 chỉ khoảng 0,44 MB cho cùng 5.000 thay đổi. Mỗi commit thêm bản ghi riêng của transaction và buộc ghi khối log đang dở, nên khối đó bị ghi lại nhiều lần.
- **Sắc thái InnoDB mà sách không bàn** (**Bổ sung ngoài sách**): ở W1, ngoài 1 trang của bảng còn khoảng 650 trang trong undo tablespace bị bẩn. Mỗi transaction được gán một rollback segment theo vòng (2 undo tablespace × 128 rollback segment = 256 trang header, khớp 128 + 128 trang SYSTEM), và ghi undo log của nó. Vì vậy nhiều transaction nhỏ tạo ra I/O nền mà một transaction lớn không có.
- **Write amplification ở tầng database** (liên hệ tr. 79): doublewrite làm mỗi trang flush được ghi hai lần.

**Dọn dẹp và khôi phục.** `wait_clean` luôn trả `innodb_max_dirty_pages_pct` về 90 và `innodb_max_dirty_pages_pct_lwm` về 10 (đã kiểm tra sau vòng cuối: 0 trang bẩn, 90 và 10). Cột `grp` đã bị tăng; nếu cần dữ liệu gốc, nạp lại bảng. Tắt server.

---

<a id="exp4"></a>

## Thí nghiệm 4 — Một truy vấn dùng một CPU

**Mục tiêu.** Kiểm chứng tr. 75–76: mỗi truy vấn chỉ dùng một CPU; latency cần CPU nhanh, throughput cần nhiều CPU; quá nhiều truy vấn song song làm cạn CPU.

**Giả thuyết (Dự đoán).** Một truy vấn CPU-bound (quét bảng đã nằm trong buffer pool) có latency phía server gần như không đổi khi số client ≤ số lõi rảnh; vượt quá thì latency tăng vì phải chờ CPU.

**Chuẩn bị.** Buffer pool 256 MB, chạy `CALL ch04.scan_all()` hai lần để bảng nằm trong bộ nhớ. Kiểm tra `Innodb_buffer_pool_reads` trước và sau thí nghiệm: cùng là 1.130, tức không có đọc đĩa trong lúc đo.

**Các bước và lệnh đo.** Dùng `mysqlslap` (có sẵn trong thư mục `bin` của MySQL), tổng cộng 48 truy vấn chia đều cho các client; đo latency phía server bằng statement digest của Performance Schema; lặp 3 vòng.

```text
mysql -h127.0.0.1 -P3314 -uroot -e "TRUNCATE performance_schema.events_statements_summary_by_digest"
mysqlslap -h127.0.0.1 -P3314 -uroot --create-schema=ch04 --no-drop --query="SELECT SUM(LENGTH(pad)) FROM t" --concurrency=<c> --iterations=1 --number-of-queries=48 --csv
```

```sql
SELECT COUNT_STAR, ROUND(AVG_TIMER_WAIT/1e9,1) AS avg_ms,
       ROUND(MIN_TIMER_WAIT/1e9,1) AS min_ms, ROUND(MAX_TIMER_WAIT/1e9,1) AS max_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE 'SELECT SUM ( `LENGTH` ( `pad` ) ) FROM `t`%';
```

**Kết quả đã đo** (latency trung bình phía server của mỗi truy vấn, ms; mỗi ô là một vòng 48 truy vấn):

| Số client | Vòng 1 | Vòng 2 | Vòng 3 | **Trung vị** |
| --- | --- | --- | --- | --- |
| 1 | 102,4 | 97,3 | 94,1 | **97,3** |
| 2 | 110,3 | 98,5 | 103,3 | **103,3** |
| 4 | 106,3 | 100,0 | 116,8 | **106,3** |
| 8 | 113,5 | 128,5 | 123,0 | **123,0** |
| 16 | 221,8 | 189,1 | 171,5 | **189,1** |

Ở một vòng với 8 client, digest ghi 47 thay vì 48 truy vấn (có lẽ một truy vấn chưa kịp được tổng hợp khi đọc). Cột `CPU_TIME` của Performance Schema (có từ 8.0.28) luôn bằng 0 trên máy Windows này nên không dùng được.

Tổng thời gian phía client do `mysqlslap` báo **rất nhiễu** trên máy này: với 1 client khoảng 9,8–10,3 giây cho 48 truy vấn (khoảng 200 ms mỗi truy vấn, gấp đôi latency phía server), với 8 client lúc 0,75 giây lúc 1,8 giây. Tôi không giải thích được phần chênh ở phía client nên **không** dùng con số này để kết luận về throughput.

**Diễn giải và đối chiếu với sách.**

- Latency của **một** truy vấn không giảm khi thêm client: khoảng 97 ms với 1 client, khoảng 103–106 ms với 2–4 client. Truy vấn chạy trên một lõi; lõi khác rảnh cũng không giúp được nó (tr. 76).
- Khi số client tiến tới số lõi thật (8) trong khi máy còn chạy tiến trình khác, latency bắt đầu tăng; ở 16 client (bằng số luồng phần cứng, hai luồng SMT chia một lõi) latency gần gấp đôi. Đây là cách (1) của cạn CPU: quá nhiều truy vấn chạy song song (tr. 75).
- Hệ quả cho chọn CPU (tr. 76): muốn mỗi truy vấn nhanh hơn thì cần lõi nhanh hơn; muốn phục vụ nhiều truy vấn đồng thời mà latency không tăng thì cần đủ lõi cho mức concurrency đó (Summary, tr. 96).
- Giới hạn của thí nghiệm: laptop có tăng tốc xung nhịp động và tải nền từ tiến trình khác, nên các con số chỉ cho thấy xu hướng.

**Dọn dẹp và khôi phục.** `--no-drop` bảo đảm `mysqlslap` không xóa schema `ch04`. Tắt server.

---

<a id="exp5"></a>

## Thí nghiệm 5 — Diễn tập đọc `vmstat`, `iostat`, `mpstat` trên Linux (**Chưa thực thi**)

**Lý do chưa thực thi.** Máy thực hành chạy Windows; `vmstat`, `iostat`, `mpstat` (gói procps và sysstat) không có, và quy định của dự án không cho dựng máy ảo hay container. Dưới đây là quy trình để tự chạy trên một **máy ảo Linux dùng một lần**, không phải production.

**Mục tiêu.** Luyện cách phân biệt CPU-bound, I/O-bound, nút thắt một lõi và áp lực bộ nhớ bằng chỉ số hệ điều hành (tr. 90–95), dùng chính MySQL làm nguồn tải có kiểm soát.

**Giả thuyết (Dự đoán)** cho từng kịch bản:

| Kịch bản | Cách tạo tải | `vmstat` dự đoán | `iostat` / `mpstat` dự đoán |
| --- | --- | --- | --- |
| A. CPU-bound | Buffer pool lớn hơn dữ liệu, cache ấm; `mysqlslap` chạy quét bảng với số client gấp đôi số CPU | `r` > số CPU, `us` cao, `id` gần 0, `wa` gần 0, `bi` gần 0 | `iostat`: `r/s` thấp; `mpstat`: mọi CPU bận |
| B. I/O-bound đọc | Buffer pool nhỏ (ví dụ 64 MB), dữ liệu vài GB; thủ tục `lookups` rải đều với 8–16 client | `wa` tăng, `b` > 0, `bi` cao | `r/s` cao, `r_await` và hàng đợi tăng theo số client; `r/s` xấp xỉ tốc độ tăng của `Innodb_buffer_pool_reads` |
| C. Nút thắt một lõi | Một truy vấn nặng, một client, trên máy nhiều lõi | `us` khoảng 100/N phần trăm, `r` khoảng 1 | `mpstat`: một CPU gần 100%, các CPU khác rảnh |
| D. Áp lực bộ nhớ | **Chỉ trên VM dùng một lần**: cấp RAM ít, đặt buffer pool lớn hơn RAM khả dụng, chạy quét | `si`, `so` khác 0 và thành đợt; latency nhảy vọt | Có thể thấy OOM killer trong `dmesg` |
| E. Ghi đồng bộ | `upd_spread` với một commit mỗi hàng, so với gom trong một transaction | `bo` thấp nhưng commit chậm | `w/s` và `w_await`; so `sync_binlog=1` với `0` (chỉ trong môi trường thử) |

**Các bước và lệnh đo.**

1. Dựng MySQL 8.0 trên VM Linux, nạp dữ liệu bằng SQL ở phần chuẩn bị (tăng số hàng cho kịch bản B, D).
2. Ở ba terminal, chạy `vmstat 5`, `iostat -dxk 5` (bản sysstat mới có thể dùng `iostat -dx 5`), `mpstat -P ALL 5`. **Bỏ dòng đầu** của mỗi công cụ (trung bình từ lúc khởi động, tr. 92).
3. Song song, mỗi 5 giây đọc `Innodb_buffer_pool_reads`, `Innodb_buffer_pool_read_requests`, `Innodb_os_log_fsyncs` để đối chiếu tầng MySQL với tầng hệ điều hành.
4. Mỗi kịch bản ghi lại ít nhất 6 mẫu ở trạng thái ổn định.
5. Với `iostat`, tính Little's law hai cách như ở [ghi chú chương, mục 2.7.3](../chapter-04.md#little-law): IOPS × `await` (so với độ dài hàng đợi) và IOPS × `svctm` nếu cột này còn (so với `%util`).

**Tiêu chí đánh giá.** Mỗi kịch bản phải "nhận diện được" chỉ từ chỉ số, theo bảng dấu hiệu ở [mục 5 của ghi chú](../chapter-04.md#takeaways). Nếu kịch bản B mà `%util` gần 100% trong khi `r_await` vẫn thấp và hàng đợi ngắn, đó là ví dụ sống của nhận xét tr. 95.

**Kết quả.** **Chưa thực thi.** Không có số liệu.

**Dọn dẹp và khôi phục.** Kịch bản D có thể làm VM treo hoặc mất SSH; chỉ chạy trên VM có console truy cập được và xóa VM sau khi xong. Trả `sync_binlog` về 1 sau kịch bản E.

---

## Dọn dẹp chung

- Instance cổng 3314 đã được tắt bằng `mysqladmin -h127.0.0.1 -P3314 -uroot shutdown` sau mỗi thí nghiệm; đã kiểm tra không còn tiến trình lắng nghe ở cổng 3314.
- Datadir vẫn nằm trong thư mục scratchpad (`hpm\mysql\ch04\ws\data`); xóa thư mục đó là gỡ sạch toàn bộ thí nghiệm. Muốn xóa dữ liệu mà giữ instance: `DROP DATABASE ch04;`.
