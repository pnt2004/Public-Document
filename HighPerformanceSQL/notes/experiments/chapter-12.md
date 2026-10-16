# Bài tập quyết định — Chương 12: MySQL trên cloud

[← Về ghi chú chương 12](../chapter-12.md) · [Đáp án tự kiểm tra](../answers/chapter-12.md)

> **Môi trường**
>
> - **Không có thí nghiệm MySQL nào được chạy cho chương này**, và không khởi động instance MySQL nào. Các chủ đề của chương (dịch vụ managed, vCPU dùng chung, đĩa mạng, host bị dời sang máy khác, API mở rộng đĩa) cần một môi trường cloud thật; một instance trên laptop Windows không mô phỏng được chúng.
> - Đây là **bài tập quyết định trên giấy**. Số liệu đầu vào là **Ví dụ minh họa do AI xây dựng**, trừ chỗ ghi rõ là của sách. Kết quả là **phép tính** (Bổ sung ngoài sách), đã được kiểm lại bằng một script Python 3 ngắn; đó **không phải Kết quả đã đo** trên hệ thống thật.
> - Phần nào cần chạy trên cloud hay trên một VM Linux được ghi **Chưa thực thi** kèm quy trình.
> - Nhận định về dịch vụ cloud là **theo sách, thời điểm 2021**. Không dùng giá thật; chi phí được tính bằng "đơn vị giá" trừu tượng.

## Mục lục

