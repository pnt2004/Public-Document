# Đáp án — Chương 10: Backup and Recovery

Quay về ghi chú: [../chapter-10.md](../chapter-10.md) · Thực hành: [../experiments/chapter-10.md](../experiments/chapter-10.md)

> Đáp án diễn giải từ sách (kèm trang), cộng phần **Bổ sung ngoài sách** khi cần. Số liệu ghi **Kết quả đã đo** lấy từ bài thực hành trên MySQL 8.0.42, Windows 11.

---

<a id="q2-0-1"></a>

### Câu 2.0.1 — Restore khác recovery thế nào?

*Phân biệt restore và recovery theo định nghĩa của sách. Cho hai ví dụ bước thuộc recovery nhưng không thuộc restore.*

**Restore** là lấy dữ liệu từ backup rồi nạp vào MySQL (logical) hoặc đặt file vào chỗ MySQL cần (raw). **Recovery** là toàn bộ quá trình cứu hệ thống sau sự cố: gồm restore và mọi bước để server hoạt động đầy đủ trở lại (tr. 258).

Ví dụ bước thuộc recovery nhưng không phải restore: khởi động lại MySQL, đổi cấu hình, làm ấm cache (tr. 258); phát lại binlog, kiểm tra dữ liệu đã restore, mở lại server với truy cập đầy đủ (bước 8–10, tr. 281). Hệ quả thực tế: RTO phải tính cả các bước này, không chỉ thời gian chạy lệnh restore.

