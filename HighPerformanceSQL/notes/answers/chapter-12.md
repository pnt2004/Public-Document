# Đáp án — Chương 12: MySQL trên cloud

[← Về ghi chú chương 12](../chapter-12.md)

> "tr. N" là số trang in; trang PDF = N + 22. Nhận định về dịch vụ cloud là **theo sách, thời điểm 2021**. Phần ghi **Bổ sung ngoài sách** là kiến thức hay phép tính thêm của người viết ghi chú. Hãy tự trả lời trước khi đọc.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Quy trình replication của chương 9 có dùng nguyên được trong Aurora cluster?

*Một đồng nghiệp nói: "Aurora tương thích MySQL, nên mọi quy trình replication của chương 9 (GTID failover, đo lag, kiểm tra nhất quán) đều dùng nguyên được trong cluster." Câu này sai ở đâu? Khi nào bạn vẫn phải làm việc với binlog trên Aurora?*

**Sai ở hai chỗ.**

1. **"Tương thích" chỉ nói về giao diện.** Sách nhấn mạnh data store MySQL-compatible có giao diện SQL nhưng cơ chế bên trong có thể hoàn toàn khác Oracle MySQL (tr. 314).
2. **Replication trong Aurora cluster không phải binlog replication.** Đó là cơ chế độc quyền của Amazon. Mọi instance trong cluster dùng chung một tầng storage, nên replication trong cluster làm ở tầng block storage (tr. 315). Không có luồng binlog từ source tới replica để đo lag hay so GTID theo cách chương 9 mô tả. Quy trình promotion dựa trên GTID và `AUTO_POSITION` (chương 9, tr. 240) vì vậy không áp dụng nguyên vẹn bên trong cluster.

**Khi nào vẫn cần binlog.** Aurora vẫn hỗ trợ ghi binlog theo định dạng của community server, cho hai nhu cầu sách nêu (tr. 315):

- replicate từ một Aurora cluster sang **cluster khác**;
- các mục đích khác của binlog, ví dụ **change data capture** (chú thích 2, tr. 315).

**Bổ sung ngoài sách.** Theo bài báo SIGMOD 2017 mà sách dẫn, instance đọc dùng chung volume và nhận luồng redo từ instance ghi. Vì vậy thêm một instance đọc không cần chép dữ liệu, khác hẳn việc dựng replica MySQL từ backup.