- [Bài tập 1 — Ước lượng vCPU khi rời datacenter](#bt1)
- [Bài tập 2 — Tần suất chạy code auto-extend](#bt2)
- [Bài tập 3 — Lợi ích chi phí và khoảng an toàn khi nhắm 90%](#bt3)
- [Bài tập 4 — Runbook host reboot và script @reboot](#bt4)
- [Bài tập 5 — Managed hay VM bằng ma trận trọng số](#bt5)

---

<a id="bt1"></a>

## Bài tập 1 — Ước lượng vCPU khi rời datacenter

**Mục tiêu.** Dùng công thức của sách (tr. 319) để ước lượng số vCPU, và thấy kết quả nhạy thế nào với các giả định: p95 hay đỉnh, mức mục tiêu, đơn vị vCPU.

**Giả thuyết (Dự đoán).** Kết quả thay đổi gần tỷ lệ thuận với mức dùng đo được và tỷ lệ nghịch với mức mục tiêu; sai đơn vị (core vật lý so với hyperthread) có thể làm lệch tới hai lần.

**Chuẩn bị.**

- Công thức sách: vCPU = (Core Count × 95% Total CPU Usage) × 2 (tr. 319). Dạng tổng quát (**Bổ sung ngoài sách**): vCPU ≈ C × U ÷ U_mục_tiêu, với C là số CPU, U là mức dùng đo được, U_mục_tiêu là mức muốn chạy trên cloud.
- Mục tiêu của sách: 50% lúc bình thường, đỉnh 65–70% (tr. 320).
- Script kiểm tra phép tính (chạy được với Python 3, không cần thư viện):

```python
def vcpu(cores, usage, target):
    return cores * usage / target

for cores, usage, target in [(40, 0.30, 0.50), (40, 0.30, 0.65), (40, 0.22, 0.50), (64, 0.25, 0.50)]:
    print(cores, usage, target, round(vcpu(cores, usage, target), 1))
```

**Các bước.**

1. Lấy mức dùng CPU của server nguồn trong **30 ngày** (sách dùng 30 ngày, tr. 319). Tính cả **đỉnh** và **p95**.
2. Xác định C theo **cùng đơn vị** với vCPU của nhà cung cấp. Trên Linux, `nproc` cho số CPU logic (gồm hyperthread). **Chưa thực thi** trên server thật.
3. Áp công thức với mục tiêu 50%; làm tròn lên type gần nhất, hoặc xem custom type (chú thích 5, tr. 319: custom có thể đắt hơn).
4. Lặp lại với mục tiêu 65% để biết "sàn" nếu chấp nhận chạy nóng hơn.

**Kết quả (phép tính, Bổ sung ngoài sách).**

| Trường hợp | C | U | Mục tiêu | vCPU | Làm tròn |
| --- | --- | --- | --- | --- | --- |
| Ví dụ của sách (tr. 319), U là **đỉnh** | 40 | 30% | 50% | 24,0 | 24 |
| Chấp nhận chạy nóng hơn | 40 | 30% | 65% | 18,5 | 19–20 |
| Dùng p95 giả định thấp hơn đỉnh | 40 | 22% | 50% | 17,6 | 18 |
| Câu 2.2.1 | 64 | 25% | 50% | 32,0 | 32 |

**Kiểm tra đơn vị** (**Bổ sung ngoài sách**). Nếu "40 core" là core vật lý có hyperthreading (80 CPU logic) và mức 30% do hệ điều hành báo trên 80 CPU logic, thì tải thực là 30% × 80 = 24 CPU logic đang bận. Ở mục tiêu 50%, cần 48 CPU logic, tức **48 vCPU** nếu một vCPU là một hyperthread, gấp đôi con số 24. Ngược lại, nếu "core" trong công thức đã là CPU logic thì 24 là đúng. Phải biết rõ mình đang đếm cái gì.

**Diễn giải và đối chiếu với sách.**

- Hệ số × 2 của sách chính là ÷ 50% (tr. 319–320).
- Dùng **đỉnh** (như ví dụ của sách) cho kết quả bảo thủ: ngày cao điểm cũ sẽ chạy ở 50%. Dùng **p95** thì lúc bình thường quanh 50%, đỉnh có thể lên 65–70%, khớp với hộp cảnh báo ở tr. 320.
- vCPU dùng chung nên latency dao động hơn server riêng (tr. 319): con số này chỉ là điểm xuất phát; phải kiểm lại sau khi chuyển.

**Kiểm lại sau khi chuyển (Chưa thực thi).** Đo mức dùng CPU p50 và p95, cột `st` của `vmstat` ([Chương 4](../chapter-04.md#vmstat), tr. 93) và p99 latency truy vấn trong một tuần. Nếu p95 CPU vượt 70% kéo dài, thêm CPU (tr. 320).

**Dọn dẹp và khôi phục.** Không có thay đổi trạng thái nào.

---

<a id="bt2"></a>

## Bài tập 2 — Tần suất chạy code auto-extend

**Mục tiêu.** Làm đúng gợi ý của sách ở tr. 324: dựa trên throughput của đĩa, tính thời gian để lấp đầy phần còn trống, rồi suy ra code kiểm tra phải chạy thường xuyên tới mức nào.

**Giả thuyết (Dự đoán).** Với một ngưỡng phần trăm cố định (90%), đĩa **nhỏ và nhanh** có thời gian phản ứng ngắn tới mức cron vài phút một lần không còn an toàn.

**Chuẩn bị.**

- S: kích thước đĩa; ngưỡng p (ví dụ 0,9); T: throughput ghi tối đa của đĩa (MB/s); t_kt: chu kỳ kiểm tra; t_mr: thời gian mở rộng xong (gọi API cộng mở rộng filesystem).
- Công thức (**Bổ sung ngoài sách**):
  - t_đầy = (1 − p) × S ÷ T.
  - Điều kiện an toàn: t_kt + t_mr < t_đầy.
  - Ngưỡng tối đa theo thời gian: p_max = 1 − (t_kt + t_mr) × T ÷ S.

```python
def t_full(size_gb, p, mbps):
    return (1 - p) * size_gb * 1000 / mbps          # giây, 1 GB = 1000 MB

def p_max(size_gb, mbps, check_s, extend_s):
    return 1 - (check_s + extend_s) * mbps / 1000 / size_gb

for s in (200, 500, 1000, 4000):
    print(s, round(t_full(s, 0.9, 250)), round(p_max(s, 250, 300, 120), 3), round(p_max(s, 250, 60, 120), 3))
```

**Các bước.**

1. Lấy T là throughput **tối đa** của volume (hoặc giới hạn disk của machine type nếu thấp hơn, tr. 319). Sách nói tính "dựa trên throughput của đĩa" (tr. 324); dùng mức tối đa là giả định xấu nhất.
2. Tính t_đầy cho từng cỡ đĩa ở ngưỡng 90%.
3. Đo t_mr trên staging (**Chưa thực thi**): gọi API mở rộng, mở rộng filesystem, ghi lại tổng thời gian và latency I/O trong lúc đó.
4. Chọn t_kt sao cho t_kt + t_mr < t_đầy cho **mọi** đĩa; nếu không được, hạ ngưỡng cho đĩa đó theo p_max.

**Kết quả (phép tính, Bổ sung ngoài sách; T = 250 MB/s, t_mr giả định 120 giây).**

| S | t_đầy ở 90% | Đạt với cron 5 phút? (300 + 120 = 420 s) | p_max với cron 5 phút | p_max với cron 1 phút |
| --- | --- | --- | --- | --- |
| 200 GB | 80 s | Không | 47,5% (phần trống cần 105 GB, hơn nửa đĩa) | 77,5% (phần trống cần 45 GB) |
| 500 GB | 200 s | Không | 79,0% | 91,0% |
| 1 TB | 400 s | Không (sát) | 89,5% | 95,5% |
| 2 TB | 800 s | Có | 94,8% | 97,8% |
| 4 TB | 1.600 s | Có | 97,4% | 98,9% |

**Diễn giải và đối chiếu với sách.**

- Sách chỉ đưa nguyên tắc: code phải chạy thường xuyên hơn thời gian lấp đầy phần còn trống (tr. 324). Bảng cho thấy hệ quả: **cùng mốc 90%** nhưng khoảng an toàn từ 80 giây tới gần 27 phút.
- Cách bền hơn là đặt ngưỡng theo **thời gian còn lại tới khi đầy** rồi đổi ra phần trăm cho từng đĩa. Đĩa lớn hưởng lợi nhiều từ mức dùng cao; đĩa nhỏ mà nhanh thì không.
- Chương 2 dùng đúng phép tính này để đặt ngưỡng cảnh báo dung lượng, và nhắc rằng ngay cả khi cloud tự mở rộng volume, vẫn cần ngưỡng đủ thời gian lập kế hoạch ([Chương 2](../chapter-02.md#disk-growth), tr. 30).
- Hai cảnh báo còn lại của sách phải có trong thiết kế: **trần** để tránh volume 64 TB, và **thử API mở rộng dưới tải** vì có thể stall (tr. 324).

**Dọn dẹp và khôi phục.** Không có thay đổi trạng thái. Nếu thử bước 3 trên staging: không thu nhỏ được volume đã mở rộng ở nhiều nhà cung cấp (**Bổ sung ngoài sách**, cần kiểm tra), nên chỉ thử trên volume tạm rồi xóa cả volume.

---

<a id="bt3"></a>

## Bài tập 3 — Lợi ích chi phí và khoảng an toàn khi nhắm 90%

**Mục tiêu.** Lượng hóa câu "có thể tạo khác biệt đáng kể về chi phí dung lượng đã cấp" (tr. 324), và đặt nó cạnh cái giá là thời gian phản ứng ngắn hơn.

**Giả thuyết (Dự đoán).** Từ mục tiêu 50% lên 90%, dung lượng phải trả tiền giảm gần một nửa, nhưng thời gian tới khi đầy giảm gần chín lần.

**Chuẩn bị.**

- Dữ liệu đang dùng U = 900 GB; throughput tối đa T = 250 MB/s (**Ví dụ minh họa do AI xây dựng**).
- Với đĩa mạng, bạn trả tiền cho dung lượng **provisioned** (tr. 324). Gọi giá là g "đơn vị giá" cho mỗi GB mỗi tháng (trừu tượng; không dùng giá thật).
- Dung lượng cấp ở mức mục tiêu m: P = U ÷ m. Chi phí: g × P. Phần trống: P − U. Thời gian tới khi đầy: (P − U) ÷ T.

**Các bước.** Tính P, chi phí tương đối so với mục tiêu 50%, và t_đầy cho m = 50%, 70%, 80%, 90%.

**Kết quả (phép tính, Bổ sung ngoài sách).**

| Mục tiêu m | Dung lượng cấp P | Chi phí so với m = 50% | Phần trống | t_đầy ở 250 MB/s |
| --- | --- | --- | --- | --- |
| 50% | 1.800 GB | 100% | 900 GB | 3.600 s = 60 phút |
| 70% | 1.286 GB | 71% (giảm 29%) | 386 GB | 1.543 s ≈ 25,7 phút |
| 80% | 1.125 GB | 63% (giảm 37,5%) | 225 GB | 900 s = 15 phút |
| 90% | 1.000 GB | 56% (giảm 44%) | 100 GB | 400 s ≈ 6,7 phút |

**Diễn giải và đối chiếu với sách.**

- Lợi ích là thật: cùng dữ liệu, mục tiêu 90% trả tiền cho ít hơn 44% dung lượng so với 50%. Nhân lên cả fleet, đó là "khác biệt đáng kể" mà sách nói (tr. 324).
- Cái giá: khoảng an toàn giảm từ 60 phút xuống khoảng 6,7 phút. Vì vậy auto-extend **không phải tùy chọn** khi nhắm 90%; nó là điều kiện bắt buộc, kèm tần suất kiểm tra đúng ([bài tập 2](#bt2)).
- Đây là một ví dụ nhỏ của thông điệp cuối chương: không có bữa trưa miễn phí (tr. 324). Tiết kiệm chi phí được mua bằng tự động hóa và rủi ro vận hành.

**Dọn dẹp và khôi phục.** Không có thay đổi trạng thái.

---

<a id="bt4"></a>

## Bài tập 4 — Runbook host reboot và script @reboot

**Mục tiêu.** Biến bốn gợi ý của sách (tr. 323) thành một runbook và một phác thảo script, trong đó điều kiện "phải hỏi được một hệ thống xem máy này có nên nhận ghi không" được thực hiện rõ ràng.

**Giả thuyết (Dự đoán).** Với đĩa mạng, boot disk SSD và thiết lập ACID, chờ source quay lại cho thời gian gián đoạn ghi ngắn hơn và ít rủi ro hơn một lần unplanned failover thủ công; script tự tắt `read_only` mà không hỏi nguồn sự thật có thể gây split brain.

**Chuẩn bị.**

*Phép tính error budget* (**Bổ sung ngoài sách**). SLO availability ghi 99,95% mỗi tháng 30 ngày cho phép 43.200 phút × 0,05% = **21,6 phút** gián đoạn. Một lần reboot 5 phút (mức sách nói "thường" đạt được, tr. 323) tiêu khoảng **23%** budget tháng đó.

*Bảng quyết định chờ hay failover* (dựa trên tr. 321–323 và chương 9 tr. 239–241):

| Điều kiện | Chờ | Failover |
| --- | --- | --- |
| Đĩa dữ liệu gắn qua mạng, VM đang khởi động lại | ✔ (tr. 322) | |
| Đĩa dữ liệu local, VM sang host khác | | ✔ (dữ liệu có thể mất, tr. 321) |
| VM chưa quay lại sau ngưỡng đặt trước (ví dụ 10 phút) | | ✔ (theo quy trình chương 9, tr. 240) |
| Replica bất đồng bộ, chưa chắc có đủ dữ liệu | ✔ (không mất dữ liệu nếu ACID, ch. 9 tr. 241) | |
| Có công cụ failover tự động đã kiểm chứng, replica bán đồng bộ | | có thể ✔ |

*Cấu hình nền* (**Ví dụ minh họa do AI xây dựng, Chưa thực thi**; tệp cấu hình MySQL trên VM Linux):

```ini
[mysqld]
# Mọi lần khởi động đều bắt đầu ở chế độ chỉ đọc (chương 9, tr. 240).
super_read_only = ON
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
```

*Phác thảo script* (**Ví dụ minh họa do AI xây dựng, Chưa thực thi**). Đăng ký bằng dòng crontab `@reboot /usr/local/bin/mysql-after-reboot.sh` (tr. 323). Địa chỉ `topology.internal` là giả định, đại diện cho "nguồn sự thật" (service discovery hay công cụ quản lý topology). Thông tin đăng nhập MySQL đặt trong option file có quyền đọc hạn chế, không viết trong script.

```bash
#!/usr/bin/env bash
set -euo pipefail

HOST="$(hostname -f)"
CLUSTER="orders"

notify() {
  # Gửi tin nhắn tới kênh chat hoặc email của các đội liên quan (tr. 323).
  curl -fsS -X POST -d "text=$1" "https://chat.example.internal/hooks/db-oncall" || true
}

notify "Host ${HOST} vừa khởi động lại bất ngờ, MySQL đang khởi động ở chế độ chỉ đọc."

# 1. Chờ mysqld sẵn sàng (crash recovery có thể mất vài phút).
for _ in $(seq 1 120); do
  if mysql -N -e "SELECT 1" >/dev/null 2>&1; then break; fi
  sleep 5
done

# 2. Hỏi nguồn sự thật: source hiện tại của cluster là máy nào?
EXPECTED="$(curl -fsS "https://topology.internal/clusters/${CLUSTER}/source" || echo UNKNOWN)"

# 3. Chỉ nhận ghi lại khi nguồn sự thật xác nhận máy này vẫn là source.
if [ "${EXPECTED}" = "${HOST}" ]; then
  mysql -e "SET GLOBAL read_only = OFF;"   # cũng tắt super_read_only (chương 9, tr. 240)
  notify "Host ${HOST} đã quay lại và nhận ghi trở lại."
else
  notify "Host ${HOST} quay lại nhưng nguồn sự thật nói source là '${EXPECTED}'. Giữ chỉ đọc, cần người xem."
fi
```

**Các bước (Chưa thực thi).**

1. Trên một cặp VM thử nghiệm (source và replica, đĩa mạng), cài cấu hình và script trên.
2. Tạo sự cố giả: dừng cứng VM source qua console của nhà cung cấp, rồi khởi động lại.
3. Đo: thời gian từ lúc dừng tới lúc `@@read_only = 0`; replica có tự nối lại không (`SHOW REPLICA STATUS`, MySQL 8.0.22 trở lên).
4. Lặp lại với nhánh "đã failover": trong lúc source cũ vắng, đổi bản ghi trong nguồn sự thật sang replica; kiểm tra source cũ **giữ** chế độ chỉ đọc.
5. Kiểm tra cấu hình chặn page "host down" trong 5 phút đầu (tr. 323).

Truy vấn kiểm tra trạng thái nhanh (cú pháp MySQL 8.0):

```sql
SELECT @@hostname, @@read_only, @@super_read_only,
       @@innodb_flush_log_at_trx_commit, @@sync_binlog;
```

**Kết quả.** **Chưa thực thi**: cần VM trên cloud. Không có số liệu đo.

**Diễn giải và đối chiếu với sách.**

- Script làm đúng bốn gợi ý ở tr. 323: tự động nhận ghi lại không cần người, chạy lúc khởi động bằng `@reboot`, có kiểm tra "máy này có nên nhận ghi không", và gửi thông báo.
- `SET GLOBAL` thay vì `SET PERSIST` giữ đúng ý chương 9: source cũ khi quay lại mặc định ở `super_read_only` (tr. 240).
- Nếu không hỏi được nguồn sự thật (`UNKNOWN`), script chọn **an toàn**: giữ chỉ đọc. Mất thêm vài phút ghi còn hơn split brain.

**Dọn dẹp và khôi phục.** Xóa dòng crontab, xóa script, xóa cặp VM thử nghiệm và volume đi kèm; trả bản ghi trong nguồn sự thật về trạng thái ban đầu.

---

<a id="bt5"></a>

## Bài tập 5 — Managed hay VM bằng ma trận trọng số

**Mục tiêu.** Thực hành lời khuyên của phần Summary: đặt đánh đổi vào bối cảnh doanh nghiệp và độ trưởng thành của nó (tr. 324), bằng cách làm **trọng số** hiện rõ trên giấy.

**Giả thuyết (Dự đoán).** Cùng bảng điểm cho hai phương án, chỉ đổi trọng số theo bối cảnh là đủ để kết luận lật ngược.

**Chuẩn bị.** Tiêu chí lấy từ chương; **điểm (1–5) và trọng số (1–5) là Ví dụ minh họa do AI xây dựng**.

| Tiêu chí (trang sách) | Điểm managed | Điểm VM |
| --- | --- | --- |
| Tốc độ bắt đầu (tr. 314) | 5 | 2 |
| Ít công vận hành (tr. 313, 322) | 5 | 2 |
| Quan sát và chẩn đoán sâu (tr. 314, 318) | 2 | 5 |
| Linh hoạt topology và backup (tr. 314, 318) | 2 | 5 |
| Kiểm soát phiên bản và hành vi MySQL (tr. 314, 317) | 2 | 5 |
| Chi phí ở quy mô lớn (tr. 313, 319, 324) | 2 | 4 |

**Các bước.** Gán trọng số cho hai tổ chức giả định, nhân điểm với trọng số rồi cộng.

| Tiêu chí | Trọng số: startup 5 kỹ sư | Trọng số: công ty có đội DBA, nhiều ràng buộc compliance |
| --- | --- | --- |
| Tốc độ bắt đầu | 5 | 1 |
| Ít công vận hành | 5 | 2 |
| Quan sát và chẩn đoán sâu | 1 | 5 |
| Linh hoạt topology và backup | 1 | 4 |
| Kiểm soát phiên bản và hành vi | 2 | 4 |
| Chi phí ở quy mô lớn | 1 | 4 |

**Kết quả (phép tính, Bổ sung ngoài sách).**

| Tổ chức | Tổng điểm managed | Tổng điểm VM | Nghiêng về |
| --- | --- | --- | --- |
| Startup 5 kỹ sư | 60 | 44 | Managed |
| Công ty có đội DBA | 49 | 87 | VM |

**Diễn giải và đối chiếu với sách.**

- Con số tự nó không có ý nghĩa; điểm và trọng số đều là giả định. Giá trị của bài tập là buộc mọi người **nói ra trọng số** của mình, đúng tinh thần sách: dùng đánh đổi để dẫn dắt thảo luận (tr. 316, 324).
- Trước khi chấm điểm managed, phải qua một **điều kiện loại**: engine có tương thích đúng bản MySQL ứng dụng cần không (tr. 314). Không qua thì ma trận không cần tính.
- Trọng số đổi theo thời gian: khi công ty trưởng thành, cùng một ma trận có thể cho kết luận khác. Nên tính lại định kỳ.

**Dọn dẹp và khôi phục.** Không có thay đổi trạng thái.
