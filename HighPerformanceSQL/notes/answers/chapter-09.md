# Đáp án — Chương 9: Replication

Về ghi chú chương: [../chapter-09.md](../chapter-09.md) · Bài thực hành: [../experiments/chapter-09.md](../experiments/chapter-09.md)

> Đáp án diễn giải từ sách (kèm trang in) và từ thí nghiệm đã chạy trên MySQL 8.0.42 (ghi "Kết quả đã đo"). Phần suy luận thêm ghi **Bổ sung ngoài sách**. Hãy tự trả lời trước khi đọc.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Vì sao replication là bất đồng bộ và lag không có cận trên?

*Vì sao replication mặc định của MySQL được gọi là bất đồng bộ, và vì sao điều đó dẫn tới "lag không có cận trên"? Nêu hai tình huống làm lag tăng mạnh.*

Source ghi event vào binary log rồi trả lời client mà **không chờ** replica. Replica tự đọc event về relay log rồi tự phát lại theo nhịp của nó (tr. 227–229). Không có cơ chế nào buộc replica theo kịp trong một khoảng thời gian cho trước, nên độ trễ giữa thời gian thực và dữ liệu trên replica **không có cận trên** (tr. 228).

Tình huống làm lag tăng:
- **Truy vấn lớn** trên source: replica chỉ áp dụng sau khi source commit, rồi mất thêm thời gian phát lại; sách nói có thể tụt vài giây, vài phút, thậm chí vài giờ (tr. 228). Kết quả đã đo: `UPDATE` 200.000 hàng mất 2,3 giây trên source, lag heartbeat lên khoảng 4,7 giây ([TN3](../experiments/chapter-09.md#tn3)).
- **Replica dừng hay mạng chập chờn**: I/O thread không kéo được event, hoặc SQL thread dừng vì lỗi; lag tăng đều theo thời gian (TN3: dừng SQL thread 10 giây thì heartbeat lag lên 10,5 giây).
- Các nguyên nhân khác sách nêu ở tr. 252–253: replica phát lại đơn luồng hay workload không song song được, cấu hình durability đắt so với khả năng đĩa.

[về mục 2.1](../chapter-09.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Thêm replica khi ghi tăng gấp đôi

*Một hệ có 1 source và 5 replica bị nghẽn vì lượng ghi tăng gấp đôi. Đồng nghiệp đề xuất thêm replica thứ 6. Đề xuất này giúp được gì và không giúp được gì?*

**Không giúp** phần ghi. Mỗi replica phải phát lại **toàn bộ** ghi của source; thêm replica chỉ khiến mỗi lần ghi được làm thêm một lần nữa. Cả hệ bị giới hạn bởi số lần ghi mà phần yếu nhất làm được (tr. 228). Source vẫn phải tự xử lý mọi ghi, và còn thêm một dump thread phục vụ replica mới.

**Có thể giúp** nếu nghẽn thật ra nằm ở **đọc** (replica hiện có quá tải đọc): thêm replica chia tải đọc (tr. 228, 242–243). Nhưng nếu các replica đang lag vì không theo kịp ghi, replica mới cũng sẽ lag y như vậy.

Hướng đúng cho ghi: tối ưu truy vấn và cấu hình trước, rồi sharding — "scale đọc bằng replica, scale ghi bằng sharding" (tr. 253; [Chương 11](../chapter-11.md#s2-6)).

[về mục 2.1](../chapter-09.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Nâng cấp 5.7 lên 8.0: replica hay source trước?

*Khi chuẩn bị nâng cấp từ 5.7 lên 8.0, bạn nên đưa bản 8.0 vào vai replica hay vai source trước? Vì sao?*

**Replica trước.** Replication gần như tương thích ngược: server mới thường làm replica được cho server cũ. Chiều ngược lại thường không được, vì server cũ có thể không hiểu tính năng, cú pháp SQL hay định dạng file mà server mới dùng (ví dụ không thể replicate từ 5.6 sang 5.5) (tr. 228). Sách cũng nêu đúng cách dùng này: dựng một replica chạy bản mới để kiểm tra truy vấn trước khi nâng cấp mọi instance (tr. 229), và khuyên thử replication trước khi nâng cấp giữa các bản (tr. 228).

**Bổ sung ngoài sách.** Khi mọi replica đã lên 8.0, promote một replica 8.0 thành source bằng planned promotion (tr. 240); sau thời điểm đó, các server 5.7 còn lại không nên làm replica của nó nữa.

[về mục 2.1](../chapter-09.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — SQL thread dừng, I/O thread vẫn chạy

*SQL thread của replica dừng vì lỗi nhưng I/O thread vẫn chạy. Các event mới của source đang nằm ở đâu, và dữ liệu đọc từ replica lúc này phản ánh thời điểm nào?*

Event mới vẫn được I/O thread chép vào **relay log** của replica (bước 2 của tr. 229), vì hai thread làm việc độc lập (tr. 229). Nhưng chúng **chưa được phát lại**, nên dữ liệu đọc từ replica dừng ở transaction cuối cùng mà SQL thread đã áp dụng trước khi lỗi, và ngày càng cũ.

Kết quả đã đo ([TN3](../experiments/chapter-09.md#tn3)): khi dừng SQL thread, `Retrieved_Gtid_Set` vẫn tăng (backlog GTID 1 → 11 transaction), `Executed_Gtid_Set` đứng yên, `Seconds_Behind_Source` là `NULL`. Hệ quả vận hành: relay log chiếm thêm đĩa (tr. 254), và nếu replica này nằm trong read pool thì phải rút nó ra (tr. 243).

[về mục 2.1](../chapter-09.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Replica có index mà source không có

*Trên một replica, đồng nghiệp lỡ tạo thêm một index mà source không có. Ở định dạng STATEMENT, câu `DELETE ... LIMIT 10` không có `ORDER BY` có thể gây gì? Ở định dạng ROW thì sao, và vì sao?*

**STATEMENT.** Replica thực thi lại **câu SQL** (tr. 230). Optimizer của replica có thể chọn access path khác (dùng index mới, hoặc ngược lại không có index mà source dùng). Với `LIMIT` không `ORDER BY`, tập hàng bị xóa là các hàng **đầu tiên mà physical execution gặp**, nên hai server có thể xóa **10 hàng khác nhau** (tr. 230, 250). Không có lỗi nào được báo: số hàng vẫn bằng nhau.

Kết quả đã đo ([TN2](../experiments/chapter-09.md#tn2)): source dùng range trên `idx_k` và xóa id 9991–10000; replica thiếu index quét theo khóa chính và xóa id 9501–9510; replica vẫn `Yes / Yes`, errno 0; chỉ checksum lộ ra khác biệt. Source chỉ phát Note 1592 "Unsafe statement".

**ROW.** Binlog chứa **ảnh của từng hàng** đã bị xóa trên source (tr. 230–231). Replica định vị chính các hàng đó theo khóa, không cần tái lập quyết định của optimizer. Kết quả đã đo: cùng câu ở ROW, cả ba server xóa cùng id 9981–9990. Nếu hàng cần xóa hay sửa không tồn tại trên replica (vì đã lệch trước đó), ROW làm replication **dừng với lỗi 1032**, tức lệch bị lộ ra thay vì âm thầm.

Cách phòng theo sách: dùng row-based hoặc viết câu tất định, ví dụ thêm `ORDER BY` theo khóa duy nhất (tr. 250).

[về mục 2.2](../chapter-09.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Gắn lại replica khi source được dựng lại từ backup

*Source crash và được dựng lại từ backup. Với replication theo file và position, việc gắn lại replica khó ở đâu? GTID cùng auto-position giải quyết điều đó thế nào?*

Với file và position, replica nhớ "đang đọc `binlog.000002` ở vị trí 2749". Source dựng lại từ backup có binlog **bắt đầu lại từ đầu**, nên vị trí cũ vô nghĩa. Người vận hành phải đọc event để dò điểm gắn: chọn sớm quá thì replica **áp dụng lặp** event, muộn quá thì **bỏ sót** event (tr. 231).

Với GTID, mỗi transaction có định danh duy nhất `server_uuid:số` được ghi cùng transaction vào binlog, và replica ghi nhận GTID nào đã hoàn tất (tr. 231). Khi kết nối với `SOURCE_AUTO_POSITION=1` (chú thích 3, tr. 232), replica gửi **tập GTID đã có**, và source chỉ gửi những transaction nằm ngoài tập đó. Không ai phải tính vị trí.

Kết quả đã đo ([TN1](../experiments/chapter-09.md#tn1)): tắt replica khi có GTID 1–3, source ghi tới GTID 8; bật replica lên, log ghi "Starting GTID-based replication" và replica tự có đủ 1–8.

[về mục 2.2](../chapter-09.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Vị trí applier trong bảng InnoDB và `relay_log_recovery`

*Vì sao lưu vị trí của applier trong bảng InnoDB an toàn hơn lưu trong file? Vì sao bật `relay_log_recovery` thì không cần `sync_relay_log`?*

**Bảng thay vì file** (tr. 233). Khi vị trí nằm trong file, việc commit transaction đã áp dụng và việc ghi file vị trí là **hai bước** đồng bộ riêng. Crash giữa hai bước làm file chứa vị trí sai, và khi khởi động lại replica có thể áp dụng lặp hay bỏ sót. Khi vị trí nằm trong bảng InnoDB, replication cập nhật **dữ liệu và vị trí trong cùng một transaction**, nên hai thứ luôn khớp nhau sau crash (thao tác nguyên tử).

**`relay_log_recovery`** (tr. 233). Khi phát hiện crash, replica **vứt bỏ toàn bộ relay log** và lấy lại phần còn thiếu từ source. Relay log hỏng hay dở dang vì crash không còn là vấn đề, vì nó không được dùng nữa. Do đó không cần trả giá `fsync` relay log thường xuyên (`sync_relay_log`) để bảo vệ nó.

Kết quả đã đo ([TN1](../experiments/chapter-09.md#tn1)): trên 8.0.42, `relay_log_info_repository=TABLE` là mặc định (và đã deprecated), còn `relay_log_recovery` mặc định **OFF**, phải tự bật.

[về mục 2.2](../chapter-09.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Tăng worker mà lag không giảm

*Bạn tăng `replica_parallel_workers` từ 4 lên 16 nhưng lag không giảm. Nêu hai nguyên nhân gốc khả dĩ và cách kiểm chứng bằng dữ liệu.*

Nguyên nhân khả dĩ:

1. **Workload trên source không có song song để khai thác.** Với `LOGICAL_CLOCK`, chỉ các transaction cùng một binary log group commit mới được áp dụng song song (tr. 234). Một job ghi tuần tự trong một session tạo chuỗi transaction nối đuôi nhau. Kết quả đã đo ([TN3](../experiments/chapter-09.md#tn3)): 1.200 transaction tuần tự đều chạy trên worker 1 (1.200 / 0 / 0 / 0).
2. **Một transaction khổng lồ.** Một transaction chỉ do một worker áp dụng; các transaction sau phải chờ nó (thứ tự commit được giữ với `replica_preserve_commit_order`, tr. 235).
3. Khả dĩ khác: chế độ `DATABASE` mà mọi ghi rơi vào một database (tr. 234); nút thắt là `fsync` hay I/O của replica, không phải CPU (TN3 phần E); overhead của coordinator (tr. 235).

Kiểm chứng:
- Cách sách gợi ý: đếm số transaction mỗi worker xử lý bằng Performance Schema (`events_transactions_summary_by_thread_by_event_name` nối với `replication_applier_status_by_worker`) trong lúc ghi nặng (tr. 235–236). Nếu vài worker đầu làm gần hết việc, thêm worker không giúp.
- **Bổ sung ngoài sách:** xem `last_committed` và `sequence_number` trong `mysqlbinlog`; nếu mỗi transaction có `last_committed` bằng `sequence_number` của transaction trước thì không có gì để song song.

[về mục 2.2](../chapter-09.md#s2-2)

<a id="q2-2-5"></a>

### Câu 2.2.5 — "Có semisync rồi nên failover không mất transaction"

*Một đồng nghiệp nói: "Ta bật semisync rồi, nên failover không bao giờ mất transaction đã commit." Câu này sai ở đâu?*

Sai ở hai điểm:

1. **Semisync rơi về bất đồng bộ khi hết timeout.** Nếu không replica nào xác nhận trong thời hạn, MySQL chuyển sang replication bất đồng bộ và **không làm transaction thất bại** (tr. 237). Từ lúc đó, transaction commit trên source có thể không có ở replica nào. Sách kết luận semisync không phải công cụ chống mất dữ liệu và khuyên không dựa vào nó cho toàn vẹn dữ liệu (tr. 237).
2. **ACK chỉ nghĩa là đã ghi vào relay log**, chưa chắc đã áp dụng (tr. 237). Nếu replica được chọn làm đích chưa xử lý xong relay log, phải để nó áp dụng hết trước khi mở ghi; và nếu chọn **một replica khác** (không phải cái đã ACK) thì transaction đó có thể không có.

Kết quả đã đo ([TN5](../experiments/chapter-09.md#tn5)): cắt ACK của cả hai replica, `INSERT` chờ 3,015 giây rồi **thành công**, `Rpl_semi_sync_source_status` thành `OFF`; ba transaction lúc đó chỉ có trên source. Sau khi source "chết" và failover, chúng biến mất khỏi topology, và trỏ source cũ về thì lỗi 1062.

[về mục 2.2](../chapter-09.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — Vì sao chặn ghi trên source trước khi mở ghi trên đích

*Trong planned promotion, vì sao phải bật `super_read_only` trên source (bước 3) **trước** khi tắt `read_only` trên đích (bước 5)? Nếu đảo thứ tự thì có thể xảy ra gì?*

Bước 3 làm **đóng băng** tập transaction của source. Chỉ khi tập đó không đổi nữa, bước 4 ("chờ tới khi đồng bộ, so GTID để chắc chắn", tr. 240) mới có nghĩa: đích chứa **mọi** transaction của source.

Nếu đảo thứ tự, trong một khoảng thời gian **cả hai server cùng nhận ghi** (**Bổ sung ngoài sách**, suy ra từ tr. 240 và tr. 249–250):
- ghi trên source cũ sau thời điểm đích ngừng replicate từ nó sẽ không bao giờ tới đích, tức mất hoặc lệch dữ liệu;
- hai bên có thể cấp **trùng giá trị `AUTO_INCREMENT`** hay cùng sửa một hàng; khi source cũ được trỏ về đích (bước 7), replication có thể lỗi trùng khóa;
- đây chính là tình trạng "ghi vào nhiều server cùng lúc" mà sách cấm (tr. 250).

Kết quả đã đo ([TN5](../experiments/chapter-09.md#tn5)): sau bước 3, lần ghi của ứng dụng vào source cũ bị từ chối với lỗi 1290; bước 4 trả về 0 (đã đồng bộ) và `GTID_SUBSET` = 1.

[về mục 2.3](../chapter-09.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Những bước mất đi trong unplanned promotion

*Unplanned promotion bỏ những bước nào của planned promotion? Hệ quả của việc bỏ các bước đó với dữ liệu là gì?*

Unplanned promotion là bản rút gọn vì **không có source sống để kiểm tra** (tr. 240). So với bảy bước của planned promotion, nó bỏ:
- bước 2 theo nghĩa đầy đủ (kiểm lag so với source sống);
- bước 3 (chặn ghi trên source; source đã chết);
- bước 4 (chờ đồng bộ và so GTID với source).

Hệ quả: chọn đích chỉ dựa trên **dữ liệu đã được replicate** (tr. 240). Mọi transaction source đã commit nhưng chưa tới replica nào sẽ **không có** trên source mới, và bạn **không biết chắc** có bao nhiêu (tr. 240–241). Chỉ khi source cũ quay lại mới đo được bằng phép trừ tập GTID (**Bổ sung ngoài sách**). Kết quả đã đo ([TN5](../experiments/chapter-09.md#tn5)): `only_on_old_source = ...:3006-3008`, tức ba transaction.

[về mục 2.3](../chapter-09.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Khi nào không failover

*Khi nào sách cho rằng **không failover** là lựa chọn tốt hơn? Cần điều kiện cấu hình nào để việc "chờ" không mất dữ liệu?*

Sách nói phản xạ đầu tiên với downtime thường là failover, nhưng vì khó biết đích thiếu bao nhiêu dữ liệu, đôi khi **không failover** tốt hơn (tr. 240). Lý do (tr. 241):
- unplanned promotion ít được luyện tập, phải tra tài liệu, phải xem xét replica nào là ứng viên; tất cả tốn thời gian;
- có khi **chờ** server hay tiến trình MySQL quay lại còn **nhanh hơn**.

Điều kiện để chờ không mất dữ liệu: source chạy **cấu hình ACID đầy đủ** theo chương 5 (tr. 241), cụ thể là `innodb_flush_log_at_trx_commit=1` và `sync_binlog=1` (tr. 232–233). Khi đó mọi transaction đã báo commit đều nằm trên đĩa, source khởi động lại không mất gì, và replica tiếp tục từ chỗ dừng (tr. 241).

**Bổ sung ngoài sách:** "chờ" chỉ hợp lý khi có tín hiệu máy sẽ quay lại (ví dụ restart tiến trình, reboot) và thời gian chờ dự kiến nằm trong mức downtime chấp nhận được; hỏng phần cứng hay mất cả máy thì chờ không có ý nghĩa.

[về mục 2.3](../chapter-09.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — Source cũ quay lại với `super_read_only`

*Vì sao source cũ phải khởi động lại với `super_read_only` bật sẵn? Mô tả một chuỗi sự kiện cụ thể dẫn tới hỏng dữ liệu nếu không làm vậy.*

Sách yêu cầu source cũ khi online lại phải mặc định bật `super_read_only` để **tránh ghi nhầm** (tr. 240). Ví dụ minh họa do AI xây dựng:

1. Source A chết, B được promote, ứng dụng ghi vào B.
2. A khởi động lại **có thể ghi**. Một số instance ứng dụng còn cache DNS hay cấu hình cũ, hoặc một cron job trỏ cứng vào A, và bắt đầu ghi vào A.
3. A và B giờ cùng nhận ghi, cấp trùng `AUTO_INCREMENT`, và có các GTID mà bên kia không có.
4. Khi trỏ A về B, replication lỗi trùng khóa (như lỗi 1062 đã đo ở [TN5](../experiments/chapter-09.md#tn5)), hoặc tệ hơn là chạy được nhưng dữ liệu hai bên lệch âm thầm. Dữ liệu ghi vào A phải đối soát thủ công.

Kết quả đã đo: A khởi động với `--super-read-only=ON` từ chối mọi `INSERT` của root (lỗi 1290). Lưu ý: phải đặt trong **cấu hình khởi động**, vì `SET GLOBAL` mất khi restart (**Bổ sung ngoài sách**).

[về mục 2.3](../chapter-09.md#s2-3)

<a id="q2-3-5"></a>

### Câu 2.3.5 — Delayed replica và automation failover

*Topology có một delayed replica 6 giờ và hệ failover tự động chọn "replica có lag nhỏ nhất". Có rủi ro gì không? Nếu hệ chọn theo "replica có nhiều GTID nhất" thì còn rủi ro nào?*

Với tiêu chí "lag nhỏ nhất", delayed replica thường không bị chọn khi các replica khác khỏe. Nhưng nếu mọi replica thường đều đang lỗi (`Seconds_Behind_Source` là `NULL` hay không đo được), một hệ ngây thơ có thể bỏ qua chúng và chọn delayed replica, tức promote một server **trễ 6 giờ**: mất 6 giờ dữ liệu. Sách nói rõ phải **loại delayed replica khỏi danh sách ứng viên**, và điều này càng quan trọng khi failover ghi được tự động hóa (tr. 233).

Với tiêu chí "nhiều GTID nhất": chọn đúng hơn về lượng dữ liệu (đúng tinh thần "replica bạn chắc có đủ dữ liệu", tr. 240). Rủi ro còn lại (**Bổ sung ngoài sách**):
- một replica có **GTID lạ** (ghi nhầm trực tiếp) có thể trông như "nhiều GTID nhất" dù thiếu transaction của source; nên so tập GTID theo từng `server_uuid` và kiểm GTID lạ ([TN5](../experiments/chapter-09.md#tn5) phần A loại được một ứng viên như vậy);
- replica có nhiều GTID nhất chưa chắc **đủ năng lực** làm source (sách muốn đích failover có cấu hình giống source, tr. 242–243).

[về mục 2.3](../chapter-09.md#s2-3)

---

<a id="q2-4-1"></a>

### Câu 2.4.1 — Ứng dụng không chịu được đọc cũ

*Một ứng dụng không chịu được việc người dùng không thấy dữ liệu mình vừa ghi. Sách gợi ý topology nào, và cái giá phải trả là gì?*

**Active/passive**: mọi đọc và ghi tới một source; replica passive không phục vụ traffic. Lý do chính để chọn mô hình này là không phải lo replication lag, tránh vấn đề read-after-write (tr. 241).

Cái giá (tr. 242):
- khả năng scale đọc bị **gắn vào năng lực một server**; chạm giới hạn thì phải chuyển sang read pool hoặc sharding;
- các replica passive "nằm chờ", nhưng vẫn phải có cấu hình giống source để gánh traffic khi failover, và cần n+2 (vật lý) hoặc n+1/n+2 (cloud).

**Bổ sung ngoài sách:** có thể lai: dùng read pool cho phần chịu được dữ liệu cũ, còn các luồng "đọc ngay sau khi ghi" thì đọc từ source.

[về mục 2.4](../chapter-09.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — Phân tải read pool không đồng nhất, mục tiêu CPU

*Read pool gồm một replica 32 core (đích failover) và ba replica 8 core. Bạn phân tải đọc thế nào? Vì sao sách nhắm CPU khoảng 50–60% thay vì 90%?*

Dùng **traffic weighting** theo năng lực: node 32 core nhận gấp **4 lần** traffic của mỗi node 8 core để được tận dụng (tr. 243). **Bổ sung ngoài sách (phép tính):** tổng trọng số 4 + 1 + 1 + 1 = 7, nên node 32 core nhận khoảng 4/7 tổng đọc, mỗi node 8 core khoảng 1/7.

CPU 50–60% (tr. 243):
- CPU là chỉ số sử dụng đáng tin nhất cho đọc; khi CPU tăng, server tốn nhiều thời gian **context switching** hơn và **latency tăng**; cần cân bằng giữa latency và mức sử dụng.
- Pool cần **khoảng đệm** cho node hỏng (tr. 243): một node rời pool thì tải của nó dồn sang các node còn lại; ở 90% thì không còn chỗ.
- Node 32 core còn là **đích failover**: khi nó phải lên làm source, phần đọc nó gánh phải chuyển sang các node khác.

[về mục 2.4](../chapter-09.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Active-active chia theo id chẵn/lẻ

*Vì sao chia ghi theo id chẵn/lẻ giữa hai cosource vẫn không làm active-active nhất quán? Ngoài nhất quán, active-active còn làm hỏng điều gì của chiến lược failover?*

Chia chẵn/lẻ chỉ bảo đảm read-after-write cho **cùng một hàng** (hàng chẵn luôn ghi và đọc ở một bên). Truy vấn chạm hàng **của bên kia** thì không nhất quán: đọc id 1–6 thì gửi đi đâu? Nếu bên kia có cập nhật chưa tới đây vì lag thì truy vấn thấy dữ liệu trộn giữa hai thời điểm (tr. 244).

Về failover (tr. 244–245):
- **Năng lực:** mỗi bên là đích failover của bên kia; nếu cả hai đều đang gánh traffic, dồn hết sang một bên dễ **hết CPU**.
- **Working set:** bên nhận thêm traffic phải nạp working set của bên kia; buffer pool của InnoDB bị **churn**.
- Sách kết luận: dùng server passive để "có ích" là ảo giác; bạn đưa không nhất quán vào ứng dụng và luôn lo không đủ năng lực failover; **mất chiến lược failover là mất khả năng chống chịu** (tr. 245).

[về mục 2.4](../chapter-09.md#s2-4)

<a id="q2-4-4"></a>

### Câu 2.4.4 — Ring ba source mất một node

*Trong ring ba source, một server hỏng phần cứng. Điều gì xảy ra với các cập nhật mới, và vì sao gắn thêm replica cho mỗi source không giải quyết được gốc rễ?*

Mỗi server là replica của server trước và source của server sau (tr. 246). Một server offline thì **vòng bị đứt**: cập nhật không còn chạy quanh vòng, nên ghi ở một số server không tới được các server khác (tr. 246).

Gắn replica cho mỗi source chỉ cho bạn một **bản thay thế**; vòng **vẫn đứt** cho tới khi promote replica vào đúng vị trí cũ (tr. 246), tức vẫn cần một thao tác failover trên chính topology phức tạp. Gốc rễ là cấu trúc mà **mọi node đều là mắt xích bắt buộc** và nhiều node cùng nhận ghi (vi phạm quy tắc tr. 250). Sách: topology này "ngược với đơn giản và không có ưu điểm nào" (tr. 246).

[về mục 2.4](../chapter-09.md#s2-4)

<a id="q2-4-5"></a>

### Câu 2.4.5 — Khi nào dùng multisource

*Khi nào multisource replication là lựa chọn hợp lý theo sách, và khi nào nó trở thành topology không khuyến khích?*

Hợp lý cho việc **một lần**, ví dụ **gộp** hai cluster đã tách (dữ liệu video và dữ liệu user) về một chỗ: một cluster thứ ba với hai **replication channel**, nạp và bắt kịp, rồi một downtime rất ngắn để đóng băng ghi ở hai source và chuyển code sang database gộp (tr. 246–247). Lưu ý giới hạn: không thể cấu hình một replica dùng multisource nhiều lần với **cùng một source** (tr. 247).

Không khuyến khích khi bạn xây một **topology lâu dài** quanh nó. Dùng tạm thời để gộp dữ liệu, với mục tiêu quay về active/passive hay active/read pool, vẫn chấp nhận được (tr. 247).

[về mục 2.4](../chapter-09.md#s2-4)

---

<a id="q2-5-1"></a>

### Câu 2.5.1 — Vì sao `Seconds_Behind_Source` không đáng tin

*Nêu ít nhất bốn lý do khiến `Seconds_Behind_Source` không đáng tin. Heartbeat khắc phục chúng thế nào, và nó vẫn **không** trả lời được câu hỏi gì?*

Năm lý do của sách (tr. 248–249):
1. Tính bằng thời gian hiện tại trừ timestamp trong event binlog, nên chỉ báo được khi **đang xử lý** một câu lệnh.
2. Thường báo `NULL` khi các replication thread không chạy.
3. Một số lỗi (`max_allowed_packet` lệch, mạng không ổn) làm replication dừng nhưng giá trị lại là **0**.
4. Có khi không tính được dù thread chạy, và báo 0 hoặc `NULL`.
5. Transaction **rất dài** làm số liệu nhảy (mở một giờ rồi commit thì báo trễ một giờ rồi về 0).

Heartbeat (tr. 249): source cập nhật một timestamp **mỗi giây**; lag = thời gian hiện tại trên replica trừ timestamp đó. Nó đo trực tiếp "dữ liệu trên replica hiện hành tới lúc nào", nên không phụ thuộc thread nào đang chạy hay timestamp của event nào; sách nói nó miễn nhiễm với mọi vấn đề trên.

Heartbeat (và mọi chỉ số lag khác) **không** cho biết replica **mất bao lâu để bắt kịp**; điều đó phụ thuộc sức replica và lượng ghi source tiếp tục xử lý (tr. 249).

Kết quả đã đo ([TN3](../experiments/chapter-09.md#tn3)): lý do 2 đúng (NULL khi dừng SQL hay I/O thread); lý do 5 đúng với applier một luồng (SBS 18–21 giây, heartbeat 1–4,6 giây), không thấy với applier 4 luồng mặc định.

[về mục 2.5](../chapter-09.md#s2-5)

<a id="q2-5-2"></a>

### Câu 2.5.2 — I/O thread đứt vì mạng

*I/O thread của replica bị đứt vì mạng. `Seconds_Behind_Source` báo gì? Số GTID "đã nhận chưa thực thi" báo gì? Chỉ số nào còn phản ánh đúng độ trễ?*

- `Seconds_Behind_Source`: thường **`NULL`** (tr. 248). Kết quả đã đo: `NULL` suốt 4 giây dừng I/O thread.
- GTID đã nhận chưa thực thi: **trống**, vì SQL thread đã áp dụng hết những gì nhận được; replica **không biết** source có thêm gì (Bổ sung ngoài sách, đã đo ở [TN3](../experiments/chapter-09.md#tn3)). Nếu cảnh báo dựa trên chỉ số này thì sẽ im lặng đúng lúc có vấn đề.
- **Heartbeat** vẫn đúng: timestamp trên replica đứng yên trong khi đồng hồ chạy, nên lag tăng đều (đã đo 0,9 → 4,9 giây) (tr. 249). Một cách khác là so `gtid_executed` của replica với của source, nhưng cần truy cập source.

[về mục 2.5](../chapter-09.md#s2-5)

<a id="q2-5-3"></a>

### Câu 2.5.3 — Nguyên nhân lệch và quy tắc phòng ngừa

*Ghép mỗi nguyên nhân gây lệch dữ liệu ở tr. 249 với quy tắc (hoặc cấu hình) phòng ngừa tương ứng. Nguyên nhân nào không có quy tắc nào chặn được hoàn toàn?*

| Nguyên nhân (tr. 249) | Phòng ngừa |
| --- | --- |
| Ghi nhầm vào replica | `super_read_only` trên mọi replica (tr. 250) |
| Dual source với cả hai bên nhận ghi | Không ghi vào nhiều server cùng lúc; một source nhận mọi ghi (tr. 250) |
| Truy vấn không tất định + statement-based | Dùng row-based hoặc câu tất định (`ORDER BY` trên khóa) (tr. 250) |
| MySQL crash khi chạy chế độ kém durable | Cấu hình durable của chương 5 và các thiết lập crash-safe (tr. 232–233); cẩn thận khi tạm giảm durability (tr. 253) |
| Bug của MySQL | **Không chặn được hoàn toàn**; chỉ giảm rủi ro bằng thử nghiệm kỹ khi nâng cấp và giám sát (tr. 254), và rebuild replica khi gặp lỗi (tr. 250) |

**Bổ sung ngoài sách:** vì bug (và sai sót con người) không chặn được hoàn toàn, nên vẫn cần **kiểm tra nhất quán định kỳ** (checksum) để phát hiện lệch, như [TN2](../experiments/chapter-09.md#tn2) cho thấy lệch có thể hoàn toàn im lặng.

[về mục 2.5](../chapter-09.md#s2-5)

<a id="q2-5-4"></a>

### Câu 2.5.4 — `read_only` không đủ; hậu quả của ghi nhầm khi dùng GTID

*Vì sao `read_only` không đủ để bảo vệ replica? Hậu quả lâu dài của một lần ghi nhầm vào replica trong topology dùng GTID là gì?*

`read_only` chỉ chặn user **không có** quyền `SUPER`. DBA (hay công cụ chạy bằng tài khoản quản trị) vẫn có thể chạy `DELETE` hay `ALTER` mà không biết mình đang ở replica. `super_read_only` chỉ cho replication ghi (tr. 250). Kết quả đã đo ([TN1](../experiments/chapter-09.md#tn1)): với `read_only=ON`, root `INSERT` thành công; với `super_read_only=ON`, lỗi 1290.

Hậu quả lâu dài (**Bổ sung ngoài sách**, đã quan sát ở [TN5](../experiments/chapter-09.md#tn5)): lần ghi nhầm có GTID mang `server_uuid` của replica (một *errant transaction*), và GTID đó **ở lại mãi** trong `gtid_executed` của replica. Ngoài việc dữ liệu lệch với source, nếu replica này được promote thì các server khác sẽ nhận và áp dụng thay đổi đó, nên một thay đổi chưa từng có trên source lan khắp topology; nếu binlog chứa nó đã bị xóa thì các replica báo lỗi khi trỏ về. Vì vậy cần kiểm GTID lạ trước khi chọn đích failover.

[về mục 2.5](../chapter-09.md#s2-5)

---

<a id="q2-6-1"></a>

### Câu 2.6.1 — Hai replica cùng server ID

*Hai replica bị cấu hình cùng server ID. Sách mô tả những triệu chứng nào trên source và trên replica? Vì sao lỗi này "khó nắm bắt"?*

Triệu chứng (tr. 251):
- Trên source: chỉ thấy **một trong hai** replica kết nối tại mỗi thời điểm (bình thường mọi replica đều kết nối).
- Trên replica: thông báo **ngắt và nối lại** thường xuyên trong log lỗi, nhưng **không nhắc gì** tới server ID.
- Tùy phiên bản: replicate đúng nhưng chậm, hoặc sai (bỏ sót event, lặp event gây lỗi trùng khóa hay hỏng dữ liệu thầm lặng); tải source tăng vì các replica "đánh nhau"; log lỗi phình rất nhanh.

Khó nắm bắt vì nếu không nhìn kỹ thì mọi thứ **trông như vẫn chạy**, và thông báo lỗi không chỉ ra nguyên nhân. Phòng ngừa: cẩn thận khi dựng, giữ danh sách ánh xạ replica với server ID, hoặc dùng octet cuối của IP nếu cùng subnet (tr. 251).

Kết quả đã đo trên 8.0.42 ([TN5](../experiments/chapter-09.md#tn5) phần C): `SHOW REPLICAS` chỉ liệt kê một replica (khớp sách), nhưng hai dump thread cùng tồn tại, cả hai replica nhận dữ liệu, không thấy ngắt và nối lại trong 2,5 phút. Hành vi phụ thuộc phiên bản như sách đã nói.

[về mục 2.6](../chapter-09.md#s2-6)

<a id="q2-6-2"></a>

### Câu 2.6.2 — `replicate_do_db=sakila` và `USE test`

*Replica đặt `replicate_do_db=sakila`. Trên source, ứng dụng chạy `USE test;` rồi `UPDATE sakila.actor ...`. Dự đoán kết quả trên replica với định dạng STATEMENT, với định dạng ROW, và với một câu DDL tương tự.*

- **STATEMENT:** bộ lọc xét **database mặc định** (`test`), không xét `sakila`, nên câu `UPDATE` **bị bỏ qua** và replica lệch với source (tr. 238, 252).
- **ROW (Bổ sung ngoài sách, đã đo):** với DML ở định dạng ROW, bộ lọc xét database của **bảng** được sửa, nên câu `UPDATE sakila.actor` **được** áp dụng. Kết quả đã đo ([TN2](../experiments/chapter-09.md#tn2) phần D): `INSERT INTO misc.t` khi `USE other` bị lọc ở STATEMENT nhưng tới replica ở ROW.
- **DDL:** DDL luôn được ghi dạng câu lệnh, nên lại theo database mặc định. Đã đo: `CREATE TABLE misc.ddl_from_other` khi `USE other` bị lọc; `CREATE TABLE other.ddl_from_misc` khi `USE misc` thì **được áp dụng** và làm SQL thread dừng với lỗi 1049 vì database `other` không tồn tại trên replica. Đây là ví dụ cho câu "có loại câu lệnh còn làm replication dừng với lỗi vì phụ thuộc không được replicate" (tr. 252).

Bài học: cùng một filter có hai quy tắc khác nhau tùy định dạng và loại câu lệnh, nên sách khuyên tránh filter (tr. 238–239, 255).

[về mục 2.6](../chapter-09.md#s2-6)

<a id="q2-6-3"></a>

### Câu 2.6.3 — Ba cách giảm lag và chốt an toàn khi giảm durability

*Nêu ba cách giảm lag của sách. Với cách "tạm giảm durability", liệt kê các rủi ro và các chốt an toàn cần có trong script tự động.*

Ba cách (tr. 253): **multithreaded replication** (và chỉnh theo manual); **sharding** để dàn ghi ra nhiều source; **tạm giảm durability** (`sync_binlog=0`, `innodb_flush_log_at_trx_commit=0`) khi đã hết cách khác và lag do giới hạn ghi.

Rủi ro (tr. 253):
- chỉ được làm trên **replica**;
- nếu replica là nơi backup (snapshot LVM hay đĩa cloud, chú thích 7), backup có thể **không phục hồi được**;
- replica crash trong lúc đó thì nhiều khả năng phải **dựng lại từ source**;
- làm tay rất dễ **quên bật lại**.

Chốt an toàn theo quy trình của sách (tr. 253): (1) xác nhận server là replica không ghi được (`super_read_only` bật); (2) mới đổi hai thiết lập; (3) kiểm tra lag định kỳ; (4) dưới ngưỡng thì đặt lại durable. **Bổ sung ngoài sách:** dùng heartbeat làm ngưỡng thay vì `Seconds_Behind_Source`; tạm dừng job backup trên replica đó trong thời gian giảm durability; có giám sát cảnh báo nếu một replica ở trạng thái kém durable quá lâu; loại replica đó khỏi danh sách ứng viên failover cho tới khi durable trở lại.

Kết quả đã đo ([TN3](../experiments/chapter-09.md#tn3) phần E): bắt kịp 2.000 transaction nhỏ mất 4,20 s (durable) so với 0,66 s (giảm durability), trung vị 3 lần.

[về mục 2.6](../chapter-09.md#s2-6)

<a id="q2-6-4"></a>

### Câu 2.6.4 — Đĩa replica đầy khi lag

*Replica tụt 6 giờ trong lúc source chạy nhiều `LOAD DATA INFILE`. Vì sao đĩa của replica có thể đầy dù dữ liệu của nó chưa lớn thêm? Cần giám sát và cấu hình gì?*

Replica càng tụt thì càng nhiều **relay log đã nhận nhưng chưa thực thi** nằm trên đĩa (tr. 254). `LOAD DATA INFILE` tạo event lớn và file tạm; nếu replica bật `log_replica_updates` thì nó còn ghi lại các thay đổi vào **binlog của chính nó** (tr. 254). Dữ liệu chưa được áp dụng nhưng relay log và binlog đã chiếm chỗ.

Cần (tr. 248, 254): **giám sát dung lượng đĩa** trên cả source và replica; đặt giới hạn dung lượng relay log (sách gọi là `relay_log_space`; trên 8.0.42 biến là `relay_log_space_limit`, mặc định 0 = không giới hạn). Nhắc lại hậu quả khác nhau: source hết đĩa thì transaction timeout; replica hết đĩa thì tạm dừng replication chờ có chỗ (tr. 248).

[về mục 2.6](../chapter-09.md#s2-6)

<a id="q2-6-5"></a>

### Câu 2.6.5 — Temporary table và statement-based

*Vì sao temporary table và statement-based replication không hợp nhau? Cách xử lý ưu tiên của sách là gì?*

Với statement-based, replica thực thi lại các câu tạo và dùng temporary table. Temporary table chỉ sống trong session của replication thread; nếu replica crash hay bị tắt, chúng **biến mất**. Khi chạy lại, mọi câu lệnh sau tham chiếu tới temporary table đó **thất bại** (tr. 252).

Ưu tiên của sách: dùng **row-based replication** (tr. 252). **Bổ sung ngoài sách:** với row-based, thay đổi trên bảng thường được ghi dưới dạng ảnh hàng, nên replica không cần tái tạo temporary table trung gian. Cách thứ hai: đặt tên temporary table nhất quán (ví dụ tiền tố `temporary_`) và dùng quy tắc replication để bỏ qua chúng hoàn toàn (tr. 252).

[về mục 2.6](../chapter-09.md#s2-6)

---

<a id="q2-7-1"></a>

### Câu 2.7.1 — Ba quyết định thể hiện "giữ đơn giản"

*Nêu ba quyết định cụ thể trong chương thể hiện phương châm "giữ đơn giản", mỗi quyết định kèm rủi ro mà nó tránh được.*

Ví dụ (chọn ba):
1. **Chỉ dùng active/passive hoặc active/read pool** (tr. 241): tránh topology không bảo trì nổi; tránh active-active (không nhất quán, mất năng lực failover, tr. 244–245) và ring (một node hỏng là vòng đứt, tr. 246).
2. **Không dùng filter** trừ khi thật cần (tr. 238–239, 255): tránh lọc theo database mặc định một cách bất ngờ, mất khả năng point-in-time recovery, chặn nhầm event và routine.
3. **Binlog chỉ một định dạng, ROW** (tr. 231): tránh nội dung binlog khó đoán của MIXED và lệch dữ liệu của statement-based.
4. **Không cấu hình sẵn replication chiều ngược** (dual source active-passive) (tr. 245): vì dựng replication là bước dễ tự động hóa trong failover; cấu hình sẵn chỉ trói vào một node và gây nhầm lẫn.
5. **Multisource chỉ dùng tạm** (tr. 247): tránh topology lâu dài phức tạp.

[về mục 2.7](../chapter-09.md#s2-7)

<a id="q2-7-2"></a>

### Câu 2.7.2 — Bản sao đầy đủ kể cả quyền

*Vì sao sách nhấn mạnh replica phải là bản sao đầy đủ **kể cả quyền**? Điều gì có thể xảy ra trong failover nếu quyền bị lọc khỏi replication?*

Sách khuyên dùng replication để phản chiếu **toàn bộ** dữ liệu, kể cả quyền, và giữ replica giống source về mọi mặt (tr. 255). Replica là **đích failover**: ngay khi nó thành source, ứng dụng phải kết nối và có đúng quyền như trước.

Nếu lọc bảng hệ thống để chặn `GRANT` lan sang replica, bạn còn chặn luôn event và routine (tr. 238). Khi failover (**Ví dụ minh họa do AI xây dựng**): user mới hay quyền mới chỉ có trên source cũ; sau khi promote, ứng dụng gặp lỗi đăng nhập hay thiếu quyền, hoặc user replication không tồn tại khiến các replica không trỏ về được. Sự cố kiểu này lộ ra đúng lúc bất tiện nhất, khi đang disaster recovery (tr. 238–239). Trong thí nghiệm, user `repl` được tạo trên source **trước** khi dựng replica và replicate sang mọi server, nên mọi lần trỏ lại ở TN5 đều dùng được ngay.

[về mục 2.7](../chapter-09.md#s2-7)

<a id="q2-7-3"></a>

### Câu 2.7.3 — Cấu hình replication tối thiểu cho hệ OLTP mới

*Tổng hợp: liệt kê cấu hình replication tối thiểu cho một hệ OLTP mới theo khuyến nghị của chương (định dạng, định danh vị trí, crash safety, bảo vệ replica, đo lag, topology), mỗi mục kèm trang.*

| Hạng mục | Khuyến nghị | Trang |
| --- | --- | --- |
| Định dạng | `binlog_format=ROW` | tr. 231 |
| Vị trí | GTID bật (`gtid_mode=ON`, `enforce_gtid_consistency=ON`), `SOURCE_AUTO_POSITION=1` | tr. 232 |
| Crash safety | `innodb_flush_log_at_trx_commit=1`, `sync_binlog=1`, vị trí replication trong bảng, `relay_log_recovery=ON` | tr. 232–233 |
| Bảo vệ replica | `super_read_only=ON` trong cấu hình khởi động của mọi replica, kể cả source cũ sau failover | tr. 240, 250 |
| Song song | Applier đa luồng, chỉnh số worker theo mức bận đo bằng Performance Schema; `replica_preserve_commit_order` | tr. 235–236 |
| Đo lag | Heartbeat (ví dụ `pt-heartbeat`), không dựa vào `Seconds_Behind_Source` | tr. 249 |
| Giám sát | Đĩa trên source và replica, trạng thái và lỗi thread, độ trễ của delayed replica | tr. 248 |
| Topology | Active/passive (n+2 vật lý; n+1 trên cloud nếu dữ liệu nhỏ) hoặc active/read pool với ít nhất một đích failover giống source | tr. 241–243 |
| Tránh | Filter, ring, active-active; semisync không thay cho bảo đảm dữ liệu | tr. 237, 244–246, 255 |
| Quy trình | Planned và unplanned promotion viết sẵn, luyện tập; `server_id` duy nhất và tường minh | tr. 240–241, 251–252 |

**Bổ sung ngoài sách:** trên 8.0.42, các mục định dạng, `sync_binlog`, flush, bảng vị trí và applier đa luồng đã là mặc định; cần tự đặt GTID, `relay_log_recovery`, `super_read_only` và `server_id` ([TN1](../experiments/chapter-09.md#tn1)).

[về mục 2.7](../chapter-09.md#s2-7)
