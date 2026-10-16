# Đáp án — Chương 2: Giám sát MySQL theo tư duy reliability engineering

Quay lại ghi chú: [../chapter-02.md](../chapter-02.md). "tr. N" là số trang in của sách; nhãn **Bổ sung ngoài sách**, **Ví dụ minh họa do AI xây dựng**, **Kết quả đã đo** dùng như trong ghi chú chương. Hãy tự trả lời trước khi đọc.

---

<a id="q2-0-1"></a>

### Câu 2.0.1 — Outcome và output; vì sao SRE thuộc high performance MySQL

*Phân biệt outcome và output trong giám sát MySQL, mỗi loại cho hai ví dụ. Vì sao tác giả cho rằng SRE là một phần của high performance MySQL?*

- **Output** là thứ hệ thống hay đội làm ra, đo từ bên trong: số query mỗi giây, CPU, tỉ lệ trúng buffer pool, số query chậm đã tối ưu trong quý.
- **Outcome** là kết quả khách hàng nhận được: tỉ lệ lần tải trang dưới vài trăm mili giây, tỉ lệ lần checkout thành công (ví dụ của sách ở tr. 23).

Tác giả dẫn Nicole Forsgren: thước đo nên tập trung vào outcome, không phải output (tr. 20). Quản lý MySQL hiệu quả cần giám sát tốt sức khỏe database (tr. 20), và "hiệu năng" chỉ có ý nghĩa khi nó thay đổi trải nghiệm khách hàng. SLO cho biết lúc nào công tối ưu hiệu năng là cần thiết và lúc nào không (tr. 22), nên giám sát theo service level là điểm xuất phát của việc tối ưu, không phải một chủ đề tách rời.