[về mục 2.0](../chapter-10.md#s2-0)

<a id="q2-0-2"></a>

### Câu 2.0.2 — Vì sao crash recovery không cứu được `DROP TABLE`?

*Vì sao InnoDB crash recovery không cứu được một `DROP TABLE` vô tình, dù nó "phục hồi" được dữ liệu sau khi server sập?*

Crash recovery đối chiếu datafile với logfile để datafile chỉ chứa thay đổi của transaction **đã commit**, và phát lại những thay đổi đã commit chưa kịp vào datafile (tr. 258). Một `DROP TABLE` đã thực thi là một thay đổi đã commit hợp lệ, nên crash recovery giữ nó. Crash recovery bảo vệ **tính nhất quán sau sự cố mất điện**, không bảo vệ trước **thay đổi sai về nghiệp vụ**. Muốn lấy lại bảng, phải có bản sao ở thời điểm **trước** lệnh đó (backup) và binlog để tiến tới ngay trước nó (tr. 261).

[về mục 2.0](../chapter-10.md#s2-0)

<a id="q2-0-3"></a>

### Câu 2.0.3 — Hỏi lại gì khi nghe "hot backup"?

*Sách tránh các từ hot, warm, cold. Nếu đồng nghiệp nói "công cụ X làm hot backup", bạn sẽ hỏi lại những câu gì để biết mức gián đoạn thật?*

Sách tránh các từ này vì không có nghĩa thống nhất, và có công cụ mang chữ "hot" mà không làm điều tác giả coi là hot backup; thay vào đó sách mô tả mỗi kỹ thuật **làm gián đoạn server tới mức nào** (tr. 257–258). Các câu nên hỏi:

- Công cụ giữ **lock** gì (khóa toàn cục, khóa bảng, backup lock), giữ bao lâu, chặn ghi hay chỉ chặn DDL?
- Có cần **dừng** server hay replication không?
- Tốn bao nhiêu I/O, CPU, mạng trên server (backup load), có tùy chọn **throttle** không (tr. 262)?
- Bản backup nhất quán ở mức nào: một bảng, một database, toàn instance, kèm tọa độ binlog hay không?

**Kết quả đã đo** (thí nghiệm 1): cả `mysqldump` mặc định lẫn `--single-transaction` đều chạy "khi server online", nhưng chế độ mặc định làm tốc độ ghi rơi từ khoảng 780–955 xuống 109–163 hàng/giây. Hai thứ cùng được gọi là "online" có thể khác nhau rất xa.

[về mục 2.0](../chapter-10.md#s2-0)

<a id="q2-0-4"></a>

### Câu 2.0.4 — Vì sao thêm LVM lúc cần snapshot là quá muộn?

*Sách lấy LVM làm ví dụ cho một lựa chọn phải có từ lúc dựng server. Vì sao đến lúc cần snapshot mới thêm LVM là "quá muộn"?*

Snapshot LVM chỉ chụp được **logical volume** của LVM. Nếu datadir nằm trên một partition thường, muốn có snapshot phải tạo volume LVM mới rồi chuyển dữ liệu sang, nghĩa là migration và thường là downtime (tr. 257). Ngoài ra sách đòi các điều kiện chỉ thỏa được khi thiết kế từ đầu: mọi file InnoDB (tablespace và transaction log), thường cả datadir, nằm trên **một** logical volume, và volume group phải còn **chỗ chưa cấp phát** cho copy-on-write; sách dặn "khi dựng hệ thống, chừa chỗ" (tr. 274).

[về mục 2.0](../chapter-10.md#s2-0)

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Copy thư mục dữ liệu khi server đang chạy

*Một nhà cung cấp hosting nói họ "copy thư mục dữ liệu MySQL mỗi đêm, khi server vẫn chạy". Vì sao sách cho rằng backup này đáng ngờ? Trong điều kiện nào một bản copy file khi server chạy lại dùng được?*

Sách nói kiểu copy này **nhiều khả năng cho ra backup hỏng, vô dụng** (tr. 259). Cơ chế (**Bổ sung ngoài sách**): InnoDB giữ dirty page trong buffer pool, ghi xuống datafile dần dần; lệnh copy đọc các file ở thời điểm khác nhau, có thể đọc page đang ghi dở, và lấy redo log không khớp với datafile; crash recovery trên bản copy có thể thất bại hoặc cho dữ liệu sai.

Bản copy khi server chạy dùng được khi có một trong các cơ chế:

- **Snapshot nguyên tử** của mọi file tại một khoảnh khắc (mọi file InnoDB trên một volume), với MySQL 8+, chỉ InnoDB, GTID và full ACID, rồi để crash recovery chạy khi restore (tr. 274, 276).
- Công cụ **chép redo song song** và prepare như XtraBackup (tr. 270, 278).
- Hoặc đơn giản là **tắt server** trước (offline, tr. 261).

Và trong mọi trường hợp, chỉ tin backup sau khi đã khởi động một instance trên nó và chạy `CHECK TABLES` (tr. 265).

[về mục 2.1](../chapter-10.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Bốn lý do backup và phạm vi cần khôi phục

*Với mỗi lý do trong bốn lý do của sách, hãy nêu phạm vi cần khôi phục và mốc thời gian cần khôi phục.*

| Lý do (tr. 258–259) | Phạm vi | Mốc thời gian |
| --- | --- | --- |
| Disaster recovery | Thường là cả server, có khi cả data center | Càng gần lúc sự cố càng tốt (RPO nhỏ) |
| Người ta đổi ý | Một bảng, vài hàng, một database | Ngay trước lúc xóa; cần PITR |
| Kiểm toán | Dữ liệu hay schema của một phần hệ thống | Một thời điểm có thể rất cũ, nên cần lưu giữ dài hạn; sách khuyên logical backup định kỳ cho mục đích pháp lý (tr. 265) |
| Kiểm thử | Toàn bộ (hoặc phần lớn) dữ liệu | Gần đây, làm đều đặn |

Bảng cho thấy vì sao sách hỏi "cần phục hồi **cái gì**" như một câu hỏi riêng khi đặt RPO, RTO (tr. 260).

[về mục 2.1](../chapter-10.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Xác suất thấp không phải lý do bỏ backup

*Vì sao "xác suất từng loại thảm họa rất thấp" không phải là lý do để bỏ qua backup?*

Sách nói xác suất **một** loại thảm họa cụ thể khá thấp, nhưng **cộng lại** thì đáng kể (tr. 258). Danh sách rất dài: phần cứng, bug, kết nối nhầm server, cháy, tấn công, bug MySQL. Hơn nữa, có loại "sự cố" không hiếm chút nào: người ta xóa cố ý rồi muốn lấy lại xảy ra thường hơn bạn nghĩ (tr. 259).

**Bổ sung ngoài sách (phép tính).** Nếu có 10 loại sự cố độc lập, mỗi loại xác suất 1% một năm, xác suất gặp ít nhất một loại trong năm là 1 − 0,99¹⁰ ≈ 9,6%; qua 5 năm là 1 − 0,99⁵⁰ ≈ 39,5%. Trong khi hậu quả của việc mất dữ liệu thường không đảo ngược được.

[về mục 2.1](../chapter-10.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — RPO và RTO bị chi phối bởi gì?

*Phân biệt RPO và RTO. Đại lượng nào bị chi phối chủ yếu bởi tần suất sao lưu binlog, đại lượng nào bởi tốc độ restore?*

RPO là lượng dữ liệu chấp nhận mất; RTO là thời gian chấp nhận chờ để lấy lại (tr. 260).

- **RPO** khi mất trọn server bị chi phối bởi **tần suất đưa binlog ra khỏi máy** (sách: không chịu mất quá 30 phút thì backup binlog ít nhất 30 phút một lần, tr. 269); nếu không có binlog thì bởi chu kỳ backup (tr. 261).
- **RTO** bị chi phối bởi **recovery time**: chép backup về, restore (nạp dump hay chép file và crash recovery), phát lại binlog, kiểm tra (tr. 262, 281). Với logical backup, đoạn nạp lại thường lớn nhất và khó đoán (tr. 264).

Lưu ý (**Bổ sung ngoài sách**): cả hai phụ thuộc loại sự cố. Với `DROP TABLE` khi server còn sống, RPO có thể gần 0 nhờ binlog tại chỗ; với mất data center, RPO là đoạn binlog chưa kịp ra offsite.

[về mục 2.2](../chapter-10.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Vì sao backup dễ hơn phục hồi, và cách bù

*Sách nêu năm lý do khiến làm backup dễ hơn xây quy trình phục hồi. Chọn ba lý do và đề xuất một biện pháp cụ thể cho mỗi lý do.*

Năm lý do (tr. 259–260): backup đến trước; backup được tự động hóa và tinh chỉnh liên tục; backup là thường nhật còn phục hồi là khủng hoảng; bảo mật cản đường (volume mã hóa không ai mở được, phải lấy một file từ file mã hóa khổng lồ); một người thiết kế backup có thể vắng mặt lúc cần.

Biện pháp (**Bổ sung ngoài sách**, theo tinh thần khuyến nghị ở tr. 261 và Summary tr. 286):

- **Thường nhật so với khủng hoảng** → biến phục hồi thành thường nhật: diễn tập tự động hằng ngày vào staging, đo thời gian.
- **Bảo mật cản đường** → đưa bước lấy khóa giải mã và trích một bảng từ backup mã hóa vào chính bài diễn tập; lưu khóa theo quy trình có nhiều người truy cập được.
- **Một người** → runbook viết sẵn; luân phiên người chạy diễn tập; đào tạo nhiều người (sách nói rõ "train several people", tr. 260).

[về mục 2.2](../chapter-10.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Phép thử `DROP DATABASE`

*Áp dụng phép thử `DROP DATABASE` cho: RAID 10, một replica bất đồng bộ, một replica trễ 1 giờ. Cái nào qua được, trong điều kiện nào?*

- **RAID 10: trượt.** RAID nhân bản block; lệnh `DROP` được ghi xuống mọi đĩa trong mirror (tr. 260).
- **Replica bất đồng bộ: trượt.** Lệnh có trong binlog và được replica áp trong vòng vài giây (tr. 260). Thí nghiệm 3, bước 7C: phát lại toàn bộ binlog thì bảng lại mất.
- **Replica trễ 1 giờ: qua có điều kiện.** Chỉ qua nếu sự cố được **phát hiện** và replica được **dừng** trước khi nó áp sự kiện (trong 1 giờ), và nếu replica nhất quán với source (tr. 268). Phát hiện sau 2 giờ thì cũng trượt. Sách vẫn nói rõ replication không phải backup (tr. 268).

[về mục 2.2](../chapter-10.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Khôi phục vài hàng so với khôi phục toàn server

*Yêu cầu "khôi phục đúng các đơn hàng bị xóa nhầm lúc 14:05" khác yêu cầu "khôi phục toàn server" ở điểm nào khi thiết kế backup?*

Sách liệt kê "cần phục hồi cái gì" là câu hỏi riêng: cả server, một database, một bảng, hay vài transaction, câu lệnh (tr. 260).

- **Vài hàng ở một thời điểm** cần **point-in-time recovery**: backup cộng binlog liên tục (tr. 261, 269), và khả năng trích **một phần** dữ liệu: restore ra server riêng rồi trích bảng, hay logical backup tách theo bảng để lấy riêng một bảng (tr. 283). Thời gian lấy lại phụ thuộc kích thước bảng hơn là kích thước server.
- **Toàn server** cần cách restore nhanh khối lớn: raw backup (tr. 260–261), cùng cấu hình server và file hệ điều hành (tr. 266). PITR là tùy chọn, tùy RPO.

Một thiết kế tốt thường cần cả hai: raw backup cho toàn server, binlog cho PITR, logical backup định kỳ để trích từng bảng (tr. 265).

[về mục 2.2](../chapter-10.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — Tốc độ tối thiểu và cách giảm backup load

*Database 1,5 TB, ban đêm server chỉ tải 50% trong 8 giờ. Tính tốc độ chép tối thiểu để backup xong trong cửa sổ, và nêu ba cách sách gợi ý để giữ backup load dưới 50%.*

**Phép tính (Bổ sung ngoài sách):** 1,5 × 10¹² B / (8 × 3 600 s) = 1,5 × 10¹² / 28 800 ≈ 52 × 10⁶ B/s ≈ **52 MB/s** trung bình, chưa tính biên cho lúc tải tăng.

Cách giữ load dưới 50% (tr. 262): dùng `ionice` và `nice` cho việc chép hay nén; đổi mức nén hoặc **nén trên backup server**; dùng `lzo` hay `pigz` để nén nhanh; dùng `O_DIRECT` hay `fadvise()` để không làm bẩn cache của hệ điều hành; dùng tùy chọn throttle của XtraBackup, MySQL Enterprise Backup, hoặc `pv --rate-limit` cho script tự viết. Ngoài ra có thể chụp backup từ **replica** để không chạm vào source (tr. 268).

Đánh đổi: giới hạn tốc độ thấp hơn làm backup kéo dài; nếu dùng snapshot thì snapshot phải mở lâu hơn và cần nhiều chỗ copy-on-write hơn (tr. 277).

[về mục 2.3](../chapter-10.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Khi nào logical lớn hơn hay nhỏ hơn raw?

*Khi nào logical backup lớn hơn raw backup, khi nào nhỏ hơn? Dựa vào cơ chế nào?*

- **Logical lớn hơn** khi biểu diễn văn bản kém hiệu quả hơn cách engine lưu: một số nguyên 4 byte có thể thành tới 12 ký tự ASCII (tr. 264). Bảng nhiều cột số, ít index, page đã đầy đặn dễ rơi vào trường hợp này.
- **Logical nhỏ hơn** khi bảng có **nhiều index** (dump chỉ chứa dữ liệu hàng và câu `CREATE TABLE`, index được dựng lại khi nạp) và khi tablespace có nhiều **chỗ trống** hay vùng dùng cho việc khác như insert buffer, rollback segment (tr. 264–265). Chú thích 4 (tr. 264): theo kinh nghiệm tác giả, logical thường nhỏ hơn, nhưng không phải luôn luôn.

**Kết quả đã đo** (thí nghiệm 2): bảng 1 triệu hàng có 2 secondary index: `.ibd` 140 MB, dump 61,5 MB; cùng dữ liệu không secondary index: `.ibd` 80 MB, dump vẫn 61,5 MB. Phần lớn chênh lệch đến từ index; phần còn lại từ chỗ trống và overhead của page.

[về mục 2.3](../chapter-10.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Vì sao thời gian restore logical khó đoán?

*Vì sao sách gọi thời gian restore logical là "khó đoán trước"? Giải thích bằng những việc server phải làm khi nạp dump, và vai trò của kích thước dữ liệu so với bộ nhớ.*

Khi nạp dump, server phải **parse và thông dịch** từng câu lệnh, **chuyển** sang định dạng lưu trữ, và **dựng lại index** (tr. 264); nếu binlog bật thì ghi thêm binlog (tr. 282). `mysqldump` và việc nạp qua `mysql` là **đơn luồng** (tr. 269). Thời gian vì vậy phụ thuộc phần cứng, schema, index và dữ liệu (tr. 286).

Phần "khó đoán" đến từ quan hệ với bộ nhớ: khi bảng InnoDB **không vừa bộ nhớ**, mỗi lần chèn vào secondary index có thể cần đọc page từ đĩa (**Bổ sung ngoài sách**: chèn vào index theo thứ tự ngẫu nhiên), nên tốc độ có thể rơi mạnh khi dữ liệu lớn lên; sách nói lúc đó raw restore có thể nhanh hơn **một bậc độ lớn hoặc hơn** (tr. 264).

**Kết quả đã đo** (thí nghiệm 2): restore chậm hơn dump 11–16 lần ở 250 000 tới 2 triệu hàng. Ở quy mô nhỏ này tốc độ nạp chưa rơi theo kích thước, nên không thể ngoại suy tuyến tính lên dữ liệu lớn hơn nhiều lần bộ nhớ: phải đo bằng diễn tập.

[về mục 2.3](../chapter-10.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — Chuỗi incremental có một bản hỏng

*Full backup Chủ nhật, backup hằng ngày tới thứ Bảy. Bản thứ Tư bị hỏng, thứ Sáu cần restore. Với differential và với incremental, bạn phục hồi được tới đâu? Binlog thay đổi kết luận thế nào?*

- **Differential** (đổi từ full gần nhất, tr. 266): thứ Sáu cần Full Chủ nhật + Diff thứ Năm. Bản thứ Tư hỏng **không ảnh hưởng**; phục hồi được tới thời điểm backup thứ Năm (hay thứ Sáu nếu đã có).
- **Incremental** (đổi từ backup gần nhất bất kỳ, tr. 266): cần Full + thứ Hai + thứ Ba + thứ Tư + thứ Năm theo đúng thứ tự. Bản thứ Tư hỏng làm **đứt chuỗi**: chỉ phục hồi được tới thời điểm backup thứ Ba. Đây đúng là rủi ro "nhiều thế hệ gắn chặt nhau, một thế hệ hỏng làm vô hiệu các thế hệ khác" (tr. 267).
- **Có binlog** liên tục từ thời điểm backup thứ Ba trở đi: restore tới thứ Ba rồi phát lại binlog tới bất kỳ thời điểm nào sau đó, kể cả ngay trước sự cố (tr. 261, 269). Binlog vì vậy là đường lui cho chuỗi incremental, với điều kiện chuỗi binlog cũng không đứt và được giữ đủ lâu (tr. 261).

[về mục 2.3](../chapter-10.md#s2-3)

<a id="q2-3-5"></a>

### Câu 2.3.5 — Điều kiện để backup từ replica đáng tin

*Liệt kê điều kiện để một backup chụp từ replica là đáng tin, và lý do của từng điều kiện.*

| Điều kiện | Lý do |
| --- | --- |
| Replica **nhất quán** với source: dựng lại từ source, bật `super_read_only`, kiểm tra bằng `pt-table-checksum` | Lệch dữ liệu trên replica là chuyện thường và MySQL không tự phát hiện; backup một replica lệch là backup dữ liệu sai (tr. 268, 286) |
| Không ghi gì vào replica ngoài replication | Ghi vào là đổi chính dữ liệu đang backup (tr. 268) |
| Dùng **GTID** | Backup tự mang vị trí của nó; dễ nhân bản replica, áp binlog cho PITR, thăng replica (tr. 268) |
| Replica **đuổi kịp** source trước lần backup | Nếu không, backup cũ hơn bạn tưởng, RPO thực tế lớn hơn (tr. 268) |
| Không có temporary table mở khi dừng replication | Có thể không khởi động lại replication được (tr. 268) |
| **Không hạ durability** trên replica để giảm lag | Chương 9 cảnh báo có thể không phục hồi được từ backup chụp ở đó (chương 9, tr. 253) |
| Không dùng replication filter | Chương 9 nói filter làm PITR không còn khả thi (chương 9, tr. 238) |

[về mục 2.3](../chapter-10.md#s2-3)

---

<a id="q2-4-1"></a>

### Câu 2.4.1 — Binlog không bao giờ bị xóa trên server ít ghi

*Server ít ghi, `binlog_expire_logs_seconds` đặt 3 ngày, file binlog hiện tại sau hai tuần vẫn chưa đầy 1 GB. Binlog cũ có bị xóa không? Vì sao, và bạn xử lý thế nào?*

**Không** (trừ khi server khởi động lại). Thời hạn chỉ được xét khi server khởi động hoặc khi binlog **xoay vòng**; nếu binlog không bao giờ đầy để xoay vòng thì server không xóa gì (tr. 269). **Kết quả đã đo** (thí nghiệm 5): đặt thời hạn 2 giây lúc runtime không xóa file nào; chỉ ở lần `FLUSH BINARY LOGS` kế tiếp, các file có thời gian sửa cũ hơn thời hạn mới bị xóa, còn file vừa bị sửa thì được giữ.

Xử lý (**Bổ sung ngoài sách**): chạy `FLUSH BINARY LOGS` định kỳ (cũng giúp sao lưu binlog theo từng file đã đóng), hoặc giảm `max_binlog_size`, hoặc dùng `PURGE BINARY LOGS BEFORE ...` có kiểm soát. Không xóa file bằng tay (tr. 269).

[về mục 2.4](../chapter-10.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — RPO 30 phút và RPO của `DROP TABLE`

*RPO là 30 phút cho kịch bản mất trọn server. Bạn sao lưu binlog thế nào? Với kịch bản `DROP TABLE` nhầm khi server vẫn sống, RPO thực tế là bao nhiêu?*

- Mất trọn server: sao lưu binlog ra máy khác **ít nhất mỗi 30 phút** (tr. 269); an toàn hơn là kéo liên tục bằng `mysqlbinlog --read-from-remote-server --raw --stop-never` (**Bổ sung ngoài sách**), khi đó RPO chỉ còn vài giây. Lưu ý file đang ghi dở chỉ được sao lưu nếu dùng cách kéo liên tục, hoặc nếu bạn xoay vòng binlog trước khi chép.
- `DROP TABLE` khi server vẫn sống: binlog trên server còn nguyên, nên có thể phát lại tới **ngay trước** lệnh hỏng. RPO thực tế gần **0**. **Kết quả đã đo** (thí nghiệm 3): cả `orders` (tới ngay trước `DROP`) và `customers` (kể cả các ghi sau sự cố) đều khớp checksum với trạng thái mong đợi.

[về mục 2.4](../chapter-10.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Vì sao không xóa binlog bằng tay?

*Vì sao sách dặn không xóa binlog bằng tay? Nêu ít nhất hai hậu quả có thể xảy ra.*

Sách chỉ nói: dùng `binlog_expire_logs_seconds` để MySQL tự xóa, không xóa file bằng tay (tr. 269). Hậu quả có thể (**Bổ sung ngoài sách**):

1. File `binlog.index` vẫn liệt kê file đã mất, nên `SHOW BINARY LOGS`, `PURGE` hay việc khởi động có thể báo lỗi hoặc cư xử bất thường.
2. Replica đang đọc (hoặc sẽ cần đọc) file đó bị đứt replication.
3. Chuỗi binlog cho PITR bị thủng; nếu file đó chưa được sao lưu thì không phát lại qua được đoạn ấy.

`PURGE BINARY LOGS TO` hay `BEFORE` cập nhật index và không xóa file replica đang đọc, nên là cách an toàn để xóa thủ công khi cần.

[về mục 2.4](../chapter-10.md#s2-4)

<a id="q2-4-4"></a>

### Câu 2.4.4 — Giữ binlog bao lâu?

*Liên hệ giữa thời gian giữ binlog và số thế hệ backup theo khuyến nghị ở tr. 261. Full backup mỗi tuần thì nên giữ binlog bao lâu?*

Sách khuyên giữ binlog trên server **đủ để phục hồi từ ít nhất hai thế hệ raw backup**, để có thể dựng replica từ backup rồi cho nó theo source đang sống mà không phải tự áp binlog; và sao lưu binlog **độc lập**, giữ trong kho backup ít nhất tới bản logical backup gần nhất (tr. 261). Summary nhắc lại: giữ đủ thế hệ backup và binlog để phục hồi hay dựng replica ngay cả khi backup mới nhất không dùng được (tr. 286).

Với full backup mỗi tuần (**Bổ sung ngoài sách, phép tính**): hai thế hệ là khoảng 14 ngày, cộng biên (ví dụ thêm vài ngày phòng backup trễ hay hỏng), nên khoảng 16–21 ngày. Mặc định 30 ngày của MySQL 8.0 (đã đo trên 8.0.42) thỏa yêu cầu này, nhưng phải kiểm đĩa đủ chỗ với workload ghi nhiều (row-based binlog lớn hơn, tr. 269).

[về mục 2.4](../chapter-10.md#s2-4)

---

<a id="q2-5-1"></a>

### Câu 2.5.1 — Vì sao bản sao "lệch thời điểm" của XtraBackup lại nhất quán?

*XtraBackup chép datafile ở các thời điểm khác nhau. Giải thích vì sao sau bước prepare, backup vẫn nhất quán, và bản backup phản ánh thời điểm nào.*

Trong lúc chép datafile, một luồng nền đọc đuổi theo redo log (tr. 270). XtraBackup ghi lại LSN và dùng nó để chạy crash recovery trên file đã chép (tr. 278). Bước **prepare** chính là crash recovery của InnoDB trên bản sao: với mỗi page, áp các thay đổi trong redo có LSN lớn hơn LSN của page, đưa mọi page về cùng một điểm cuối (tr. 270, 285). Kết quả nhất quán và chứa mọi transaction đã commit **tính tới lúc kết thúc quá trình chép file** (tr. 270), tức thời điểm luồng chép log dừng, không phải lúc bắt đầu backup.

**Bổ sung ngoài sách:** đây là cùng nguyên lý với phục hồi sau mất điện; transaction chưa commit tại điểm cuối được rollback bằng undo.

[về mục 2.5](../chapter-10.md#s2-5)

<a id="q2-5-2"></a>

### Câu 2.5.2 — XtraBackup có cần kết nối MySQL không?

*Tr. 270 nói XtraBackup không cần kết nối MySQL, nhưng output ở tr. 278 cho thấy nó chạy lệnh SQL. Giải thích hai phát biểu này cùng đúng ở mức nào.*

Phần **chép dữ liệu** InnoDB (đọc file, đọc đuổi redo log, và về sau là prepare) diễn ra bên ngoài server, không qua giao thức SQL: đó là ý của tr. 270. Nhưng để bản backup có **thông tin replication khớp với dữ liệu**, XtraBackup "khóa ở một số điểm" (tr. 278): output cho thấy nó chạy `LOCK TABLES FOR BACKUP`, `FLUSH NO_WRITE_TO_BINLOG BINARY LOGS`, đọc LSN và vị trí binlog từ `p_s.log_status`, `FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS`, rồi `UNLOCK TABLES` (tr. 278–279). Vậy: chép dữ liệu thì không cần kết nối; phối hợp tọa độ và khóa ngắn thì có. Sách cũng dặn dùng `--user`, `--password` với tài khoản đủ quyền (tr. 279), điều chỉ có nghĩa nếu công cụ có kết nối.

[về mục 2.5](../chapter-10.md#s2-5)

<a id="q2-5-3"></a>

### Câu 2.5.3 — Song song hóa và giới hạn của nó

*Vì sao mydumper (hay một công cụ song song bất kỳ) nhanh hơn `mysqldump`, và song song hóa bị giới hạn bởi điều gì?*

`mysqldump` đơn luồng (tr. 269); mydumper dùng nhiều luồng cho cả backup lẫn restore (tr. 270), nên dùng được nhiều lõi CPU và băng thông I/O cùng lúc. **Kết quả đã đo** (thí nghiệm 2): dump 3 database (2,25 triệu hàng) mất 3,84 s với `mysqldump` và 1,98 s với `mysqlpump` 4 luồng.

Giới hạn (**Bổ sung ngoài sách**): (1) bảng lớn nhất, nếu công cụ không chia bảng theo khoảng khóa; (2) băng thông đĩa, CPU và mạng chung; (3) càng song song thì **backup load** càng cao, ngược với mục tiêu giữ tải thấp (tr. 262); (4) restore song song vẫn phải dựng index và ghi redo; (5) các luồng phải cùng nhìn một snapshot nhất quán, đòi phối hợp lúc bắt đầu.

[về mục 2.5](../chapter-10.md#s2-5)

---

<a id="q2-6-1"></a>

### Câu 2.6.1 — Ba chế độ của `mysqldump` ở thí nghiệm 1

*Ở thí nghiệm 1, bản dump `--skip-lock-tables` lệch +132 tới +257, còn hai chế độ kia lệch 0. Giải thích cơ chế của từng chế độ và vì sao chỉ một chế độ vừa đúng vừa không chặn ghi.*

- **`--skip-lock-tables`**: mỗi bảng được đọc bằng một câu `SELECT` riêng, mỗi câu thấy snapshot tại lúc nó bắt đầu. `checking` được đọc trước, `savings` đọc sau; các lần chuyển tiền commit ở giữa đã được cộng vào `savings` nhưng chưa bị trừ trong bản `checking` đã chụp, nên tổng **dư** đúng bằng số lần chuyển đó.
- **Mặc định**: `LOCK TABLES ... READ` cho mọi bảng của database trước khi đọc; luồng ghi phải chờ, nên dữ liệu không đổi trong lúc dump: đúng nhưng **chặn ghi** (tốc độ ghi rơi xuống 109–163 hàng/giây).
- **`--single-transaction`**: `START TRANSACTION WITH CONSISTENT SNAPSHOT` ở mức REPEATABLE READ; mọi câu `SELECT` sau đó đọc từ **cùng một** read view MVCC, bất kể ghi đồng thời; InnoDB đọc nhất quán không cần khóa hàng, nên luồng ghi vẫn chạy (849–897 hàng/giây).

Chỉ chế độ thứ ba vừa đúng vừa không chặn ghi, nhờ MVCC của InnoDB. Giới hạn (**Bổ sung ngoài sách**): chỉ áp dụng cho engine có MVCC, và không bảo vệ trước DDL đồng thời. Sách đặt nền cho đánh đổi này khi nói backup online vẫn có thể gián đoạn dịch vụ tùy yêu cầu nhất quán (tr. 262).

[về mục 2.6](../chapter-10.md#s2-6)

<a id="q2-6-2"></a>

### Câu 2.6.2 — Vì sao snapshot không phải backup?

*Vì sao snapshot copy-on-write "không phải backup"? Dựng một kịch bản cụ thể mà mọi snapshot đều vô dụng.*

Snapshot copy-on-write chỉ chứa **phần khác biệt** giữa dữ liệu sống và dữ liệu lúc chụp; các block chưa sửa vẫn **dùng chung** với volume gốc. Nếu một block chưa sửa bị hỏng trên volume gốc, mọi snapshot đều thấy cùng block hỏng (tr. 272).

**Ví dụ minh họa do AI xây dựng.** Một bảng lưu trữ ít khi ghi nằm trên các block không đổi suốt nhiều tuần. Một lỗi firmware làm hỏng vài sector chứa bảng đó. Bảy snapshot hằng ngày đều trỏ về đúng các sector ấy cho phần dữ liệu không đổi, nên cả bảy đều đọc ra dữ liệu hỏng. Thêm nữa, nếu cả mảng đĩa hỏng hay kẻ tấn công có quyền root xóa volume group, snapshot mất cùng volume. Vì vậy sách nói dùng snapshot để "đóng băng" dữ liệu trong lúc **chép ra** backup (tr. 272).

[về mục 2.6](../chapter-10.md#s2-6)

<a id="q2-6-3"></a>

### Câu 2.6.3 — Tính dung lượng snapshot

*Server ghi 20 MB/giây, ước lượng 40% lượng ghi rơi vào block chưa chép, việc chép snapshot sang server khác mất 90 phút. Cần dành bao nhiêu chỗ cho snapshot theo cách của sách? Nếu bạn throttle việc chép xuống còn một nửa tốc độ thì sao?*

Theo cách tính ở tr. 277 (tỷ lệ × tốc độ ghi × thời gian giữ snapshot):

- 0,4 × 20 MB/s × 5 400 s = **43 200 MB ≈ 43,2 GB**, cộng biên an toàn (sách: "err on the side of caution").
- Chép chậm một nửa thì thời gian giữ snapshot gấp đôi (10 800 s): 0,4 × 20 × 10 800 = **86 400 MB ≈ 86,4 GB**.

**Bổ sung ngoài sách:** đây là ước lượng an toàn vì tỷ lệ ghi rơi vào block chưa chép giảm dần theo thời gian (mỗi block chỉ bị chép một lần, tr. 277). Ví dụ cho thấy đánh đổi ẩn: giảm backup load bằng throttle (tr. 262) làm tăng rủi ro snapshot hết chỗ, khiến job backup lỗi I/O giữa chừng (tr. 273).

[về mục 2.6](../chapter-10.md#s2-6)

<a id="q2-6-4"></a>

### Câu 2.6.4 — Ba điều kiện của snapshot không khóa

*Sách nói backup bằng snapshot LVM không khóa chỉ "dễ" khi có ba điều kiện. Nêu ba điều kiện và hậu quả nếu thiếu từng điều kiện.*

Điều kiện (tr. 276): MySQL 8+ chỉ dùng bảng **InnoDB**; dùng **GTID**; chế độ **ACID đầy đủ**. Kèm điều kiện hạ tầng: mọi file InnoDB và datadir trên một logical volume (tr. 274).

Hậu quả nếu thiếu (**Bổ sung ngoài sách**):

- **Có bảng không phải InnoDB** (hay bản trước 8.0 với file `.frm`): các file này không được crash recovery bảo vệ, snapshot giữa lúc ghi có thể cho bảng hỏng hay lệch; phải khóa (ví dụ `FLUSH TABLES WITH READ LOCK`) trong lúc chụp.
- **Không có GTID**: bản restore không tự biết nó ứng với điểm nào trong binlog; phải khóa ngắn để ghi lại tọa độ binlog lúc chụp, nếu không thì không dựng replica hay làm PITR chính xác được.
- **Không full ACID** (`sync_binlog` khác 1, `innodb_flush_log_at_trx_commit` khác 1): transaction đã báo commit có thể chưa nằm trên đĩa lúc chụp, hoặc binlog và redo lệch nhau; bản restore có thể thiếu transaction hay không khớp binlog. Chương 9 cảnh báo điều này với replica dùng để backup (chương 9, tr. 253).

[về mục 2.6](../chapter-10.md#s2-6)

<a id="q2-6-5"></a>

### Câu 2.6.5 — Vì sao vị trí binlog phải khớp LSN

*Trong output XtraBackup, vì sao phải lấy vị trí binlog cùng lúc với LSN, và điều gì xảy ra nếu hai tọa độ này lệch nhau?*

Vị trí binlog nói "bản backup đã chứa mọi transaction tới đây"; LSN nói "dữ liệu InnoDB được phục hồi tới đây". Khi dùng backup để làm PITR hay dựng replica, bạn bắt đầu phát lại binlog từ đúng vị trí đó (tr. 268–269). XtraBackup khóa ở một số điểm để thông tin replication **khớp** với dữ liệu (tr. 278), và đọc cả hai từ `p_s.log_status` (tr. 278).

Nếu lệch (**Bổ sung ngoài sách**): vị trí binlog **sớm hơn** dữ liệu thì một số transaction bị áp **hai lần** (có thể lỗi trùng khóa, hay âm thầm sai với các câu cập nhật tương đối); vị trí **muộn hơn** thì một số transaction bị **bỏ sót**. Cả hai cho ra dữ liệu sai mà không báo. **Kết quả đã đo** (thí nghiệm 5): `performance_schema.log_status` trả về trong một lần đọc cả `gtid_executed`, file và vị trí binlog, và LSN InnoDB.

[về mục 2.6](../chapter-10.md#s2-6)

---

<a id="q2-7-1"></a>

### Câu 2.7.1 — Vì sao cô lập server khi restore

*Vì sao phải khởi động MySQL ở chế độ hạn chế truy cập khi restore, nhất là với logical backup? Nêu hai cách làm.*

Trong lúc phục hồi, cần MySQL không truy cập được với mọi thứ trừ tiến trình phục hồi, tới khi đã kiểm tra xong; điều này đặc biệt quan trọng với logical backup vì chúng được **nạp từng phần** (tr. 282). Ứng dụng kết nối giữa chừng sẽ đọc dữ liệu dở dang, hoặc ghi xen vào làm hỏng kết quả restore và làm khó việc kiểm tra.

Cách làm: khởi động với `--skip-networking` và một `--socket` riêng (tr. 282); hoặc (**Bổ sung ngoài sách**) bật `offline_mode`, chặn ở firewall, hay restore lên một instance riêng ở cổng riêng chỉ nghe localhost (cách bài thực hành dùng).

[về mục 2.7](../chapter-10.md#s2-7)

<a id="q2-7-2"></a>

### Câu 2.7.2 — Nạp 200 GB trong một transaction

*Bạn restore một dump 200 GB bằng một transaction duy nhất, binlog bật. Những vấn đề nào có thể xảy ra, và sửa quy trình thế nào?*

Vấn đề:

- **Rollback segment phình khổng lồ** (tr. 282); nếu lỗi giữa chừng, rollback cũng rất lâu.
- **Binlog** thêm chi phí, có thể không cần (tr. 282). **Bổ sung ngoài sách:** một transaction khổng lồ phải được đệm toàn bộ trước khi ghi binlog lúc commit, có thể chạm giới hạn `max_binlog_cache_size`, và tạo một sự kiện khổng lồ làm replica trễ lớn.
- Không biết trước mất bao lâu, nên RTO không kiểm soát được (tr. 264).

Sửa: nạp theo **khúc vừa phải, commit sau mỗi khúc** (tr. 282); tắt binlog cho phiên nạp nếu không cần replicate (`SET SQL_LOG_BIN = 0`, tr. 283); giải nén và nạp trong một pipeline (tr. 283); cô lập server (tr. 282); thông báo người dùng, đo thời gian nạp trong diễn tập. **Kết quả đã đo** (thí nghiệm 2): tắt binlog cho phiên nạp 1 triệu hàng giảm thời gian từ 22,5 s xuống 13,6 s. `mysqldump` mặc định đã chia dữ liệu thành nhiều câu `INSERT` khoảng 1 MB (quan sát ở thí nghiệm 2); với autocommit mặc định của client, mỗi câu là một transaction riêng, nên tránh được bẫy "một transaction" (**Bổ sung ngoài sách**).

[về mục 2.7](../chapter-10.md#s2-7)

<a id="q2-7-3"></a>

### Câu 2.7.3 — Phát lại binlog bỏ qua đúng lệnh `DROP`

*Sau một `DROP TABLE` nhầm lúc 14:05, ứng dụng vẫn ghi vào các bảng khác tới 14:30. Mô tả cách phát lại binlog để lấy lại bảng mà không mất các thay đổi hợp lệ sau 14:05, theo vị trí và theo GTID.*

Nền tảng của sách: restore backup rồi phát lại binlog tới thời điểm mong muốn (tr. 261, 269); không ghi đè binlog hiện tại bằng bản cũ (tr. 282). Cách làm cụ thể là **Bổ sung ngoài sách**, đã chạy ở thí nghiệm 3:

1. Restore backup lên **server phục hồi riêng**, khởi động hạn chế truy cập.
2. Tìm sự kiện: `SHOW BINLOG EVENTS` hay `mysqlbinlog -v` để lấy vị trí bắt đầu của sự kiện GTID ngay trước câu `DROP`, `End_log_pos` của câu `DROP`, và GTID của nó.
3. **Theo vị trí:** `mysqlbinlog --start-position=<tọa độ trong backup> --stop-position=<vị trí sự kiện GTID của DROP>` đưa vào `mysql`; rồi `mysqlbinlog --start-position=<End_log_pos của DROP>` cho phần sau (nhớ xử lý mọi file binlog tiếp theo).
4. **Theo GTID:** `mysqlbinlog --exclude-gtids='<uuid>:<n>'` cho các file binlog từ điểm backup; transaction đã có trong `GTID_PURGED` của backup được server tự bỏ qua.
5. Kiểm tra (checksum, số hàng), rồi hoặc trích riêng bảng bị mất đưa về production, hoặc chuyển ứng dụng sang server phục hồi.

**Kết quả đã đo:** cả hai cách cho checksum khớp tuyệt đối cho cả hai bảng; `gtid_executed` có một lỗ đúng tại GTID của `DROP`.

[về mục 2.7](../chapter-10.md#s2-7)

<a id="q2-7-4"></a>

### Câu 2.7.4 — Tablespace và transaction log không khớp

*Vì sao InnoDB từ chối khởi động khi tablespace và transaction log không khớp? Điều này nói gì về cách backup raw file?*

Sách nói nếu thay tablespace mà không thay transaction log (hay ngược lại), InnoDB sẽ từ chối khởi động; đó là lý do phải backup transaction log **cùng** datafile (tr. 284). Cơ chế (**Bổ sung ngoài sách**): crash recovery dựa vào LSN; redo log mô tả thay đổi từ một checkpoint ứng với trạng thái của datafile. Nếu datafile "mới hơn" log hay log thuộc về một bộ datafile khác, InnoDB không thể biết phải áp gì và áp từ đâu mà vẫn bảo đảm nhất quán, nên nó dừng thay vì đoán.

Hệ quả cho backup raw: mọi file InnoDB (tablespace, redo, undo) phải được lấy ở **cùng một thời điểm**: tắt server (tr. 261), snapshot nguyên tử của **một** volume chứa tất cả (tr. 274), hay công cụ chép redo và prepare (tr. 270). Sách cũng nói lý tưởng là backup cả datadir cùng lúc (tr. 265).

[về mục 2.7](../chapter-10.md#s2-7)

<a id="q2-7-5"></a>

### Câu 2.7.5 — Prepare sớm hay prepare lúc restore

*So sánh prepare ngay sau backup với prepare lúc restore của XtraBackup về RTO và về rủi ro.*

Sách: nếu không dùng streaming, có thể prepare ngay sau backup, khi đó bạn lưu một backup đã prepare và bớt việc phải làm lúc restore (tr. 285).

| | Prepare ngay sau backup | Prepare lúc restore |
| --- | --- | --- |
| RTO | Ngắn hơn: lúc khẩn cấp chỉ còn chép file và khởi động | Dài hơn: thêm giải nén, giải mã (nếu có) và crash recovery (tr. 284–285) |
| Phát hiện backup hỏng | Sớm: prepare lỗi là biết ngay, khi còn thời gian làm lại (**Bổ sung ngoài sách**) | Muộn: chỉ biết lúc đang khủng hoảng |
| Ràng buộc | Không dùng được với streaming (tr. 285); tốn CPU, I/O ở máy backup ngay sau backup | Linh hoạt lưu trữ dạng một file nén, mã hóa |

Lựa chọn phụ thuộc RTO: nếu RTO chặt, nên prepare sớm, hoặc ít nhất prepare trong diễn tập hằng ngày để vừa đo vừa kiểm.

[về mục 2.7](../chapter-10.md#s2-7)

---

<a id="q2-8-1"></a>

### Câu 2.8.1 — Hai phép thử đòi hỏi gì khác nhau

*Phép thử `DROP TABLE` và phép thử "mất data center" đòi hỏi những thuộc tính khác nhau nào của hệ thống backup?*

- **Phép thử `DROP TABLE`** (và "bị hack", tr. 286) đòi **bản sao ở quá khứ**, độc lập với dữ liệu đang sống: backup theo thời điểm cộng binlog để làm PITR (tr. 261, 269), khả năng trích một phần dữ liệu (tr. 283), và (với "bị hack") **tách biệt quyền truy cập** giữa production và kho backup (tr. 261).
- **Phép thử mất data center** đòi **khoảng cách**: backup và binlog ở nơi khác, cùng cấu hình server và file hệ điều hành để dựng lại từ đầu (tr. 266), và quy trình phục hồi chạy được mà không dựa vào hạ tầng đã mất.

Replica ở data center khác có thể qua phép thử thứ hai nhưng trượt phép thử thứ nhất; snapshot trên cùng mảng đĩa có thể qua phép thử thứ nhất nhưng trượt phép thử thứ hai (tr. 260, 272, 286).

[về mục 2.8](../chapter-10.md#s2-8)

<a id="q2-8-2"></a>

### Câu 2.8.2 — Quy trình thử phục hồi mỗi ngày

*Thiết kế một quy trình "thử phục hồi mỗi ngày" theo gợi ý của Summary: gồm bước nào, đo chỉ số gì, cảnh báo khi nào?*

Summary gợi ý thử phục hồi mỗi ngày bằng cách restore backup vào server development hay staging, rồi dump từ instance đó để có logical backup (tr. 286). Một thiết kế (**Ví dụ minh họa do AI xây dựng**):

1. Lấy backup mới nhất; restore vào staging (prepare nếu cần, sửa quyền, khởi động hạn chế truy cập).
2. Phát lại binlog tới một mốc (ví dụ cuối ngày hôm trước).
3. Kiểm tra: khởi động thành công, `CHECK TABLES` hay `mysqlcheck` (tr. 265), `SHOW TABLE STATUS` ở từng database và xem error log (tr. 286), so số hàng hay checksum vài bảng then chốt với nguồn.
4. Dump logical từ staging cho lưu trữ dài hạn (tr. 265, 286).

Chỉ số: thời gian từng bước và tổng (RTO thật), kích thước backup, thành công hay thất bại, tuổi backup. Cảnh báo khi: diễn tập thất bại; tổng thời gian vượt một ngưỡng so với RTO (ví dụ 80%); backup quá cũ. Giám sát **độc lập** với công cụ backup (tr. 261), lưu chỉ số ít nhất một năm (chương 2, tr. 35), và lưu bằng chứng cho kiểm toán (chương 13, tr. 339).

[về mục 2.8](../chapter-10.md#s2-8)

<a id="q2-8-3"></a>

### Câu 2.8.3 — Backup vài giờ, restore hàng tuần

*Vì sao "backup xong trong vài giờ có thể mất hàng tuần để restore"? Liệt kê các yếu tố sách nêu và cơ chế của mỗi yếu tố.*

Sách nêu: phần cứng, schema, index và dữ liệu (tr. 286). Cơ chế:

- **Bất đối xứng công việc**: dump là đọc và in ra; restore logic phải parse SQL, chuyển định dạng, dựng index (tr. 264), ghi binlog nếu bật (tr. 282).
- **Index**: mỗi secondary index phải được cập nhật cho mỗi hàng; nhiều index thì càng chậm (tr. 264).
- **Dữ liệu so với bộ nhớ**: khi bảng không vừa bộ nhớ, việc cập nhật index cần I/O ngẫu nhiên; raw restore có thể nhanh hơn một bậc độ lớn (tr. 264).
- **Đơn luồng**: `mysqldump` và nạp qua `mysql` đơn luồng (tr. 269).
- **Dữ liệu tăng dần**: job backup vẫn "xanh" trong khi thời gian nạp lại đã vượt xa RTO mà không ai biết (tr. 286).

**Kết quả đã đo** (thí nghiệm 2): ngay ở quy mô nhỏ, restore chậm hơn dump 11–16 lần; ở quy mô dữ liệu lớn hơn bộ nhớ, sách cảnh báo tỷ lệ có thể còn tệ hơn nhiều.

[về mục 2.8](../chapter-10.md#s2-8)
