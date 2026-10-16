# Đáp án — Chương 4: Operating System and Hardware Optimization

Quay lại ghi chú: [../chapter-04.md](../chapter-04.md). Hãy tự trả lời trước khi đọc. "tr. N" là số trang in; **Bổ sung ngoài sách**, **Ví dụ minh họa do AI xây dựng** và **Kết quả đã đo** theo quy ước của bộ ghi chú.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Hai cách khiến CPU bị cạn

*Sách nêu hai cách khiến CPU bị cạn. Phân biệt chúng, và cho biết mỗi cách gợi ý hướng điều tra khác nhau ra sao.*

Theo tr. 75, CPU bão hòa khi (1) MySQL cố chạy **quá nhiều truy vấn song song**, hoặc (2) **một số ít truy vấn chạy quá lâu** trên CPU.

- Cách (1) là vấn đề **concurrency**: tổng nhu cầu vượt số lõi, truy vấn phải xếp hàng chờ CPU. Hướng điều tra: số kết nối đang chạy, connection pool của ứng dụng, đỉnh tải; `vmstat` thường có `r` lớn hơn số CPU.
- Cách (2) là vấn đề **hiệu quả của từng truy vấn**: mỗi truy vấn tự nó làm quá nhiều việc. Hướng điều tra: truy vấn nào có nhiều rows examined, sắp xếp, bảng tạm (Performance Schema ở chương 3; index và truy vấn ở chương 7, 8).

Phân biệt hai cách quan trọng vì giải pháp khác nhau: cách (1) cần giới hạn concurrency hoặc thêm lõi; cách (2) cần làm truy vấn ít việc hơn, và thêm lõi gần như không giúp (mỗi truy vấn chỉ dùng một CPU, tr. 76).

