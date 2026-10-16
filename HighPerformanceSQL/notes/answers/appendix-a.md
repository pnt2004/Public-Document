# Đáp án — Phụ lục A: Nâng cấp MySQL

Về ghi chú: [Phụ lục A](../appendix-a.md) · Bài thực hành: [experiments/appendix-a.md](../experiments/appendix-a.md)

> Đáp án dẫn trang in của sách ("tr. N", trang PDF = N + 22). Phần nào không có trong sách được ghi **Bổ sung ngoài sách**; số liệu đo ghi **Kết quả đã đo** kèm tên thí nghiệm.

---

<a id="q2-1-1"></a>

### Câu 2.1.1 — Cơ sở cài đặt rộng và dot-20 rule

*Vì sao chính việc MySQL có cơ sở cài đặt rộng lại là lý do để **không** lên ngay bản mới nhất? Liên hệ với dot-20 rule.*

Cơ sở cài đặt rộng nghĩa là rất nhiều người khác đang chạy và thử MySQL (tr. 343). Lợi ích đó chỉ có với những bản **đã được nhiều người chạy đủ lâu**. Một bản vừa phát hành chưa được cộng đồng "thử hộ", nên bug và regression của nó chưa lộ ra. Lên ngay bản mới nhất là tự mình làm người thử đầu tiên, tức chấp nhận rủi ro vô tình đưa bug hay regression vào môi trường (tr. 343).

Dot-20 rule (chú thích 1, tr. 343) nói đúng điều này: phần mềm chỉ thật sự trưởng thành từ bản "chấm 20", sau khi đủ nhiều người dùng và đủ nhiều bản sửa lỗi. Tác giả nhấn mạnh đây không phải quy tắc cứng, chỉ là cách nói về đánh đổi giữa mới và ổn định.

Mặt kia của đánh đổi: tụt quá xa cũng có giá, vì bạn có thể đang gặp bug không lộ rõ và bỏ lỡ tính năng tối ưu hiệu năng (tr. 343). Vậy câu trả lời không phải "càng cũ càng tốt" mà là "đủ chín, nhưng không quá xa".