[về mục 2.1](../chapter-12.md#s2-1)

---

<a id="q2-1-2"></a>

### Câu 2.1.2 — Hình 12-1 với nhu cầu vừa xuyên region vừa ưu tiên ghi

*Theo Hình 12-1, một hệ production cần replication dưới một giây giữa các region và đồng thời muốn ưu tiên write availability sẽ được dẫn tới đâu? Vì sao câu hỏi thứ ba của hình thực chất là một câu hỏi về ưu tiên?*

**Đường đi trên hình** (tr. 317): không phải staging → "Có" với replication dưới một giây giữa các region → "Có" với ưu tiên write availability → **Aurora Multi-Master**.

**Vì sao đó là câu hỏi ưu tiên.** Multi-Master là giải pháp HA khi write availability **trong một region** là ưu tiên cao nhất, và không thể trộn với Global Database trong cùng triển khai (tr. 316). Như vậy, chọn Multi-Master nghĩa là **bỏ** nhu cầu replication xuyên region dưới một giây mà bạn vừa trả lời "Có". Sách nói thẳng: không thể đạt cùng lúc multiwriter HA và replication xuyên region dưới một giây (tr. 316). Câu hỏi thứ ba vì thế buộc bạn chọn: **write availability hay replication theo vùng**. Sách khuyên dùng chính các lựa chọn của Aurora để trình bày đánh đổi này và dẫn dắt thảo luận sản phẩm khó (tr. 316).

**Lưu ý thêm** (theo sách, 2021): Multi-Master chạy lõi MySQL 5.6, có số node tối đa, và là giải pháp "rất có chủ kiến" về availability và consistency; cần cân nhắc kỹ ràng buộc trước khi chọn (tr. 316).

[về mục 2.1](../chapter-12.md#s2-1)

---

<a id="q2-1-3"></a>

### Câu 2.1.3 — Vì sao Cloud SQL tắt SUPER và plugin; hệ quả cho audit log

*Cloud SQL chạy community server nhưng tắt `SUPER` và việc nạp plugin. Giải thích nguyên nhân gốc, và hệ quả với một đội cần audit log để đạt chứng nhận compliance.*

**Nguyên nhân gốc theo sách.** Cloud SQL tắt một số tính năng **để phục vụ tính multitenant và tính managed** của sản phẩm (tr. 317).

**Diễn giải (Bổ sung ngoài sách).** `SUPER` cho phép thao tác ảnh hưởng toàn server: đổi biến toàn cục, kill session của người khác, cấu hình replication, ghi bất chấp `read_only` (chỉ `super_read_only` chặn được, chương 5 tr. 122). Nạp plugin là chạy code tùy ý trong process `mysqld`. Nếu khách hàng làm được những việc đó, nhà cung cấp không còn giữ được bảo đảm về HA, bảo mật và cách ly giữa các tenant. Từ MySQL 8.0, `SUPER` deprecated và được chia thành dynamic privilege (ví dụ `SYSTEM_VARIABLES_ADMIN`, `CONNECTION_ADMIN`, `BINLOG_ADMIN`); dịch vụ managed có thể chỉ cấp một phần.

**Hệ quả cho audit log.** Chương 13 đề xuất ghi log truy cập dữ liệu ngay ở tầng database bằng **plugin**: Percona audit log plugin nếu chạy bản của Percona, hoặc MySQL Enterprise Audit plugin ([Chương 13](../chapter-13.md#data-access-logging), tr. 334). Trên một dịch vụ cấm nạp plugin, cách đó không dùng được. Sách cho biết Cloud SQL có **giải pháp audit logging riêng** để hỗ trợ compliance (chú thích 3, tr. 317). Đội cần kiểm tra giải pháp đó có đáp ứng yêu cầu bằng chứng của kiểm toán viên không (ghi được truy cập đọc lẫn ghi, định dạng, nơi lưu, ai xóa được), thay vì giả định dùng được plugin quen thuộc.

[về mục 2.1](../chapter-12.md#s2-1)

---

<a id="q2-1-4"></a>

### Câu 2.1.4 — Ba hạn chế của managed và công cụ bị mất

*Nêu ba hạn chế về quan sát hoặc kiểm soát của managed MySQL, và với mỗi hạn chế, chỉ ra một công cụ hay kỹ thuật của các chương trước mà bạn mất.*

Sách liệt kê các hạn chế ở tr. 314. Ví dụ ba cặp (phần "mất gì" là diễn giải của người viết ghi chú):

| Hạn chế (tr. 314) | Công cụ hay kỹ thuật bị mất |
| --- | --- |
| Không truy cập hệ điều hành và filesystem | `vmstat`, `iostat`, `mpstat`, `pt-diskstats` trên host để xem CPU, steal, I/O ([Chương 4](../chapter-04.md#os-status), tr. 92–95); chọn filesystem, scheduler |
| Không dựng được topology nâng cao | Các topology của [Chương 9](../chapter-09.md), ví dụ delayed replica (tr. 233) mà sách nêu là lợi thế của VM (tr. 318) |
| Backup và restore chỉ theo phương án của nhà cung cấp | Tự chọn XtraBackup, LVM snapshot, chiến lược backup binlog riêng ([Chương 10](../chapter-10.md)) |

Các hạn chế khác cũng đúng: bị giới hạn trong process (không đặt được mọi biến, quyền; ví dụ `SUPER` trên Cloud SQL, tr. 317); chỉ quan sát được những gì nhà cung cấp đưa; gặp sự cố phải mở ticket và chờ (tr. 314).

[về mục 2.1](../chapter-12.md#s2-1)

---

<a id="q2-1-5"></a>

### Câu 2.1.5 — Khi nào dùng RDS Proxy; chuyện gì xảy ra nếu không có

*Sách khuyên dùng RDS Proxy trong trường hợp nào? Dự đoán điều gì xảy ra với một Aurora cluster không có proxy khi latency tăng và ứng dụng phản ứng bằng cách mở thêm kết nối.*

**Khi nào** (tr. 315). Khi đặt database **mission-critical** lên Aurora, và nhất là khi biết phía ứng dụng có khả năng gây **connection storm**. RDS Proxy giúp lượng kết nối mới không đè lên database.

**Dự đoán khi không có proxy** (dựa trên định nghĩa connection storm ở chương 2, tr. 31). Latency tăng → ứng dụng mở thêm kết nối → database tốn tài nguyên để xử lý lượng kết nối mới → còn ít tài nguyên cho truy vấn → latency tăng thêm → ứng dụng lại mở thêm kết nối. Số kết nối trống trong `max_connections` giảm đột ngột; khi chạm trần, kết nối mới bị từ chối và availability bị đe dọa. Đây là một **vòng phản hồi dương**.

**Bổ sung ngoài sách — vì sao proxy cắt được vòng đó.** Proxy giữ một pool kết nối cố định tới database và cho nhiều kết nối phía ứng dụng dùng chung. Kết nối mới từ ứng dụng dừng ở proxy (có thể xếp hàng), không biến thành kết nối mới tới MySQL. Đây cũng là vai trò connection pooling của ProxySQL và Vitess ở [Chương 11](../chapter-11.md) (tr. 303, 306).

[về mục 2.1](../chapter-12.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Ước lượng vCPU cho server 64 core

*Dùng công thức của sách, ước lượng số vCPU cho một server 64 core có mức dùng CPU p95 là 25%, mục tiêu 50%. Hệ số × 2 đến từ đâu? Trong trường hợp nào kết quả có thể lệch tới hai lần?*

**Phép tính** (**Bổ sung ngoài sách**, theo công thức tr. 319): 64 × 25% × 2 = **32 vCPU**. Nếu không có type 32 thì làm tròn lên type gần nhất hoặc xem custom machine type, nhớ rằng custom có thể đắt hơn (chú thích 5, tr. 319).

**Hệ số × 2.** Đó là 1 ÷ 0,5, tức chia cho mức sử dụng mục tiêu 50%. Ví dụ của sách hỏi "cần bao nhiêu core để chạy ở 50% utilization" (tr. 319). Mục tiêu 50% đến từ hộp cảnh báo: CPU nên ở 50% lúc bình thường, đỉnh 65–70% (tr. 320). Dạng tổng quát: vCPU ≈ core × mức dùng ÷ mức mục tiêu.

**Khi nào lệch tới hai lần** (**Bổ sung ngoài sách**). Khi đơn vị không nhất quán. Ở nhiều nhà cung cấp, một vCPU là một luồng phần cứng (hyperthread). Nếu "64 core" là core vật lý có hyperthreading (128 luồng) và mức 25% được tính trên core vật lý, thì 32 vCPU chỉ tương đương 16 core vật lý, tức một nửa năng lực cần. Hãy đếm bằng cùng một đơn vị (ví dụ số CPU logic hệ điều hành thấy) và kiểm tra định nghĩa vCPU của nhà cung cấp. Ngoài ra, chọn "đỉnh" hay "p95" (công thức ghi 95%, ví dụ dùng đỉnh) cũng làm kết quả khác nhau.

[về mục 2.2](../chapter-12.md#s2-2)

---

<a id="q2-2-2"></a>

### Câu 2.2.2 — Vì sao chọn loại đĩa khó nhất; cách sửa; ghi nhiều gặp nút thắt đĩa trước

*Vì sao sách coi chọn loại đĩa là quyết định phức tạp nhất, trong khi machine type thì "dynamic"? Nêu hai cách sửa khi đã chọn sai loại đĩa, và vì sao workload ghi nhiều gặp nút thắt đĩa trước.*

**Vì sao khó nhất** (tr. 320). Machine type đổi được bằng resize, có khi chỉ cần một lần reboot nhanh để thêm CPU. Loại đĩa thì một khi đã chứa dữ liệu, chuyển sang loại khác thường phải gắn đĩa thứ hai và chép dữ liệu sang. Thêm vào đó, đúng sai phụ thuộc mạnh vào workload, mà workload thì phải đo hoặc ước lượng trước (tr. 320–322).

**Hai cách sửa.**

1. Theo sách: gắn một đĩa thứ hai loại mới và **chép dữ liệu sang** (tr. 320).
2. **Bổ sung ngoài sách:** dựng một **replica mới** trên loại đĩa mới (từ backup hay snapshot, rồi đuổi binlog), sau đó **failover có kế hoạch** sang nó theo [Chương 9](../chapter-09.md#planned-promotion) (tr. 239–240). Cách này tránh chép dữ liệu ngay trên source đang phục vụ.

**Vì sao ghi nhiều gặp nút thắt đĩa trước** (tr. 321). Đọc có thể được bộ nhớ che chắn: working set nằm trong buffer pool thì không cần xuống đĩa. Ghi thì **luôn** xuống đĩa. **Bổ sung ngoài sách:** cache chỉ hoãn và gom được các lần ghi, không xóa chúng (chương 4, tr. 76–77); với thiết lập ACID, mỗi commit phải flush redo log và đồng bộ binlog, nên latency commit bị chặn dưới bởi fsync; trên cloud lại không có write cache có pin để đệm (tr. 321).

[về mục 2.2](../chapter-12.md#s2-2)

---

<a id="q2-2-3"></a>

### Câu 2.2.3 — Đĩa local, RAID 1, host hỏng

*Source chạy trên đĩa local có software RAID 1. Host vật lý hỏng và VM khởi động lại trên host khác. Dự đoán chuyện gì xảy ra với dữ liệu, và lời khuyên "cứ để server quay lại" của sách còn dùng được không?*

**Dự đoán.** Dữ liệu trên đĩa local **nhiều khả năng mất**. Sách nói đĩa local được coi là đĩa cho dữ liệu tạm: phần cứng chạy VM hỏng thì có thể mất toàn bộ dữ liệu trên đĩa local; thậm chí tắt instance rồi bật lại có thể ra host khác với **đĩa trống** (tr. 321). Software RAID 1 nhân bản giữa các đĩa **của cùng host**: nó giúp khi một đĩa hỏng, không giúp khi cả host hỏng hay VM bị dời. Sách chỉ nói RAID "ít nhất giảm khả năng" mất dữ liệu (tr. 321).

**Lời khuyên "chờ server quay lại"** (tr. 322) **không còn dùng được** trong trường hợp này, vì không có dữ liệu để quay lại. Phải **failover** sang replica ([Chương 9](../chapter-09.md#s2-3), tr. 239), rồi dựng lại một replica mới thay cho máy cũ. Điều này cho thấy lời khuyên "chờ" ngầm giả định **đĩa gắn qua mạng**, nơi dữ liệu tồn tại độc lập với host.

**Bài học.** Chọn đĩa local thì độ bền phải đến từ **replica trên host khác** và **backup tự làm** (LVM, XtraBackup; tr. 321), và runbook host reboot phải khác với trường hợp đĩa mạng.

[về mục 2.2](../chapter-12.md#s2-2)

---

<a id="q2-2-4"></a>

### Câu 2.2.4 — Script @reboot tắt read_only không hỏi ai

*Một script `@reboot` tắt `read_only` mỗi khi source khởi động lại, không hỏi ai. Mô tả kịch bản khiến script này gây hại, và sửa nó theo điều kiện sách nêu ở tr. 323.*

**Kịch bản gây hại** (**Bổ sung ngoài sách**). Source sập; vì lâu không thấy nó quay lại, người trực (hoặc công cụ) **failover** sang replica, ứng dụng bắt đầu ghi vào source mới. Sau đó VM cũ khởi động lại, script tự tắt `read_only`. Nếu còn ứng dụng hay kết nối nào trỏ tới máy cũ (DNS cũ, cấu hình cũ), giờ có **hai nơi nhận ghi** (*split brain*): dữ liệu phân kỳ và rất khó hợp nhất.

**Điều kiện của sách** (tr. 323). Gợi ý tắt `read_only` tự động bằng `@reboot` chỉ đúng khi **truy vấn được một hệ thống để xác định máy này có nên nhận ghi không**. Kết hợp với chương 9: source cũ khi quay lại phải mặc định bật `super_read_only` (tr. 240).

**Cách sửa** (**Bổ sung ngoài sách**):

1. Tệp cấu hình đặt `super_read_only=ON` để mọi lần khởi động đều chỉ đọc.
2. Script hỏi nguồn sự thật (service discovery, công cụ quản lý topology). Không hỏi được thì **giữ chỉ đọc** và báo người.
3. Chỉ khi nguồn sự thật xác nhận máy này vẫn là source và chưa có failover, chạy `SET GLOBAL read_only = OFF;` (theo chú thích 6 của chương 9, tr. 240, lệnh này cũng tắt `super_read_only`).
4. Dùng `SET GLOBAL`, không dùng `SET PERSIST`, để lần khởi động sau vẫn bắt đầu chỉ đọc.

Phác thảo đầy đủ ở [bài tập 4](../experiments/chapter-12.md#bt4).

[về mục 2.2](../chapter-12.md#s2-2)

---

<a id="q2-2-5"></a>

### Câu 2.2.5 — Tần suất kiểm tra auto-extend cho đĩa 500 GB

*Đĩa mạng 500 GB, throughput tối đa 250 MB/s, ngưỡng auto-extend 90%. Code kiểm tra phải chạy thường xuyên tới mức nào? Nêu thêm hai rủi ro sách cảnh báo về auto-extend.*

**Phép tính** (**Bổ sung ngoài sách**, theo cách sách gợi ý ở tr. 324). Phần trống ở mốc 90%: 10% × 500 GB = 50 GB = 50.000 MB. Thời gian lấp đầy ở throughput tối đa: 50.000 ÷ 250 = **200 giây** (khoảng 3,3 phút). Code phải chạy **thường xuyên hơn** khoảng đó, và chính xác hơn: chu kỳ kiểm tra **cộng** thời gian gọi API và mở rộng filesystem phải nhỏ hơn 200 giây. Ví dụ, nếu mở rộng mất 2 phút thì chu kỳ phải dưới 80 giây. Nếu không làm được, hãy hạ ngưỡng cho đĩa này (đặt ngưỡng theo thời gian còn lại tới khi đầy, không theo phần trăm cố định).

**Hai rủi ro khác sách cảnh báo** (tr. 324):

- Tiến trình **chạy mất kiểm soát**, cứ mở rộng mãi, có thể tạo ra một volume **64 TB** và một hóa đơn đắt bất ngờ. **Bổ sung ngoài sách:** đặt trần kích thước và số lần mở rộng tối đa mỗi ngày.
- Lệnh gọi API mở rộng có thể làm **đĩa stall** một lúc; phải **thử dưới tải** để chắc không ảnh hưởng người dùng.

Xem bảng t_đầy cho nhiều cỡ đĩa ở [bài tập 2](../experiments/chapter-12.md#bt2).

[về mục 2.2](../chapter-12.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — Được và mất của managed và VM

*Với managed MySQL và MySQL trên VM, mỗi bên nêu hai thứ bạn được và hai thứ bạn trả giá, kèm trang sách.*

| | Được | Trả giá |
| --- | --- | --- |
| **Managed** | Bắt đầu rất nhanh: vài click hay `terraform apply` có database kèm replica và backup định kỳ (tr. 314). Nhiều việc vận hành được làm sẵn, ví dụ Aurora lo snapshot backup, audit logging, replication trong region (tr. 314); Cloud SQL lo HA, mã hóa at rest, nâng cấp (tr. 318) | Thường đắt hơn, ít kiểm soát (tr. 313). Không truy cập OS, filesystem; sự cố phải mở ticket; không có topology nâng cao; backup giới hạn (tr. 314). Vẫn có downtime bảo trì (tr. 318) |
| **VM** | Toàn quyền như bare metal: replica ở region khác, delayed replica, backup tùy biến, soi OS và filesystem (tr. 318). Tối ưu chi phí được, ví dụ resize theo mùa (tr. 319), auto-extend đĩa (tr. 324) | Tốn thời gian và overhead vận hành (tr. 313). Tự lo sizing đĩa, backup (tr. 322), host reboot (tr. 322–323); loại đĩa khó đổi (tr. 320) |

Ý chung của sách: không có bữa trưa miễn phí, lựa chọn nào cũng có đánh đổi (tr. 324).

[về mục 2.3](../chapter-12.md#s2-3)

---

<a id="q2-3-2"></a>

### Câu 2.3.2 — Cùng ứng dụng, khuyến nghị khác nhau theo độ trưởng thành

*Cùng một ứng dụng, vì sao khuyến nghị có thể khác nhau cho một startup năm kỹ sư và một công ty có đội DBA với yêu cầu compliance? Gắn lập luận với câu "đặt đánh đổi vào bối cảnh và độ trưởng thành" của sách.*

Sách nói việc hữu ích nhất là đặt đánh đổi vào bối cảnh **cách doanh nghiệp vận hành** và **giai đoạn trưởng thành** của nó (tr. 324). Cùng một bảng đánh đổi, nhưng **trọng số** khác nhau. (Phần dưới là **Ví dụ minh họa do AI xây dựng**; sách không định nghĩa các giai đoạn.)

- **Startup năm kỹ sư.** Nguồn lực khan hiếm nhất là thời gian và kiến thức vận hành. Managed giảm gánh nặng nhận thức và cho bắt đầu nhanh (tr. 313–314); chi phí cao hơn và ít kiểm soát là cái giá chấp nhận được. Khuyến nghị nghiêng về managed, kèm kiểm thử tương thích phiên bản (tr. 314).
- **Công ty có đội DBA và yêu cầu compliance.** Đã có người gánh được host reboot, backup, sizing đĩa (tr. 322). Nhu cầu quan sát sâu, topology riêng (delayed replica, replica xuyên region), kiểm soát chi tiết backup và bucket (tr. 318, 323) và tối ưu chi phí ở quy mô lớn (tr. 319, 324) có trọng số cao hơn. Khuyến nghị có thể nghiêng về VM, hoặc managed nhưng phải kiểm tra kỹ audit logging và phương án backup có đáp ứng kiểm soát compliance ([Chương 13](../chapter-13.md)).

Điểm mấu chốt: **câu trả lời đổi khi trọng số đổi**, nên phải trình bày đánh đổi kèm bối cảnh, không đưa một đáp án chung.

[về mục 2.3](../chapter-12.md#s2-3)

---

<a id="q2-3-3"></a>

### Câu 2.3.3 — Khi nào snapshot đĩa mạng dùng được làm backup

*Nối chương 9, 10 và 12: dưới những điều kiện nào một snapshot đĩa mạng trên cloud dùng được làm backup? Điều gì có thể làm nó vô dụng?*

**Điều kiện theo chương 12** (tr. 321): có **thiết lập tuân thủ ACID** (`innodb_flush_log_at_trx_commit=1`, `sync_binlog=1`, chú thích 6) và một giải pháp backup **thiết kế đúng**. Khi đó snapshot ở bất kỳ lúc nào cũng phục hồi được qua crash recovery bình thường.

**Điều có thể làm nó vô dụng.**

- **Hạ durability** trên máy dùng để snapshot. Chương 9 cảnh báo: tạm đặt `sync_binlog=0` và `innodb_flush_log_at_trx_commit=0` trên replica để giảm lag có thể khiến **không phục hồi được từ backup**, và chú thích 7 nói điều này thường đúng khi backup bằng LVM snapshot hoặc snapshot đĩa trên cloud (tr. 253).
- **Coi snapshot là bản sao độc lập khi nó không phải.** Chương 10, "Backup Myth #2": snapshot LVM, ZFS, SAN không phải backup thật vì là copy-on-write, dùng chung block với dữ liệu đang chạy (tr. 272). **Bổ sung ngoài sách:** snapshot block storage của cloud lớn thường lưu tách khỏi volume, nhưng cần kiểm tra với nhà cung cấp của bạn.
- **Bổ sung ngoài sách:** data file, redo log, undo và binlog nằm trên **nhiều volume** snapshot riêng lẻ, nên các phần không cùng thời điểm; không bao giờ **thử phục hồi**; snapshot chỉ nằm cùng tài khoản và region với dữ liệu gốc.

**Tóm lại.** Snapshot là một công cụ backup tốt khi thiết lập durability đúng, dữ liệu nằm cùng một volume (hoặc snapshot nhất quán nhiều volume), có bản sao ở nơi khác, và **đã thử phục hồi**. Binlog backup lên bucket (tr. 323) bổ sung khả năng phục hồi tới một thời điểm giữa hai snapshot (chương 10, tr. 269).

[về mục 2.3](../chapter-12.md#s2-3)
