# Đáp án — Chương 13: Compliance với MySQL

Liên kết: [về ghi chú chương 13](../chapter-13.md) · [bài thực hành](../experiments/chapter-13.md)

> Đáp án diễn giải nội dung sách kèm trang ("tr. N" là số trang in; trang PDF = N + 22). Phần không có trong sách ghi **Bổ sung ngoài sách**; kết quả thí nghiệm ghi **Kết quả đã đo** (MySQL 8.0.42 Community, Windows 11, instance riêng). Đây không phải tư vấn pháp lý (tr. 325).

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Control là gì, vì sao một việc phủ nhiều khung

*Sách định nghĩa control là gì? Vì sao định nghĩa này khiến một việc kỹ thuật như "mỗi service một database user" có thể phục vụ cùng lúc nhiều khung compliance?*

Control là quy trình và quy tắc mà công ty **tự định nghĩa và thực hành** để giảm xác suất một kết cục rủi ro không mong muốn (tr. 326). Điểm then chốt: control được định nghĩa theo **rủi ro cần giảm**, không theo tên luật.

Nhiều khung cùng lo một số rủi ro giống nhau: truy cập trái phép, thay đổi không được ghi lại, mất dữ liệu. "Mỗi service một user" giảm rủi ro truy cập vượt nhu cầu và thu nhỏ blast radius khi lộ credential (tr. 330, 332). Nó vì vậy làm bằng chứng cho SOX (chỉ người cần mới truy cập dữ liệu doanh thu, tr. 326), PCI DSS (quản lý truy cập dữ liệu chủ thẻ, tr. 327), HIPAA (kiểm soát truy cập ePHI, tr. 327), SOC 2 (quản lý truy cập instance, tr. 326). Sách nói thẳng: cùng một việc có thể phủ nhiều kiểm soát, và cần hiểu mục đích của từng kiểm soát để đưa đúng bằng chứng (tr. 328).