[về mục 2.1](../appendix-a.md#s2-1)

<a id="q2-1-2"></a>

### Câu 2.1.2 — Chẩn đoán hành vi lạ bằng release notes

*Production có hành vi lạ không giải thích được. Sách gợi ý quy trình nào? Vì sao bắt đầu từ phiên bản đang chạy chứ không chỉ đọc release notes của bản mới nhất?*

Quy trình (tr. 344): xác định phiên bản MySQL đang chạy, rồi đọc release notes của **các bản sau nó, lần lượt tới bản mới nhất**. Rất có thể hiện tượng là một bug của MySQL đã được ghi nhận và sửa. Nếu đúng, bạn có lý do để nâng cấp.

Vì sao bắt đầu từ bản đang chạy (**Bổ sung ngoài sách**, suy luận từ cách sách mô tả):

- Bản sửa có thể nằm ở **bất kỳ** bản trung gian nào, không chỉ bản mới nhất. Release notes mỗi bản chỉ liệt kê thay đổi của bản đó.
- Đọc liên tục từ bản đang chạy cho bạn **toàn bộ** những gì sẽ thay đổi khi nâng cấp: tính năng mới, thay đổi hành vi, tính năng deprecated (tr. 344). Đó chính là danh sách vùng cần thử nghiệm ở bước 3 của vòng đời nâng cấp.
- Nó cũng giúp chọn phiên bản đích: bản sớm nhất có bản sửa có thể ít thay đổi khác hơn bản mới nhất.

[về mục 2.1](../appendix-a.md#s2-1)

<a id="q2-1-3"></a>

### Câu 2.1.3 — Tính năng mới trong point release là con dao hai lưỡi

*Vì sao việc đưa tính năng mới vào point release là "con dao hai lưỡi"? Nó buộc bạn đọc release notes theo cách nào?*

Nhiều người nghĩ point release (ví dụ 8.0.21 lên 8.0.22) chỉ sửa lỗi, còn minor version mới có tính năng. Oracle thì thường đưa tính năng mới vào point release, và chúng có thể ảnh hưởng workload (tr. 344).

- **Lưỡi tốt:** bạn có tính năng mới mà không phải chờ hay làm một lần nâng cấp lớn. Thêm cột tức thì ở tr. 348 là ví dụ của kiểu lợi ích này.
- **Lưỡi xấu:** một lần nâng cấp tưởng là "chỉ vá lỗi" thực ra mang mã mới, tức rủi ro regression mới. Bạn không thể dựa vào số phiên bản để đoán mức rủi ro.

Hệ quả: phải đọc **tất cả** release notes, kể cả các thay đổi nhỏ, trước khi nâng cấp (tr. 344; bước 1 ở tr. 344 cũng nói "including any minor changes").

**Bổ sung ngoài sách:** từ giữa năm 2023, Oracle tách bản Innovation (có tính năng mới) và bản LTS (chủ yếu sửa lỗi); dòng 8.0 từ 8.0.34 được thông báo là chỉ còn sửa lỗi. Chọn nhánh LTS làm giảm phần "lưỡi xấu" này, nhưng vẫn nên đọc release notes vì bản sửa lỗi cũng có thể đổi hành vi.

[về mục 2.1](../appendix-a.md#s2-1)

---

<a id="q2-2-1"></a>

### Câu 2.2.1 — Release notes và upgrade notes

*Release notes và upgrade notes khác nhau thế nào? Tài liệu nào cho bạn danh sách "vùng cần thử nghiệm"?*

| | Release notes | Upgrade notes |
| --- | --- | --- |
| Nội dung (tr. 344) | Tính năng mới, thay đổi, tính năng deprecated, thường có danh sách bug đã sửa | Tổng quan chi tiết cách nâng cấp và các điều quan trọng cần biết trước khi tiếp tục |
| Câu hỏi trả lời | **Cái gì** sẽ khác? | **Làm sao** nâng cấp cho đúng? |

Danh sách vùng cần thử nghiệm chủ yếu đến từ **release notes**: mỗi thay đổi hành vi, tính năng mới hay deprecated là một chỗ workload của bạn có thể bị ảnh hưởng. Sách nói sau khi đọc cả hai loại, bạn sẽ hiểu rõ các mối lo và vùng cần tập trung thử nghiệm (tr. 345). Upgrade notes bổ sung các điều kiện tiên quyết và cảnh báo về chính quá trình nâng cấp, ví dụ biến cấu hình bị đổi tên hay bỏ (tr. 345).

[về mục 2.2](../appendix-a.md#s2-2)

<a id="q2-2-2"></a>

### Câu 2.2.2 — Lùi từ 8.0.26 về 8.0.25

*Bạn nâng source từ 8.0.25 lên 8.0.26. Hai giờ sau phát hiện một truy vấn quan trọng chậm hẳn. Có những cách lùi nào, mỗi cách mất gì?*

Trước hết, **không thể** đơn giản cài lại binary 8.0.25 lên datadir hiện tại: từ 8.0, kể cả point release cũng không hạ cấp được nếu không export rồi import toàn bộ dữ liệu (cảnh báo ở tr. 345).

Các lựa chọn:

1. **Restore backup chụp trước khi nâng cấp** (tr. 345). Mất mọi thay đổi ghi trong hai giờ qua, trừ khi có cách áp lại (**Bổ sung ngoài sách**: ví dụ áp lại binary log, nhưng phải kiểm tra binlog của bản mới có áp được lên bản cũ không). Thời gian khôi phục tỉ lệ với kích thước dữ liệu ([Chương 10](../chapter-10.md)).
2. **Quay về snapshot hệ thống tệp** chụp trước khi nâng cấp ([Chương 10](../chapter-10.md#myth-snapshot), tr. 273). Nhanh hơn, nhưng cũng mất thay đổi sau snapshot.
3. **Export toàn bộ dữ liệu rồi import vào một server 8.0.25** (tr. 345). Không mất dữ liệu, nhưng rất chậm với dữ liệu lớn và cần dừng ghi trong lúc làm.
4. **Không lùi, xử lý tại chỗ** (**Bổ sung ngoài sách**): tìm thay đổi optimizer liên quan trong release notes, viết lại truy vấn, thêm index hay dùng hint cho riêng truy vấn đó. Thường là lựa chọn ít tốn kém nhất khi chỉ một truy vấn bị ảnh hưởng.

Bài học của sách: vì lùi đắt như vậy, nâng cấp phải có kế hoạch từ trước (tr. 345) và phải thử kỹ (tr. 345–346). Nâng source **trước** replica như trong câu hỏi là thứ tự rủi ro: nếu nâng một replica trước, bạn đã có thể thấy regression của truy vấn đọc này với workload thật và chỉ cần depool replica đó (tr. 346).

[về mục 2.2](../appendix-a.md#s2-2)

<a id="q2-2-3"></a>

### Câu 2.2.3 — Điểm không quay lại

*Vì sao "điểm không quay lại" thường là lúc promote node đã nâng cấp làm source, chứ không phải lúc cài binary mới lên một replica?*

Khi node bản mới vẫn chỉ là replica:

- source và dữ liệu "gốc" vẫn chạy bản cũ;
- replica bản mới vẫn nhận thay đổi từ source cũ, vì server mới thường làm replica được cho server cũ (Chương 9, tr. 228);
- nếu nó có vấn đề, chỉ cần depool (tr. 346) rồi dựng lại từ backup hoặc từ một node bản cũ. Không dữ liệu nào bị mất.

Khi node bản mới được promote làm source và bắt đầu nhận ghi:

- dữ liệu mới chỉ còn được ghi trên phiên bản mới;
- các node bản cũ **không làm replica được** cho source mới hơn (tr. 228), nên không còn node bản cũ nào giữ dữ liệu cập nhật;
- muốn quay lại bản cũ thì chỉ còn restore backup hoặc export rồi import (tr. 345), cả hai đều đắt hoặc mất dữ liệu.

Vì vậy nên dồn mọi thử nghiệm vào trước lúc promote. Đây là suy luận ghép tr. 228 với tr. 345–346 (**Bổ sung ngoài sách**); phụ lục không dùng cụm "điểm không quay lại".

[về mục 2.2](../appendix-a.md#s2-2)

---

<a id="q2-3-1"></a>

### Câu 2.3.1 — Chạy tốt trên dev, chậm trên production

*Một truy vấn chạy tốt trên môi trường dev đã nâng cấp, nhưng chậm hẳn trên production sau khi nâng cấp. Giải thích theo tầng logical và physical, và vì sao môi trường dev không bắt được.*

- **Tầng logical** (parser, ngữ nghĩa): truy vấn chạy được và cho kết quả đúng ở cả dev lẫn production, nên tầng này không có vấn đề. Môi trường dev bắt tốt đúng loại lỗi này: mục tiêu chính của nó là làm lộ lỗi cú pháp hiển nhiên (tr. 345).
- **Tầng physical** (optimizer, access path, join order, cost): đây là nơi regression xảy ra. Bản mới có thể chọn plan khác: index khác, thứ tự join khác, có thêm bảng tạm hay sort.
- **Vì sao dev không bắt được** (tr. 345 và **Bổ sung ngoài sách**): dev thường không có dữ liệu cỡ production. Sách lấy ví dụ truy vấn chạm 10 hàng trên dev nhưng 10 triệu hàng trên production (tr. 345). Với 10 hàng, mọi plan đều rẻ, dữ liệu nằm hết trong buffer pool, nên một plan tồi vẫn chạy nhanh. Optimizer chọn plan dựa trên statistics và ước lượng cardinality; bảng nhỏ cho statistics khác, nên plan trên dev có khi còn khác plan trên production. Chi phí của một plan tồi chỉ hiện ra khi số hàng phải đọc đủ lớn.

Cách bắt loại regression này là các phương pháp có dữ liệu và workload thật: production mirror (tr. 345) hoặc nâng cấp một replica (tr. 346), rồi so plan bằng `EXPLAIN` và đo bằng `EXPLAIN ANALYZE` trên hai phiên bản.

[về mục 2.3](../appendix-a.md#s2-3)

<a id="q2-3-2"></a>

### Câu 2.3.2 — Replica không thử được traffic ghi

*Vì sao nâng cấp một replica trước không thử được traffic ghi, dù replica vẫn nhận mọi thay đổi qua replication?*

Sách nêu nhược điểm này nhưng không giải thích (tr. 346). Lý do (**Bổ sung ngoài sách**, dựa trên cách replication hoạt động ở [Chương 9](../chapter-09.md#how-replication-works)):

- Ứng dụng chỉ gửi câu ghi tới **source**. Replica không chạy các câu `INSERT`, `UPDATE`, `DELETE` của ứng dụng; nó nhận các **sự kiện** trong binary log của source rồi áp lại bằng luồng replication.
- Với row-based replication, sự kiện là ảnh của hàng trước và sau thay đổi, không phải câu SQL gốc. Cách bản mới **phân tích và tối ưu** câu ghi (ví dụ `UPDATE ... WHERE` dùng index nào) không được thử.
- Concurrency cũng khác: trên source, nhiều client ghi đồng thời tranh lock với nhau; trên replica, các thay đổi được áp bởi một hay vài luồng applier. Hành vi lock và latency của câu ghi dưới tải thật không được tái hiện.

Cái replica **có** thử được: read traffic thật (tr. 346), và luồng replication từ source cũ sang replica mới, điều bạn sẽ dựa vào trong suốt quá trình nâng cấp dần (Chương 9, tr. 228).

[về mục 2.3](../appendix-a.md#s2-3)

<a id="q2-3-3"></a>

### Câu 2.3.3 — Vì sao pt-upgrade cần hai hệ giống nhau

*pt-upgrade cần hai hệ giống hệt nhau, chỉ nâng cấp một. Sao không so bản mới trực tiếp với production đang chạy?*

pt-upgrade chạy cùng tập truy vấn trên hai đích rồi báo khác biệt về số hàng, dữ liệu hàng và lỗi (tr. 346). Một khác biệt chỉ có ý nghĩa nếu ta biết **nó do đâu**.

Với hai hệ giống nhau (cùng dữ liệu, cấu hình, phần cứng), chỉ khác phiên bản, thì phiên bản là biến duy nhất: mọi khác biệt quan sát được có thể quy cho nó. Đây là thiết kế thí nghiệm có nhóm đối chứng (**Bổ sung ngoài sách**).

Nếu so với production đang chạy:

- dữ liệu production thay đổi liên tục, nên cùng một `SELECT` chạy ở hai thời điểm có thể trả kết quả khác, dù phiên bản không liên quan;
- production chịu tải nền khác hẳn, nên khác biệt thời gian không phản ánh phiên bản;
- chạy thêm truy vấn thử nghiệm lên production còn tạo tải cho hệ thống đang phục vụ người dùng.

Vì vậy sách khuyên: thu truy vấn (slow log hoặc binary log), dựng hai hệ giống nhau, chỉ nâng cấp một, rồi chạy pt-upgrade trên cả hai (tr. 346).

[về mục 2.3](../appendix-a.md#s2-3)

<a id="q2-3-4"></a>

### Câu 2.3.4 — `--validate-config` đã đủ chưa?

*Bạn chỉ chạy `mysqld --validate-config` trên tệp cấu hình cũ và thấy không có lỗi. Như vậy đã đủ để kết luận cấu hình sẵn sàng chưa?*

Chưa đủ. **Kết quả đã đo** ở [TN4](../experiments/appendix-a.md#tn4) trên 8.0.42:

- `--validate-config` bắt được tùy chọn **đã bị bỏ** (`query_cache_size`, `query_cache_type` báo `unknown variable` và dừng). Đây là loại lỗi làm server không khởi động được.
- Nhưng nó **không báo gì** với tùy chọn chỉ bị deprecated (`innodb_log_file_size`, `expire_logs_days`, `default-authentication-plugin`). Cảnh báo deprecated chỉ hiện trong error log khi server khởi động thật, hoặc khi tùy chọn được dùng lúc chạy.

Ngoài ra (**Bổ sung ngoài sách**):

- `--validate-config` kiểm tra tùy chọn có hợp lệ không, không kiểm tra **giá trị mặc định mới** của các biến bạn không đặt. Sách nhắc ví dụ `sql_mode` mặc định đổi giữa các phiên bản ([Chương 5](../chapter-05.md#safety-table), tr. 122): tệp cấu hình hợp lệ nhưng hành vi ứng dụng vẫn có thể đổi.
- Kiểm tra phải chạy bằng **binary của phiên bản đích**, không phải binary hiện tại.

Vậy nên rà cấu hình gồm: kiểm tĩnh bằng binary mới, khởi động thử và đọc cảnh báo trong error log, và so các giá trị mặc định quan trọng giữa hai phiên bản. Sách gói tất cả trong một câu: phiên bản mới hay đổi tên hoặc bỏ biến, nên phải rà tệp cấu hình (tr. 345).

[về mục 2.3](../appendix-a.md#s2-3)

---

<a id="q2-4-1"></a>

### Câu 2.4.1 — `read_only` trong bước Verify target

*Bước "Verify target" dùng `read_only` khi không có hệ thống tra traffic. Nó chặn tốt loại sai nào, bỏ lọt loại nào? Vì sao thiết kế này nghiêng về "bỏ qua"?*

**Chặn tốt:** nâng cấp nhầm **source đang nhận ghi**. Source thường có `read_only = 0`, nên quy tắc "ghi được thì bỏ qua" giữ nó an toàn (tr. 346). Theo chương 5, replica nên luôn bật `read_only` (tr. 122), nên server ghi được nhiều khả năng là source.

**Bỏ lọt hoặc sai** (**Bổ sung ngoài sách**):

- `read_only = 1` **không** có nghĩa là không nhận traffic: replica trong read pool vẫn phục vụ đọc. Vì vậy sách ưu tiên một hệ thống cho biết database có đang nhận traffic không và chỉ coi `read_only` là phương án thay thế (tr. 346). Thực tế vẫn cần depool replica trước ([Chương 9](../chapter-09.md#read-pool), tr. 243–244).
- Một replica cấu hình sai (quên bật `read_only`) sẽ bị **bỏ qua**, dù lẽ ra nâng cấp được.

**Vì sao nghiêng về bỏ qua:** hai loại sai có giá rất khác nhau. Bỏ qua nhầm một replica chỉ làm chậm tiến độ; lần chạy sau, sau khi sửa cấu hình, sẽ bắt được nó. Nâng cấp nhầm source thì dừng ghi của cả ứng dụng và có thể không lùi được (tr. 345). Một quy trình chạy tự động trên hàng trăm server nên chấp nhận loại sai rẻ để tránh loại sai đắt.

**Kết quả đã đo** ([TN4](../experiments/appendix-a.md#tn4)): với `read_only = 0`, câu kiểm tra mẫu trả "SKIP"; sau `SET GLOBAL super_read_only = ON`, cả hai biến đều 1 và câu kiểm tra trả "OK to upgrade".

[về mục 2.4](../appendix-a.md#s2-4)

<a id="q2-4-2"></a>

### Câu 2.4.2 — Chạy lại playbook

*Điều gì xảy ra khi chạy lại playbook trên một server đã nâng cấp? Tính chất đó giúp gì khi nâng cấp hàng trăm server?*

Bước 1 còn kiểm xem server đã được nâng cấp chưa, nên chạy lại playbook trên server đó **không làm gì** (tr. 346). Tính chất này gọi là **idempotent** (**Bổ sung ngoài sách**: chạy nhiều lần cho cùng kết quả như chạy một lần).

Lợi ích khi nâng cấp đội máy lớn:

- Có thể chĩa runbook vào **bất kỳ** server nào, hay cả đội máy. Nó chỉ nâng cấp những node chưa nâng cấp và không nhận traffic (tr. 348).
- Nâng cấp được theo **từng đợt**: mỗi lần depool thêm vài replica rồi chạy lại cùng playbook, không cần danh sách thủ công "đã làm, chưa làm".
- Khi một lần chạy dừng giữa chừng (lỗi mạng, một node hỏng), chỉ cần chạy lại: các node đã xong bị bỏ qua.
- Giảm lỗi người: không phải đăng nhập từng máy, ít gõ nhầm hay làm nhầm server (tr. 348).

[về mục 2.4](../appendix-a.md#s2-4)

<a id="q2-4-3"></a>

### Câu 2.4.3 — Bước 7 với 5.6 lên 5.7 và 5.7 lên 8.0.28

*Bạn bật `super_read_only` trên mọi replica. So sánh bước 7 khi nâng 5.6 lên 5.7 với khi nâng 5.7 lên 8.0.28.*

- **5.6 lên 5.7:** bản mới cũ hơn 8.0, nên phải chạy `mysql_upgrade` sau khi khởi động `mysqld` (tr. 347). Vì `mysql_upgrade` phải ghi vào bảng hệ thống, còn `super_read_only` chặn cả tài khoản quản trị, phải **tạm đặt `super_read_only = OFF`** cho bước này (tr. 347), rồi bật lại sau. Đặt `read_only = OFF` cũng tắt `super_read_only` (Chương 9, chú thích 6, tr. 240), nên cần cẩn thận bật lại đúng biến.
- **5.7 lên 8.0.28:** 8.0 đã chuyển việc của `mysql_upgrade` vào lúc server khởi động, không cần chạy bước riêng (chú thích 2, tr. 347). **Bổ sung ngoài sách:** mốc chính xác là 8.0.16, nên 8.0.28 thuộc nhóm này. Bước 7 trở thành **kiểm tra error log** xem quá trình nâng cấp lúc khởi động đã hoàn tất chưa. **Kết quả đã đo** ([TN4](../experiments/appendix-a.md#tn4)) trên 8.0.42: chạy `mysql_upgrade` chỉ in thông báo deprecated; khởi động với `--upgrade=FORCE` ghi các dòng "Server upgrade from ... started" và "... completed" vào error log.

Lưu ý **Bổ sung ngoài sách:** nếu đích là 8.0.0–8.0.15 thì vẫn phải chạy `mysql_upgrade` như bản cũ.

[về mục 2.4](../appendix-a.md#s2-4)

---

<a id="q2-5-1"></a>

### Câu 2.5.1 — Vì sao INSTANT gần như không phụ thuộc số hàng

*Vì sao thời gian INSTANT gần như không phụ thuộc số hàng, còn INPLACE và COPY tỉ lệ với kích thước bảng? Những số đo nào ở TN1 cho thấy INSTANT không rebuild bảng?*

**Cơ chế** (tr. 348 nói "không cần dựng lại cả bảng"; chi tiết là **Bổ sung ngoài sách**):

- INSTANT chỉ ghi định nghĩa cột mới vào data dictionary và tăng row version của bảng. Hàng cũ không được đọc hay ghi lại; khi đọc, cột mới được điền giá trị mặc định từ metadata. Công việc không phụ thuộc số hàng.
- INPLACE (khi thao tác cần rebuild) đọc toàn bộ primary key, sắp xếp, dựng tablespace mới. COPY chép từng hàng qua tầng handler sang bảng tạm. Cả hai phải chạm mọi hàng, nên tỉ lệ với kích thước bảng.

**Bằng chứng** (**Kết quả đã đo**, [TN1](../experiments/appendix-a.md#tn1), 8.0.42, 1.000.000 hàng, 116 MiB):

| Dấu hiệu | INSTANT | INPLACE | COPY |
| --- | --- | --- | --- |
| Thời gian (trung vị 3 lần) | 0,0215 s | 6,84 s | 10,90 s |
| `TABLE_ID`, `SPACE` | **giữ nguyên** | đổi | đổi |
| Stage `alter table (...)` | **không có** | có (read PK and internal sort ...) | `copy to tmp table` |
| Dữ liệu ghi | **0,03–0,96 MiB** | 111–139 MiB | 212–221 MiB |
| `Handler_write` | 24 | 25 | 1.000.025 |
| `TOTAL_ROW_VERSIONS` | **tăng 1** | về 0 | về 0 |

`TABLE_ID` và `SPACE` không đổi chứng tỏ không có bảng hay tablespace mới; không có stage `alter table` chứng tỏ không có pha đọc và sắp xếp dữ liệu; lượng ghi gần 0 chứng tỏ không trang dữ liệu nào được viết lại. Ngoài ra, ở TN2, INSTANT trên bảng 3 hàng cũng mất khoảng 0,02 s, như trên bảng 1 triệu hàng.

[về mục 2.5](../appendix-a.md#s2-5)

<a id="q2-5-2"></a>

### Câu 2.5.2 — Migration vốn tức thì bỗng chạy lâu

*Một migration `ALTER TABLE big ADD COLUMN x INT` (không ghi `ALGORITHM`) trước nay chạy tức thì. Hôm nay nó chạy rất lâu. Nêu nguyên nhân có thể và cách phòng.*

Nguyên nhân có thể (**Bổ sung ngoài sách**, có kiểm chứng trên 8.0.42):

1. **Hết row version.** Mỗi lần thêm hay xóa cột tức thì tốn một row version; tối đa 64 (8.0.29+). Khi đã đủ 64, câu `ADD COLUMN` không ghi `ALGORITHM` **không báo lỗi** mà tự chuyển sang rebuild. **Kết quả đã đo** ([TN2](../experiments/appendix-a.md#tn2)): trên bảng 1 triệu hàng, câu đó mất 13,42 s, `TABLE_ID` đổi, `TOTAL_ROW_VERSIONS` về 0; cùng câu có `ALGORITHM=INSTANT` thì báo lỗi 4092 ngay.
2. **Thao tác hay bảng không đủ điều kiện:** cột kèm index trong cùng câu, bảng vừa được thêm `FULLTEXT` index, bảng đổi sang `ROW_FORMAT=COMPRESSED` (TN2). Khi đó MySQL cũng tự chọn INPLACE hay COPY.
3. **Chờ metadata lock**, không phải rebuild: một transaction dài đang giữ bảng ([TN3](../experiments/appendix-a.md#tn3)). Dấu hiệu là trạng thái `Waiting for table metadata lock` trong processlist.

Cách phòng:

- Ghi rõ `ALGORITHM=INSTANT` trong migration để thất bại nhanh thay vì âm thầm rebuild.
- Theo dõi `TOTAL_ROW_VERSIONS` trong `information_schema.INNODB_TABLES`; khi gần 64, lên lịch rebuild có kiểm soát, ví dụ bằng công cụ online schema change có throttling ([Chương 6](../chapter-06.md#external-tools), tr. 150).
- Đặt `lock_wait_timeout` nhỏ cho phiên chạy DDL, kiểm tra transaction dài trước khi chạy.

Liên hệ sách: đây là lý do tr. 348 nhấn mạnh "thử nghiệm kỹ" ngay cả với tính năng mang lại lợi ích rõ ràng.

[về mục 2.5](../appendix-a.md#s2-5)

<a id="q2-5-3"></a>

### Câu 2.5.3 — ALTER INSTANT treo và kéo theo SELECT

*Một câu `ALTER ... ALGORITHM=INSTANT` trên production treo, kéo theo hàng loạt `SELECT` cũng treo. Giải thích cơ chế.*

Cơ chế (**Kết quả đã đo**, [TN3](../experiments/appendix-a.md#tn3), đối chiếu Chương 3, tr. 63–64):

1. Một transaction đang mở đã đọc bảng, nên giữ **shared metadata lock** (`SHARED_READ`) tới khi `COMMIT` hay `ROLLBACK`. Metadata lock được giữ tới hết transaction (Chương 3, tr. 63).
2. `ALTER` dù là INSTANT vẫn cần **exclusive metadata lock** để đổi định nghĩa bảng. Nó chờ ở trạng thái `Waiting for table metadata lock`. Trong TN3, `metadata_locks` cho thấy ALTER giữ `SHARED_UPGRADABLE` và chờ `EXCLUSIVE` (`PENDING`).
3. Các câu `SELECT` mới cần `SHARED_READ`, nhưng phải xếp hàng **sau** yêu cầu exclusive đang chờ. Trong TN3, một `SELECT` đọc 10 hàng theo primary key mất 7,95 s, đúng bằng thời gian tới khi ALTER hết `lock_wait_timeout` (10 s) và bỏ cuộc với lỗi 1205.

Vậy INSTANT nghĩa là **không chép dữ liệu**, không có nghĩa là **không khóa**. Nếu `lock_wait_timeout` để mặc định (rất lớn), ALTER chờ lâu và mọi truy vấn trên bảng dồn lại phía sau.

Cách xử lý: tìm phiên giữ lock bằng `performance_schema.metadata_locks` (Chương 3, tr. 63–64) rồi quyết định chờ hay kết thúc nó; khi chạy DDL thì đặt `lock_wait_timeout` nhỏ, thử lại sau, và kiểm tra transaction dài trước (**Bổ sung ngoài sách**).

[về mục 2.5](../appendix-a.md#s2-5)
