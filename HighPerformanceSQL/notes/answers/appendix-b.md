# Đáp án — Phụ lục B: MySQL trên Kubernetes

Về ghi chú: [Phụ lục B](../appendix-b.md)

> Đáp án dẫn trang in của sách ("tr. N", trang PDF = N + 22). Phần nào không có trong sách được ghi **Bổ sung ngoài sách**. Phụ lục này không có bài thực hành, nên không có **Kết quả đã đo**.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Lợi ích stateless không chuyển nguyên sang MySQL

*Vì sao hai lợi ích chính của Kubernetes với workload stateless (co giãn, hiệu quả tài nguyên) không chuyển nguyên sang MySQL? Tác giả đề xuất thu hẹp mục tiêu thế nào?*

Sách nói Kubernetes mạnh với workload stateless vì mang lại co giãn và hiệu quả sử dụng tài nguyên tính toán (tr. 349). Lý do chúng không chuyển nguyên sang MySQL (**Bổ sung ngoài sách**, nối với các chương trước):

- **Co giãn.** Thêm pod web server là thêm năng lực xử lý ngay, vì mọi pod như nhau. Thêm một pod MySQL nghĩa là dựng một replica: phải chép dữ liệu, bắt kịp replication, và chỉ giúp được phần **đọc**. Muốn tăng năng lực ghi phải sharding ([Chương 11](../chapter-11.md#s2-6)). Bớt pod cũng không "rẻ": pod đó giữ dữ liệu và có thể là source.
- **Hiệu quả tài nguyên.** Kubernetes đạt hiệu quả nhờ xếp nhiều workload lên chung node và co giãn theo tải. MySQL lại cần bộ nhớ ổn định cho buffer pool ([Chương 5](../chapter-05.md#buffer-pool)) và độ trễ đĩa, CPU ổn định ([Chương 4](../chapter-04.md)); chia node với workload co giãn làm mất tính ổn định đó. Đây cũng là lý do sách hỏi về node chuyên dụng (tr. 350).

Tác giả đề xuất **thu hẹp chiến thắng** thành: chỉ dùng Kubernetes để **provisioning và cấu hình** hệ thống cho tài nguyên database (tr. 349–350). Đi kèm là ba điều phải nói rõ từ đầu: workload database được quản lý tách khỏi stateless, cần bộ kỹ năng vận hành khác, và xử lý container hỏng theo cách khác (tr. 350).

[về mục 2.1](../appendix-b.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Hai lựa chọn control plane

*So sánh hai lựa chọn: operator làm tất cả, và Kubernetes chỉ để provisioning. Vì sao phải quyết định sớm?*

Hai lựa chọn sách nêu (tr. 350):

| | Operator làm tất cả | Kubernetes chỉ để provisioning |
| --- | --- | --- |
| Phạm vi | Provisioning, failover, quản lý kết nối tới database | Dựng và cấu hình; quản lý database đang chạy bằng "các cách khác" |
| Ưu (**Bổ sung ngoài sách**) | Một chỗ điều khiển; ít mã ghép nối | Giữ công cụ và quy trình failover, backup đã kiểm chứng của đội database |
| Nhược (**Bổ sung ngoài sách**) | Thừa hưởng cách failover và lỗi của operator; phải hiểu sâu nó | Phải ghép hai hệ thống; trách nhiệm dễ chia không rõ |

Vitess, được giới thiệu ở chương 11, là ví dụ của mô hình đầy đủ: VTGate lo truy cập của ứng dụng, topology store lo danh mục instance, và có operator để chạy lớp đó trên Kubernetes (tr. 304–305).

**Vì sao quyết định sớm:** sách nói kỳ vọng với control plane sẽ chi phối nhiều chi tiết vận hành về sau (tr. 350). Cụ thể, câu trả lời cho phần lớn 8 câu hỏi ở mục The Finer Details (tr. 350) phụ thuộc vào nó: ai làm failover khi rút node, backup do ai chạy, nâng cấp MySQL theo thứ tự nào. Đổi mô hình khi đã có nhiều database chạy trên đó nghĩa là làm lại cả quy trình vận hành (**Bổ sung ngoài sách**).

[về mục 2.1](../appendix-b.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Dữ liệu trong container hay trên volume

*Câu hỏi "dữ liệu nằm trong container hay trên volume gắn riêng" liên quan thế nào tới câu "nếu mọi thứ hỏng, dựng lại thế nào, có mất dữ liệu không"?*

Câu hỏi thứ hai trong danh sách của sách (tr. 350) hỏi bạn sẽ gắn volume vào container và quản lý phục hồi container **tách** khỏi dữ liệu, hay để dữ liệu **là một phần của** container. Câu hỏi về chế độ hỏng ở tr. 351 là phép thử cho lựa chọn đó.

**Bổ sung ngoài sách:**

- **Dữ liệu trong container:** hệ tệp của container mất khi container bị xóa. Pod chết là bản sao dữ liệu đó mất. Dựng lại nghĩa là chép lại toàn bộ dữ liệu từ một replica hay từ backup, thời gian tỉ lệ với kích thước dataset. Nếu node đó là source và replication bất đồng bộ, giao dịch chưa kịp replicate sẽ mất ([Chương 9](../chapter-09.md#async-replication)).
- **Dữ liệu trên volume bền:** pod chết, volume còn. Pod mới gắn lại volume, InnoDB chạy crash recovery rồi tiếp tục. Câu hỏi "dựng lại thế nào" có câu trả lời nhanh hơn. Nhưng vẫn phải lo trường hợp volume hỏng hay không gắn lại được, và khi đó lại quay về backup ([Chương 10](../chapter-10.md)).

Vậy lựa chọn ở câu hỏi 2 quyết định phần lớn **RTO** và một phần **RPO** của kịch bản hỏng. Sách muốn bạn có câu trả lời cho cả hai trước khi bắt đầu (tr. 351).

[về mục 2.1](../appendix-b.md#s2-1)

<a id="q2-1-4"></a>

### Câu 2.1.4 — Bắt đầu từ stateless, rồi dataset nhỏ và ít quan trọng

*Vì sao tác giả khuyên bắt đầu với workload stateless, rồi tới database vài GB và ít quan trọng? Liên hệ với các chương trước.*

Sách (tr. 351): MySQL không nên là "chuột bạch" đầu tiên. Hãy chứng minh tính khả thi và học các góc sắc bằng workload stateless trước; khi chọn trường hợp đầu tiên cho database, bắt đầu với dataset nhỏ (vài GB trên đĩa) và ít quan trọng, để mọi đội làm quen với mô hình vận hành mới mà ít rủi ro cho doanh nghiệp.

Lý do, nối với các chương trước:

- **Tách hai nguồn rủi ro.** Nếu cả nền tảng lẫn workload đều mới, khi có sự cố sẽ khó biết lỗi do Kubernetes hay do MySQL trên Kubernetes. Workload stateless giúp đội làm chủ nền tảng trước, khi lỗi còn rẻ (**Bổ sung ngoài sách**).
- **Dataset nhỏ thì khôi phục nhanh.** Dataset khôi phục lâu nhất quyết định thời gian khôi phục ([Chương 2](../chapter-02.md#backup-restore-time), tr. 35), và kích thước instance là chỉ số để biết dữ liệu có quá lớn cho mục tiêu khôi phục ([Chương 13](../chapter-13.md#dr-planning), tr. 340). Với vài GB, dựng lại replica hay restore từ backup chỉ mất ít phút.
- **Blast radius nhỏ.** Chương 11 nói Vitess ưa instance nhỏ vì bán kính ảnh hưởng khi hỏng nhỏ hơn (tr. 305). Dữ liệu ít quan trọng thì hậu quả của một sự cố trong giai đoạn học cũng nhỏ.
- **Học dần mô hình vận hành.** Sách nhấn mạnh đây là mô hình vận hành mới, mọi thứ càng lớn càng phức tạp (tr. 350); bắt đầu nhỏ để ba đội (database, Kubernetes, tính năng) cùng học.

[về mục 2.1](../appendix-b.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Tận dụng khoản đầu tư sẵn có

*"Tận dụng khoản đầu tư Kubernetes sẵn có" là lý do tốt hay chưa đủ? Khi nào nó trở thành lý do không tốt?*

Sách coi mong muốn này là **dễ hiểu** và hợp lý (tr. 349, 351): quản lý cluster Kubernetes cần người chuyên trách, nên tổ chức muốn tận dụng chuyên môn đó. Nhưng sách cũng nói nó phải được **cân bằng với nhu cầu ổn định** của tầng data store (tr. 351). Vậy đây là lý do **cần nhưng chưa đủ**.

Nó trở thành lý do không tốt khi (**Bổ sung ngoài sách**, suy ra từ tr. 349–351):

- nó là lý do **duy nhất**, không kèm giá trị cụ thể muốn đạt (câu hỏi "giá trị gì", tr. 349);
- chưa trả lời được các câu hỏi về backup, restore, nâng cấp, rút node (tr. 350) và câu hỏi "có mất dữ liệu không" (tr. 351);
- bắt đầu từ database lớn, quan trọng, ngược với lời khuyên ở tr. 351;
- đội database và đội Kubernetes chưa thống nhất trách nhiệm và SLO (tr. 351).

Liên hệ [Chương 12](../chapter-12.md#no-free-lunch): không có bữa trưa miễn phí; đánh đổi phải được đặt trong bối cảnh cách doanh nghiệp vận hành (tr. 324).

[về mục 2.2](../appendix-b.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Một rủi ro, một phần thưởng và chỉ số đánh giá

*Nêu một rủi ro và một phần thưởng cụ thể khi đưa MySQL lên Kubernetes, và chỉ số nào giúp đánh giá từng cái.*

Một cách trả lời (**Ví dụ minh họa do AI xây dựng**; các chỉ số là **Bổ sung ngoài sách**):

- **Phần thưởng: provisioning thống nhất, nhanh hơn** (tr. 349–350; tốc độ làm việc của kỹ sư, tr. 351). Chỉ số: thời gian từ lúc một đội yêu cầu tới lúc có cụm MySQL dùng được; số bước thủ công trong quy trình; số yêu cầu đội database phải xử lý bằng tay mỗi tháng.
- **Rủi ro: mất dữ liệu hay gián đoạn khi pod hoặc node hỏng** (tr. 351), ví dụ khi đội Kubernetes rút node để nâng cấp cluster (tr. 350). Chỉ số: RTO và RPO đo trong diễn tập có kiểm soát (so tập GTID trên source cũ và mới, [Chương 9](../chapter-09.md#gtid)); số lần failover ngoài kế hoạch mỗi tháng; latency p99 so với cùng workload trên VM ([Chương 2](../chapter-02.md#percentiles)).

Điểm quan trọng: cả hai phía đều phải đo được, để quyết định "theo lăng kính rủi ro và phần thưởng" (tr. 351) dựa trên số liệu chứ không dựa trên cảm nhận. Chỉ mở rộng phạm vi khi các chỉ số rủi ro ở pilot nằm trong SLO đã thống nhất.

[về mục 2.2](../appendix-b.md#s2-2)
