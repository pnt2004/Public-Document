# Đáp án — Chương 5: Tối ưu thiết lập của server

[← Về ghi chú chương 5](../chapter-05.md)

> "tr. N" là số trang in; trang PDF = N + 22. Phần ghi **Bổ sung ngoài sách** là kiến thức hay phép tính thêm của người viết ghi chú; **Kết quả đã đo** lấy từ [bài thực hành](../experiments/chapter-05.md) trên MySQL 8.0.42 (Windows 11, instance thử nghiệm). Hãy tự trả lời trước khi đọc.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — `SET GLOBAL` có làm báo cáo nhanh lên ngay?

*Lúc 10:00 một DBA chạy `SET GLOBAL sort_buffer_size = 8M` để chữa một báo cáo chậm. Ứng dụng dùng connection pool giữ kết nối hàng giờ. Dự đoán báo cáo có nhanh lên ngay không, vì sao, và cách làm đúng hơn là gì?*

**Dự đoán: không nhanh lên ngay.** `sort_buffer_size` có giá trị session. Giá trị session được **khởi tạo từ global lúc kết nối được tạo**; đổi global không đổi session hiện tại hay bất kỳ session nào đang mở (hộp Mẹo, tr. 102). Báo cáo chạy trên kết nối đã mở từ trước 10:00 vẫn dùng giá trị cũ, cho tới khi pool mở kết nối mới.

**Kết quả đã đo** (TN1): kết nối B mở trước khi kết nối A chạy `SET GLOBAL sort_buffer_size = 4194304`; sau đó B vẫn thấy 1 MiB, và chính A cũng vẫn 1 MiB; chỉ kết nối mới C nhận 4 MiB.

**Cách đúng hơn.**

1. Trước hết chẩn đoán: báo cáo chậm có thật vì sort tràn ra đĩa không? Sách nhấn mạnh bắt đầu từ truy vấn và response time (tr. 100).
2. Nếu đúng là cần sort buffer lớn, tăng **ở session** của báo cáo, ngay trước truy vấn (tr. 104). Tăng global cho mọi kết nối là lãng phí vì đây là buffer theo kết nối.
3. Sau mỗi thay đổi global, kiểm `SHOW GLOBAL VARIABLES` (tr. 102), và nhớ sửa file cấu hình nếu muốn giữ (tr. 102).