[về mục 2.1](../chapter-13.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Khung nào đòi log cả truy cập đọc

*Trong các khung sách nêu, khung nào đòi log cả truy cập đọc tới dữ liệu nhạy cảm? Điều đó loại bỏ ngay một cách theo dõi thay đổi nào ở mục sau?*

**HIPAA**: sách nêu "activity logging" **mỗi khi ePHI được truy cập** (tr. 327). Ở phần data access logging, sách cũng nói PCI và HIPAA coi dữ liệu nhạy cảm tới mức **mọi truy cập** phải được ghi (tr. 334). FedRAMP đòi kiểm toán "truy cập và thay đổi" (tr. 327).

Hệ quả: **trigger** bị loại, vì trigger chỉ theo dõi được thao tác ghi và không mở rộng được để theo dõi đọc (tr. 334). **Kết quả đã đo** ở thí nghiệm 5: một phiên đọc 1.000 hàng không để lại dấu vết nào trong bảng audit dựng bằng trigger. Muốn log đọc phải dùng audit plug-in ở tầng server (tr. 334), hay ghi ở proxy, dịch vụ managed.

[về mục 2.1](../chapter-13.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Ba thực hành cho SOC 2

*Một công ty SaaS chưa niêm yết, không xử lý thẻ hay dữ liệu y tế, nhưng có khách hàng doanh nghiệp đòi báo cáo SOC 2. Ba thực hành database nào sách nói họ cần có nếp?*

Sách nói kỹ sư database ở công ty muốn đạt SOC 2 cần thực hành đã thành nếp về (tr. 326):

1. **quản lý thay đổi database** (change management): ở chương này là version control cho schema và configuration management cho user (tr. 336–337);
2. **quy trình backup và restore**: tự động, có cảnh báo, có test, có log tập trung, có số liệu thời gian khôi phục (tr. 338–340);
3. **quản lý truy cập** tới instance: user riêng, least privilege, gỡ user không dùng (tr. 332, 337–338).

Không cần SOX (chưa niêm yết), PCI DSS (không có thẻ) hay HIPAA (không có dữ liệu y tế), đúng ý sách rằng không phải quy định nào cũng áp cho mọi doanh nghiệp, và có quy định chỉ áp cho một phần dữ liệu (tr. 326).

**Bổ sung ngoài sách:** SOC 2 Type 2 đánh giá kiểm soát trong một khoảng thời gian, nên bằng chứng phải liên tục (lịch sử PR, log backup theo ngày), không chỉ là ảnh chụp cấu hình.

[về mục 2.1](../chapter-13.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — Quyền được lãng quên và câu DELETE

*Vì sao "quyền được lãng quên" của GDPR khó hơn một câu `DELETE` trong MySQL? Nêu ít nhất ba nơi dữ liệu có thể còn tồn tại.*

Sách định nghĩa right to be forgotten là con đường pháp lý để cá nhân yêu cầu xóa dữ liệu của mình khỏi hệ thống của bất kỳ bên xử lý nào (tr. 327). Sách không bàn chi tiết cách làm trong MySQL.

**Bổ sung ngoài sách.** `DELETE` chỉ xóa bản "đang sống" trong bảng. Dữ liệu còn có thể nằm ở:

- **binary log**: với `binlog_format=ROW` (mặc định 8.0), row event chứa ảnh trước của hàng bị xóa, tới khi binlog bị purge;
- **backup** logic hay vật lý, cho tới hết thời hạn giữ backup;
- **replica**, nhất là replica cố ý trễ;
- bảng tổng hợp, bảng audit, kho phân tích được nạp từ binlog hay từ dump;
- general log hoặc slow log nếu có ghi giá trị trong câu SQL (thí nghiệm 4 cho thấy chuỗi trong `SELECT` thường hiện nguyên văn trong general log).

Vì vậy một quy trình xóa phải gắn với chính sách giữ binlog, giữ backup, và danh sách hệ thống dẫn xuất. Mâu thuẫn giữa log bất biến và nghĩa vụ xóa được DDIA bàn ở [chương 11](../../../DDIA/ddia-notes/chapter-11.md#immutability-limits).

[về mục 2.1](../chapter-13.md#s2-1)

<a id="q2-1-5"></a>

### Câu 2.1.5 — Schrems II và giới hạn của sách

*Theo sách, Schrems II thay đổi cách suy luận về kiến trúc dữ liệu thế nào so với GDPR ban đầu? Vì sao ghi chú khuyên không dùng phát biểu của sách như kết luận pháp lý ở thời điểm hiện tại?*

Theo sách (tr. 328): phán quyết 2020 cho rằng Privacy Shield không đủ bảo vệ người ở EU khỏi giám sát của chính phủ Mỹ qua FISA; vì vậy, theo cách hiểu của tác giả, dữ liệu định danh của người ở EU do công ty Mỹ thu thập phải ở lại EU và không để người ở Mỹ truy cập. GDPR ban đầu chủ yếu nói về đồng ý, giới hạn truy cập và quyền xóa; Schrems II thêm câu hỏi **dữ liệu nằm ở đâu và ai truy cập được từ đâu** (data sovereignty, tr. 325). Sách cũng nói cách thực thi còn chưa rõ, mỗi doanh nghiệp tự xác định phạm vi, và nên giả định nó sẽ chạm tới bạn nếu có khách ở châu Âu (tr. 328).

Lý do không dùng như kết luận pháp lý:

- Chương này tự nói nó không phải tư vấn pháp lý và khuyên hỏi pháp chế (tr. 325).
- Phát biểu phản ánh thời điểm 2021, khi việc thực thi "chưa rõ" (tr. 328).
- **Bổ sung ngoài sách:** phán quyết thực tế vô hiệu Privacy Shield và đặt điều kiện cho điều khoản hợp đồng mẫu, chứ không viết thành một quy tắc đơn giản "phải ở lại EU"; sau đó EU và Mỹ thông qua khung EU–US Data Privacy Framework (2023). Tình hình pháp lý đã đổi, nên kiến trúc phải dựa vào ý kiến pháp chế hiện hành.

[về mục 2.1](../chapter-13.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Xoay vòng credential là một lần deploy

*Vì sao sách nói xoay vòng credential "là một lần deploy", kể cả khi credential chỉ là một dòng cấu hình? Hệ quả với availability là gì?*

Credential nằm trong cấu hình của **mọi instance** đang chạy của ứng dụng. Đổi nó trên database là chưa đủ; thay đổi phải **lan ra toàn fleet**, và thường phải **restart** ứng dụng để chúng đọc cấu hình mới (tr. 331). Như mọi lần deploy, việc này có thể làm gián đoạn dịch vụ nếu làm cùng lúc trên mọi instance, nên phải điều phối để **không ảnh hưởng availability** của dịch vụ nói chung (tr. 331). Cách làm cụ thể phụ thuộc cách đưa cấu hình tới ứng dụng và pipeline deploy (tr. 331).

Hệ quả: cần một quy trình **đã luyện tập**, tự động hoặc có runbook, chạy được cả theo lịch lẫn khẩn cấp (tr. 337). **Kết quả đã đo** (thí nghiệm 2): với cách đổi username, drop user cũ trước khi mọi instance đã chuyển làm phiên còn sót bị `ERROR 1142` ngay; với dual password, phiên đang mở không bị cắt, nhưng pool phải được tái tạo để thật sự dùng mật khẩu mới.

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Bãi mìn kích thước secret

*Một đội lưu mật khẩu database trong dịch vụ secret của cloud. Hai năm sau, kiểm soát mới đòi quản lý cả private key TLS như secret. Sách cảnh báo "bãi mìn" nào, và hai lựa chọn khó chịu nào xuất hiện?*

Đó là **space limitations** (tr. 331). Nhiều dịch vụ secret của cloud giả định secret ngắn, cỡ một cặp user và mật khẩu. Khi phải lưu chuỗi dài như SSH key hay certificate, có thể vượt kích thước tối đa của một key. Khi lượng secret lớn lên, tổ chức phát hiện cần **một giải pháp secret khác** cho secret dài. Hai lựa chọn khi đó (tr. 331):

1. **Di chuyển** toàn bộ sang giải pháp mới, có thể ảnh hưởng uptime;
2. **Vận hành hai giải pháp** song song, với tooling và tích hợp riêng cho từng cái, kèm độ phức tạp của nó.

Bài học: kiểm tra giới hạn kích thước **ngay khi chọn** giải pháp, dựa trên danh sách secret mà kiểm soát có thể đòi (tr. 329 liệt kê cả SSH private key và certificate key).

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Câu hỏi về failure mode của secret manager

*Ứng dụng chạy ở ba region và đọc credential từ secret manager lúc khởi động. Liệt kê các câu hỏi về failure mode mà sách muốn bạn trả lời trước khi chọn giải pháp.*

Theo phần regional availability (tr. 331–332) và phần đầu mục secrets (tr. 329–330):

- Secret được **lấy** thế nào lúc runtime?
- Chuyện gì xảy ra nếu ứng dụng **không gọi được** secret manager? Có failure mode mới nào do phụ thuộc này?
- Dịch vụ của cloud có **tự nhân bản secret sang region khác** không, hay phải tự xây?
- Secret manager có thành **single point of failure** không? Đánh đổi khi nó không sẵn sàng đã được viết rõ chưa?
- Secret nào được **cache**, cache thế nào? Đã thống nhất trước với đội pháp chế và đội bảo mật chưa (tr. 330)?

**Bổ sung ngoài sách:** với ứng dụng chỉ đọc lúc khởi động, sự cố secret manager thường không làm sập instance đang chạy, nhưng chặn **khởi động mới** (autoscale, deploy, restart sau crash). Đó chính là lúc hệ thống cần nó nhất.

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-4"></a>

### Câu 2.2.4 — Hai thực hành, hai cơ chế giảm rủi ro

*"Không dùng chung user" và "không đưa credential vào repo" giảm rủi ro theo hai cơ chế khác nhau. Mỗi cái giảm phần nào: xác suất bị lộ hay phạm vi ảnh hưởng khi bị lộ?*

- **Không đưa credential vào repo** (tr. 330) giảm **xác suất bị lộ**: repo bị sao chép, fork, công khai nhầm, và lịch sử commit giữ secret mãi. Quét secret trước khi merge là lớp "trust but verify".
- **Không dùng chung user** (tr. 330, 332) giảm **phạm vi ảnh hưởng** (blast radius) khi lộ đã xảy ra: chỉ một service phải nhận credential mới, và quyền của credential bị lộ chỉ là quyền của service đó.

Sách nhấn mạnh rằng lộ credential xảy ra với công ty mọi cỡ (tr. 332), nên cần cả hai lớp: không thể đưa xác suất về 0, vì vậy phải giới hạn thiệt hại. Hai thực hành này cũng làm thay đổi khẩn cấp về sau đơn giản hơn (tr. 330).

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-5"></a>

### Câu 2.2.5 — Dữ liệu PHI vào cluster cũ

*Trong ví dụ công ty marketing mở sản phẩm y tế (tr. 332), điều gì sẽ xảy ra nếu dữ liệu PHI được thêm vào cluster cũ? Vì sao sách nhấn mạnh "từ đầu"?*

Nếu PHI nằm chung cluster, kiểm soát HIPAA (kiểm soát truy cập, mã hóa, log mọi truy cập, tr. 327) sẽ phải áp cho **cả cluster**, kéo theo dataset cũ và mọi ứng dụng phụ thuộc nó. Đó là "gánh nặng không đáng có" mà sách muốn tránh (tr. 332): thêm log truy cập, quyền chặt hơn, kiểm toán rộng hơn cho dữ liệu vốn không cần.

"Từ đầu" vì tách **trước** khi dữ liệu trộn vào nhau rẻ hơn nhiều so với tách **sau**. **Bổ sung ngoài sách:** tách về sau cần di chuyển dữ liệu đang sống, đổi code truy vấn đang `JOIN` qua hai miền dữ liệu, và chứng minh với kiểm toán rằng không còn PHI sót ở cluster cũ, kể cả trong backup và binlog. Đây là một trường hợp của functional sharding mà chương 2 định nghĩa, có tính cả access control (tr. 34).

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-6"></a>

### Câu 2.2.6 — Có role mà vẫn ERROR 1142

*Một user `svc_x` đã được `GRANT 'r_orders_rw'`. `SHOW GRANTS FOR svc_x` hiện đúng dòng gán role. Ứng dụng vẫn bị `ERROR 1142` khi `SELECT`. Nguyên nhân gốc có thể là gì trong MySQL 8.0, và kiểm tra bằng cách nào?*

**Bổ sung ngoài sách (Kết quả đã đo, thí nghiệm 1 và 2).** Trong MySQL 8.0, role được gán nhưng **chưa được kích hoạt** trong phiên. Mặc định `activate_all_roles_on_login = OFF`, nên role chỉ có hiệu lực khi là **default role** của user, hoặc khi ứng dụng tự chạy `SET ROLE`. Đo được: chưa có default role thì `CURRENT_ROLE()` trả về `NONE` và `SELECT` bị `ERROR 1142`.

Kiểm tra:

- `SELECT CURRENT_ROLE();` trong phiên của ứng dụng;
- `SELECT * FROM mysql.default_roles WHERE USER = 'svc_x';`
- `SHOW GRANTS FOR 'svc_x'@'…' USING 'r_orders_rw';` để xem quyền hiệu lực khi role được bật.

Sửa: `SET DEFAULT ROLE ALL TO 'svc_x'@'…';` (hoặc tạo user với `CREATE USER … DEFAULT ROLE …`). Bài học cho quy trình xoay vòng của sách (tr. 337): so `SHOW GRANTS` là cần nhưng chưa đủ; đo được hai user có `SHOW GRANTS` giống hệt mà một user vẫn lỗi.

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-7"></a>

### Câu 2.2.7 — Least privilege cho người và cho code

*Least privilege áp cho "cả người lẫn code" (tr. 332). Cho một ví dụ về mỗi loại trong vận hành MySQL, và chỉ ra bằng chứng kiểm toán tương ứng.*

Sách nói least privilege áp cho cả truy cập của người và của code ứng dụng (tr. 332). Ví dụ (**Ví dụ minh họa do AI xây dựng**):

- **Code:** service billing chỉ có `SELECT` trên bảng hóa đơn, không có quyền trên bảng thẻ. Bằng chứng: định nghĩa role và user trong repo configuration management, có PR và review (tr. 337); đo được user như vậy nhận `ERROR 1142` khi chạm bảng khác (thí nghiệm 1).
- **Người:** chỉ một nhóm nhỏ được duyệt thay đổi schema của cluster tài chính. Bằng chứng: quyền duyệt trên repository schema riêng của cluster đó và lịch sử ai yêu cầu, ai duyệt, chạy ra sao (tr. 336). Sách nói kiểm toán thường hỏi ai có quyền thay đổi database, và giữ vòng tròn người vận hành nhỏ là đúng least privilege (tr. 336).

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-8"></a>

### Câu 2.2.8 — Bốn lý do không dùng trigger

*Nêu bốn lý do sách không khuyên dùng trigger để theo dõi thay đổi. Kết quả đo ở thí nghiệm 5 ủng hộ và làm rõ lý do thứ nhất thế nào?*

Bốn lý do (tr. 334):

1. Trigger gây giảm hiệu năng ghi, và ảnh hưởng đúng vào lúc tệ nhất.
2. Trigger là đặt business logic trong database.
3. Code trong database dễ né quy trình test, staging, deploy, và thành bất ngờ trong sự cố.
4. Trigger chỉ theo dõi ghi, không mở rộng được cho đọc.

**Kết quả đã đo** (thí nghiệm 5, MySQL 8.0.42, laptop Windows nhiều nhiễu):

- **Ủng hộ:** `Handler_write` tăng đúng bằng số hàng bị cập nhật (200.001), binlog của một câu cập nhật 200.001 hàng tăng từ 7.633.958 lên 34.400.659 byte (khoảng 4,5 lần), thời gian trung vị từ 2,30 s lên 22,08 s.
- **Làm rõ:** với 2.000 câu cập nhật một hàng, mỗi câu tự commit, thời gian không khác biệt rõ (3,90 s so với 4,13 s), vì fsync lúc commit chi phối. Chi phí ghi vẫn gấp đôi nhưng chỉ lộ rõ ở workload ghi hàng loạt hoặc khi hệ thống đang gần giới hạn, khớp với cụm "lúc tệ nhất".

Ngoài ra (**Bổ sung ngoài sách**, đã đo): trigger chạy với quyền definer, nên `CURRENT_USER()` trong trigger ghi `root@localhost` thay vì user thực sự thay đổi.

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-9"></a>

### Câu 2.2.9 — Latency tăng đều mà plan không đổi

*Server MySQL có audit plug-in gửi log qua rsyslog. Một ngày p99 latency của mọi truy vấn tăng, nhưng không truy vấn nào có plan thay đổi. Theo cảnh báo của sách, nguyên nhân gốc nào cần nghi, và vì sao khó debug?*

Cần nghi **pipeline chuyển log audit bị chậm** (hộp cảnh báo tr. 336). Khi đường chuyển tệp hay sự kiện chậm lại, plug-in phải **buffer** sự kiện, và việc buffer này có thể ảnh hưởng hiệu năng của chính database server. Khó debug vì **triệu chứng duy nhất** là truy vấn chậm hơn (tr. 336): không có lỗi, plan không đổi, không có truy vấn "thủ phạm".

Hướng xử lý theo sách: ghi rõ output của plug-in được lưu thế nào trên host, kể cả lưu tạm; lên kế hoạch **chaos testing** cho toàn pipeline log (tr. 336); cấu hình rsyslog có chủ đích, không tin vào mặc định (tr. 335–336). **Bổ sung ngoài sách:** kiểm tra độ trễ của collector, đĩa nơi lưu tạm, và cách plug-in xử lý khi buffer đầy (chờ hay bỏ sự kiện, tùy cấu hình của Percona).

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-10"></a>

### Câu 2.2.10 — Vì sao my.cnf không đủ

*Vì sao sách nói kiểm tra `my.cnf` không đủ để chứng minh audit plug-in đang chạy? Kiểm tra đúng phải nhìn vào đâu?*

Vì plug-in có thể bị **tắt ngay lúc đang chạy** mà không cần restart server (tr. 335). Tệp cấu hình trên đĩa chỉ cho biết trạng thái sẽ có **ở lần khởi động tới**, không phải trạng thái hiện tại. Sách khuyên truy vấn **trạng thái hiện tại** từ shell: danh sách plug-in đang active, và biến policy cho thấy plug-in thật sự đang ghi truy vấn (tr. 335). Nếu muốn ghi cả đăng nhập, phải sửa lệnh kiểm tra cho phù hợp (tr. 335).

**Bổ sung ngoài sách:** cùng nguyên tắc áp cho Performance Schema: thí nghiệm 3 cho thấy thay đổi `setup_instruments` lúc chạy mất sau restart. Trạng thái runtime và trạng thái cấu hình có thể khác nhau theo cả hai chiều, nên giám sát phải đọc cả hai.

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-11"></a>

### Câu 2.2.11 — Bước dễ sai thứ tự khi xoay vòng

*Trong 5 bước xoay vòng credential của sách, bước nào dễ gây lỗi nếu làm sai thứ tự? Dựa vào kết quả đo, giải thích chuyện gì xảy ra với một phiên đang mở khi user cũ bị drop.*

Bước 5 (drop cặp cũ) phải đến **sau** bước 4 (restart mọi instance để chắc chắn cặp mới được dùng) (tr. 337). Bước 2 cũng dễ sai nếu chỉ so `SHOW GRANTS` mà quên default role (câu 2.2.6).

**Kết quả đã đo** (thí nghiệm 2): một phiên của `svc_orders_v1` đang chạy `SLEEP(6)`. `DROP USER` chạy thành công, không cảnh báo. Phiên vẫn kết nối, nhưng câu `SELECT` và `INSERT` tiếp theo bị `ERROR 1142`. Kết nối mới bằng v1 bị `ERROR 1045`. Nghĩa là connection pool còn kết nối cũ sẽ lỗi **ngay** ở truy vấn kế tiếp, không phải lúc tái kết nối.

**Bổ sung ngoài sách:** trước khi drop, kiểm `performance_schema.threads` (theo `PROCESSLIST_USER`) và `performance_schema.accounts` để biết host nào còn dùng user cũ; có thể `ACCOUNT LOCK` trước như bước đệm đảo ngược được.

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-12"></a>

### Câu 2.2.12 — Chính sách sáu tháng và Performance Schema

*Vì sao chính sách "gỡ user không hoạt động sáu tháng" không thể chỉ dựa vào `performance_schema.users`? Đề xuất cách bổ sung.*

Sách nói bảng `users` chỉ giữ lịch sử trong **đời của tiến trình server** hoặc tới **kích thước tối đa** của bảng, cái nào tới trước (tr. 338). Một server bị restart (nâng cấp, failover, bảo trì) trong sáu tháng sẽ mất toàn bộ lịch sử trước đó. **Kết quả đã đo** (thí nghiệm 3): sau restart, `TOTAL_CONNECTIONS` của mọi user về 0. Ngoài ra bảng chỉ ghi user **đã** kết nối (tr. 338), đăng nhập sai không tạo hàng, và bảng gom theo tên user chứ không theo host (đo được).

Cách bổ sung:

- Theo sách: dùng **audit log** (ví dụ plug-in của Percona) để biết user có kết nối trong một số tuần gần đây (tr. 338).
- **Bổ sung ngoài sách:** một job định kỳ chụp `performance_schema.users` hoặc `accounts` của mọi server vào kho bền (kèm `Uptime` để biết lúc nào bộ đếm bị reset), rồi tính "lần cuối thấy kết nối" theo từng user trên toàn fleet. Gỡ hai bước: `ACCOUNT LOCK` trước, `DROP USER` sau thời gian ân hạn.

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-13"></a>

### Câu 2.2.13 — Backup 3 giờ, monitoring check 30 giây

*Hệ thống monitoring của công ty chỉ cho check chạy tối đa 30 giây, còn backup mất 3 giờ. Theo sách, làm sao vẫn có bằng chứng backup thành công trong monitoring?*

Chạy backup như một monitoring check chỉ được khi monitoring chịu được check chạy lâu (tr. 339). Không được thì dùng **bread crumbs** (tr. 339): quy trình backup (và test backup) ghi lại dấu vết khi **chạy xong thành công**, ví dụ sửa một tệp chứa timestamp ở cuối mỗi lần chạy. Monitoring chỉ làm check **nhanh** là bread crumb tồn tại.

Yêu cầu đi kèm (tr. 339): cơ chế chạy theo lịch phải gửi sự kiện tới monitoring và hệ thống ticket, để lỗi được cảnh báo và mỗi lỗi thành work item được theo dõi. **Bổ sung ngoài sách:** check nên kiểm cả **độ mới** của timestamp (ví dụ dưới 26 giờ cho backup hằng ngày), không chỉ sự tồn tại, và bread crumb nên nằm ở nơi tập trung để không mất khi host bị thay.

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-14"></a>

### Câu 2.2.14 — Log backup cục bộ

*Vì sao sách coi log backup nằm trên đĩa cục bộ của một instance là không ổn cho kiểm toán, dù log đó đầy đủ?*

Sách muốn giải pháp giả định server **dễ thay thế và bị thay nhiều lần**, không phải máy "may đo" (tr. 340). Nếu máy bị cho nghỉ và thay trước kỳ kiểm toán, log cục bộ **mất cùng máy**. Tài sản liên quan nghiệp vụ, như log quy trình backup, phải nằm ở **nơi tập trung** mà ai có đúng quyền đều truy cập được (tr. 340). Thêm nữa, sách nói khi số cluster tăng thì không thể báo cáo thành công, thất bại bằng cách đọc logfile thủ công (tr. 339); log tập trung cho phép tìm kiếm và tổng hợp, đúng tinh thần structured log ở Hình 13-1 (tr. 333).

[về mục 2.2](../chapter-13.md#s2-2)

<a id="q2-2-15"></a>

### Câu 2.2.15 — Vòng tự cải thiện và MTTR

*Một cluster tăng 40% dung lượng mỗi năm. Giải thích vòng "tự cải thiện" mà sách đề xuất giúp phát hiện sớm rằng cluster sắp vượt mục tiêu MTTR, và hai quyết định có thể đưa ra.*

Sách đề xuất (tr. 340): cho script backup và script test backup **gửi số liệu thời gian** của mỗi lần chạy. Khi đó có số liệu theo từng cluster về thời gian backup và thời gian khôi phục, test. Theo dõi số liệu này theo thời gian cho biết dataset nào đang **quá lớn** so với MTTR doanh nghiệp kỳ vọng. Chương 2 đã gợi ý dùng kích thước instance làm chỉ số cho việc này (tr. 34–35, được nhắc ở tr. 340).

Với cluster tăng 40% mỗi năm, đường thời gian khôi phục sẽ đi lên; khi nó tiến gần MTTR, có dữ liệu để chọn một trong hai (tr. 340):

1. **ưu tiên chia nhỏ dataset** về kích thước chấp nhận được (ví dụ functional sharding);
2. **xem lại SLA khôi phục** với doanh nghiệp.

"Tự cải thiện" vì mỗi lần test restore vừa là bằng chứng SOC 2, vừa là một điểm dữ liệu cho dự báo. **Bổ sung ngoài sách:** giả định thời gian khôi phục tăng tuyến tính theo kích thước chỉ là xấp xỉ; nên dùng số đo thật vì tốc độ khôi phục còn phụ thuộc loại backup, I/O và số index phải dựng lại.

[về mục 2.2](../chapter-13.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — Mật khẩu "luôn được lưu mã hóa"

*Summary nói "ensure that your passwords are always stored encrypted". Phía MySQL server đã làm gì cho yêu cầu này, và phần nào vẫn là trách nhiệm của đội vận hành?*

Sách chỉ nêu yêu cầu (tr. 341), không nói cơ chế.

**Bổ sung ngoài sách, Kết quả đã đo** (thí nghiệm 4, MySQL 8.0.42):

- `mysql.user` lưu **hash** (`caching_sha2_password`, tiền tố `$A$005$`), không lưu mật khẩu. Nói chính xác là hash, không phải mã hóa đảo ngược được.
- `CREATE USER`, `ALTER USER`, `SET PASSWORD` được viết lại thành `IDENTIFIED BY <secret>` trong general log và trong `events_statements_history`.
- Binary log ghi lại câu dưới dạng `IDENTIFIED WITH … AS '<hash>'`: không có mật khẩu dạng thô, nhưng có hash.

Phần còn lại là của đội vận hành: secret trong cấu hình ứng dụng, biến môi trường, repo (tr. 330), secret manager (tr. 329–332); quyền đọc binlog và backup vì chúng chứa hash và dữ liệu (tr. 340); và mọi secret ứng dụng ghi vào bảng bằng câu SQL thường, vì những câu đó **không** được viết lại (đo được: chuỗi trong `SELECT` hiện nguyên văn trong general log).

[về mục 2.3](../chapter-13.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Pipeline log đáng tin trước khi cần

*Vì sao "có pipeline log đáng tin trước khi cần log truy cập" lại quan trọng, xét cả góc compliance lẫn góc hiệu năng?*

**Góc compliance.** Kiểm toán cần **hồ sơ theo thời gian**; không thể tạo lại log truy cập của quá khứ khi kiểm toán viên đã hỏi. Sách muốn bằng chứng được "nướng sẵn" để kiểm toán hằng năm không thành cuộc chạy đua gom bằng chứng (tr. 333), và nói compliance không thể gắn thêm khi cần (tr. 329).

**Góc hiệu năng.** Pipeline log là một phụ thuộc của database: nếu đường chuyển chậm, plug-in buffer sự kiện và truy vấn chậm đi, với triệu chứng duy nhất là truy vấn chậm hơn (tr. 336). Ghi ra đĩa cục bộ thì có nguy cơ đầy đĩa (tr. 335). Dựng và chaos-test pipeline **trước** khi bật log trên production cho phép hiểu các failure mode này trong điều kiện an toàn (tr. 336), thay vì phát hiện chúng giữa một sự cố.

[về mục 2.3](../chapter-13.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Một việc của Summary, nhiều khung

*Chọn một trong năm việc của Summary và chỉ ra nó phục vụ ít nhất hai khung compliance khác nhau ở mục 2.1.*

Ví dụ với "**thay đổi schema được kiểm soát và ghi log**" (tr. 341):

- **SOC 2**: đòi thực hành quản lý thay đổi database đã thành nếp (tr. 326).
- **SOX**: mọi thay đổi trên dữ liệu ảnh hưởng doanh thu phải có log và lý do; cần audit trail (tr. 326). Version control cho thấy ai yêu cầu, ai duyệt, chạy ra sao (tr. 336).
- **FedRAMP**: gồm configuration management và kiểm toán thay đổi (tr. 327).

Ví dụ khác, "**tách user ứng dụng**": phục vụ SOX (chỉ người cần mới truy cập, tr. 326), PCI DSS (quản lý truy cập dữ liệu chủ thẻ, tr. 327), HIPAA (kiểm soát truy cập ePHI, tr. 327) và SOC 2 (quản lý truy cập instance, tr. 326). Đây là ý "cùng một việc phủ nhiều kiểm soát" ở tr. 328.

[về mục 2.3](../chapter-13.md#s2-3)