[về mục 2.0](../chapter-02.md#s2-0)

<a id="q2-0-2"></a>

### Câu 2.0.2 — Ba câu hỏi của SRE và câu hỏi về toil

*Ba câu hỏi mà nguyên lý SRE giúp trả lời là gì? Vì sao câu "cân bằng tính năng mới với toil" cũng là câu hỏi của đội database?*

Ba câu hỏi (tr. 19): trải nghiệm khách hàng có chấp nhận được không; có nên tập trung vào độ tin cậy và resilience không; cân bằng tính năng mới với toil thế nào.

Đội database cũng có thời gian hữu hạn. Sách nói khi DBA thành SRE hay DBRE, đội phải tối ưu cách dùng thời gian, và service level giúp cân giữa xử lý hiệu năng, scaling với xây công cụ nội bộ (tr. 20). Khi khách hàng hài lòng, giám sát SLO đẩy đội sang việc khác như giảm toil (tr. 29). **Bổ sung ngoài sách:** toil của đội database là những việc như tay dựng lại replica, tay dọn binary log khi đĩa đầy; tự động hóa chúng chính là "tính năng" của đội nền tảng dữ liệu, và cũng cạnh tranh thời gian với việc sửa sự cố.

[về mục 2.0](../chapter-02.md#s2-0)

<a id="q2-0-3"></a>

### Câu 2.0.3 — Vì sao khó nối chỉ số MySQL với trải nghiệm khách hàng

*Sách nói giám sát truyền thống là "đường đã trải nhựa" còn áp dụng SRE cho MySQL thì ít được hiểu. Điều gì làm việc nối chỉ số MySQL với trải nghiệm khách hàng trở nên khó?*

- Chỉ số MySQL chủ yếu là output nội bộ; Performance Schema nằm quá sâu trong nội bộ để tự nó trả lời câu hỏi về lời hứa độ tin cậy (tr. 25).
- Nhiều thứ khách hàng cảm nhận **không** đi qua đồng hồ của server: latency do chờ kết nối, mạng, retry; sách yêu cầu đo latency phía client (tr. 27) và kiểm availability từ client hoặc từ xa (tr. 26).
- Lỗi phía client không phải lúc nào cũng nghĩa là hỏng; phải chọn lỗi nào đáng làm SLI, và đó là bài toán giao tiếp, xã hội không kém kỹ thuật (tr. 27–28).
- Database phục vụ nhiều tính năng với mức quan trọng khác nhau, nên phải biết dữ liệu nào phục vụ tính năng nào (tr. 22–23).

[về mục 2.0](../chapter-02.md#s2-0)

<a id="q2-1-1"></a>

### Câu 2.1.1 — Giám sát trong mô hình DBA gác cổng

*Trong mô hình "DBA gác cổng", giám sát hoạt động thế nào và vì sao nó mang tính phản ứng?*

Giám sát dựa vào việc đào sâu hiệu năng của **một server**, ví dụ profile một server đang chạy kém (tr. 20). Nó phản ứng vì được kích hoạt **sau** khi đã có vấn đề, và vì không có định nghĩa chung nào về "khách hàng không hài lòng" để biết trước. Trong mô hình đó chỉ đội DBA được biết database vận hành ra sao (tr. 20), nên đội tính năng không thể cùng theo dõi và cảnh báo sớm.

[về mục 2.1](../chapter-02.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Service level giúp chia thời gian của DBRE

*Service level giúp đội DBRE chia thời gian giữa những loại việc nào? Cho ví dụ quyết định khác nhau khi SLO đang đạt và khi đang trượt.*

Sách nói service level giúp xác định khi nào khách hàng không hài lòng, để cân thời gian giữa **vấn đề hiệu năng, thách thức scaling** và **công cụ nội bộ** (tr. 20).

**Ví dụ minh họa do AI xây dựng.** SLO latency của checkout đang đạt với dư địa lớn: đội dành sprint để tự động hóa việc dựng replica. SLO đang trượt vì một query mới thiếu index: đội dừng việc công cụ, sửa query, và theo sách thì các đội liên quan cũng không nên đẩy tính năng mới tới khi SLO đạt lại (tr. 22).

[về mục 2.1](../chapter-02.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Profiling sâu một server còn chỗ đứng không

*Profiling sâu một server (ví dụ bằng Performance Schema) còn chỗ đứng trong mô hình SRE không? Nó bổ sung cho SLI ra sao?*

Còn. Sách nói đào sâu một server "vẫn có nhiều giá trị" (tr. 20), và Performance Schema giúp tìm nút thắt, làm được nhiều hơn với cùng cấu hình, tiết kiệm hạ tầng, trả lời "vì sao lâu vậy?" (tr. 25). Nhưng nó **không** dùng một mình để biết có đạt lời hứa độ tin cậy không (tr. 25). Phân vai: SLI cho biết **có** đang làm khách hàng đau không và ở tính năng nào; profiling cho biết **vì sao**. Thứ tự hợp lý là SLI hoặc chỉ báo sớm báo động trước, profiling chẩn đoán sau (xem tình huống 2 trong ghi chú).

[về mục 2.1](../chapter-02.md#s2-1)

<a id="q2-2-1"></a>

### Câu 2.2.1 — SLI, SLO, SLA cho checkout

*Phân biệt SLI, SLO, SLA bằng ví dụ cho tính năng checkout. Vì sao SLA là tùy chọn còn SLO thì không nên thiếu?*

**Ví dụ minh họa do AI xây dựng**, theo định nghĩa ở tr. 21:

- **SLI:** tỉ lệ request checkout trả về thành công (2xx, kể cả 202) trên tổng request hợp lệ, đo ở load balancer.
- **SLO:** SLI ≥ 99% trong mỗi tháng dương lịch (sách dùng đúng con số "lỗi chập chờn chỉ 1% thời gian trong một tháng dương lịch", tr. 23).
- **SLA:** hợp đồng với khách hàng trả tiền (ví dụ một nhà bán hàng trên nền tảng): nếu tỉ lệ thành công dưới 99,5% trong tháng thì hoàn 10% phí.

SLA là tùy chọn vì đó là quyết định kinh doanh: hứa SLA bán thêm được bao nhiêu so với rủi ro doanh thu khi vi phạm (tr. 21). SLO thì là công cụ nội bộ để quyết định làm tính năng hay làm độ tin cậy (tr. 22–23); không có SLO, mọi cuộc tranh luận về ưu tiên thiếu chỉ số khách quan.

[về mục 2.2](../chapter-02.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Vì sao SLO cần khoảng thời gian

*Vì sao SLO phải gắn với một khoảng thời gian? Cho ví dụ hai cách hiểu khác nhau của "99,9% uptime" khi không ghi khoảng thời gian.*

Sách nói SLO phải là giá trị trên một khoảng thời gian để mọi người hiểu SLO giống nhau (tr. 21), và có khoảng thời gian thì đội mới cân được giữa tính năng và resilience (tr. 23).

**Bổ sung ngoài sách (phép tính).** 99,9% theo **năm** cho phép khoảng 8 giờ 45 phút downtime; một sự cố 3 giờ vẫn đạt. 99,9% theo **tháng** chỉ cho phép khoảng 43 phút 48 giây; cùng sự cố 3 giờ là trượt nặng. Theo **ngày** chỉ còn 1 phút 26 giây. Hai người cùng nói "99,9%" có thể đang nghĩ tới hai mức chịu đựng khác nhau hàng chục lần, và sẽ ra quyết định khác nhau về việc có phải dừng tính năng hay không.

[về mục 2.2](../chapter-02.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Tự tính 99,95% mỗi tháng và so với Table 2-1

*Tự tính downtime cho phép mỗi tháng của 99,95% và so với Table 2-1. Bạn thấy gì, và điều đó dạy gì về cách dùng bảng tra?*

**Bổ sung ngoài sách (phép tính).** Tháng = 365/12 ≈ 30,42 ngày ≈ 43.800 phút. Downtime = 0,0005 × 43.800 = 21,9 phút = **21 phút 54 giây**.

Table 2-1 (tr. 22) in **31 phút 54 giây**: lệch đúng 10 phút, nhiều khả năng là lỗi in. Bảng còn vài ô khác cũng lệch (ví dụ 99,999% mỗi ngày phải là 0,86 giây, sách in 0,14 giây; xem bảng tự tính ở [mục 2.2.1](../chapter-02.md#nines)).

Bài học: bảng tra là để cảm nhận độ lớn; khi đặt SLO thật, hãy tự tính bằng công thức (1 − availability) × độ dài kỳ, và ghi rõ giả định về độ dài tháng, tuần. Thông điệp của sách vẫn đúng: mỗi số 9 thêm vào cắt ngân sách downtime khoảng 10 lần.

[về mục 2.2](../chapter-02.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Vì sao SLO 100% hay quá chặt lại có hại

*Vì sao đặt SLO 100%, hoặc mục tiêu chặt hơn nhu cầu (750 ms thay vì 2 giây), lại có hại dù nghe có vẻ tốt cho khách hàng?*

- Mục đích của SLO là cho biết, bằng chỉ số khách quan, đội có được đổi mới hay độ ổn định đang nguy (tr. 22). Thất bại là không tránh được (tr. 23), nên SLO 100% sẽ **luôn** bị vi phạm; đội sẽ mãi ở trạng thái "dừng tính năng", và SLO mất vai trò công cụ ra quyết định.
- Thời gian kỹ sư hữu hạn (tr. 22); mỗi số 9 thêm vào đắt hơn rất nhiều (Table 2-1, tr. 22).
- Mục tiêu 750 ms khi khách hàng hài lòng với 2 giây là gánh nặng vô lý (tr. 22): tốn công mà không đổi outcome. Mục tiêu đúng là **mức tối thiểu tuyệt đối** để khách hàng hài lòng (tr. 22).
- Phải nói rõ với đội chăm sóc khách hàng rằng "100% uptime" không hợp lý (tr. 26).

[về mục 2.2](../chapter-02.md#s2-2)

<a id="q2-2-5"></a>

### Câu 2.2.5 — Một data set, hai nhóm stakeholder

*Một data set phục vụ hai nhóm: checkout (query ngắn, cần nhanh) và báo cáo nội bộ (query quét lớn). Dự đoán chuyện gì xảy ra nếu chỉ có một SLO chung, và sách gợi ý hướng xử lý nào?*

**Dự đoán.** Query báo cáo quét nhiều dữ liệu, chiếm disk I/O và đẩy dữ liệu nóng của checkout ra khỏi bộ nhớ; checkout chậm lúc báo cáo chạy. Đây đúng là ca "khó thấy" mà sách nêu: một tác vụ định kỳ chiếm hết disk I/O làm mọi thứ khác chậm (tr. 21). Với một SLO chung: hoặc nó lỏng tới mức checkout chậm mà vẫn "đạt", hoặc nó chặt tới mức báo cáo luôn làm nó trượt.

**Hướng của sách.** Nhiệm vụ then chốt là phát hiện khi một data set thành nút thắt cho các query profile rất khác nhau của các nhóm stakeholder, rồi **tách nhu cầu** để cho mỗi nhóm SLI và SLO hợp lý (tr. 23). Cách tách cụ thể sách nêu sau là functional sharding: đưa bảng của một chức năng sang cluster riêng để quản lý riêng uptime, hiệu năng (tr. 34), và dùng SLO để biết khi nào cần chia (tr. 39). **Bổ sung ngoài sách:** một bước nhẹ hơn là chuyển báo cáo sang replica, chấp nhận dữ liệu trễ.

[về mục 2.2](../chapter-02.md#s2-2)

<a id="q2-3-1"></a>

### Câu 2.3.1 — Tách câu SLI "99,5% dưới 2 ms và không lỗi"

*Câu "99,5% request database phục vụ dưới 2 ms và không lỗi" chứa những SLI nào? Mỗi SLI nên đo ở đâu, và câu này còn thiếu gì theo chính định nghĩa SLO của sách?*

- **SLI latency:** tỉ lệ request database xong dưới 2 ms. Nên đo **phía client**, vì latency phía server bỏ sót chờ kết nối, mạng, retry (tr. 27).
- **SLI lỗi:** tỉ lệ request không trả lỗi, cũng đo phía client; bộ đếm phía server dùng để chẩn đoán (tr. 28).
- **Availability** nằm ngầm: request không kết nối được cũng là lỗi (availability là trả lời request mà không lỗi, tr. 26).

Sách nhấn mạnh không thể xác nhận cả câu bằng **một** chỉ số (tr. 23–24). Theo định nghĩa SLO của chính sách, câu này **thiếu khoảng thời gian** (tr. 21), ví dụ "trong mỗi cửa sổ 30 ngày". **Bổ sung ngoài sách:** câu cũng chưa nói "request" gồm loại nào (đọc, ghi, hay mọi query), và có tính request bị retry thành công là tốt hay không.

[về mục 2.3](../chapter-02.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Ba probe availability và chỉ báo sớm

*So sánh ba probe availability: `SELECT 1`, đọc bảng thật, ghi rồi đọc. Probe nào sẽ báo "xanh" khi một replica vừa được promote nhưng còn read-only? Vì sao kiểm từ xa vẫn chưa đủ, và chỉ báo sớm nào bù cho nó?*

- `SELECT 1` xác nhận MySQL nhận và parse được query nhưng không chạm tầng lưu trữ (tr. 26).
- Đọc bảng thật xác nhận tầng lưu trữ đọc được (tr. 26).
- Ghi rồi đọc xác nhận ghi thành công (tr. 26).

Khi source mới còn read-only, **cả `SELECT 1` lẫn probe đọc** đều xanh; chỉ probe ghi bắt được. **Kết quả đã đo** (TN3, MySQL 8.0.42): với `super_read_only` bật, `SELECT 1` trả 1, còn lệnh ghi nhận lỗi 1290.

Kiểm từ xa giúp theo dõi mục tiêu availability nhưng **không** cho thấy trước khi sự cố xảy ra (tr. 27). Chỉ báo sớm sách đề xuất là `Threads_running` (so với số core) và mức gần chạm `max_connections` (tr. 27).

[về mục 2.3](../chapter-02.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Chẩn đoán hai trạng thái kết nối

*Máy 32 core, `max_connections` = 1000. Trường hợp A: `Threads_connected` 900, `Threads_running` 6. Trường hợp B: `Threads_connected` 400, `Threads_running` 350 và đang tăng. Chẩn đoán từng trường hợp và nêu hành động.*

**A — kết nối không dùng** (trạng thái thứ nhất, tr. 31). 90% `max_connections` đã bị chiếm mà chỉ 6 kết nối đang làm việc: tầng ứng dụng mở nhiều kết nối không cần, có nguy cơ chạm trần "vô cớ". Server chưa quá tải, nhưng một lần scale-out ứng dụng hay một connection storm (tr. 31) sẽ đẩy nó qua giới hạn và sinh lỗi "too many connections" (tr. 28). Hành động: cảnh báo theo tỉ lệ `Threads_connected` / `max_connections` (tr. 31); thu nhỏ pool, đặt timeout cho kết nối rảnh (**Bổ sung ngoài sách**); cân nhắc ProxySQL để tách số kết nối frontend khỏi database (tr. 30).

**B — quá tải thật** (trạng thái thứ hai, tr. 31). 350 thread đang chạy, gấp hơn 10 lần số core (ngưỡng tr. 27) và vượt xa khoảng 100 (tr. 31): query không xong kịp, chồng lên nhau, có thể dẫn tới khóa cứng CPU hoặc hệ điều hành giết MySQL (tr. 27, 31). Hành động: giảm tải chiến thuật bằng `KILL` hoặc `pt-kill`, rồi phân tích query để tìm nguyên nhân (tr. 31). **Bổ sung ngoài sách:** kiểm tra xem các thread đang dùng CPU hay đang chờ lock, vì `Threads_running` đếm cả thread đang chờ; nếu là chờ lock thì giết query đang **giữ** lock mới có tác dụng.

[về mục 2.3](../chapter-02.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — Lỗi theo tốc độ, lỗi một lần, và cơ chế lock wait timeout

*Lỗi nào chỉ đáng báo động khi tốc độ tăng, lỗi nào một lần là đủ? Giải thích cơ chế khiến lock wait timeout tăng vọt báo hiệu sắp mất khả năng ghi.*

**Theo tốc độ** (tr. 28): lock wait timeout, aborted connections, các bộ đếm `Connection_errors_xxx`. Từng lỗi lẻ thường hết khi retry, nhưng tốc độ tăng là dấu hiệu rắc rối.

**Một lần là đủ** (tr. 28): lỗi read-only (source mới còn read-only, hoặc access layer gửi ghi sang replica), "too many connections", "cannot create new thread". Chúng không phải lỗi chập chờn; retry không giải quyết, và chuyển thẳng thành lỗi 5xx.

**Cơ chế** (tr. 28, phần cơ chế InnoDB là **Bổ sung ngoài sách**): một transaction giữ row lock lâu (hoặc nhiều transaction cùng nhắm vào vài hàng nóng); các transaction khác chờ tới `innodb_lock_wait_timeout` rồi thất bại. Ứng dụng retry, thêm người vào hàng chờ, hàng chờ dài hơn, nhiều lần chờ hết hạn hơn. Khi tranh chấp leo thang như sách mô tả, phần lớn lệnh ghi chạm các hàng đó đều thất bại dù đã retry: với người dùng, đó là mất khả năng ghi. **Kết quả đã đo** (TN3): 5 lần chờ cho đúng 5 lỗi 1205, `Innodb_row_lock_waits` tăng 5, tổng thời gian chờ khoảng 10 giây; các bộ đếm này giúp chẩn đoán phía server.

[về mục 2.3](../chapter-02.md#s2-3)

<a id="q2-3-5"></a>

### Câu 2.3.5 — Ngưỡng đĩa và giám sát auto-increment trên 8.0

*(a) Đĩa ghi tối đa 400 MB/s, đội cần 30 phút để phản ứng: theo cách tính của sách, ngưỡng dung lượng trống tối thiểu là bao nhiêu, và vì sao theo dõi tốc độ tăng tốt hơn? (b) Vì sao trên MySQL 8.0, query giám sát auto-increment có thể báo sai, và vì sao không gian khóa có thể cạn nhanh hơn số hàng?*

**(a)** Sách gợi ý dùng tốc độ ghi tối đa của đĩa để tính bao lâu thì đầy ở lưu lượng tối đa, và cần ngần ấy thời gian báo trước (tr. 30). **Bổ sung ngoài sách (phép tính):** 400 MB/s × 1.800 giây = 720.000 MB, khoảng **720 GB** trống. Con số này rất thận trọng vì hiếm khi đĩa bị ghi kín liên tục; trên đĩa nhỏ nó có thể bắt cảnh báo khi đĩa còn khá trống. Theo dõi **tốc độ tăng** là cách lý tưởng của sách (tr. 29): nó báo theo thời gian còn lại thực tế tới khi đầy, và bắt được những cú tăng bất thường như transaction dài với undo log lớn, `ALTER TABLE`, log quá mức hay đổi kiểu insert (tr. 29).

**(b)** **Kết quả đã đo** (TN5, MySQL 8.0.42):

- `information_schema.TABLES.AUTO_INCREMENT` được cache theo `information_schema_stats_expiry` (mặc định 86.400 giây). Sau khi chèn thêm 20 hàng, query vẫn trả giá trị cũ; phải đặt biến này về 0 trong phiên mới thấy giá trị mới.
- Với `innodb_autoinc_lock_mode` = 2 (mặc định 8.0), `INSERT ... SELECT` cấp giá trị theo lô và bỏ phí phần thừa: 100 hàng đẩy bộ đếm `TINYINT UNSIGNED` lên 128. Vì vậy phải theo dõi bộ đếm, không theo dõi số hàng, đúng như sách nói "không gian số nguyên còn lại" (tr. 33).
- Ngoài ra query kiểu của sách trả trùng dòng với primary key nhiều cột, và cột `max_value` thực chất là phần trăm đã dùng.

[về mục 2.3](../chapter-02.md#s2-3)

<a id="q2-4-1"></a>

### Câu 2.4.1 — Đánh giá năng lực theo nhịp kinh doanh

*Vì sao phải đánh giá năng lực database theo nhịp kinh doanh (đỉnh mùa cao điểm) thay vì theo trung bình? Nêu ba khía cạnh của tầng database mà sách nói sẽ tăng ở đỉnh.*

Mùa cao điểm là lúc mọi SLO bị thử thách nhiều nhất và bị khách hàng quan trọng nhất soi kỹ nhất, và đỉnh có thể lớn hơn trung bình **nhiều bậc độ lớn** (tr. 36). Ba khía cạnh (tr. 36): số request mỗi giây phải phục vụ; tải kết nối từ server ứng dụng; tác động doanh thu nếu lệnh ghi lỗi chập chờn. Ví dụ: thương mại điện tử cuối tháng 11 tới hết năm, phần mềm nhân sự tháng 11 ở Mỹ, bán hoa ngày Valentine (tr. 36). **Bổ sung ngoài sách:** latency tăng phi tuyến khi tài nguyên gần bão hòa, nên năng lực "dư gấp đôi trung bình" vẫn có thể vỡ ở đỉnh gấp mười.

[về mục 2.4](../chapter-02.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — Đồ thị trung bình IOutil và câu hỏi tăng gấp đôi

*Đồ thị trung bình theo ngày của IOutil trong 6 tháng luôn quanh 36%. Sếp hỏi: "năm sau lưu lượng gấp đôi, I/O có chịu được không?" Vì sao không nên trả lời từ đồ thị này, và cần dữ liệu gì?*

Đây gần như đúng ví dụ của sách: dữ liệu dài hạn gộp thành trung bình san phẳng đỉnh, nên nếu muốn biết disk I/O utilization có tăng gấp đôi được không, đồ thị trung bình rất dễ cho cảm giác an toàn giả (tr. 38). Cần xem **đỉnh** của dữ liệu nhiều tháng (tr. 38): giá trị lớn nhất theo từng khoảng, hoặc dữ liệu chi tiết quanh giờ cao điểm và mùa cao điểm (tr. 36).

**Kết quả đã đo** (TN4 phần B, dữ liệu tổng hợp): chuỗi có trung bình ngày 35,8% nhưng đỉnh 92% mỗi tối. Theo trung bình, gấp đôi là khoảng 72%, "chịu được"; theo đỉnh, gấp đôi là bất khả. **Bổ sung ngoài sách:** cũng nên xem thời gian chờ mỗi I/O (await), vì trên SSD, `%util` chưa phản ánh hết mức bão hòa.

[về mục 2.4](../chapter-02.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Trung bình, p95, p99 trong TN4

*Trong TN4, trung bình 35,4 ms, p95 22,9 ms, p99 524,8 ms. Con số nào nên đặt cạnh một SLO latency, vì sao trung bình gây hiểu lầm, và vì sao p99 có thể lớn hơn cả max đo được?*

- Đặt cạnh SLO **percentile cùng loại với SLO**: SLO "99% câu dưới X ms" thì so với p99; sách gợi ý vẽ p95 để khớp với SLO của ứng dụng và dễ hiểu với đội khác (tr. 38). Ở đây p99 524,8 ms cho thấy một SLO "99% dưới 100 ms" đang trượt.
- Trung bình gây hiểu lầm vì phân phối có hai cụm (khoảng 15 ms và khoảng 500 ms): 35,4 ms không ứng với request nào (không có câu nào giữa khoảng 26 ms và 501 ms), vừa phóng đại trải nghiệm thường gặp, vừa che đuôi chậm. Percentile dựa trên sắp xếp và bỏ phần cao nhất (tr. 38), nên mô tả được từng phần của phân phối.
- p99 > max (524,8 so với 514,6) vì `QUANTILE_99` của Performance Schema là **cận trên của bucket** histogram chứa percentile đó (**Kết quả đã đo**, giải thích là **Bổ sung ngoài sách**). Đây là ước lượng "cao", sai số cỡ độ rộng bucket.

[về mục 2.4](../chapter-02.md#s2-4)

<a id="q2-4-4"></a>

### Câu 2.4.4 — Khi nào gọi on-call vì replication lag

*Replication lag vừa là SLI cấp tính vừa là xu hướng dài hạn. Khi nào nên gọi on-call vì lag, khi nào chỉ nên ghi nhận xu hướng? Lag dài hạn nói gì về hệ thống?*

- **Gọi on-call** khi lag thực sự ảnh hưởng khách hàng, tức ứng dụng đọc từ replica (ví dụ người dùng không thấy bình luận của chính mình, tr. 32), **và** có hành động khắc phục được. Sách cảnh báo: không phải lúc nào cũng khắc phục ngay được; nếu không đọc từ replica, đừng cảnh báo gắt; cảnh báo ngoài giờ phải luôn hành động được (tr. 32).
- **Ghi nhận xu hướng** trong các trường hợp còn lại, xem định kỳ.
- **Ý nghĩa dài hạn:** dù chưa bao giờ ảnh hưởng khách hàng, lag cho thấy ít nhất thỉnh thoảng lượng ghi ở source vượt khả năng ghi của replica với cấu hình hiện tại: "con chim hoàng yến" cho năng lực ghi (tr. 32). Lắng nghe nó giúp chuẩn bị trước: chiến lược giảm lag ([chương 9](../chapter-09.md#excessive-lag)), hoặc chia ghi bằng sharding ([chương 11](../chapter-11.md#s2-6)).

[về mục 2.4](../chapter-02.md#s2-4)

<a id="q2-4-5"></a>

### Câu 2.4.5 — Vì sao nghĩ tới sharding lúc sự cố là quá muộn

*Vì sao sách nói nếu chỉ nghĩ tới sharding khi đã có sự cố năng lực thì đã quá muộn? Chỉ số nào trong chương giúp nhận ra sớm "tiền thân của nhu cầu shard"?*

Việc chia dữ liệu thành phần quản lý được không bắt đầu khi dữ liệu đã quá lớn cho một cluster, mà từ rất sớm, khi còn đang xác định mục tiêu cho trải nghiệm khách hàng (tr. 35). Lúc đã có sự cố do năng lực, bạn vừa phải chữa cháy vừa phải làm một thay đổi kiến trúc lớn, nên đã muộn (tr. 35). Khi quy mô tăng, giữ cùng SLO đã ngày càng khó và đắt (tr. 38–39).

Chỉ số báo sớm trong chương:

- **thời gian restore** tới khi bắt kịp replication, so với mục tiêu MTTR: sách gọi nó là tiền thân của nhu cầu shard (tr. 35);
- **xu hướng replication lag**: dấu hiệu năng lực ghi (tr. 32);
- **tăng trưởng dung lượng đĩa** so với khả năng chứa của một cluster (tr. 30);
- **xung đột giữa các query profile** của các nhóm stakeholder trên cùng data set (tr. 23);
- chính **SLI và SLO**: chúng cho biết khi nào cần đầu tư scale để mỗi cluster còn quản lý được (tr. 39).

[về mục 2.4](../chapter-02.md#s2-4)

<a id="q2-5-1"></a>

### Câu 2.5.1 — Mục tiêu và giám sát chủ động

*Liệt kê ba vùng nên đặt mục tiêu và các vùng nên giám sát chủ động theo Summary. Vì sao sách tách "mục tiêu" khỏi "giám sát chủ động"?*

- **Ba vùng đặt mục tiêu:** latency, availability, errors (tr. 39).
- **Giám sát chủ động:** tăng trưởng kết nối, dung lượng đĩa, disk I/O và latency (tr. 39). Mục 2.3.6 còn có replication lag, auto-increment, thời gian restore (tr. 31–35).

Lý do tách: mục tiêu đo xem khách hàng **đang** hài lòng không (tr. 29, 39). Giám sát chủ động theo dõi những tín hiệu **không gắn trực tiếp** với tác động lên khách hàng (tr. 29) nhưng báo trước sự cố. Nếu chỉ đo khi sự cố đang xảy ra, bạn luôn đến muộn; sách khuyên dành công cho những thứ giúp **ngăn** sự cố (tr. 39). Hai loại cũng dẫn tới hành động khác nhau: trượt SLO thì dừng tính năng; chỉ báo chủ động xấu đi thì lên kế hoạch (mở rộng đĩa, đổi kiểu khóa, chia dữ liệu).

[về mục 2.5](../chapter-02.md#s2-5)

<a id="q2-5-2"></a>

### Câu 2.5.2 — Vì sao phải xem lại SLI và SLO

*Vì sao SLI và SLO phải được xem lại định kỳ? Nêu hai sự kiện cụ thể buộc bạn sửa một SLO của tầng database.*

Sách nói SLI và SLO không đặt một lần là xong: doanh nghiệp lớn lên, bạn hiểu trải nghiệm khách hàng sâu hơn, và hiểu biết đó phải dẫn tới chỉ số và mục tiêu tốt hơn (tr. 39).

**Ví dụ minh họa do AI xây dựng**, hai sự kiện:

1. **Khách hàng phàn nàn trong khi SLO vẫn đạt.** SLI không đại diện cho trải nghiệm (đi ngược yêu cầu "luôn đại diện trải nghiệm khách hàng", tr. 39), ví dụ đo latency phía server trong khi vấn đề nằm ở connection pool; phải chuyển sang đo phía client (tr. 27).
2. **Một tính năng mới quan trọng hơn ra đời**, ví dụ thanh toán trả góp. Sách kỳ vọng SLI và SLO khác nhau theo tác động và doanh thu của tính năng (tr. 22–23); tính năng mới cần SLO riêng, có thể chặt hơn.

Cũng có thể là kiến trúc đổi (thêm proxy làm tăng latency nền) hay SLO luôn đạt với dư địa quá lớn (có thể nới để tiết kiệm công).

[về mục 2.5](../chapter-02.md#s2-5)

<a id="q2-5-3"></a>

### Câu 2.5.3 — Xếp loại sáu chỉ số

*Dùng toàn chương, xếp sáu chỉ số sau vào "SLI", "chỉ báo sớm" hay "theo dõi dài hạn" và nêu hành động khi chúng xấu đi: tỉ lệ checkout thành công, `Threads_running`, tỉ lệ lỗi lock wait timeout, phần trăm auto-increment, thời gian restore, p99 latency phía client.*

| Chỉ số | Loại | Hành động khi xấu đi |
| --- | --- | --- |
| Tỉ lệ checkout thành công | SLI availability (tr. 23, 26) | Đang tiêu nhanh ngân sách lỗi thì điều tra ngay; trượt SLO thì dừng tính năng mới (tr. 22) |
| `Threads_running` | Chỉ báo sớm (tr. 27) | Vượt số core hoặc khoảng 100: gọi on-call, giảm tải bằng `KILL` hoặc `pt-kill`, phân tích query (tr. 31) |
| Tỉ lệ lock wait timeout | Chỉ báo sớm theo tốc độ (tr. 28) | Tăng tốc: tìm transaction giữ lock, hàng nóng; tránh để thành mất ghi |
| Phần trăm auto-increment | Theo dõi dài hạn (tr. 33) | Ticket; lên kế hoạch đổi kiểu khóa sang lớn hơn ([chương 6](../chapter-06.md)) |
| Thời gian restore | Theo dõi dài hạn (tr. 34–35) | Gần mục tiêu MTTR: chỉnh MTTR, định nghĩa lại chức năng then chốt, rút ngắn restore, hoặc functional sharding (tr. 35) |
| p99 latency phía client | SLI latency (tr. 27, 38) | So với SLO; lệch thì dùng Performance Schema, slow log để chẩn đoán (tr. 24–25) |

Ranh giới không cứng: replication lag chẳng hạn vừa có thể là SLI cấp tính vừa là xu hướng dài hạn (tr. 32).

[về mục 2.5](../chapter-02.md#s2-5)