[về mục 2.1](../chapter-05.md#s2-1)

---

<a id="q2-1-2"></a>

### Câu 2.1.2 — `SET GLOBAL ... = DEFAULT` trả về giá trị nào?

*Server được khởi động với `--sort_buffer_size=1M`. Sau một loạt thử nghiệm, DBA chạy `SET GLOBAL sort_buffer_size = DEFAULT` để "trả về như lúc khởi động". Giá trị nhận được là gì? Vì sao sách khuyên không dùng `DEFAULT` cho biến global?*

**Giá trị nhận được là giá trị biên dịch sẵn**, không phải 1 MiB truyền ở dòng lệnh, cũng không phải giá trị trong file cấu hình (tr. 102–103). **Kết quả đã đo** (TN1): 262144 byte.

**Vì sao sách khuyên tránh.** Với biến session, `DEFAULT` có nghĩa "lấy giá trị global", rất tiện để trả session về trạng thái lúc mới kết nối. Với biến global, `DEFAULT` "có lẽ không làm điều bạn muốn" (tr. 102–103): người ta thường nghĩ nó khôi phục trạng thái lúc khởi động, nhưng thật ra nó bỏ qua mọi thiết lập từ file và dòng lệnh. Muốn khôi phục, hãy đặt lại đúng con số cũ (đọc từ file cấu hình hay từ ghi chép trước khi đổi).

**Bổ sung ngoài sách.** `performance_schema.variables_info` giúp biết giá trị ban đầu đến từ đâu: trước khi đổi, `sort_buffer_size` có nguồn `COMMAND_LINE`; sau `SET GLOBAL ... = DEFAULT`, nguồn thành `DYNAMIC` (đo được trong TN1).

[về mục 2.1](../chapter-05.md#s2-1)

---

<a id="q2-1-3"></a>

### Câu 2.1.3 — Ba cách tăng `max_connections`

*So sánh ba cách tăng `max_connections`: (a) `SET GLOBAL` rồi sửa `my.cnf`; (b) `SET PERSIST`; (c) chỉ sửa `my.cnf`. Điều gì xảy ra ngay lúc đó và sau khi khởi động lại với mỗi cách? Rủi ro vận hành của (b) là gì?*

| Cách | Ngay lúc đó | Sau khi khởi động lại |
| --- | --- | --- |
| (a) `SET GLOBAL` + sửa `my.cnf` | Có hiệu lực ngay (biến global, dynamic) | Giữ, vì server đọc lại `my.cnf` |
| (b) `SET PERSIST` | Có hiệu lực ngay, và được ghi xuống đĩa | Giữ (tr. 103) |
| (c) Chỉ sửa `my.cnf` | **Không có gì xảy ra**: sửa file không tự đổi giá trị lúc chạy (tr. 104) | Có hiệu lực |

Cách (a) là quy trình trước 8.0: phải làm trên từng instance và tự giữ hai nơi đồng bộ (tr. 103). Cách (c) dễ gây hiểu lầm "đã đổi rồi" trong khi server vẫn chạy giá trị cũ.

**Rủi ro của (b)** (**Bổ sung ngoài sách**).

- Giá trị nằm trong `mysqld-auto.cnf` trong datadir, **không** nằm trong `my.cnf` mà đội đưa vào version control (tr. 104). Người đọc `my.cnf` sẽ hiểu sai cấu hình thật; hai nguồn có thể mâu thuẫn.
- **Kết quả đã đo** (TN1): khởi động lại với `--no-defaults` thì server bỏ qua `mysqld-auto.cnf`, `max_connections` về 151 dù đã persist 300. Ai khởi động "sạch" để gỡ lỗi sẽ âm thầm mất cấu hình đã persist.
- Cách giảm rủi ro: chọn một kênh chính; nếu dùng `SET PERSIST`, đưa `performance_schema.persisted_variables` vào quy trình kiểm tra cấu hình.

[về mục 2.1](../chapter-05.md#s2-1)

---

<a id="q2-1-4"></a>

### Câu 2.1.4 — Buffer theo kết nối và cách cấp phát

*Vì sao sách khuyên không tăng giá trị global của biến theo kết nối như `read_buffer_size`? `read_rnd_buffer_size` khác `read_buffer_size` ở điểm nào về cách cấp phát, và điều đó đổi mức rủi ro ra sao?*

**Vì nhân lên theo số kết nối.** Giá trị global là mặc định cho **mọi** session mới. Một số buffer được cấp **nguyên khối** khi cần, dù truy vấn không dùng hết, nên một giá trị global lớn có thể là "lãng phí khổng lồ" khi nhân với hàng trăm kết nối (tr. 104). Kèm theo là rủi ro hết bộ nhớ hay swap (tr. 104). Cách đúng: tăng ở session cho truy vấn cần (tr. 104).

**Khác biệt cấp phát** (tr. 104):

- `read_buffer_size`: không cấp cho tới khi truy vấn cần, nhưng khi cần thì cấp **toàn bộ** giá trị đã đặt.
- `read_rnd_buffer_size`: không cấp cho tới khi cần, và chỉ cấp **lượng cần dùng**; sách nói tên `max_read_rnd_buffer_size` mới mô tả đúng.

**Hệ quả với rủi ro** (**Bổ sung ngoài sách**, suy ra từ mô tả của sách). Với biến kiểu "cấp nguyên khối", chi phí bộ nhớ gần bằng giá trị đặt mỗi khi buffer được dùng, nên tăng global là rủi ro trực tiếp. Với biến kiểu "tối đa", giá trị lớn chỉ là trần; chi phí thật phụ thuộc truy vấn. Rủi ro thấp hơn nhưng không bằng 0: trần cao cho phép một truy vấn xấu dùng nhiều bộ nhớ. Bài học chung của tr. 103: không đoán được hành vi từ tên biến, phải đọc manual.

[về mục 2.1](../chapter-05.md#s2-1)

---

<a id="q2-1-5"></a>

### Câu 2.1.5 — Vì sao benchmark sáng thứ Hai có thể đánh lừa?

*Một thay đổi cấu hình được kết luận là "tốt hơn" nhờ benchmark chạy sáng thứ Hai. Theo sách, vì sao kết luận đó có thể sai, và nên quan sát gì để đánh giá?*

**Vì sao có thể sai** (tr. 104–105):

- **Benchmark không phải workload thật.** Sách nói benchmark "không đủ vì chúng không thật" (tr. 104).
- **Workload đổi theo thời gian.** Tác giả từng thấy người ta tưởng cấu hình mới nhanh hơn, nhưng tổng thể lại chậm hơn vì một workload khác vào giờ khác hay ngày khác (tr. 104). Cấu hình tốt cho giữa buổi sáng có thể không đúng cho giữa buổi chiều (tr. 105).
- **Nếu không đo hiệu năng thật, có thể làm hại mà không biết** (tr. 104).

**Nên quan sát gì.**

1. **SLO** của dịch vụ ([Chương 2](../chapter-02.md#sli-slo-sla)) trước và sau thay đổi, qua đủ các chu kỳ tải (tr. 104).
2. **Status variable theo thời gian**, nhiều snapshot ở cao điểm và thấp điểm (tr. 108).
3. **Version control** cho file cấu hình để nối một lần vi phạm SLO với đúng thay đổi (tr. 104).
4. Tối ưu cho **cao điểm** rồi dừng ở mức "đủ tốt" (tr. 105).

[về mục 2.1](../chapter-05.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Hit ratio tăng mà I/O không giảm

*Dùng một ví dụ cụ thể, giải thích vì sao hit ratio của buffer pool có thể tăng từ 65% lên 97% mà hệ thống không đọc ít trang từ đĩa hơn. Chỉ số nào phản ánh chi phí tốt hơn?*

**Ví dụ (Kết quả đã đo, TN3, buffer pool 128 MiB).** Một lần quét toàn bảng khoảng 300 MB: 35.654 lần đọc logic, 12.468 trang đọc từ đĩa, hit ratio 65,0%. Cùng lần quét đó nhưng trong cùng khoảng đo có thêm 200.000 lookup vào 1000 id "nóng": 437.672 lần đọc logic, 11.443 trang đọc từ đĩa, hit ratio **97,4%**.

**Vì sao** (**Bổ sung ngoài sách**, minh họa cho lập luận tr. 105–106). Hit ratio = 1 − (đọc đĩa / đọc logic). Các lookup nóng thêm rất nhiều lần đọc logic rẻ vào mẫu số nhưng gần như không thêm đọc đĩa. Tỉ lệ đẹp lên trong khi phần tốn thời gian (đọc đĩa của lần quét) không đổi. Đây đúng là ý của chú thích 2 (tr. 106): có thể tạo ra hit ratio bất kỳ, bất kể hệ thống chạy tệ thế nào.

**Chỉ số tốt hơn.** Số trang đọc từ đĩa **mỗi giây** (`Innodb_pages_read`, `Innodb_buffer_pool_reads` lấy hiệu số theo thời gian), độ trễ I/O, và cuối cùng là **response time của truy vấn** (tr. 100). Lưu ý thêm (đo được): `Innodb_buffer_pool_reads` không đếm trang nạp bằng read-ahead, nên nên xem cả `Innodb_pages_read`.

[về mục 2.2](../chapter-05.md#s2-2)

---

<a id="q2-2-2"></a>

### Câu 2.2.2 — Tương quan ngẫu nhiên giữa kích thước cache và hit ratio

*Khi tăng buffer pool, hit ratio thường tăng theo. Vì sao sách gọi đó là tương quan ngẫu nhiên, không phải căn cứ để định cỡ cache? Vậy định cỡ dựa vào gì?*

**Theo sách** (tr. 105). Hit ratio **không liên quan** tới việc cache quá lớn hay quá nhỏ: (1) nó phụ thuộc **workload**, có workload không cache được dù cache lớn mấy; (2) bản thân cache hit "vô nghĩa". Đôi khi cache nhỏ thì hit rate thấp và tăng cache thì hit rate tăng, nhưng đó là tương quan ngẫu nhiên, không nói gì về hiệu năng hay kích thước đúng.

**Minh họa (Kết quả đã đo, TN3).** Ở **cùng** buffer pool 128 MiB, 20.000 lookup trên 1000 id nóng đạt 99,97%, còn 20.000 lookup ngẫu nhiên trên 1 triệu id đạt 84,2%. Tăng lên 512 MiB thì cả lần quét lẫn lookup ngẫu nhiên đạt gần 100%, vì toàn bảng đã nằm vừa. Con số phần trăm không cho biết trước 512 MiB là đủ hay 256 MiB là đủ; điều cho biết là kích thước **working set** so với pool.

**Định cỡ dựa vào** (tổng hợp từ tr. 109–110 và [Chương 4](../chapter-04.md)):

- **Working set** (dữ liệu và index hay dùng) so với bộ nhớ có thể cấp, sau khi chừa cho kết nối và hệ điều hành (tr. 109).
- Không cần lớn hơn nhiều so với tổng dữ liệu (tr. 110).
- Kiểm chứng bằng tốc độ đọc đĩa và response time trước và sau (**Bổ sung ngoài sách**), không bằng hit ratio.

[về mục 2.2](../chapter-05.md#s2-2)

---

<a id="q2-2-3"></a>

### Câu 2.2.3 — Công thức bộ nhớ khi crash

*Công thức "bộ nhớ tối đa" mà MySQL in ra khi crash sai ở đâu? Bạn quan sát bộ nhớ thật bằng cách nào?*

**Sai ở đâu** (tr. 106). Công thức có từ "thời xa xưa"; không đáng tin, thậm chí không hữu ích để biết bộ nhớ tối đa trong trường hợp xấu nhất. Các biến thể trên internet thêm yếu tố nhưng vẫn sai. Lý do gốc: **không thể đặt cận trên** cho bộ nhớ MySQL dùng, vì MySQL không phải server kiểm soát chặt việc cấp phát bộ nhớ. Ví dụ ngay trong chương: một truy vấn có thể có nhiều join buffer (tr. 102); prepared statement và data dictionary tiêu thụ khó đoán (tr. 109); có buffer cấp nguyên khối, có buffer cấp theo nhu cầu (tr. 104).

**Quan sát bộ nhớ thật** (**Bổ sung ngoài sách**).

- Instrument bộ nhớ của Performance Schema, theo thread và theo loại cấp phát ([Chương 3](../chapter-03.md#memory-usage), tr. 64), và các view tổng hợp của `sys`.
- Bộ nhớ tiến trình ở mức hệ điều hành và dấu hiệu swap (tr. 109).
- Từ 8.0.28 có thể đặt giới hạn cho bộ nhớ do kết nối người dùng cấp phát (`connection_memory_limit`, `global_connection_memory_limit`); chúng không bao phủ mọi bộ nhớ của server, nên không thay được việc đo.

[về mục 2.2](../chapter-05.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — Chép nguyên file mẫu cho 8.0.42

*Dự đoán điều gì xảy ra nếu chép nguyên file mẫu của sách (đã điền `<value>`) cho MySQL 8.0.42 trên Linux, và trên Windows. Bài học rút ra từ chú thích 3 là gì?*

**Kết quả đã đo trên Windows** (TN2, `--validate-config`): server **từ chối** cấu hình. `innodb_flush_method = O_DIRECT` không hợp lệ trên Windows, làm InnoDB không nạp được tùy chọn; `log_slow_queries` là `unknown variable`. Exit code 1.

**Dự đoán trên Linux** (chưa kiểm vì máy là Windows). `O_DIRECT` hợp lệ trên Linux, nhưng `log_slow_queries` vẫn là biến không tồn tại (đã bị loại bỏ, theo tài liệu MySQL từ 5.6), nên server vẫn từ chối khởi động. Ngoài ra, `innodb_log_file_size` được chấp nhận nhưng là tham số deprecated từ 8.0.30: server ghi cảnh báo và dùng nó để tính `innodb_redo_log_capacity` (đã đo trên Windows: cảnh báo MY-013907).

**Bài học** (chú thích 3, tr. 107). Các phiên bản MySQL bỏ, deprecate hay đổi tùy chọn; phải đối chiếu với tài liệu của **đúng phiên bản**. Thực hành tốt (**Bổ sung ngoài sách**): kiểm file bằng `mysqld --validate-config` (có từ 8.0.16) trước khi khởi động lại; đọc error log sau khi khởi động để bắt cảnh báo deprecated. Sách cũng nhắc nguyên tắc chung: đọc manual và thử cẩn thận mọi thay đổi (tr. 100).

[về mục 2.3](../chapter-05.md#s2-3)

---

<a id="q2-3-2"></a>

### Câu 2.3.2 — `innodb_dedicated_server`

*`innodb_dedicated_server` làm gì? Nêu một tình huống sách cho là rất có ích và một tình huống bạn không nên dùng nó. Ở 8.0.30 trở lên, danh sách biến nó quản lý khác sách thế nào?*

**Làm gì** (tr. 108). Xem server có bao nhiêu bộ nhớ và tự đặt các biến InnoDB phù hợp cho server database chuyên dụng; theo sách là `innodb_buffer_pool_size`, `innodb_log_file_size`, `innodb_log_files_in_group`, `innodb_flush_method`. Thường dùng 50–75% RAM cho buffer pool (tr. 109). Summary nói nó lo khoảng 90% cấu hình hiệu năng (tr. 124).

**Rất có ích** (tr. 108): trên cloud, VM 128 GB khởi động lại thành 256 GB; MySQL tự cấu hình lại, không cần sửa file.

**Không nên dùng** (**Bổ sung ngoài sách**, suy ra từ giả định "chuyên dụng" và tr. 109): host còn chạy dịch vụ khác dùng nhiều bộ nhớ (ứng dụng, cache, agent nặng), hoặc nhiều instance MySQL chung một máy. 75% RAM cho một instance lúc đó dễ dẫn tới swap, mà swap là dấu hiệu hệ điều hành thiếu bộ nhớ (tr. 109).

**Khác ở 8.0.30+** (**Bổ sung ngoài sách**, đã đối chiếu `mysqld --verbose --help` của 8.0.42): tùy chọn co giãn `innodb_buffer_pool_size` và **`innodb_redo_log_capacity`**, và đặt `innodb_flush_method = O_DIRECT_NO_FSYNC` nếu nền tảng hỗ trợ. Hai biến `innodb_log_file_size`, `innodb_log_files_in_group` đã deprecated từ 8.0.30. Theo tài liệu MySQL, biến nào bạn đặt tường minh thì giá trị tường minh được dùng.

[về mục 2.3](../chapter-05.md#s2-3)

---

<a id="q2-3-3"></a>

### Câu 2.3.3 — Vì sao ghi rõ socket và pid file?

*Vì sao sách khuyên ghi rõ đường dẫn socket và pid file thay vì để mặc định biên dịch sẵn, dù không khuyên chọn vị trí khác?*

Hai lý do (tr. 107–108):

1. **Bug.** Một số phiên bản MySQL có bug khi để socket và pid file theo mặc định biên dịch sẵn.
2. **Ổn định qua nâng cấp.** Sách nói rõ không khuyên chọn chỗ khác; điều quan trọng là `my.cnf` **nêu rõ** vị trí, để chúng không đổi và không làm hỏng mọi thứ khi bạn nâng cấp server.

Liên hệ: phần `[client]` của file phải khớp các giá trị đó để `mysql`, `mysqladmin` biết cách kết nối (tr. 108). Nếu server và client hiểu socket ở hai nơi khác nhau, client sẽ báo không kết nối được dù server đang chạy (**Bổ sung ngoài sách**).

[về mục 2.3](../chapter-05.md#s2-3)

---

<a id="q2-3-4"></a>

### Câu 2.3.4 — Đọc hai snapshot status

*Bạn có hai snapshot `SHOW GLOBAL STATUS` cách nhau 60 giây. Phân biệt bộ đếm tích lũy và giá trị tức thời, cho mỗi loại một ví dụ trong chương. Vì sao sách khuyên chụp cả ở giờ cao điểm lẫn thấp điểm?*

**Phân loại** (**Bổ sung ngoài sách**, ví dụ lấy từ chương):

- **Bộ đếm tích lũy**: chỉ tăng từ lúc khởi động; chỉ có nghĩa khi lấy hiệu số rồi chia cho 60 giây. Ví dụ: `Threads_created` (tr. 111, mục tiêu dưới 10 mỗi giây), `Innodb_buffer_pool_reads`.
- **Giá trị tức thời**: trạng thái lúc chụp. Ví dụ: `Threads_connected` (tr. 111), `Innodb_buffer_pool_pages_dirty` (tr. 110).
- **Mốc nước cao**: `Max_used_connections` (tr. 121), giá trị lớn nhất từng đạt.

Lệnh `mysqladmin extended-status -ri60` mà sách dùng (tr. 108) in sẵn hiệu số giữa các lần (cờ `-r`), nên bộ đếm tích lũy hiện ra thành "tốc độ".

**Vì sao cao điểm và thấp điểm** (tr. 108, 104–105). Giá trị tuyệt đối và thay đổi theo thời gian đều cần; workload thay đổi trong ngày, và cấu hình nên được tối ưu cho **cao điểm**. Một snapshot lúc thấp điểm có thể cho thấy mọi thứ ổn trong khi cao điểm đang thiếu thread cache hay chạm `max_connections`.

[về mục 2.3](../chapter-05.md#s2-3)

---

<a id="q2-4-1"></a>

### Câu 2.4.1 — Ngân sách bộ nhớ cho máy 64 GB

*Một server 64 GB chỉ chạy MySQL dùng `innodb_dedicated_server`. Ước lượng phần bộ nhớ còn lại và kể ba nhóm tiêu thụ cần tính tới. Vì sao không nên tính theo kịch bản `max_connections` nhân với tổng mọi buffer?*

**Ước lượng** (**Bổ sung ngoài sách**, phép tính). Sách nói dedicated server thường dùng 50–75% RAM, để lại ít nhất 25% (tr. 109). Theo tài liệu MySQL 8.0, máy trên 4 GB được đặt buffer pool khoảng 75%: 0,75 × 64 = 48 GB; còn khoảng **16 GB**.

**Ba nhóm cần tính tới** (tr. 109):

1. **Bộ nhớ theo kết nối**: giữ kết nối và thread, bộ nhớ cơ bản cho mỗi truy vấn; phần khó đoán như prepared statement, data dictionary; truy vấn có bảng tạm lớn hay stored procedure phức tạp.
2. **Hệ điều hành**: agent giám sát, công cụ quản lý cấu hình, job định kỳ; mục tiêu là không swap.
3. **Các thiết lập bộ nhớ khác** của MySQL ngoài buffer pool (ví dụ log buffer, thread cache khoảng 256 KB mỗi thread, tr. 111, 113).

**Vì sao không dùng kịch bản xấu nhất** (tr. 109, 106). Về lý thuyết 100 kết nối có thể cùng chạy truy vấn lớn, nhưng thực tế khó xảy ra; tính như vậy sẽ cấp quá ít cho buffer pool. Hơn nữa, kiểu công thức "nhân lên" chính là thứ sách bảo đừng tin, vì không thể đặt cận trên cho bộ nhớ MySQL (tr. 106). Thay vào đó: chừa biên hợp lý, rồi **đo** bộ nhớ thật ở cao điểm và theo dõi swap.

[về mục 2.4](../chapter-05.md#s2-4)

---

<a id="q2-4-2"></a>

### Câu 2.4.2 — Khởi động lại server buffer pool 200 GB

*Bạn phải khởi động lại một server có buffer pool 200 GB, nhiều trang bẩn, trong cửa sổ bảo trì ngắn. Lập kế hoạch theo sách. Vì sao ép `innodb_fast_shutdown` không thật sự rút ngắn chu kỳ?*

**Kế hoạch** (tr. 110–111):

1. **Trước cửa sổ bảo trì**, hạ `innodb_max_dirty_pages_pct` lúc chạy để InnoDB thôi "lười" và flush tích cực hơn.
2. **Theo dõi** `Innodb_buffer_pool_pages_dirty` (hoặc qua `innotop`) tới khi số trang bẩn đủ nhỏ.
3. **Tắt server** khi trang bẩn đã ít; shutdown sẽ nhanh vì còn ít trang phải ghi.
4. **Khởi động lại** với dump và load buffer pool (mặc định bật) để làm ấm; chấp nhận một khoảng thời gian nạp, vẫn nhanh hơn chờ cache tự đầy.
5. Trả `innodb_max_dirty_pages_pct` về giá trị cũ (**Bổ sung ngoài sách**: thay đổi động mất khi tắt, trừ khi đã persist).

**Vì sao `innodb_fast_shutdown` không rút ngắn chu kỳ** (tr. 110). Tắt nhanh tức là bỏ qua việc ghi trang bẩn, nhưng các thay đổi đó vẫn phải được áp dụng: lần khởi động sau, InnoDB phải **recovery nhiều hơn** từ redo log. Tổng thời gian tắt cộng bật không giảm; công việc chỉ dời sang lúc khởi động.

**Kết quả đã đo** (quy mô nhỏ, TN3): sau khi nạp dữ liệu, 3815 trang bẩn được flush nền về 0 trong khoảng 10 giây khi server rảnh; tệp dump có 1894 dòng và việc nạp lại xong trong khoảng 1 giây. Ở 200 GB, các con số này sẽ lớn hơn nhiều.

[về mục 2.4](../chapter-05.md#s2-4)

---

<a id="q2-4-3"></a>

### Câu 2.4.3 — `innodb_max_dirty_pages_pct` và flush "lười"

*Hạ `innodb_max_dirty_pages_pct` có bảo đảm luôn ít trang bẩn hơn không? Giải thích cơ chế "lười" và lý do InnoDB chọn nó.*

**Không bảo đảm** (tr. 110). Biến này là **ngưỡng** để InnoDB thôi lười, không phải trần cứng.

**Cơ chế** (tr. 110). Mặc định InnoDB flush trang bẩn bằng **luồng nền**, gộp nhiều lần ghi và ghi tuần tự. Gọi là "lười" vì nó trì hoãn flush trừ khi cần chỗ cho dữ liệu khác. Khi tỉ lệ trang bẩn vượt ngưỡng, InnoDB flush nhanh hết mức để kéo xuống. Page cleaner đã được tối ưu nhiều, có thể chạy nhiều luồng.

**Vì sao chọn lười** (tr. 110, 113, và [Chương 4](../chapter-04.md#caching-writes) tr. 76–77). Trì hoãn ghi cho phép (1) một trang bị sửa nhiều lần chỉ cần ghi một lần, (2) gộp nhiều trang thành ghi tuần tự hơn. Redo log đã bảo đảm durability, nên trang bẩn không cần ghi ngay khi commit. Flush gấp từng trang sẽ biến lợi ích này thành random I/O.

**Bổ sung ngoài sách.** Ở 8.0.42, mặc định `innodb_max_dirty_pages_pct` = 90 và có thêm `innodb_max_dirty_pages_pct_lwm` = 10 (đo được), mức "nước thấp" để bắt đầu flush trước khi tới ngưỡng chính. Ngoài ra, redo log có kích thước cố định cũng ép flush khi sắp hết chỗ ([2.5.1](../chapter-05.md#s2-5-1)), bất kể tỉ lệ trang bẩn.

[về mục 2.4](../chapter-05.md#s2-4)

---

<a id="q2-4-4"></a>

### Câu 2.4.4 — Định cỡ thread cache

*`Threads_connected` dao động 500–700, `Threads_created` tăng khoảng 30 mỗi giây. Đặt `thread_cache_size` bao nhiêu theo sách, và kiểm chứng bằng chỉ số nào? Bao giờ thì cache quá lớn trở thành vấn đề?*

**Giá trị** (tr. 111). Sách dùng đúng ví dụ này: dao động 500–700 thì cache **200** là đủ. Ở 700 kết nối gần như không còn thread nào trong cache; ở 500 kết nối có 200 thread sẵn sàng khi tải quay lên 700.

**Kiểm chứng** (tr. 111). Lấy hiệu số `Threads_created` theo thời gian: mục tiêu **dưới 10 thread mới mỗi giây**, thường dễ đạt dưới 1. Trạng thái hiện tại 30 mỗi giây là dấu hiệu cache quá nhỏ so với dao động.

**Kết quả đã đo** (TN3, quy mô nhỏ): với cache mặc định (9), 31 kết nối tuần tự không tạo thread nào; với cache 0, 31 kết nối tạo 31 thread.

**Khi nào quá lớn** (tr. 111). Thường cache lớn không cần thiết nhưng cũng không tốn mấy (khoảng 256 KB mỗi thread trong cache hay đang ngủ). Chỉ khi con số rất lớn, cỡ **hàng nghìn** thread, mới nên giảm, vì một số hệ điều hành xử lý kém khi có quá nhiều thread, kể cả khi phần lớn đang ngủ.

[về mục 2.4](../chapter-05.md#s2-4)

---

<a id="q2-4-5"></a>

### Câu 2.4.5 — Thread cache, thread pool, connection pooling

*Phân biệt thread cache, thread pool và connection pooling ở tầng truy cập. Cơ chế nào giới hạn được số truy vấn chạy đồng thời trong server?*

| Cơ chế | Là gì | Giới hạn truy vấn đồng thời? |
| --- | --- | --- |
| **Thread cache** (tr. 111) | Giữ sẵn thread rảnh để kết nối mới khỏi phải tạo thread; vẫn **một thread cho mỗi kết nối** | Không. Chỉ giảm chi phí tạo, hủy thread |
| **Thread pool** (**Bổ sung ngoài sách**) | Plugin của server: một nhóm thread có hạn phục vụ nhiều kết nối, xếp hàng truy vấn | **Có**, ở tầng server |
| **Connection pooling ở tầng truy cập** (ProxySQL, [Chương 11](../chapter-11.md#connection-pooling), tr. 306–307) | Proxy giữ ít kết nối tới database, phục vụ nhiều kết nối từ ứng dụng | **Có**, gián tiếp: số kết nối vào database bị giới hạn |
| `innodb_thread_concurrency` (tr. 119) | Giới hạn số thread trong kernel InnoDB | Một phần: chỉ trong InnoDB, các thread chờ vẫn tồn tại |

Ghi chú: chương 1 (tr. 2) nói thread pooling "thường làm ở tầng truy cập" và "bàn ở chương 5", nhưng chương 5 không có nội dung đó; phần tầng truy cập nằm ở chương 11 (xem [kiểm chứng](../chapter-05.md#kiem-chung-roadmap)). **Kết quả đã đo:** bản Community 8.0.42 trên máy chỉ có `thread_handling = one-thread-per-connection` và không kèm plugin thread pool.

[về mục 2.4](../chapter-05.md#s2-4)

---

<a id="q2-5-1"></a>

### Câu 2.5.1 — Redo log và kích thước cố định

*Giải thích vì sao redo log biến random I/O thành sequential I/O, và vì sao kích thước cố định của log buộc InnoDB phải flush trang bẩn. Dự đoán điều gì xảy ra nếu log quá nhỏ với một workload ghi nặng.*

**Random thành sequential** (tr. 113). Thay đổi của một transaction rơi vào trang dữ liệu và index ở **vị trí ngẫu nhiên** trong tablespace. Flush các trang đó ở mỗi commit cần random I/O, đắt trên ổ truyền thống. Thay vào đó InnoDB **ghi nối tiếp** bản ghi thay đổi vào log; khi log đã an toàn trên đĩa thì transaction đã bền, và có crash thì phát lại log. Trang bẩn được luồng nền flush sau, gộp lại và tuần tự hơn.

**Kích thước cố định** (tr. 113). Log được ghi vòng tròn. InnoDB không được ghi đè một bản ghi log mà thay đổi trong đó chưa vào datafile, vì đó là bản ghi bền vững duy nhất của transaction đã commit. Vì vậy, trước khi quay vòng đè lên phần cũ, các trang bẩn tương ứng phải được flush.

**Dự đoán khi log quá nhỏ** (**Bổ sung ngoài sách**). Phần log còn trống hết nhanh, InnoDB phải flush trang bẩn **gấp** thay vì từ tốn: ghi kém gộp, nhiều random I/O, và truy vấn ghi có thể khựng lại chờ có chỗ trong log. Đó là lý do sách xếp kích thước log là một trong hai thiết lập quan trọng nhất (tr. 113, 124). Đổi lại, log lớn thì recovery sau crash phải phát lại nhiều hơn. Trên 8.0.30+, biến cần chỉnh là `innodb_redo_log_capacity` (đổi online được, đã đo).

[về mục 2.5](../chapter-05.md#s2-5)

---

<a id="q2-5-2"></a>

### Câu 2.5.2 — Mất gì với 0, 1, 2

*So sánh `innodb_flush_log_at_trx_commit` = 0, 1, 2 về mất dữ liệu khi (a) chỉ `mysqld` crash, (b) cả máy mất điện. Vì sao ổ đĩa "giả" flush còn nguy hiểm hơn đặt giá trị 2?*

| Giá trị | (a) `mysqld` crash | (b) Mất điện cả máy |
| --- | --- | --- |
| 1 | Không mất transaction đã commit | Không mất, nếu đĩa và hệ điều hành flush thật |
| 2 | **Không mất**: log đã được ghi vào cache của hệ điều hành ở mỗi commit, tiến trình chết thì hệ điều hành vẫn giữ | Có thể mất, thường tới khoảng 1 giây |
| 0 | Có thể mất khoảng 1 giây: commit không ghi gì, log buffer còn trong bộ nhớ của `mysqld` | Có thể mất khoảng 1 giây |

*(Theo tr. 114. "Khoảng 1 giây" là thường lệ; nếu một lần flush bị nghẽn thì có thể mất nhiều hơn.)*

Mấu chốt (tr. 114): **ghi** log ra tệp chỉ là chuyển từ bộ nhớ InnoDB sang **cache của hệ điều hành**, vẫn trong RAM; chỉ **flush** mới đưa tới bộ nhớ bền vững.

**"Giả" flush nguy hiểm hơn** (tr. 115). Khi controller hay hệ điều hành báo đã flush nhưng thật ra chỉ đặt dữ liệu vào một cache khác (ví dụ cache của ổ đĩa không có pin), mất điện có thể làm mất dữ liệu mà InnoDB tin là đã bền. Hậu quả có thể là **hỏng dữ liệu**, không chỉ mất vài transaction cuối, vì InnoDB đã ra quyết định (ví dụ cho phép ghi đè phần log cũ, ghi trang) dựa trên giả định sai rằng dữ liệu đã nằm an toàn. Với giá trị 2, bạn biết mình chấp nhận mất tối đa khoảng một giây; với flush giả, bạn không biết mình đang chấp nhận gì.

[về mục 2.5](../chapter-05.md#s2-5)

---

<a id="q2-5-3"></a>

### Câu 2.5.3 — Vì sao nới một biến không đủ

*Trong TN4, chỉ đặt `innodb_flush_log_at_trx_commit = 2` (giữ `sync_binlog = 1`) tăng thông lượng khoảng 1,9 lần, còn nới cả hai thì khoảng 13 lần. Giải thích nguyên nhân gốc. Vì sao sách vẫn khuyên giữ `sync_binlog = 1`?*

**Nguyên nhân gốc (Kết quả đã đo, TN4).** Với binary log bật, mỗi commit ở cấu hình mặc định chờ **hai loại fsync**: redo (khoảng 1,3 mỗi commit) và binlog (đúng 1,0 mỗi commit). Thông lượng của một client bị chặn bởi tổng thời gian chờ các fsync đó.

- (2, 1): fsync redo gần như biến mất, nhưng fsync binlog vẫn 1 mỗi commit: 1.234 commit/giây so với 645.
- (1, 0): fsync binlog biến mất, fsync redo còn: 954 commit/giây.
- (2, 0): không còn fsync nào trên đường commit: 8.214 commit/giây.

Nghĩa là nút thắt là **"còn fsync nào trên đường commit không"**, chứ không phải riêng biến nào. Đây là phép đo trên laptop, có nhiễu; thứ hạng đáng tin hơn con số tuyệt đối. Ứng dụng thật có round trip mạng cho từng commit nên chênh lệch sẽ nhỏ hơn (**Dự đoán**).

**Vì sao vẫn giữ `sync_binlog = 1`** (tr. 118). Không giữ 1 thì crash rất có thể làm binlog **lệch** với dữ liệu transaction; điều đó dễ làm hỏng replication và khiến phục hồi không thể làm được, nhất là với GTID ([Chương 9](../chapter-09.md#gtid)). Sách đánh giá sự an toàn vượt xa chi phí I/O. Muốn nhanh mà vẫn an toàn thì dùng phần cứng có write cache bền (tr. 115, 118–119) và tận dụng group commit (TN4: 8 client đạt khoảng 3.048 commit/giây ở (1, 1)).

[về mục 2.5](../chapter-05.md#s2-5)

---

<a id="q2-5-4"></a>

### Câu 2.5.4 — Nhiều tệp tablespace và file-per-table

*Vì sao chia system tablespace thành nhiều tệp trên nhiều đĩa không phân tải như mong muốn? `innodb_file_per_table` được và mất gì?*

**Nhiều tệp không phân tải** (tr. 116). InnoDB **nối các tệp nối đuôi nhau**: đầy tệp đầu mới ghi sang tệp sau. Tại một thời điểm, phần lớn hoạt động dồn vào một tệp, nên tải không được trải đều như kiểu striping. Muốn chia tải, dùng RAID controller.

**`innodb_file_per_table`** (tr. 116–117):

| Được | Mất |
| --- | --- |
| Thu hồi dung lượng dễ khi xóa bảng (xóa tệp `.ibd`) | Có thể lãng phí chỗ nhiều hơn tổng thể: chỗ trống thừa nằm rải trong từng tệp `.ibd` thay vì phân mảnh bên trong một tablespace chung |
| Dễ quản lý, quan sát: xem kích thước bảng qua kích thước tệp, nhanh hơn `SHOW TABLE STATUS` | `DROP TABLE` từng chậm, có thể làm cả server khựng (unlink chậm trên vài filesystem; quét buffer pool tìm trang), theo sách đã hết từ 8.0.23 (hộp Cảnh báo, tr. 117) |
| Tablespace chung nhỏ hơn | Vẫn cần tablespace chung cho dữ liệu hệ thống |

Khuyến nghị của sách: bật `innodb_file_per_table` và giới hạn kích thước tablespace chung (tr. 117). **Kết quả đã đo** (TN2): ở 8.0.42 nó đã bật mặc định; sau khi nạp khoảng 360 MB, `ibdata1` vẫn 12 MiB, và undo nằm ở `undo_001`, `undo_002` riêng.

[về mục 2.5](../chapter-05.md#s2-5)

---

<a id="q2-5-5"></a>

### Câu 2.5.5 — Đĩa đầy vì history list

*Đĩa đầy dần trong khi lượng dữ liệu nghiệp vụ không tăng; history list length lên hàng triệu. Nêu nguyên nhân gốc theo sách và hai hướng xử lý. Vì sao `innodb_max_purge_lag` là "cái ác nhỏ hơn"?*

**Nguyên nhân gốc** (tr. 117). Một hay nhiều transaction **mở lâu** (kể cả khi không làm gì) ở REPEATABLE READ. InnoDB không thể xóa phiên bản cũ của hàng vì các transaction đó còn có thể cần nhìn thấy chúng; phiên bản cũ nằm trong tablespace (ở 8.0 là undo tablespace) và lớn dần theo mỗi cập nhật. History list length cho biết lượng việc purge đang tồn (tr. 118); đo được trong TN5 là nó đếm **số transaction** chờ purge.

**Hai hướng xử lý.**

1. **Gỡ nguyên nhân**: tìm và kết thúc transaction mở lâu. **Kết quả đã đo** (TN5): khi snapshot đóng, history list 20.000 về 0 trong khoảng 2 giây ở quy mô nhỏ. Có thể chỉnh purge (`innodb_purge_threads`, `innodb_purge_batch_size`) nếu purge thật sự không theo kịp (tr. 117).
2. **Hãm ghi**: đặt `innodb_max_purge_lag` khác 0, để khi số transaction chờ purge vượt ngưỡng, InnoDB trì hoãn các truy vấn cập nhật cho purge theo kịp (tr. 118). Ví dụ của sách: 1 KB mỗi transaction, chịu được 100 MB chưa purge, thì đặt 100000.

**Vì sao "cái ác nhỏ hơn"** (tr. 118). Không hãm thì InnoDB cứ ghi tới khi đầy đĩa hay chạm giới hạn tablespace, và phiên bản chưa purge làm bảng, index lớn hơn nên **mọi truy vấn** chậm đi. Hãm ghi cũng làm chậm, nhưng có kiểm soát và tránh được kết cục tệ hơn.

[về mục 2.5](../chapter-05.md#s2-5)

---

<a id="q2-6-1"></a>

### Câu 2.6.1 — Thứ tự ưu tiên khi gặp vấn đề concurrency

*Theo sách, bước đầu tiên khi gặp vấn đề concurrency của InnoDB trên bản cũ hơn 5.7 là gì? Trên bản mới mà vẫn nghẽn thì lựa chọn tốt nhất là gì, và giới hạn concurrency đứng ở đâu trong thứ tự ưu tiên?*

Thứ tự theo tr. 119:

1. **Bản cũ hơn 5.7: nâng cấp.** Bản cũ xếp hàng trên các mutex toàn cục như mutex của buffer pool, server gần như đứng. Ở bản mới, đa số trường hợp không cần giới hạn concurrency.
2. **Bản mới mà vẫn chạm nút thắt: shard dữ liệu** ([Chương 11](../chapter-11.md#s2-6)). Đây là "lựa chọn tốt nhất".
3. **Chỉ khi shard không khả thi: giới hạn concurrency**, bắt đầu bằng `innodb_thread_concurrency` bằng số lõi CPU rồi thử lên xuống; manual là hướng dẫn tốt nhất.

Bài học (**Bổ sung ngoài sách**): giới hạn concurrency là **giải pháp cuối**, vì nó không làm hệ thống làm được nhiều việc hơn mà chỉ xếp hàng việc lại để giảm tranh chấp.

[về mục 2.6](../chapter-05.md#s2-6)

---

<a id="q2-6-2"></a>

### Câu 2.6.2 — Hai pha và vé

*Mô tả cơ chế hai pha khi thread vào kernel InnoDB và vai trò của vé. Khi nào `innodb_thread_sleep_delay` mặc định gây hại?*

**Hai pha** (tr. 119). Kernel đã đủ số thread cho phép thì thread mới không vào được.

1. Pha 1: thread **ngủ** `innodb_thread_sleep_delay` micro giây (mặc định 10.000, tức 10 ms) rồi thử lại.
2. Pha 2: vẫn không vào được thì vào **hàng đợi** và nhường hệ điều hành.

Mục đích: giảm chi phí context switch do bộ lập lịch của hệ điều hành.

**Vé** (tr. 120). Vào được kernel, thread nhận một số vé (`innodb_concurrency_tickets`) để quay lại kernel "miễn phí" mà không qua kiểm tra. Vé giới hạn lượng việc nó làm trước khi phải xếp hàng lại; cấp theo **truy vấn**, không theo transaction; truy vấn xong thì vé thừa bị bỏ. Hiếm khi cần đổi, trừ khi có nhiều truy vấn chạy cực lâu.

**Khi sleep delay mặc định gây hại** (tr. 119):

- Có nhiều **truy vấn nhỏ**: 10 ms ngủ cộng thẳng vào latency, có thể lớn hơn chính thời gian chạy truy vấn.
- Concurrency cao mà **CPU chưa dùng hết**, nhiều thread ở trạng thái "sleeping before entering queue": thread đang ngủ trong khi CPU rảnh.

[về mục 2.6](../chapter-05.md#s2-6)

---

<a id="q2-6-3"></a>

### Câu 2.6.3 — Hai chỗ giới hạn: kernel và commit

*`innodb_thread_concurrency` và `innodb_commit_concurrency` giới hạn hai chỗ khác nhau nào? Vì sao commit là một nút thắt riêng, và nó liên hệ thế nào với group commit ở 2.5.2?*

**Hai chỗ** (tr. 119–120):

- `innodb_thread_concurrency`: số thread **trong kernel InnoDB** để truy cập dữ liệu (tranh chấp trên buffer pool và các cấu trúc khác).
- `innodb_commit_concurrency`: số thread được **commit cùng lúc**.

**Vì sao commit là nút thắt riêng** (tr. 120). Giai đoạn commit phần lớn **bị chặn bởi I/O** vì các thao tác flush. Sách gợi ý chỉnh nó nếu vẫn thấy thread "giành giật" dù `innodb_thread_concurrency` đã thấp.

**Liên hệ với group commit** (tr. 114, và **Bổ sung ngoài sách**). Group commit gom nhiều transaction đang chờ flush vào một thao tác I/O. TN4 đo được với 8 client, mỗi fsync binlog phục vụ khoảng 3,4 commit. Do đó nhiều commit đồng thời không nhất thiết là xấu: chúng giúp chia sẻ fsync. **Dự đoán**: đặt `innodb_commit_concurrency` quá thấp có thể làm nhóm commit nhỏ đi và mất bớt lợi ích này; đây là lý do chỉ chỉnh nó khi thật sự thấy thrashing. **Bổ sung ngoài sách:** theo tài liệu MySQL, biến này không đổi được lúc chạy giữa 0 và khác 0.

[về mục 2.6](../chapter-05.md#s2-6)

---

<a id="q2-7-1"></a>

### Câu 2.7.1 — `max_connections` là phanh khẩn cấp

*Vì sao sách gọi `max_connections` là "phanh khẩn cấp"? Dự đoán điều gì xảy ra nếu đặt 10.000 trên server 16 lõi khi có một stall kéo dài vài giây.*

**Theo sách** (tr. 120–121). Khi ứng dụng trục trặc hay server gặp stall, rất nhiều kết nối mới có thể được mở. Nhưng mở kết nối mà không chạy được truy vấn thì vô ích; bị từ chối bằng lỗi "too many connections" là cách **thất bại nhanh và rẻ**. Giới hạn này chặn server bị nhấn chìm.

**Dự đoán với 10.000** (**Bổ sung ngoài sách**). Trong stall, ứng dụng timeout rồi thử lại, mở thêm kết nối. Mỗi kết nối có thread, bộ nhớ riêng; truy vấn dồn lại tranh CPU, lock và I/O. Khi stall hết, server phải xử lý một hàng đợi khổng lồ, latency tăng vọt, bộ nhớ có thể cạn tới mức swap hay bị hệ điều hành giết. Với giới hạn hợp lý (ví dụ khoảng 500 khi tải thường 300, như sách gợi ý), phần vượt bị từ chối sớm, server giữ được khả năng phục hồi nhanh.

Theo dõi `Max_used_connections`: chạm `max_connections` nghĩa là đã có client bị từ chối (tr. 121).

[về mục 2.7](../chapter-05.md#s2-7)

---

<a id="q2-7-2"></a>

### Câu 2.7.2 — `skip_name_resolve` và `max_connect_errors`

*Bật `skip_name_resolve` giải quyết vấn đề gì, và đòi hỏi thay đổi gì với tài khoản? Vì sao sau đó `max_connect_errors` hết tác dụng?*

**Vấn đề** (tr. 121). Mặc định, khi client kết nối, server xác định hostname và dùng nó trong xác thực (thông tin đăng nhập là user, host, password). Để xác minh hostname, server phải tra **ngược rồi tra xuôi** DNS. Khi DNS trục trặc (sớm muộn sẽ xảy ra), các kết nối dồn ứ và timeout. `skip_name_resolve` tắt việc tra DNS khi xác thực; sách **khuyên mạnh** bật.

**Đòi hỏi** (tr. 121). Mọi grant theo hostname phải đổi sang **IP, wildcard hoặc `localhost`**, vì tài khoản dựa trên hostname sẽ bị vô hiệu.

**Vì sao `max_connect_errors` hết tác dụng** (tr. 120). Cơ chế chặn host sau nhiều lỗi kết nối dựa vào **host cache**, mà `skip_name_resolve` tắt host cache. Sách cũng nói mặc định 100 của `max_connect_errors` quá nhỏ, dễ chặn nhầm client sau một đợt lỗi mạng; có thể tăng rất lớn nếu server đã được bảo vệ khỏi brute-force.

[về mục 2.7](../chapter-05.md#s2-7)

---

<a id="q2-7-3"></a>

### Câu 2.7.3 — Đổi `sql_mode` an toàn

*Vì sao bật `ONLY_FULL_GROUP_BY` hay đổi `sql_mode` trên một ứng dụng đang chạy là rủi ro? Quy trình an toàn theo sách là gì, và nó liên quan gì tới nâng cấp phiên bản?*

**Rủi ro** (tr. 121–122). `sql_mode` thay đổi hành vi của server; đổi trên ứng dụng hiện có có thể làm server **không còn khớp kỳ vọng** của ứng dụng. Rất thường gặp truy vấn tham chiếu cột không nằm trong GROUP BY; bật `ONLY_FULL_GROUP_BY` sẽ làm chúng lỗi. Ép MySQL giống SQL chuẩn ANSI hơn cũng có thể làm vỡ client và công cụ GUI vốn kỳ vọng phương ngữ của MySQL.

**Quy trình an toàn** (tr. 121–122). Không đổi "cho vui". Bật trước ở **dev hay staging**, chỉ đưa lên production khi chắc mọi thứ chạy đúng.

**Liên quan nâng cấp** (tr. 122). Các bản sau chặt hơn về `sql_mode`; khi lên kế hoạch nâng cấp, phải kiểm tra thay đổi của `sql_mode` **mặc định**, vì nó có thể không tương thích với ứng dụng, và phải thử trước ([Phụ lục A](../appendix-a.md)). **Kết quả đã đo:** mặc định ở 8.0.42 gồm `ONLY_FULL_GROUP_BY`, `STRICT_TRANS_TABLES`, `NO_ZERO_IN_DATE`, `NO_ZERO_DATE`, `ERROR_FOR_DIVISION_BY_ZERO`, `NO_ENGINE_SUBSTITUTION`.

[về mục 2.7](../chapter-05.md#s2-7)

---

<a id="q2-7-4"></a>

### Câu 2.7.4 — `read_only`, `super_read_only`, `sysdate_is_now`

*`read_only` khác `super_read_only` thế nào? `sysdate_is_now` bảo vệ replication và point-in-time recovery ra sao?*

**`read_only` và `super_read_only`** (tr. 122). `read_only` chặn user **không có đặc quyền** ghi trên replica; replica chỉ nên nhận thay đổi qua replication, không từ ứng dụng. `super_read_only` chặt hơn: chặn cả user có quyền `SUPER`, nên chỉ replication ghi được. Sách khuyên mạnh bật cả hai; `super_read_only` ngăn bạn vô tình dùng tài khoản quản trị ghi vào replica làm nó lệch.

**`sysdate_is_now`** (tr. 122). `SYSDATE()` mặc định **không tất định**: trả thời điểm thực thi, khác nhau giữa các lần gọi, kể cả trong một câu lệnh. Khi một câu lệnh dùng nó được phát lại ở nơi khác hay lúc khác (replica, hay áp binlog để khôi phục tới một thời điểm), giá trị có thể khác bản gốc, nên dữ liệu lệch và point-in-time recovery không đáng tin. Bật tùy chọn này làm `SYSDATE()` hoạt động như `NOW()`, tất định trong câu lệnh. Đánh đổi: có thể không tương thích ngược với ứng dụng cố ý dựa vào hành vi cũ.

**Kết quả đã đo.** Ở 8.0.42, `sysdate_is_now` là **tùy chọn dòng lệnh**, không phải system variable: `SET GLOBAL sysdate_is_now = 1` báo lỗi 1193, nên chỉ bật được lúc khởi động.

[về mục 2.7](../chapter-05.md#s2-7)

---

<a id="q2-8-1"></a>

### Câu 2.8.1 — LRU hai phần và `innodb_old_blocks_time`

*Sách và tài liệu MySQL mô tả LRU hai phần khác nhau ở đâu? Giải thích cơ chế đúng, và dùng kết quả TN3 để chỉ ra `innodb_old_blocks_time` bảo vệ hot set thế nào.*

**Khác biệt.** Sách (tr. 124) viết trang mới vào phần **"young"**, chỉ chuyển sang **"old"** sau nhiều lần truy cập, và mặc định của biến là **0**. Theo tài liệu MySQL và mô tả `--help` của 8.0.42 (**Bổ sung ngoài sách**), hướng **ngược lại**, và mặc định là **1000** (từ 5.6.6; đo được 1000 trên 8.0.42).

**Cơ chế đúng** (**Bổ sung ngoài sách**). LRU chia hai phần: **new** (young) ở đầu và **old** ở cuối (khoảng 3/8, `innodb_old_blocks_pct` = 37). Trang mới đọc từ đĩa được chèn ở **điểm giữa**, tức đầu phần old. Nó chỉ được đưa lên đầu phần new khi được truy cập lại **sau ít nhất** `innodb_old_blocks_time` mili giây kể từ lần truy cập đầu. Một lần quét đọc nhiều hàng trong cùng trang trong vài mili giây, nên các truy cập đó không đủ điều kiện; trang của lần quét trôi về cuối phần old và bị loại trước, không đụng tới hot set ở phần new. Với giá trị 0, lần truy cập thứ hai (ngay lập tức, do trang có nhiều hàng) đã đưa trang lên new, đúng vấn đề sách mô tả.

**Kết quả đã đo** (TN3, pool 128 MiB, bảng `hot` khoảng 1250 trang đã nóng, rồi quét bảng 300 MB):

- 1000: lần quét đưa 1.384 và 0 trang lên young (hai vòng); sau đó đọc `hot` chỉ cần 3 và 0 trang từ đĩa.
- 0: lần quét đưa khoảng 16.860 trang lên young; sau đó đọc `hot` cần 1.645 và 1.263 trang từ đĩa, tức hot set đã bị đẩy ra.

**Kết luận.** Lời **khuyên** của sách (đặt khoảng 1000) đúng và nay đã là mặc định; **mô tả cơ chế** và giá trị mặc định trong sách là lỗi thời hoặc nhầm.

[về mục 2.8](../chapter-05.md#s2-8)

---

<a id="q2-8-2"></a>

### Câu 2.8.2 — `innodb_buffer_pool_instances`

*`innodb_buffer_pool_instances` giải quyết loại nút thắt nào? Vì sao instance thực hành 128 MiB chỉ có 1 instance, và điều đó gợi ý gì về khi nào biến này đáng quan tâm?*

**Nút thắt** (tr. 123). Tranh chấp trên **mutex toàn cục** của buffer pool khi nhiều lõi CPU cùng truy cập với workload đồng thời cao. Chia buffer pool thành nhiều phần làm tải được phân ra, các mutex không còn là điểm nóng. Sách gọi đây có lẽ là một trong những cách quan trọng nhất để tăng khả năng mở rộng trên máy nhiều lõi (có từ 5.5).

**Vì sao chỉ có 1** (**Bổ sung ngoài sách**, đo được 1). Theo tài liệu MySQL 8.0, mặc định là 8 khi buffer pool từ 1 GiB trở lên, ngược lại là 1. Instance thực hành dùng 128 MiB.

**Gợi ý.** Biến này đáng quan tâm khi có **cả hai** điều kiện: buffer pool lớn và concurrency cao trên nhiều lõi. Với pool nhỏ, chia nhỏ thêm không có lợi; với workload ít đồng thời, mutex không phải nút thắt. Đây cũng là biến hỗ trợ việc giảm thời gian quét buffer pool khi `DROP TABLE` (hộp Cảnh báo, tr. 117).

[về mục 2.8](../chapter-05.md#s2-8)

---

<a id="q2-8-3"></a>

### Câu 2.8.3 — `innodb_io_capacity`

*Vì sao giả định "một ổ 100 IOPS" là mặc định tồi với flash? Đặt `innodb_io_capacity` quá cao có thể gây hại gì?*

**Vì sao tồi** (tr. 123). InnoDB dùng con số này để quyết định flush nền nhanh tới đâu. Giả định một ổ 100 IOPS khiến InnoDB flush quá chậm trên lưu trữ nhanh: trang bẩn tích lại, rồi phải flush gấp khi chạm giới hạn (redo log, tỉ lệ trang bẩn), gây khựng. Sách nói trên lưu trữ cực nhanh như flash PCIe, đôi khi phải đặt rất cao (hàng chục nghìn) để flush **đều đặn**; lý do chi tiết "khá phức tạp để giải thích" và sách dẫn ba bài Percona (chú thích 4).

**Quá cao có hại gì** (**Bổ sung ngoài sách**; tiêu đề bài Percona đầu tiên "Give Love to Your SSDs—Reduce innodb_io_capacity_max" gợi ý hướng này). Flush quá tích cực làm mất lợi ích của việc trì hoãn: một trang bị sửa nhiều lần bị ghi nhiều lần thay vì một; I/O nền tranh băng thông với truy vấn đọc ở tiền cảnh; SSD bị ghi nhiều hơn (hao mòn). **Kết quả đã đo:** mặc định 8.0.42 là 200, `innodb_io_capacity_max` là 2000.

[về mục 2.8](../chapter-05.md#s2-8)

---

<a id="q2-8-4"></a>

### Câu 2.8.4 — `innodb_strict_mode`

*`innodb_strict_mode` mang lại lợi ích gì và vì sao sách cảnh báo về lúc restore backup?*

**Lợi ích** (tr. 123). Biến một số **cảnh báo thành lỗi**, nhất là tùy chọn `CREATE TABLE` sai hay có thể nguy hiểm. Nhờ vậy cấu hình bảng sai không âm thầm lọt qua.

**Rủi ro khi restore** (tr. 123). Bật nó thì những bảng từng tạo được có thể bị từ chối; đôi khi nó hơi bi quan và quá chặt. Nếu bạn chỉ phát hiện điều đó khi đang restore backup (một logical backup chứa các câu `CREATE TABLE` cũ), quá trình khôi phục bị dừng giữa chừng, đúng lúc bạn cần nó nhất. Vì vậy: bật thì phải kiểm mọi tùy chọn `CREATE TABLE` trước, và thử restore trên môi trường có cùng thiết lập (**Bổ sung ngoài sách**: liên hệ việc tự động kiểm tra backup ở [Chương 10](../chapter-10.md#kiem-tra-backup)).

**Kết quả đã đo.** Ở 8.0.42, `innodb_strict_mode` = ON mặc định (**Bổ sung ngoài sách:** ON từ 5.7.7), nên backup từ server cũ tắt strict mode có thể gặp đúng tình huống này khi restore vào 8.0.

[về mục 2.8](../chapter-05.md#s2-8)

---

<a id="q2-9-1"></a>

### Câu 2.9.1 — Cấu hình "đủ tốt" ba bước

*Tóm tắt cấu hình "đủ tốt" theo sách bằng ba bước. Trên MySQL 8.0.30 trở lên, biến quan trọng thứ hai trong Summary đổi thành gì, và vì sao?*

**Ba bước** (tr. 124):

1. Bắt đầu từ **file cấu hình mẫu** (đã cập nhật tên biến theo phiên bản).
2. Đặt **tùy chọn cơ bản** cho server và workload: trên server chuyên dụng là `innodb_dedicated_server`; nếu không, `innodb_buffer_pool_size` và kích thước redo log.
3. Thêm **safety option** nếu muốn. Rồi dừng; không "tune" theo tỉ lệ, công thức hay script.

**Trên 8.0.30+** (**Bổ sung ngoài sách**, đã đo trên 8.0.42). Biến thứ hai là **`innodb_redo_log_capacity`**. `innodb_log_file_size` đã deprecated: nếu đặt, nó chỉ được dùng để tính capacity, kèm cảnh báo trong error log; bản thân nó read-only, còn `innodb_redo_log_capacity` đổi được online. `innodb_dedicated_server` của 8.0.42 cũng quản lý `innodb_redo_log_capacity` thay cho hai biến cũ. Ý của sách (kích thước redo log rất quan trọng cho ghi, tr. 113) không đổi; chỉ tên biến đổi.

[về mục 2.9](../chapter-05.md#s2-9)

---

<a id="q2-9-2"></a>

### Câu 2.9.2 — Cấu hình chỉ là phần nhỏ

*Vì sao chương coi cấu hình chỉ là một phần nhỏ của hiệu năng? Cho một ví dụ vấn đề hiệu năng mà không thiết lập nào trong chương sửa được.*

**Vì sao** (tr. 99–100, 105). Chỉ vài thiết lập thật sự quan trọng; khi chúng đã đúng, lợi ích từ việc chỉnh thêm thường **nhỏ**. Phần lớn thời gian nên dành cho **schema, index và thiết kế truy vấn**. Sách còn khuyên tối ưu truy vấn và schema (ít nhất là thêm index hiển nhiên) **trước** khi chỉnh cấu hình, vì đổi chúng sau sẽ buộc đánh giá lại cấu hình (tr. 105).

**Ví dụ** (**Ví dụ minh họa do AI xây dựng**). Một truy vấn `SELECT ... FROM orders WHERE customer_email = ?` chạy hàng nghìn lần mỗi phút trên bảng 50 triệu hàng, **không có index** trên `customer_email`. Mỗi lần là một full scan. Tăng buffer pool có thể làm các lần quét đọc từ bộ nhớ thay vì đĩa, nhưng mỗi lần vẫn phải duyệt 50 triệu hàng và tốn CPU; không thiết lập nào biến full scan thành tra cứu vài hàng. Thêm index phù hợp mới sửa được gốc (xem [Chương 7](../chapter-07.md) và [Chương 8](../chapter-08.md)). Đây đúng là trường hợp sách nói nên bắt đầu từ truy vấn và response time (tr. 100).

[về mục 2.9](../chapter-05.md#s2-9)

---

<a id="q2-9-3"></a>

### Câu 2.9.3 — Danh sách kiểm tra trước khi đổi một biến

*Lập một danh sách kiểm tra năm bước trước khi đổi một biến trên production, mỗi bước dựa vào một ý của chương (kèm trang).*

Một đáp án tham khảo (thứ tự và cách diễn đạt có thể khác):

1. **Có triệu chứng thật và đã chẩn đoán**: vấn đề hiện ra ở response time hay SLO, và nguyên nhân nằm đúng ở phần server mà biến điều khiển (tr. 100, 104).
2. **Đọc manual của đúng phiên bản**: hiểu scope (global hay session), có dynamic không, tác dụng phụ khi đổi online, tên và mặc định có đổi không (tr. 100, 102–104; chú thích 3, tr. 107).
3. **Ưu tiên phạm vi nhỏ nhất**: biến theo kết nối thì đổi ở session cho truy vấn cần, không đổi global (tr. 104); không nới durability nếu chưa chấp nhận rủi ro mất dữ liệu (tr. 114–115, 118).
4. **Thử và đo trước, sau**: trên môi trường thử, rồi theo dõi SLO và status deltas ở cao điểm và thấp điểm; không tin benchmark hay hit ratio (tr. 104–105, 108).
5. **Ghi lại và có đường lui**: đưa thay đổi vào version control, nhớ rằng sửa file không tự đổi giá trị lúc chạy và ngược lại (tr. 103–104); ghi giá trị cũ để khôi phục, vì `SET GLOBAL ... = DEFAULT` không trả về giá trị lúc khởi động (tr. 102–103).

[về mục 2.9](../chapter-05.md#s2-9)