[về mục 2.1](../chapter-04.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — SSD làm I/O hiếm bão hòa, nhưng truy cập vượt buffer pool vẫn chậm 9 lần

*Vì sao SSD làm bão hòa I/O hiếm hơn, mà thí nghiệm 2 vẫn thấy truy cập ngẫu nhiên vượt buffer pool chậm hơn khoảng 9 lần truy cập nằm trong buffer pool? Hai điều này có mâu thuẫn không?*

Không mâu thuẫn. Sách nói SSD làm **bão hòa** I/O hiếm hơn và truy vấn phải chạm đĩa vẫn có hiệu năng "tạm ổn" (tr. 75); sách **không** nói đọc SSD nhanh bằng đọc RAM. Ngay tr. 76 vẫn nhấn mạnh I/O đĩa chậm hơn truy cập bộ nhớ nhiều bậc độ lớn.

**Kết quả đã đo** (thí nghiệm 2): mẫu truy cập "đều" cần khoảng 0,92 lần đọc đĩa mỗi tra cứu và mất khoảng 4,95 giây cho 20.000 tra cứu, so với khoảng 0,54 giây khi mọi trang đã nằm trong buffer pool. Ước tính (**Bổ sung ngoài sách**, phép tính): phần chênh khoảng 4,4 giây chia cho khoảng 18.400 lần đọc cho khoảng 0,24 ms mỗi lần đọc trên SSD NVMe, tức thiết bị không hề bão hòa; nhưng con số đó vẫn gấp nhiều lần mức khoảng 27 micro giây cho trọn một lần tra cứu khi trang đã nằm trong buffer pool (kể cả chi phí vòng lặp của stored procedure). "Không bão hòa" và "chậm hơn nhiều so với RAM" là hai mệnh đề khác nhau.

[về mục 2.1](../chapter-04.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Lỗi "SSH" ở tr. 75

*Câu ở tr. 75 viết SSD nhanh hơn "SSH" 10–20 lần. Bạn sửa thế nào, dựa vào căn cứ nào, và con số 10–20 lần nên hiểu với điều kiện gì?*

Sửa thành **HDD**. Căn cứ:

1. Câu ngay trước vừa giới thiệu từ viết tắt "hard disk drive (HDD)", và cả đoạn so SSD với ổ cứng.
2. SSH (Secure Shell) là giao thức đăng nhập từ xa, không phải thiết bị lưu trữ; ở tr. 91 sách dùng SSH đúng nghĩa đó (tiến trình SSH có thể bị OOM killer giết).
3. Ảnh trang PDF 97 xác nhận chữ "SSH" có thật trong bản in, nên đây là lỗi in chứ không phải lỗi trích xuất.

Về con số (**Bổ sung ngoài sách**): sách không nói "nhanh hơn" theo chỉ số nào. Với throughput tuần tự, khoảng cách SSD và HDD nhỏ hơn; với latency và IOPS ngẫu nhiên thì thường lớn hơn nhiều so với 10–20 lần. Sách tự nói cải thiện lớn nhất của flash là ở random I/O (tr. 78). Nên coi 10–20 lần là ước lượng thô.

[về mục 2.1](../chapter-04.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — Cạn bộ nhớ do đâu

*Theo sách, cạn bộ nhớ thường do đâu? Điều đó gợi ý sửa ở đâu trước khi nghĩ tới mua thêm RAM?*

Tr. 76: cạn bộ nhớ vẫn có thể xảy ra nhưng thường **chỉ khi cấp phát quá nhiều bộ nhớ cho MySQL**. Cách đặt cấu hình để tránh nằm ở "Configuring Memory Usage" của chương 5 (tr. 109). Vậy trước hết phải xem **cấu hình**: buffer pool, bộ nhớ theo kết nối, bộ nhớ chừa cho hệ điều hành (chương 5, tr. 109 nói dấu hiệu tốt nhất là hệ điều hành không swap). Mua thêm RAM không giải quyết được một cấu hình đòi bộ nhớ vượt mức có sẵn; nó chỉ dời điểm vỡ đi.

[về mục 2.1](../chapter-04.md#s2-1)

<a id="q2-2-1"></a>

### Câu 2.2.1 — Low latency cần CPU nhanh, high throughput cần nhiều CPU

*Vì sao low latency đòi CPU nhanh còn high throughput đòi nhiều CPU? Giả định nào của sách nằm sau lập luận này?*

Giả định then chốt (tr. 76): **mỗi truy vấn chỉ dùng một CPU**.

- Latency của một truy vấn CPU-bound bằng lượng việc chia cho tốc độ **một** lõi. Thêm lõi không làm nó nhanh hơn; chỉ lõi nhanh hơn (hoặc ít việc hơn) mới giúp.
- Throughput là số truy vấn hoàn tất mỗi giây. Nếu có nhiều truy vấn cùng lúc, mỗi lõi chạy một truy vấn, nên nhiều lõi thì nhiều truy vấn chạy song song mà không chờ nhau.

**Bổ sung ngoài sách:** MySQL 8.0 có vài ngoại lệ hẹp (ví dụ `innodb_parallel_read_threads` từ 8.0.14 cho `CHECK TABLE` và `SELECT COUNT(*)` không điều kiện), nhưng truy vấn thông thường vẫn chạy trên một luồng.

[về mục 2.2](../chapter-04.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — CPU trung bình 6% mà truy vấn vẫn chậm

*Một server 32 lõi báo CPU trung bình 6%, nhưng truy vấn quan trọng vẫn chậm và có vẻ tốn CPU. Theo lời khuyên ở tr. 76, bạn kiểm tra gì tiếp, và khả năng nào giải thích được tình trạng này?*

Tr. 76 khuyên đừng chỉ nhìn tổng tải mà xem **cân bằng CPU và I/O** của các truy vấn quan trọng, và xem **CPU có tải đều không**. 6% của 32 lõi chỉ khoảng 1,9 lõi: hoàn toàn có thể là **một lõi chạy 100%** và vài lõi khác lặt vặt (**Bổ sung ngoài sách**, phép tính). Một truy vấn chỉ dùng một CPU, nên nếu nó CPU-bound, nó bị giới hạn bởi một lõi bất kể máy có bao nhiêu lõi.

Kiểm tra tiếp: `mpstat` theo từng CPU (tr. 95) để thấy lõi bị dồn tải; Performance Schema để xem truy vấn đó đọc bao nhiêu hàng. Hướng xử lý: làm truy vấn ít việc hơn, hoặc nếu không thể, lõi có tần số cao hơn; thêm lõi không giúp.

[về mục 2.2](../chapter-04.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Tăng client vượt số lõi

*Dự đoán: tăng số client đồng thời chạy một truy vấn CPU-bound từ 1 lên gấp đôi số lõi thì latency từng truy vấn và throughput đổi thế nào? Đối chiếu với thí nghiệm 4.*

**Dự đoán** theo tr. 75–76: khi số client còn nhỏ hơn số lõi, latency từng truy vấn gần như không đổi và throughput tăng gần tuyến tính. Khi số client vượt số lõi thật, các truy vấn phải chờ CPU: latency từng truy vấn tăng, throughput chững lại. Đây là cách (1) của cạn CPU.

**Kết quả đã đo** (thí nghiệm 4, laptop 8 nhân 16 luồng, có tải nền khoảng 20% từ tiến trình khác): latency phía server trung vị khoảng 97 ms (1 client), 103 ms (2), 106 ms (4), 123 ms (8), 189 ms (16). Đến 4 client gần như không đổi; ở 8 client bắt đầu tăng (máy còn phải chạy tiến trình khác); ở 16 client, bằng số luồng phần cứng, latency gần gấp đôi vì hai luồng SMT chia một lõi và có hàng đợi CPU. Tổng thời gian phía client của `mysqlslap` trên Windows quá nhiễu nên không dùng để kết luận throughput.

[về mục 2.2](../chapter-04.md#s2-2)

<a id="q2-3-1"></a>

### Câu 2.3.1 — Cache khử được đọc nhưng không khử được ghi

*Vì sao cache có thể khử hoàn toàn đọc vật lý nhưng không khử được ghi?*

Đọc chỉ cần **một bản sao** của dữ liệu ở đâu đó nhanh; nếu bản sao trong bộ nhớ còn đúng, không cần chạm đĩa nữa. Khi mọi dữ liệu vừa bộ nhớ và cache đã ấm, mọi lần đọc là cache hit: vẫn có đọc logic nhưng không có đọc vật lý (tr. 76).

Ghi thì khác: thay đổi phải **bền vững**, tức phải tới thiết bị không mất dữ liệu khi mất điện. Cache trong RAM mất khi mất điện, nên sớm muộn dữ liệu phải ghi xuống đĩa. Cache chỉ **trì hoãn** và **gom** ghi, không xóa được nhu cầu ghi (tr. 76–77).

**Kết quả đã đo** (thí nghiệm 1 và 3): buffer pool đủ lớn thì quét lại đọc 0 trang từ đĩa; nhưng mọi trang bị sửa đều được flush xuống đĩa (ví dụ 4.121 trang bẩn thì 4.121 trang được ghi).

[về mục 2.3](../chapter-04.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — "Many writes, one flush" và "I/O merging"

*Phân biệt "many writes, one flush" và "I/O merging". Trong thí nghiệm 3, kịch bản nào minh họa cái nào, và giới hạn của mỗi kiểu là gì?*

- **Many writes, one flush** (tr. 77): **cùng một** mẩu dữ liệu đổi nhiều lần trong bộ nhớ, chỉ ghi lần cuối. Ví dụ của sách: bộ đếm tăng 100 lần thành 1 lần ghi. Thí nghiệm 3, kịch bản W1 (5.000 cập nhật cùng một hàng): bảng chỉ có **1** trang bẩn.
- **I/O merging** (tr. 77): **nhiều** mẩu dữ liệu khác nhau được gom để ghi bằng ít thao tác đĩa hơn. Kịch bản W2, W3 (5.000 hàng rải rác) chạm khoảng 4.100 trang khác nhau; việc flush nền của InnoDB có thể gom và sắp xếp các trang này, nhưng số trang phải ghi vẫn khoảng 4.100.

Giới hạn:

- Gom chỉ có lợi khi các thay đổi **tập trung** vào ít trang. Cập nhật rải rác làm số trang phải ghi gần bằng số thay đổi.
- Gom trang không gom được **ghi đồng bộ của commit**: W1 vẫn `fsync` redo log hàng nghìn lần. Và InnoDB còn có trang phụ (undo, header của rollback segment) bị bẩn theo mỗi transaction: W1 có khoảng 650 trang loại này (**Kết quả đã đo**; chi tiết InnoDB là **Bổ sung ngoài sách**).

[về mục 2.3](../chapter-04.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Write-ahead logging và ghi đồng bộ

*Write-ahead logging biến I/O ngẫu nhiên thành tuần tự thế nào? Cái gì vẫn phải ghi đồng bộ, và vì sao ghi đồng bộ hưởng lợi từ write-back cache có pin?*

Theo tr. 77: thay vì flush ngay các trang đã sửa (nằm rải rác, nên là I/O ngẫu nhiên và chậm), hệ thống ghi một **bản ghi thay đổi** vào log file **tuần tự**, nhanh hơn nhiều; luồng nền flush các trang về sau, và có thể tối ưu thứ tự ghi.

Cái vẫn phải ghi đồng bộ: **log** lúc commit. Để commit bền, bản ghi log phải thật sự xuống thiết bị trước khi báo thành công. Ghi đồng bộ (unbuffered) phải xong mới được coi là hoàn tất (tr. 77), nên latency của nó nằm thẳng trong thời gian commit.

Write-back cache có pin giúp vì controller có thể báo "xong" ngay khi dữ liệu nằm trong cache có pin, không cần chờ ghi xuống đĩa, mà vẫn an toàn khi mất điện (tr. 77, 85). Sách nói với BBU, bật cache ghi có thể tăng hiệu năng 20 lần trở lên cho workload flush log nhiều (tr. 85).

[về mục 2.3](../chapter-04.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — Working set và bảng lớn hơn buffer pool

*Working set là gì? Vì sao trong thí nghiệm 2, bảng 71,6 MB với buffer pool 16 MB vẫn có một mẫu truy cập chạy không cần đọc đĩa?*

Working set là phần dữ liệu ứng dụng thật sự cần để làm việc (tr. 77); Summary nói rõ hơn là dữ liệu cần thường xuyên nhất trong một khoảng thời gian (tr. 96). Hiệu năng tốt chỉ đòi working set vừa bộ nhớ, không đòi cả database (tr. 77).

Trong thí nghiệm 2, mẫu "nóng" chỉ tra cứu khóa 1..20.000. Các khóa này nằm trong khoảng 313 trang (khoảng 5 MB, gồm trang lá và vài trang trong của B+tree), nhỏ hơn buffer pool 16 MB. Sau lần đầu nạp 313 trang, các đợt sau đọc **0** trang từ đĩa (**Kết quả đã đo**). Kích thước bảng không quan trọng bằng kích thước phần đang được dùng.

**Bổ sung ngoài sách:** vùng nóng còn sống sót qua pha truy cập rải đều nhờ LRU hai đoạn của InnoDB: trang được truy cập lại sau hơn `innodb_old_blocks_time` (1000 ms trên 8.0.42) được đưa vào đoạn "young", còn trang của pha rải đều bị đẩy ra từ đoạn "old" trước khi kịp được thăng hạng.

[về mục 2.3](../chapter-04.md#s2-3)

<a id="q2-3-5"></a>

### Câu 2.3.5 — Tỷ lệ bộ nhớ/đĩa với HDD và SSD

*Vì sao tỷ lệ bộ nhớ/đĩa quan trọng với HDD nhưng ít quan trọng hơn với SSD? Điều đó có nghĩa là dùng SSD thì bỏ qua được working set không?*

Với HDD, mỗi miss rất đắt vì latency cao và IOPS thấp, nên phải giữ đủ bộ nhớ so với dữ liệu để miss hiếm; người ta tìm một tỷ lệ bộ nhớ/đĩa hiệu quả (tr. 77). SSD có latency thấp và IOPS cao hơn nhiều, nên mỗi miss rẻ hơn và tỷ lệ này kém quan trọng (tr. 77).

Nhưng **không** bỏ qua được working set. Summary vẫn đặt working set làm tiêu chí để quyết định thêm RAM hay thêm I/O (tr. 96), và miss trên SSD vẫn chậm hơn đọc RAM nhiều lần: **Kết quả đã đo** ở thí nghiệm 2 là chậm khoảng 9 lần khi working set gấp khoảng 4,4 lần buffer pool. SSD làm hình phạt nhẹ đi, không làm nó biến mất.

[về mục 2.3](../chapter-04.md#s2-3)

<a id="q2-4-1"></a>

### Câu 2.4.1 — Ba đặc tính hiệu năng của flash

*Ba đặc tính hiệu năng của flash so với HDD theo sách là gì, và đặc tính nào quan trọng nhất cho OLTP?*

Theo tr. 78, flash chất lượng cao có:

1. đọc và ghi **ngẫu nhiên** tốt hơn HDD **nhiều** (đọc thường hơi tốt hơn ghi);
2. đọc và ghi **tuần tự** tốt hơn, nhưng không ngoạn mục bằng, vì HDD vốn tương đối tốt ở tuần tự;
3. hỗ trợ **concurrency** tốt hơn nhiều; flash chỉ đạt throughput tối đa khi có nhiều thao tác đồng thời.

Quan trọng nhất theo sách: **random I/O và concurrency** (tr. 78). OLTP gồm nhiều truy vấn nhỏ, đồng thời, chạm các trang rải rác, đúng thế mạnh của flash.

[về mục 2.4](../chapter-04.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — Vì sao ghi trên flash phức tạp

*Vì sao ghi trên flash phức tạp hơn đọc? Nối "chỉ xóa được theo khối lớn" với "write amplification".*

Flash đọc được nhanh, nhiều lần, theo đơn vị nhỏ; nhưng một ô **không ghi đè được** nếu chưa xóa, và chỉ **xóa được theo khối lớn** (ví dụ 512 KB); xóa chậm và làm mòn khối (tr. 78).

Vì không ghi đè tại chỗ được, muốn "sửa" một trang nhỏ, thiết bị ghi bản mới vào chỗ trống và đánh dấu bản cũ là rác. Khi cần thu hồi một khối có cả trang rác lẫn trang còn hợp lệ, nó phải **dời** các trang còn hợp lệ đi rồi mới xóa được khối. Các lần ghi phát sinh do dời dữ liệu, ghi dữ liệu và metadata nhiều lần vì ghi không trọn khối, chính là **write amplification** (tr. 79). Phần "dời trang hợp lệ" là cách hiểu chung về flash (**Bổ sung ngoài sách**); sách chỉ nói thiết bị phải di dời trang và chạy garbage collection (tr. 79).

[về mục 2.4](../chapter-04.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — File 100 GB trên SSD 160 GB và 320 GB

*Dự đoán: cùng một file 100 GB, đặt trên SSD 160 GB hay SSD 320 GB cùng dòng thì cái nào ghi ổn định hơn khi chạy lâu? Vì sao?*

**Dự đoán:** SSD 320 GB ổn định hơn. Đây chính là ví dụ của sách (tr. 79): file 100 GB sẽ chạy khác nhau trên hai thiết bị. Trên SSD 160 GB, file chiếm khoảng 62% dung lượng; trên SSD 320 GB chỉ khoảng 31% (**Bổ sung ngoài sách**, phép tính). Càng ít chỗ trống, garbage collector càng phải làm việc nhiều để có khối sạch, hệ số write amplification càng cao, và càng hay phải chờ thao tác xóa (vài mili giây, so với vài trăm micro giây khi ghi vào khối trống). Mức chênh cụ thể tùy hãng và model; có thiết bị được thiết kế để vẫn nhanh khi gần đầy (tr. 79).

[về mục 2.4](../chapter-04.md#s2-4)

<a id="q2-4-4"></a>

### Câu 2.4.4 — SSD đầy 90% thì latency ghi tăng vọt

*Một SSD chạy tốt lúc mới, sau vài tháng đầy 90% thì latency ghi tăng vọt từng đợt. Giải thích bằng cơ chế GC và nêu hai cách giảm.*

Lúc mới, gần như mọi lần ghi rơi vào khối đã xóa sẵn, nhanh. Khi thiết bị đầy, khối trống hiếm; garbage collector phải làm nhiều hơn, write amplification tăng, và có lúc lần ghi phải **chờ xóa** xong vì không còn khối trống (tr. 79). Các lần chờ đó tạo đợt tăng vọt latency.

Cách giảm:

1. **Để chừa không gian trống**, không ghi đầy thiết bị, hoặc dùng thiết bị lớn hơn hay có vùng dự trữ lớn hơn (tr. 79).
2. **Giải phóng dữ liệu không cần** (xóa log cũ, dữ liệu lưu trữ chuyển chỗ khác) và, **Bổ sung ngoài sách**, bảo đảm filesystem gửi TRIM hay discard để thiết bị biết khối nào đã bỏ.

Ngoài ra nên giám sát mức đầy của đĩa như một chỉ số hiệu năng.

[về mục 2.4](../chapter-04.md#s2-4)

<a id="q2-5-1"></a>

### Câu 2.5.1 — Vì sao RAID 0 dễ hỏng hơn một ổ

*Vì sao xác suất hỏng của một mảng RAID 0 lại cao hơn xác suất hỏng của một ổ đơn?*

RAID 0 strip dữ liệu qua mọi ổ mà **không có dư thừa** (tr. 79–80). Mất bất kỳ một ổ nào là mất một phần của gần như mọi file lớn, tức mất cả mảng. Mảng chỉ sống khi **tất cả** ổ cùng sống.

**Bổ sung ngoài sách**, phép tính: nếu mỗi ổ hỏng trong một năm với xác suất p, độc lập nhau, thì mảng N ổ hỏng với xác suất 1 − (1 − p)^N. Ví dụ p = 2%, N = 4: khoảng 7,8%, gần gấp bốn lần một ổ. Vì vậy sách nói xác suất mảng RAID 0 hỏng **cao hơn** chứ không thấp hơn một ổ (tr. 80), và không coi RAID 0 phù hợp cho production.

[về mục 2.5](../chapter-04.md#s2-5)

<a id="q2-5-2"></a>

### Câu 2.5.2 — RAID 5 trước và sau SSD

*RAID 5 từng "đáng sợ" với database vì những lý do nào, vì sao SSD làm nó khả thi trở lại, và rủi ro nào vẫn còn?*

Lý do từng đáng sợ (tr. 80):

- Khi **một ổ hỏng**, dữ liệu phải dựng lại bằng cách đọc mọi ổ còn lại; trên HDD hiệu năng tệ, càng nhiều ổ càng tệ; giữ online khi rebuild thì cả rebuild lẫn hiệu năng đều kém.
- Khó mở rộng quá khoảng 10 ổ vì parity, vấn đề cache, và phụ thuộc nặng vào cache của controller, có thể xung đột với nhu cầu của database.
- Ghi ngẫu nhiên kém (**Bổ sung ngoài sách**: một lần ghi nhỏ cần đọc và ghi cả dữ liệu lẫn parity, khoảng 4 I/O).

SSD giúp vì IOPS và throughput cao hơn nhiều, nên các vấn đề random read/write kém không còn (tr. 80). Controller cũng được tối ưu kỹ cho RAID 5 vì nó phổ biến (tr. 80).

Rủi ro còn lại: **hai ổ hỏng là mất cả volume** (tr. 80). Mảng càng nhiều ổ, xác suất hỏng kép càng cao; đó là lý do có RAID 6 (tr. 81). Hỏng ngầm ở vùng ít đọc có thể chỉ lộ ra khi rebuild (tr. 82).

[về mục 2.5](../chapter-04.md#s2-5)

<a id="q2-5-3"></a>

### Câu 2.5.3 — RAID cache cho đọc và cho ghi

*Vì sao dùng RAID cache để cache đọc thường là lãng phí, còn cache ghi thì quan trọng?*

Cache đọc (tr. 84): hệ điều hành và database có cache **lớn hơn nhiều**. Nếu cache tầng trên hit, cache RAID không được dùng. Nếu tầng trên miss, xác suất cache RAID (nhỏ hơn nhiều và đã bị thay dữ liệu) lại hit là gần như bằng không. Dù nhìn theo hướng nào, cache đọc ở RAID là lãng phí bộ nhớ.

Cache ghi (tr. 84–85): controller báo "thành công" nhanh hơn nhiều so với ghi thật xuống đĩa, và gom ghi để làm hiệu quả hơn. Điều này đặc biệt quý với **ghi đồng bộ** như `fsync()` trên transaction log và binlog (tr. 85). Vì vậy với RAID 0, 1, 10 nên dành 100% bộ nhớ controller cho ghi, với RAID 5 thì chừa một phần cho tính parity (tr. 85). Điều kiện bắt buộc: có BBU hoặc bộ nhớ không mất dữ liệu, nếu không thì mất điện có thể làm hỏng database (tr. 85).

[về mục 2.5](../chapter-04.md#s2-5)

<a id="q2-5-4"></a>

### Câu 2.5.4 — Learning cycle làm commit chậm

*Giải thích chuỗi nhân quả "pin BBU vào learning cycle → commit chậm". Sách đưa những biện pháp nào, theo thứ tự nào?*

Chuỗi nhân quả (tr. 82–83):

1. Trong learning cycle (hoặc khi pin hỏng), controller tắt cache ghi: chuyển từ **write-back** sang **write-through**.
2. Mỗi lần ghi đồng bộ (`fsync` của redo log, của binlog khi bật `sync_binlog`) giờ phải chờ dữ liệu thật sự xuống đĩa.
3. Với `innodb_flush_log_at_trx_commit=1`, mỗi commit cần ít nhất một lần ghi đồng bộ như vậy, nên latency commit tăng mạnh.

Biện pháp theo thứ tự sách (tr. 83): (1) benchmark trước ở write-through để biết mức ảnh hưởng; (2) lên lịch learning cycle vào giờ ít tải; (3) nếu vẫn quá chậm, failover sang server khác trước khi chu kỳ bắt đầu; (4) phương án cuối: hạ durability bằng `innodb_flush_log_at_trx_commit` và `sync_binlog`, chấp nhận có thể mất dữ liệu khi crash. Về lâu dài, controller có flash-backed cache tránh được learning cycle (tr. 82).

[về mục 2.5](../chapter-04.md#s2-5)

<a id="q2-5-5"></a>

### Câu 2.5.5 — Chunk 16 KB vẫn không bảo đảm một ổ

*Vì sao chunk size 16 KB, trùng page InnoDB, vẫn không bảo đảm mỗi lần đọc một page chỉ chạm một ổ?*

Vì **căn lề** không được bảo đảm (tr. 83). Một page 16 KB chỉ nằm gọn trong một chunk 16 KB nếu nó bắt đầu đúng ranh giới chunk. Filesystem có thể phân mảnh file và căn các mảnh theo kích thước block của filesystem, thường 4 KB. Khi đó một page có thể bắt đầu giữa một chunk và vắt sang chunk kế tiếp, nằm trên ổ khác, nên một lần đọc phải chạm hai ổ. Có filesystem khôn hơn, nhưng sách khuyên đừng trông vào đó. Thêm nữa, trên thực tế nhiều controller lại không chạy tốt với chunk lớn (tr. 83), nên chọn chunk size là chuyện đo trên phần cứng thật.

[về mục 2.5](../chapter-04.md#s2-5)

<a id="q2-6-1"></a>

### Câu 2.6.1 — Latency hay băng thông

*Vì sao latency mạng, chứ không phải băng thông, là vấn đề lớn nhất với đa số ứng dụng?*

Ứng dụng điển hình làm **rất nhiều lần truyền nhỏ** qua mạng (tr. 86): mỗi truy vấn và kết quả thường chỉ vài trăm byte tới vài KB, nên chẳng mấy khi dùng hết băng thông. Nhưng mỗi lần đi về chịu một độ trễ, và các độ trễ này **cộng dồn** (tr. 86). **Ví dụ minh họa do AI xây dựng:** 100 truy vấn tuần tự với round-trip 0,5 ms tốn 50 ms chỉ cho mạng; tăng băng thông không giảm được con số này, chỉ giảm số lần đi về hoặc giảm round-trip mới giúp.

[về mục 2.6](../chapter-04.md#s2-6)

<a id="q2-6-2"></a>

### Câu 2.6.2 — 1% mất gói

*Vì sao chỉ 1% mất gói có thể làm hiệu năng giảm đáng kể?*

Theo tr. 86, khi mất gói, nhiều tầng của chồng giao thức cố sửa bằng cách **chờ một lúc rồi gửi lại**, cộng thêm thời gian. Thời gian chờ đó lớn hơn rất nhiều so với một round-trip bình thường. **Bổ sung ngoài sách:** với TCP, thời gian chờ trước khi gửi lại thường tính bằng hàng trăm mili giây, và cơ chế kiểm soát tắc nghẽn còn giảm tốc độ gửi sau khi mất gói. Một truy vấn chỉ cần dính một gói bị mất là latency của nó nhảy vọt; với hàng nghìn truy vấn mỗi giây, 1% đủ làm latency đuôi (p99) xấu đi rõ rệt.

[về mục 2.6](../chapter-04.md#s2-6)

<a id="q2-6-3"></a>

### Câu 2.6.3 — skip_name_resolve

*`skip_name_resolve` giải quyết vấn đề gì, và có hệ quả phụ nào với tài khoản người dùng?*

Vấn đề (tr. 86): khi nhận kết nối, MySQL tra DNS **xuôi và ngược**. DNS hỏng hay chậm làm kết nối bị từ chối, kết nối chậm, có thể tới mức từ chối dịch vụ. Bật `skip_name_resolve` thì MySQL không tra DNS nữa, nên loại bỏ cả nhóm sự cố này; sách coi đây là ý hay cho production.

Hệ quả phụ (tr. 86): tài khoản chỉ được dùng địa chỉ IP, `localhost`, hoặc wildcard IP trong cột `host`; tài khoản khai bằng **hostname** sẽ không đăng nhập được. Vì vậy phải rà `mysql.user` trước khi bật. **Bổ sung ngoài sách:** trong 8.0 biến này không đổi được lúc chạy, phải khởi động lại; chương 5 lưu ý nó làm `max_connect_errors` mất tác dụng (tr. 120–121).

[về mục 2.6](../chapter-04.md#s2-6)

<a id="q2-6-4"></a>

### Câu 2.6.4 — Khi nào chỉnh tham số TCP

*Khi nào nên đổi `ip_local_port_range`, `tcp_max_syn_backlog`, `tcp_fin_timeout`? Vì sao sách khuyên thường để mặc định?*

Theo tr. 86–87:

- `ip_local_port_range`: khi có **rất nhiều kết nối đồng thời** đến mức có thể cạn cổng cục bộ; mở rộng khoảng cổng.
- `tcp_max_syn_backlog`: khi hàng đợi kết nối đến hay bị đầy và client không kết nối được.
- `tcp_fin_timeout`: với server chỉ dùng cục bộ, rút ngắn thời gian chờ sau khi đóng socket khi đầu kia hỏng (mặc định khoảng một phút).

Sách khuyên phần lớn thời gian để mặc định và chỉ đổi khi có gì **bất thường**, như mạng cực kém hay số kết nối cực lớn (tr. 87). Lý do (**Bổ sung ngoài sách**): mặc định đã hợp lý cho đa số trường hợp; đổi mà không có triệu chứng thì không thu được gì nhưng thêm cấu hình phải duy trì. Ngoài ra, cổng tạm thời bị tiêu thụ chủ yếu ở phía mở kết nối (máy ứng dụng, proxy), nên cần xác định đúng máy nào đang cạn cổng.

[về mục 2.6](../chapter-04.md#s2-6)

<a id="q2-7-1"></a>

### Câu 2.7.1 — data=writeback với InnoDB

*Vì sao `data=writeback` "thường an toàn" với InnoDB, và ngoại lệ trước MySQL 8.0 là gì?*

`data=writeback` chỉ journal metadata, không đồng bộ ghi metadata với ghi dữ liệu (tr. 87). Với InnoDB, điều này thường an toàn vì InnoDB có **transaction log riêng**: khi crash, InnoDB tự khôi phục dữ liệu của nó từ redo log, không trông vào filesystem để giữ nhất quán nội dung file dữ liệu (tr. 87).

Ngoại lệ (tr. 88): ở MySQL **trước 8.0**, crash đúng lúc có thể làm hỏng file `.frm`. Ví dụ: khi nới file, metadata (kích thước mới) được log và ghi trước dữ liệu, nên phần đuôi mới của file có thể chứa rác. File `.frm` không được bảo vệ bởi transaction log của InnoDB. **Bổ sung ngoài sách:** MySQL 8.0 bỏ `.frm`, chuyển metadata vào data dictionary trong InnoDB (chương 1, tr. 17), nên ngoại lệ này không còn với 8.0.

[về mục 2.7](../chapter-04.md#s2-7)

<a id="q2-7-2"></a>

### Câu 2.7.2 — cfq và kernel hiện đại

*Vì sao `cfq` tệ cho server MySQL? Trên kernel Linux hiện đại, lời khuyên này chuyển thành gì?*

`cfq` được thiết kế cho công bằng giữa các tiến trình trên laptop, desktop, giúp tránh "chết đói" I/O. Với workload của MySQL, nó **giữ một số yêu cầu trong hàng đợi không cần thiết**, nên thời gian phản hồi rất kém (tr. 89). Sách khuyên dùng `noop` (cho thiết bị tự lập lịch như controller RAID phần cứng, SAN) hoặc `deadline` (cho cả RAID và ổ gắn trực tiếp); quan trọng nhất là không dùng `cfq` (tr. 90).

**Bổ sung ngoài sách:** từ Linux 5.0, `cfq`, `deadline`, `noop` đã bị gỡ; tầng đa hàng đợi blk-mq có `none`, `mq-deadline`, `bfq`, `kyber`. Tương đương lời khuyên của sách: `none` cho NVMe và thiết bị tự lập lịch, `mq-deadline` cho ổ SATA hay SAS; `bfq` gần tinh thần của `cfq` nên cần đo kỹ trước khi dùng cho database. Kiểm tra bằng cách đọc `/sys/block/<thiết bị>/queue/scheduler`, như sách hướng dẫn (tr. 89).

[về mục 2.7](../chapter-04.md#s2-7)

<a id="q2-7-3"></a>

### Câu 2.7.3 — Hệ điều hành swap MySQL dù buffer đúng

*Hệ điều hành có thể swap MySQL ra đĩa dù buffer đã cấu hình đúng. Giải thích cơ chế, ba công cụ điều khiển sách nêu, và rủi ro của từng cái.*

Cơ chế (tr. 91): hệ điều hành thấy MySQL làm **nhiều I/O** qua page cache, nên muốn **tăng file cache** để giữ thêm dữ liệu. Nếu không đủ bộ nhớ, nó swap thứ gì đó ra, và có thể là chính MySQL; đôi khi liên quan cách NUMA hoạt động trên Linux. Kernel cũ còn có ưu tiên phản tác dụng.

Ba công cụ (tr. 91–92):

| Công cụ | Tác dụng | Rủi ro |
| --- | --- | --- |
| `swappiness` thấp (0 hoặc 1; server nên 0 theo sách) | Kernel chỉ swap khi nhu cầu bộ nhớ ảo cực độ | **Bổ sung ngoài sách:** từ kernel 3.5, giá trị 0 có thể khiến OOM killer chạy sớm hơn; nhiều người chọn 1 |
| `innodb_flush_method=O_DIRECT` | I/O của InnoDB không qua cache hệ điều hành, nên không còn là lý do để phình file cache; chỉ áp dụng cho InnoDB | Cần filesystem hỗ trợ direct I/O (tr. 88) |
| `memlock` | Khóa MySQL trong bộ nhớ, không bị swap | Nguy hiểm: hết bộ nhớ khóa được thì MySQL crash khi cấp phát; khóa quá nhiều thì hệ điều hành thiếu bộ nhớ |

Nếu vẫn không được, phương án cuối là hạ kích thước buffer xuống mức chưa tối ưu (tr. 92).

[về mục 2.7](../chapter-04.md#s2-7)

<a id="q2-7-4"></a>

### Câu 2.7.4 — si, so hay swpd

*Trong `vmstat`, vì sao nên nhìn `si`, `so` chứ không phải `swpd`?*

`swpd` là **lượng** bộ nhớ đang nằm trong swap; nó có thể chứa trang của tiến trình đã nạp nhưng không dùng tới, nên không thật sự gây hại (tr. 90). Thứ làm hệ thống chậm là **hoạt động** swap: trang bị đẩy ra và kéo vào trong lúc đang cần. `si` và `so` đo đúng hoạt động đó, tính bằng block mỗi giây (tr. 90, 93). Tác giả muốn thấy chúng bằng 0 phần lớn thời gian, chắc chắn dưới 10 block mỗi giây, và các đợt bùng cũng xấu (tr. 93). Một máy có `swpd` vài GB mà `si`, `so` bằng 0 có thể hoàn toàn khỏe.

[về mục 2.7](../chapter-04.md#s2-7)

<a id="q2-7-5"></a>

### Câu 2.7.5 — %util 98% và Little's law

*Với mẫu `iostat` của sách, `%util` 98% có nghĩa thiết bị bão hòa không? Tính concurrency theo Little's law bằng `svctm` và bằng `await`, rồi giải thích vì sao hai kết quả khác nhau.*

Không. `%util` là phần trăm thời gian có **ít nhất một** yêu cầu đang hoạt động, không phải mức sử dụng theo lý thuyết hàng đợi; thiết bị nhiều ổ hay SSD phục vụ được nhiều yêu cầu song song mà `%util` không bao giờ vượt 100%, nên nó chỉ đáng tin với một ổ cứng vật lý đơn lẻ (tr. 95).

Tính toán (**Bổ sung ngoài sách**, số liệu tr. 94):

- Tổng IOPS = 1060,40 + 3915,00 = 4975,4.
- Theo sách, với `svctm` = 0,20 ms: 4975,4 × 0,20 / 1000 ≈ **0,995** (tr. 95).
- Với `await` = 0,74 ms: 4975,4 × 0,74 / 1000 ≈ **3,68**, khớp đúng `avgqu-sz`.

Khác nhau vì hai đại lượng đo hai thứ khác nhau. `svctm` không đo trực tiếp; sysstat **tính** nó bằng thời gian bận chia cho số I/O, nên IOPS × `svctm` chỉ cho lại tỷ lệ thời gian bận, tức xấp xỉ `%util` (0,995 so với 0,982). `await` gồm cả thời gian chờ lẫn thời gian phục vụ, nên IOPS × `await` cho số yêu cầu trung bình **trong hệ thống**, khoảng 3,7. Kết hợp với latency rất thấp (0,57–0,78 ms), không có dấu hiệu bão hòa. Các bản sysstat mới đã bỏ `svctm` vì nó gây hiểu nhầm (**Bổ sung ngoài sách**).

[về mục 2.7](../chapter-04.md#s2-7)

<a id="q2-8-1"></a>

### Câu 2.8.1 — Vì sao thích server commodity

*Vì sao tác giả thích server commodity? Nêu lý do vận hành cụ thể mà sách đưa ra.*

Tr. 96: tác giả gợi ý cân bằng hiệu năng và chi phí, và thích server commodity vì nhiều lý do. Lý do vận hành cụ thể: khi một server có vấn đề và phải **rút khỏi dịch vụ để chẩn đoán**, hoặc muốn **đổi nó với server khác** như một cách chẩn đoán, làm với máy 5.000 USD dễ hơn nhiều so với máy từ 50.000 USD trở lên. Ngoài ra MySQL, cả phần mềm lẫn workload điển hình, hợp với phần cứng commodity hơn.

[về mục 2.8](../chapter-04.md#s2-8)

<a id="q2-8-2"></a>

### Câu 2.8.2 — I/O-bound: thêm RAM hay thêm IOPS

*Áp dụng lời khuyên "trước khi ném tài nguyên vào vấn đề" cho một server I/O-bound: bạn cần đo gì để quyết định thêm RAM hay thêm IOPS?*

Summary nói câu trả lời xoay quanh **working set** (tr. 96). Cần đo (**Bổ sung ngoài sách**, quy trình chi tiết ở tình huống 3 của ghi chú):

1. **Đọc hay ghi**: `r/s` so với `w/s` (`iostat`), `bi` so với `bo` (`vmstat`). RAM thêm chỉ khử được đọc, không khử được ghi (tr. 76).
2. **Ai gây I/O**: Performance Schema theo file và theo truy vấn; có thể nguyên nhân là truy vấn quét quá nhiều, sửa truy vấn hay index rẻ hơn phần cứng.
3. **Đường cong miss theo kích thước buffer pool**: trên replica, tăng `innodb_buffer_pool_size` theo bậc và đo `Innodb_buffer_pool_reads` mỗi giây. Miss giảm mạnh thì thêm RAM; miss gần như không đổi thì working set lớn hơn bộ nhớ nhiều lần, cần IOPS hay giảm dữ liệu phải chạm.
4. **Không** dùng hit ratio làm tiêu chí (chương 5, tr. 105).

[về mục 2.8](../chapter-04.md#s2-8)

<a id="q2-8-3"></a>

### Câu 2.8.3 — Các "Big Things" của hệ điều hành

*Ba "Big Things" của hệ điều hành theo Summary là gì, và với Linux sách gợi ý thiết lập cụ thể nào?*

Tr. 97: chỉ vài "Big Things" cần làm đúng, chủ yếu liên quan **lưu trữ, mạng và quản lý bộ nhớ ảo**. Với GNU/Linux, sách gợi ý dùng **XFS**, đặt **swappiness** và **disk queue scheduler** về giá trị phù hợp server; có thể phải đổi vài tham số mạng; các chỉnh sửa khác như tắt SELinux là chuyện sở thích. Chi tiết: XFS (tr. 89), scheduler không phải `cfq` (tr. 89–90), swappiness thấp hoặc không dùng swap (tr. 91), tham số TCP (tr. 86–87).

[về mục 2.8](../chapter-04.md#s2-8)
