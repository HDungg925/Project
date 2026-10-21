# NT132 · Project 2 (TrueNAS + Ceph) · FILE NGỮ CẢNH CHO AI

> **v2.1 · 21/10/2026.** Dán file này ở đầu mỗi phiên AI mới, kèm một câu:
> "Người đang chat: <tên>, dùng máy <LAP-24 / LAP-16A / LAP-16B>. Hôm nay cần: …"
> Chi tiết (đặc tả UC, runbook, kiến trúc, nhật ký): `NT132-P2-TaiLieuNhom.md`. Chỉ dán phần cần cho phiên đó.
> **Nhãn trạng thái:** CHỐT (nhóm đã quyết) · ĐỀ XUẤT (có phương án, chờ nhóm duyệt) · CHƯA CHỐT (chưa có câu trả lời; không tự điền).
> **Nhãn độ tin cậy của lệnh/dữ kiện:** [đã chạy dd/mm] · [docs] (tài liệu chính thức) · [dự kiến] (chưa chạy) · [?] (chưa kiểm chứng).

---

## 0. Cách làm việc với nhóm

- **Vai trò:** trợ lý kỹ thuật cho nhóm 3 sinh viên năm 3 ngành An toàn thông tin (UIT), làm đồ án môn NT132.
- **Ngôn ngữ:** tiếng Việt. Giữ thuật ngữ tiếng Anh (OSD, bucket, quorum…), giải thích nghĩa ở lần xuất hiện đầu.
- **Trình độ:** nhóm lần đầu dùng TrueNAS, Ceph, S3. Kiến thức mạng/HĐH/CSDL học cách đây hơn 1 năm → nhắc khái niệm nền trước khi đi sâu; luôn có ví dụ cụ thể và lý do.
- **Mục tiêu học:** hiểu bản chất để demo trực tiếp và trả lời vấn đáp. Ai trong nhóm cũng có thể bị hỏi bất kỳ phần nào.
- **Lệnh:** đặt trong code block, ghi rõ **chạy ở đâu** (host Kali / ceph1 / ceph2 / ceph3 / TrueNAS UI / TrueNAS shell), giải thích từng lệnh.
- **Phản biện:** thấy lỗi kỹ thuật, mâu thuẫn trong file này, hoặc dữ kiện mới làm lý do của một quyết định CHỐT không còn đúng → nói ra **một lần**, nêu ảnh hưởng và cách kiểm. Ưu tiên chỉ ra sai sót hơn là đồng ý. Không nhắc lại khi không có gì mới.
- **Không bịa:** không khẳng định đường dẫn menu, tên flag, giá trị mặc định, mã lỗi nếu chưa có nhãn [đã chạy] hoặc chưa tra docs đúng phiên bản. Khi đó nói "cần kiểm" và chỉ cách kiểm.
- **ĐỀ XUẤT / CHƯA CHỐT:** không coi là quyết định. Được góp ý, nhưng ghi rõ là đề xuất.
- **Dữ kiện dễ lỗi thời (mục 7):** dòng nào kiểm cách đây hơn 30 ngày thì tra lại nếu có công cụ tìm kiếm. **Trạng thái (mục 4)** cũ hơn 7 ngày → hỏi xác nhận trước khi dựa vào.
- **Cập nhật file:** khi nhóm báo tiến độ hoặc ra quyết định, cuối câu trả lời đề xuất **bản thay thế** cho mục 4 (tối đa 15 dòng) và, nếu có, dòng thêm/sửa cho mục 2 hoặc mục 5. Không đề xuất nối thêm nhật ký (nhật ký nằm ở tài liệu nhóm, T12).
- **Bí mật:** không xin, không ghi lại access key, secret key, mật khẩu thật. Dùng placeholder `<ACCESS_KEY>`, `<SECRET_KEY>`.

---

## 1. Đề tài & phạm vi

- **Môn:** NT132 – Quản trị mạng và hệ thống (UIT). **Đề tài:** Project 2 – Building up Storage Services and Storage Gateway.
- **Đề gốc** dùng MinIO gateway. MinIO đã gỡ gateway (2022) → thầy đề xuất thay bằng Ceph. Project 1 (nhóm khác) dùng Ceph RBD cho VM → nhóm này nhấn **object storage (RGW), file↔object, IAM, erasure coding**.
- **Hình thức:** thuyết trình khoảng 16 slide, 30–35 phút, không đọc slide. Slide 6–10 là kiến trúc (Deployment ở slide 10), slide 11–15 là 5 use case. Mỗi UC phải chứng minh objective/benefit; **ít nhất 3 benefit phải xuất hiện trong demo**. Demo trực tiếp trên 1 laptop. Chưa có rubric chính thức. Hạn nộp / ngày báo cáo: **CHƯA CHỐT**.
- **"Storage gateway" trong đề tài (CHỐT 20/10):**
  - (a) **RGW là protocol gateway:** nhận request S3/HTTP, chuyển thành thao tác trên RADOS (tên đầy đủ: RADOS Gateway).
  - (b) **Cloud Sync là cầu nối file↔object theo lô** (rclone chạy theo lịch hoặc Run Now). Nó *không* phải gateway nằm trên đường đi của dữ liệu.
  - (c) Gateway file→object trực tuyến (NFS export bucket RGW): **CHƯA CHỐT** (đề xuất: Should).
- **Scope / Objectives / Benefits:** bản nháp ở tài liệu nhóm T1, **CHƯA CHỐT**. Không tự viết lại khi chưa được yêu cầu.

| UC | Mục tiêu | Đạt khi (tóm tắt) | Hạng (ĐỀ XUẤT, giả định hạn gần) |
|---|---|---|---|
| UC1 | File (SMB) ↔ object (S3) qua 2 Cloud Sync task | File trong `outbox/` thành object ở `from-nas/`; object ở `to-nas/` thành file trong `inbox/`; không file nào bị xóa | Must |
| UC2 | RGW tương thích S3 với nhiều client | Cùng endpoint dùng được bằng aws cli và rclone; presigned URL tải được rồi hết hạn | Must |
| UC3 | IAM theo least privilege | User chỉ-đọc bị 403 khi ghi, vẫn đọc được | Must (user + bucket policy); Should (IAM Accounts, group); Could (AssumeRole) |
| UC4 | Erasure coding 2+1 | Tắt 1 node: vẫn đọc/ghi, md5 khớp; bật lại: tự recovery về HEALTH_OK | Must; phần `min_size 3` là Should (video) |
| UC5 | Thresholds & Limits | Quota chặn ghi; rate limit trả 503; ngưỡng đầy chặn ghi rồi khôi phục | Must (quota, rate limit, ngưỡng đầy bằng video); Should (`rgw_max_put_size`) |

Chi tiết từng UC: tài liệu nhóm T6. Hạng ưu tiên và giả định về hạn: T10.

---

## 2. Quyết định hiện hành

| # | Nội dung | Trạng thái | Lý do | Xem lại khi |
|---|---|---|---|---|
| D1 | Ceph thay MinIO hoàn toàn | CHỐT 12/10 | MinIO gỡ gateway 2022; bản cộng đồng ngừng bảo trì; thầy đề xuất Ceph | Thầy đổi yêu cầu |
| D2 | Object storage = Ceph RGW, không dùng cloud công cộng | CHỐT 12/10 | Demo không phụ thuộc Internet; tránh phí; chưa chắc tạo được tài khoản AWS không cần thẻ ở VN | Thầy yêu cầu UC2 dùng S3 thật (Q2) |
| D3 | 1 laptop, host Kali + KVM/libvirt, mạng NAT `192.168.122.0/24` | CHỐT 12/10 | Thầy yêu cầu demo trên 1 máy; Linux host nhẹ hơn Windows | — |
| D4 | Dòng phiên bản: Ceph Tentacle 20.2.x, TrueNAS CE 25.10.x | CHỐT 12/10 | Squid hết hỗ trợ dự kiến 31/10/2026; TrueNAS 26 đang beta | Ghim bản vá cụ thể: CHƯA CHỐT |
| D5 | UC1 = 2 task Cloud Sync, chế độ COPY, đường dẫn tách | CHỐT 20/10 | SYNC xóa file ở đích; mỗi task chỉ chạy 1 chiều | Muốn dùng SYNC → bật versioning trước |
| D6 | Định nghĩa gateway (a)(b) ở mục 1 | CHỐT 20/10 | Cloud Sync không nằm trên đường đi dữ liệu | Khi chốt D9 |
| D7 | Kiến trúc 5 loại (mục 3.6); Deployment là slide riêng (slide 10) | CHỐT 20/10 | 4 loại có trong ghi chú của thầy; thầy xác nhận Deployment bắt buộc (Q3) | — |
| D8 | Máy demo = LAP-24 (của Dũng) | CHỐT 20/10 | Máy duy nhất có 24 GB RAM; nhóm chốt (Q7) | — |
| D9 | NFS export bucket RGW | CHƯA CHỐT (đề xuất Should) | Là gateway file→object thật; chỉ ghi tuần tự, client Linux phải mount `sync` | Sau khi các phần Must chạy ổn |
| D10 | RGW 2 instance (ceph1 + ceph2) | CHƯA CHỐT (đề xuất Should) | Bỏ SPOF của gateway | Có số đo RAM |
| D11 | Phân bổ RAM phương án A (mục 3.2) | ĐỀ XUẤT, chờ đo | Số đang là ước tính | Có số đo `free -h`, `ceph orch ps` lúc recovery |
| D12 | Phương án B = 1 VM Ceph, 3 OSD, EC 2+1 failure-domain osd, 7 GB | ĐỀ XUẤT (thay bản cũ 4 OSD / 6 GB) | Bản cũ thiếu RAM; cùng min_size với A → cùng kịch bản UC4 | Nhóm duyệt |
| D13 | OS cho node Ceph | CHƯA CHỐT (đề xuất Ubuntu Server 22.04 LTS) | Bảng OS của Tentacle liệt kê Ubuntu 22.04, không có 24.04 | Ceph thêm 24.04 vào bảng |
| D14 | Versioning bucket `nas-backup`, TLS cho RGW, Object Lock | CHƯA CHỐT (đề xuất: versioning Should, TLS Should, Object Lock Could) | Hướng bảo mật; minh họa chống mất dữ liệu | — |

---

## 3. Môi trường & kiến trúc (khung tóm tắt)

### 3.1 Máy

| Máy | Chủ máy | RAM | Ghi chú |
|---|---|---|---|
| LAP-24 | Dũng | 24 GB | Acer Nitro 5, i5 (đời CPU: CHƯA CHỐT, kiểm `lscpu`), Kali Xfce dual boot; VM nằm ở phân vùng `/data` 100 GB (ext4 + fstab: chưa kiểm) |
| LAP-16A, LAP-16B | CHƯA CHỐT | khoảng 16 GB | i5 (báo là đời 10, kiểm `lscpu`), Kali dual boot; khoảng 200 GB trống (nằm ở đâu: CHƯA CHỐT) |

Không có server, không có cloud.

### 3.2 Máy ảo: phương án A trên LAP-24 (ĐỀ XUẤT, chờ đo)

| VM | IP | RAM | vCPU | Đĩa (qcow2 cấp phát mỏng, ở `/data`) | Thành phần |
|---|---|---|---|---|---|
| truenas | .10 | 6 GB | 2 | 20 GB boot + 2×10 GB (ZFS mirror) | SMB, ZFS, Cloud Sync |
| ceph1 | .11 | 4.5 GB | 2 | 20 GB OS + 15 GB OSD | MON, MGR active, OSD.0, RGW, dashboard |
| ceph2 | .12 | 3.5 GB | 2 | 20 GB OS + 15 GB OSD | MON, MGR standby, OSD.1 |
| ceph3 | .13 | 3.5 GB | 2 | 20 GB OS + 15 GB OSD | MON, OSD.2 |
| Kali host | .1 | phần còn lại (thực tế dưới 6.5 GB) | — | — | client SMB/S3, trình duyệt, virt-manager |

- Tổng cấp phát đĩa 145 GB, lớn hơn 100 GB của `/data`; dùng thật ước tính 50–70 GB [dự kiến].
- TrueNAS 6 GB thấp hơn khuyến nghị 8 GB của TrueNAS. Chấp nhận có chủ đích vì pool chỉ 20 GB, không Apps, không VM.
- `osd_memory_target` = 2 GiB, tắt autotune.
- **Phương án B** (học / dự phòng, ĐỀ XUẤT): `truenas` 4 GB + `ceph-aio` 7 GB, 4 vCPU, 20 GB OS + 3×10 GB OSD, bootstrap với `--single-host-defaults`, EC 2+1 `crush-failure-domain=osd`, `osd_memory_target` 1.5 GiB. Không demo được việc mất cả một host.

### 3.3 Phiên bản

- Ceph 20.2.x Tentacle (bản vá ghim: CHƯA CHỐT; mới nhất 20.2.4 phát hành 19/08/2026). Dùng cephadm standalone của Tentacle, **không** `apt install cephadm` (gotcha H8).
- TrueNAS Community Edition 25.10.x (bản vá: CHƯA CHỐT). Không nâng lên TrueNAS 26 trong thời gian làm đồ án (ĐỀ XUẤT).
- OS node Ceph: CHƯA CHỐT (D13). Không GUI, swap 2 GB.

### 3.4 Mạng

- Mạng `default` của libvirt, NAT `192.168.122.0/24`. Host Kali là .1 (gateway). Public network và cluster network của Ceph dùng chung mạng này.
- Port chính: SMB 445 · TrueNAS UI 80/443 · RGW 80 (xác nhận bằng `ceph orch ls rgw`) · Ceph dashboard 8443 (đi theo MGR active; xem bằng `ceph mgr services`) · MON 3300/6789 · OSD 6800–7300 · SSH 22 (cephadm từ ceph1 sang ceph2, ceph3).
- Endpoint S3: `http://192.168.122.11` (chưa có TLS, xem D14).
- Cần Internet lúc cài (kéo container image từ quay.io). Demo không cần Internet.

### 3.5 Lưu trữ

- TrueNAS: pool ZFS mirror 2×10 GB (tên pool: CHƯA CHỐT) → dataset `smbshare` (share SMB2/3) gồm thư mục `outbox/`, `inbox/`.
- Ceph: EC profile `ec21` (k=2, m=1, failure-domain host) cho pool `default.rgw.buckets.data`. Mọi pool RGW khác là replicated size 3.
- Bucket: `nas-backup` (của user `truenas-sync`, prefix `from-nas/`, `to-nas/`); `nhom2-data` (UC2, UC3, UC4). Các tên user/bucket còn lại: xem T6, có thể đổi.

### 3.6 Kiến trúc: 5 sơ đồ (sơ đồ + giải thích ở tài liệu nhóm T5; mã Mermaid, ảnh, file giải thích ở thư mục `NT132-P2-kien-truc/`)

| Loại (slide) | Sơ đồ | Trả lời câu hỏi | Trạng thái |
|---|---|---|---|
| Business (6) | 1 use case diagram, mỗi UC có tag [CAP · B]; 3 actor; Quản trị viên nối UC1, UC3, UC4, UC5 | Nền tảng có những business capability nào, phục vụ ai | ĐÃ VẼ 21/10 |
| Application (7) | Component (3 tầng: client → dịch vụ → ZFS/RADOS) | Gồm thành phần phần mềm nào, nói chuyện qua giao diện nào | ĐÃ VẼ 21/10 |
| Integration (8) | Data flow, 3 luồng: UC1 PUSH · UC3 xác thực → phân quyền · UC4 đọc khi ceph3 tắt | Dữ liệu và request đi qua các thành phần thế nào | ĐÃ VẼ 21/10 |
| Infrastructure (9) | Mạng và máy chủ (số liệu theo mục 3.2, chờ đo) | Chạy trên laptop, VM, mạng nào | ĐÃ VẼ 21/10 |
| Deployment (slide 10) | Thành phần ↔ node (MON×3, MGR×2, OSD×3, RGW×1, container podman/cephadm) | Daemon nào chạy trên VM nào | ĐÃ VẼ 21/10 |

Sơ đồ dùng chung bảng màu: xanh lá = client · cam = TrueNAS · xanh dương = Ceph · tím = hạ tầng · xám nét đứt = Should/CHƯA CHỐT. Sửa sơ đồ thì sửa file `.mmd` trước, rồi chép sang T5.

### 3.7 Giới hạn & SPOF đã biết

- RGW chỉ có 1 instance trên ceph1 → tắt ceph1 là mất S3 (ảnh hưởng UC1, UC2, UC3, UC5). Demo UC4 tắt ceph3.
- TrueNAS là 1 VM. Failure domain thật là cái laptop: 3 "host" Ceph và ZFS mirror đều nằm trên cùng 1 SSD.
- EC 2+1 khi mất 1 node: dữ liệu còn nhưng không còn dự phòng.

---

## 4. Trạng thái & câu hỏi mở (cập nhật 21/10/2026; khi cập nhật thì thay cả mục này)

- **Giai đoạn:** 1 – Nền tảng. Chưa dựng lab, mọi lệnh trong tài liệu đều [dự kiến].
- **Đã có:** LAP-24 đã nâng 24 GB RAM, có phân vùng `/data` 100 GB (18/10). Thầy xác nhận 2 GB/VM chỉ là gợi ý và chấm bằng demo trực tiếp (13/10). Có ghi chú hướng dẫn thuyết trình của thầy (20/10). Thầy xác nhận Deployment Architecture bắt buộc (20/10, Q3) → kiến trúc 5 slide (6–10), tổng 16 slide. Máy demo = LAP-24 của Dũng (Q7). **5 sơ đồ kiến trúc đã vẽ xong, kèm file giải thích (21/10, mục 3.6).**
- **Việc tiếp theo:** kiểm phần cứng 3 máy (Q4 càng cần vì 8 vCPU có thể chiếm hết CPU) → kiểm `/data` → cài KVM → mỗi người dựng phương án B → nhóm duyệt Scope/Objectives/Benefits (T1) → dựng slide 6–10 từ sơ đồ.

| # | Câu hỏi | Hỏi ai | Trạng thái |
|---|---|---|---|
| Q1 | Hạn nộp / ngày báo cáo? | Thầy | CHƯA CHỐT |
| Q2 | UC2 có bắt buộc kết nối AWS S3 thật không? | Thầy | CHƯA CHỐT |
| Q3 | Deployment Architecture có bắt buộc không, đặt ở slide nào? | Thầy | CHỐT 20/10: bắt buộc; slide riêng (10), kiến trúc 6–10, tổng 16 slide |
| Q4 | Đời CPU (`lscpu`), VT-x của cả 3 máy | Nhóm | CHƯA CHỐT |
| Q5 | `/data` có phải ext4 và có trong `/etc/fstab` không? | Chủ LAP-24 | CHƯA CHỐT |
| Q6 | Khoảng 200 GB trống của 2 máy 16 GB nằm ở đâu? | Nhóm | CHƯA CHỐT |
| Q7 | Ai là TV1/TV2/TV3; chủ của LAP-16A, LAP-16B | Nhóm | Một phần CHỐT 20/10: máy demo = LAP-24 (Dũng); phần còn lại CHƯA CHỐT |
| Q8 | Duyệt Scope/Objectives/Benefits và hạng Must/Should/Could | Nhóm | CHƯA CHỐT |

---

## 5. Gotchas: quy tắc · lý do · cách kiểm

Cột "Nguồn": docs = tài liệu chính thức; nguồn ngoài = blog/spec không chính thức; kinh nghiệm = ý kiến chuyên môn, chưa đối chiếu docs; ? = chưa kiểm chứng. Khi đã chạy thử, ghi thêm [đã chạy dd/mm].

### 5.1 TrueNAS / Cloud Sync

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| CS1 | Mỗi Cloud Sync task chỉ chạy 1 chiều. UC1 dùng 2 task COPY với đường dẫn không giao nhau: A = PUSH `smbshare/outbox` → `nas-backup/from-nas/`; B = PULL `nas-backup/to-nas/` → `smbshare/inbox`. | SYNC xóa ở đích những file không có ở nguồn; 2 task SYNC trên cùng cặp đường dẫn sẽ xóa dữ liệu của nhau. | Mở lại task, đọc Direction và Transfer Mode trước khi Run Now; thử với file mẫu. | docs TrueNAS 25.10 |
| CS2 | Luôn chọn Direction tường minh khi tạo task. | Giá trị mặc định là PULL. | Xem lại task sau khi lưu. | docs TrueNAS 25.10 |
| CS3 | Chỉ dùng SYNC khi bucket đích đã bật versioning. | Có versioning thì lệnh xóa chỉ tạo delete marker, bản cũ vẫn lấy lại được. | `aws s3api get-bucket-versioning --bucket nas-backup` | kinh nghiệm |
| CS4 | Cloud Credential dùng provider "Amazon S3" và đặt Endpoint URL (Advanced Settings) = `http://192.168.122.11`. Nếu Verify lỗi, thử đặt Region. | RGW nói giao thức S3; TrueNAS mặc định trỏ về AWS. | Nút Verify Credential; log của task. | docs TrueNAS (endpoint); ? (region) |
| CS5 | Không bật Apps trên TrueNAS; giữ RAM VM ở 6 GB trừ khi đo thấy thiếu. | 6 GB đã thấp hơn khuyến nghị 8 GB; Apps tốn thêm RAM. | Dashboard bộ nhớ của TrueNAS; `free -h` trên host. | docs (FAQ 8 GB) + kinh nghiệm |
| CS6 | Share SMB chỉ dùng SMB2/3, không bật SMB1. | SMBv1 có lỗ hổng nghiêm trọng (EternalBlue/WannaCry). | Cấu hình dịch vụ SMB trên TrueNAS; `smbclient -m SMB2 …` kết nối được. | kinh nghiệm |

### 5.2 S3 client (aws cli, boto3, rclone)

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| S1 | Nếu PutObject lỗi `MissingContentLength` hoặc 400 lạ: đặt `request_checksum_calculation` và `response_checksum_validation` = `when_required` (trong profile, biến môi trường, hoặc config của boto3). | aws cli/boto3 bản mới (từ boto3 1.36) bật checksum mặc định; RGW có thể chưa hỗ trợ. | Chạy lại lệnh sau khi đặt; `aws --version`. | nguồn ngoài (spec OpenStack 2026.1); Tentacle đã hỗ trợ chưa: ? |
| S2 | Muốn PUT đơn lẻ (không multipart) thì dùng `aws s3api put-object`, không dùng `aws s3 cp`. | `aws s3 cp` tự chuyển sang multipart khi file lớn hơn ngưỡng (mặc định 8 MB). | `aws s3 cp … --debug`: có `CreateMultipartUpload` là multipart. | kinh nghiệm |
| S3 | Kiểm toàn vẹn bằng `md5sum` file gốc so với file tải về, không so ETag. | ETag của object multipart không phải md5 (có hậu tố `-N`). | `aws s3api head-object …` xem ETag có dấu `-`. | kinh nghiệm |
| S4 | Profile dùng region `us-east-1` (hoặc đúng `api_name` của zonegroup) và path-style. | Region khác có thể làm tạo bucket lỗi `InvalidLocationConstraint`; endpoint là IP nên không dùng virtual-host style được. | `radosgw-admin zonegroup get` xem `api_name`. | kinh nghiệm |
| S5 | Presigned URL: ai có URL đều dùng được cho tới khi hết hạn; đặt hạn ngắn khi demo. | URL mang sẵn chữ ký, không cần key. | Tạo URL hạn 60 s; mở lại sau 60 s → 403. | kinh nghiệm |
| S6 | Khi demo rate limit, đặt `AWS_MAX_ATTEMPTS=1`. | aws cli tự retry khi nhận 503 → demo chậm và che mất lỗi. | `--debug` thấy các lần retry. | kinh nghiệm |
| S7 | Mọi máy (host, 3 node Ceph, TrueNAS) phải đồng bộ giờ. | SigV4 từ chối request lệch giờ quá khoảng 15 phút (403 `RequestTimeTooSkewed`). | `date` trên host và từng VM; `chronyc tracking`. | kinh nghiệm |

### 5.3 RGW

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| R1 | Khi demo UC4 chỉ tắt ceph3 (hoặc ceph2), không tắt ceph1, trừ khi D10 đã xong. | RGW chỉ có 1 instance trên ceph1. | `ceph orch ps --daemon-type rgw` | kinh nghiệm |
| R2 | Rate limit: `radosgw-admin ratelimit set …` rồi `ratelimit enable …`. Giới hạn tính theo **từng** RGW instance; vượt thì trả 503. | Chỉ `set` thì chưa có hiệu lực; thêm RGW là đổi con số thực tế. | `radosgw-admin ratelimit get --ratelimit-scope=user --uid=<u>` | nguồn ngoài (blog ceph.io 2025) |
| R3 | Quota: `quota set` rồi `quota enable`. Khi demo dùng quota theo bucket. | Quota theo user cập nhật thống kê có độ trễ. | `radosgw-admin user info --uid=<u>` xem mục `bucket_quota`. | kinh nghiệm |
| R4 | Sau khi đổi `rgw_max_put_size`, kiểm giá trị đang chạy; có thể phải restart RGW. | Không chắc RGW nhận cấu hình mới khi đang chạy. | `ceph config show <tên daemon rgw> rgw_max_put_size` (tên lấy từ `ceph orch ps`). | ? |
| R5 | User thuộc IAM account mặc định không có quyền gì cho tới khi được gắn policy. | Đúng mô hình AWS IAM. | User mới chạy `aws s3 ls` → AccessDenied. | docs Ceph (accounts) |
| R6 | Viết Principal ARN đúng loại user: user thuộc account = `arn:aws:iam::RGW<17 chữ số>:user/<tên>`; user thường (tenant rỗng) = `arn:aws:iam:::user/<uid>`. | Sai ARN thì policy không khớp → AccessDenied khó hiểu. | `radosgw-admin user info --uid=<u>` xem user có thuộc account không. | docs Ceph (account ARN); kinh nghiệm (dạng tenant) |
| R7 | AssumeRole/STS (Could) cần đặt `rgw_sts_key` và bật `rgw_s3_auth_use_sts`. | Mặc định STS chưa bật. | Gọi `aws sts assume-role …` thành công. | kinh nghiệm |

### 5.4 RADOS / Erasure coding

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| E1 | EC k=2, m=1: size 3, min_size mặc định 2 (= k vì m=1). Mất 1 host vẫn đọc/ghi được nhưng **không còn dự phòng**. Ceph khuyến nghị min_size ≥ k+1. Không được nói "EC 2+1 dừng I/O khi mất 1 node". | Quy tắc mặc định: m=1 → min_size = k; m>1 → k+1. | `ceph osd pool get default.rgw.buckets.data min_size` → 2 | docs Ceph + docs Proxmox |
| E2 | Chỉ data pool của RGW dùng EC; mọi pool khác (index, meta, log, `buckets.non-ec`) phải replicated. | Pool EC không hỗ trợ omap; index và metadata của multipart dùng omap. | `ceph osd pool ls detail` (xem replicated hay erasure) | docs Ceph |
| E3 | Tạo EC data pool trước khi RGW ghi object đầu tiên (an toàn nhất: trước `ceph orch apply rgw`). Tên pool phải khớp zone. | Chưa có pool thì RGW tự tạo pool replicated; loại pool không đổi được sau khi tạo. | `radosgw-admin zone get --rgw-zone=default` xem `data_pool`; `ceph osd pool ls detail`. | kinh nghiệm |
| E4 | Đặt `min_size 3` khi đang thiếu 1 node → I/O **treo**, không báo lỗi. Phần này quay video, hoặc chỉ chiếu `ceph pg stat`; client đặt `--cli-read-timeout`. | PG không đủ mảnh để nhận I/O nên request chờ mãi. | PG không còn trạng thái `active`. | kinh nghiệm |
| E5 | Tắt node xong, chờ khoảng 20–30 s rồi mới chạy `ceph -s`. | OSD chỉ bị đánh dấu down sau thời gian heartbeat. | `ceph osd tree` | kinh nghiệm |
| E6 | Tên object đưa vào `ceph osd map` phải lấy từ `rados -p <pool> ls`. | `ceph osd map` tính vị trí cho cả tên không tồn tại. | So kết quả với `rados -p <pool> stat <tên>`. | kinh nghiệm |
| E7 | Không trộn số liệu và kịch bản của phương án A với B. B (1 VM) dùng failure-domain osd; nếu B dùng EC 2+2 thì min_size = 3, hành vi khác A. | Hai phương án tạo ra trạng thái cụm khác nhau. | `ceph osd pool get <pool> min_size`; `ceph osd erasure-code-profile get <profile>` | docs Ceph |
| E8 | Nếu tạo pool lỗi vì quá nhiều PG trên mỗi OSD: tạo pool với `pg_num` nhỏ, xem autoscale. | Với 3 OSD, mỗi OSD gánh mọi PG; mỗi pool mặc định 32 PG và RGW tạo nhiều pool. | `ceph health detail`; `ceph osd pool autoscale-status` | docs (pg_num 32); nguồn ngoài (giới hạn 250); giá trị ở Tentacle: ? |
| E9 | Ngưỡng đầy: đặt theo thứ tự nearfull < backfillfull < full, ngay trên %USE hiện tại; khôi phục ngay sau demo; làm sau cùng; không làm khi recovery chưa xong. | Sai thứ tự sinh lỗi health; quên khôi phục thì mọi UC sau bị chặn ghi; backfillfull thấp chặn recovery. | `ceph osd df`; `ceph osd dump` (các dòng `*_ratio`); `ceph health detail` | kinh nghiệm |
| E10 | Tắt `osd_memory_target_autotune` **trước khi** tạo OSD, rồi đặt `osd_memory_target` bằng số byte. | Autotune bật mặc định và có thể tính ra giá trị quá thấp trên VM nhỏ. | `ceph config show osd.0 osd_memory_target` (làm cho từng OSD) | docs Red Hat (autotune); ? (giá trị theo host đè lên) |
| E11 | Coi `osd_memory_target` là mục tiêu, không phải trần; đo RAM cả lúc recovery. | OSD dùng vượt mục tiêu khi recovery, đúng lúc demo UC4. | `ceph orch ps` (cột MEM USE) và `free -h` trong VM khi đang recovery. | kinh nghiệm |

### 5.5 KVM / host Kali

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| H1 | File qcow2 chỉ đặt trên ext4 (`/data`), không đặt trên NTFS; `/data` phải có trong `/etc/fstab`. | NTFS chậm, lỗi quyền; không tự mount thì VM không bật được. | `df -hT /data`; `findmnt /data`; `grep /data /etc/fstab` | kinh nghiệm |
| H2 | Giữ `/data` dưới 80%. | qcow2 chỉ phình chứ không tự co (trừ khi bật discard); đĩa đầy thì QEMU tạm dừng VM → cả cụm Ceph đứng. | `df -h /data`; `virsh domstate <vm> --reason` | kinh nghiệm |
| H3 | Chỉ snapshot khi VM đã tắt. Với cụm Ceph: `ceph osd set noout`, tắt, chụp và revert **cả 3 node cùng lúc**. | Snapshot có RAM tốn dung lượng bằng RAM của VM và làm giờ trong VM lùi lại; revert lẻ 1 node làm lệch trạng thái cụm. | `virsh snapshot-list <vm>`; sau khi bật: `ceph -s`, `ceph time-sync-status` | kinh nghiệm |
| H4 | Sau khi laptop sleep/resume hoặc revert snapshot: chỉnh giờ trước khi demo. | chrony mặc định chỉ nhảy giờ lúc khởi động, sau đó chỉnh rất chậm. | `chronyc tracking`; sửa bằng `sudo chronyc makestep` | kinh nghiệm |
| H5 | IP .10–.13 phải được đặt DHCP reservation theo MAC (hoặc thu hẹp dải DHCP). | Dải DHCP mặc định của mạng `default` là .2–.254, có thể cấp trùng IP. | `virsh net-dumpxml default`; `virsh net-dhcp-leases default` | kinh nghiệm |
| H6 | Dùng `virsh -c qemu:///system` (hoặc đặt `LIBVIRT_DEFAULT_URI`). | Chạy không sudo sẽ nối vào `qemu:///session`, không thấy VM nào. | `virsh uri` | kinh nghiệm |
| H7 | Không `apt full-upgrade` Kali trong 1–2 tuần trước demo. | Kali là bản rolling; cập nhật có thể làm hỏng libvirt/QEMU. | — | kinh nghiệm |
| H8 | Không cài cephadm bằng `apt` của Ubuntu; dùng cephadm standalone bản 20.2.x. | Kho Ubuntu 24.04 cung cấp Ceph Squid 19.2 (22.04 còn cũ hơn) → bootstrap ra bản cũ. | `./cephadm version`; sau bootstrap chạy `ceph versions` | docs Ceph + Ubuntu |
| H9 | Node Ceph cần Python 3, systemd, Podman hoặc Docker, chrony, LVM2; cần Internet lúc cài. | Yêu cầu của cephadm. | `cephadm check-host` | docs Ceph |

### 5.6 Demo

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| M1 | Bật VM ít nhất 10–15 phút trước giờ demo; chỉ bắt đầu khi cụm HEALTH_OK (hoặc chỉ còn WARN đã biết). | Cụm cần thời gian để peering và đồng bộ giờ. | `ceph -s` | kinh nghiệm |
| M2 | Chạy UC theo thứ tự UC1 → UC5; tắt VM truenas sau UC1 nếu RAM căng; phần ngưỡng đầy của UC5 quay video. | Giải phóng RAM trước UC4; ngưỡng đầy làm hỏng các bước sau và chặn recovery của UC4. | `free -h` trên host | kinh nghiệm |
| M3 | Mỗi UC có video dự phòng quay từ lab thật. | Đổi sang phương án B giữa buổi là không thực tế (khác máy, khác IP). | Video có trong máy demo trước buổi báo cáo | kinh nghiệm |
| M4 | Tắt chế độ sleep khi gập máy, cắm sạc khi demo. | Laptop sleep làm VM đứng và lệch giờ. | Cài đặt nguồn của Kali | kinh nghiệm |

---

## 6. Cách giải thích đã thống nhất

| Khái niệm | Cách giải thích |
|---|---|
| 3 kiểu lưu trữ | Block = ổ trống gắn vào máy; File = cây thư mục + phân quyền (SMB/NFS); Object = "quầy gửi đồ": dữ liệu + metadata + key, nằm trong bucket, gọi qua HTTP |
| S3 | Bucket, object, key. Access key = định danh; secret key = bí mật dùng để **ký** request (HMAC, SigV4), **không bao giờ được gửi qua mạng** |
| Storage gateway | "Người phiên dịch" giữa giao thức client dùng và giao thức của kho; trong đề tài có 2 nghĩa (a)(b) ở mục 1 |
| SDS | Phần mềm gom ổ đĩa của nhiều máy thường thành một kho, tự phân tán, nhân bản, sửa lỗi; scale-out |
| Ceph | RADOS = nhà kho; OSD = thủ kho từng kệ (1 OSD/ổ); MON = ban quản lý giữ sơ đồ cụm, cần **đa số** (2 trên 3) đồng ý để tránh split-brain; MGR = phòng điều hành/dashboard; RGW = quầy giao dịch S3; cephadm = công cụ cài đặt, chạy daemon trong container |
| Pool / PG / CRUSH | object → hash → PG → CRUSH → danh sách OSD; giống hash table, không cần bảng tra trung tâm; failure domain = host |
| Erasure coding | Ví dụ XOR: A=1010, B=0110, P=A⊕B=1100; mất B thì B=A⊕P. Giống RAID 5 nhưng các mảnh nằm trên nhiều máy |
| Replication vs EC | Dung lượng dùng được: replica size n → 1/n (size 3 → 1/3); EC → k/(k+m) (2+1 → 2/3; 2+2 → 1/2) |
| min_size | Số mảnh tối thiểu để PG còn nhận I/O; đánh đổi giữa tính sẵn sàng và an toàn dữ liệu |
| ZFS / TrueNAS | Pool, dataset, share, Cloud Sync (rclone bên dưới); ZFS vừa là filesystem vừa là RAID, có snapshot và checksum |
| Vì sao bỏ MinIO | Gateway bị gỡ 2022 + bản cộng đồng ngừng bảo trì + Ceph khớp ghi chú "software define storage" của thầy |

**Chưa giải thích sâu:** Paxos, BlueStore, PG autoscaler, SigV4 từng bước, logic đánh giá policy (Deny/Allow), cài TrueNAS, các lệnh cephadm từng bước.

---

## 7. Dữ kiện dễ lỗi thời

| Dữ kiện | Nguồn | Kiểm ngày |
|---|---|---|
| Ceph Tentacle mới nhất 20.2.4 (19/08/2026), hỗ trợ dự kiến tới 01/06/2027; Squid 19.2.x hết hỗ trợ dự kiến 31/10/2026 | docs.ceph.com/en/latest/releases | 19/10/2026 |
| Bảng OS của Tentacle: Ubuntu 22.04 (gói hạng A, container host H), CentOS 9 (A, H), Debian 12 (C); không có Ubuntu 24.04 | docs.ceph.com (tentacle) start/os-recommendations | 20/10/2026 |
| Kho Ubuntu 24.04 cung cấp Ceph Squid 19.2.x | ubuntu.com/ceph/docs/supported-ceph-versions | 20/10/2026 |
| TrueNAS CE 25.10.x là nhánh khuyến nghị cho người dùng thường; TrueNAS 26 đang beta | truenas.com/compare; endoflife.date/truenas | 19/10/2026 |
| TrueNAS khuyến nghị tối thiểu 8 GB RAM | truenas.com/faq | 19/10/2026 |
| Cloud Sync: SYNC xóa ở đích file không có ở nguồn; Direction mặc định PULL; có Remote Encryption (rclone crypt) | truenas.com/docs/scale/25.10 (Cloud Sync screens) | 19/10/2026 |
| RGW Cloud Transition (lifecycle → storage class `cloud-s3`); Tentacle có Cloud Restore (cần `retain_head_object=true`), nên không còn "chỉ một chiều" | docs.ceph.com radosgw/cloud-transition, cloud-restore | 19/10/2026 |
| IAM Accounts có từ Squid 19.2.0; account ID = "RGW" + 17 chữ số; user của account bắt đầu không có quyền nào | docs.ceph.com/en/squid/radosgw/account | 20/10/2026 |
| NFS export bucket RGW: chỉ ghi tuần tự cả file, không đổi tên thư mục, không có link, client phải mount `sync`, Windows không upload được | docs.ceph.com/en/squid/radosgw/nfs | 19/10/2026 |
| boto3 từ 1.36 bật checksum mặc định; RGW có thể lỗi; cách tránh: `when_required` | specs.openstack.org (glance, 2026.1) | 19/10/2026 |
| Rate limit RGW tính theo từng instance; vượt ngưỡng trả HTTP 503 | ceph.io blog (2025) rgw-rate-limiting | 19/10/2026 |
| MinIO gỡ gateway từ `RELEASE.2022-10-29T06-21-33Z`; bản cộng đồng chuyển maintenance 12/2025, archive 02/2026 | min.io blog; vonng.com; itsfoss.com | 13/10/2026 (phiên trước, chưa kiểm lại) |
| AWS từ 16/09/2026: "phần lớn khách mới không cần thẻ", 100 USD credit, triển khai dần | aws.amazon.com blog (getting-started) | 19/10/2026 |

---

## 8. Tài liệu nhóm chứa gì (`NT132-P2-TaiLieuNhom.md`)

T1 Ngữ cảnh, Scope, Objectives, Benefits (bản nháp) · T2 Business capabilities & bảng truy vết · T3 Khung 16 slide · T4 Bối cảnh môn & đề gốc · T5 Kiến trúc (5 sơ đồ Mermaid + cách đọc) · T6 Đặc tả UC1–UC5 · T7 Runbook dựng lab · T8 Kịch bản demo & đánh giá thời lượng · T9 Câu hỏi vấn đáp · T10 Nhóm, phân công, hạng ưu tiên, lộ trình · T11 Rủi ro · T12 Nhật ký · T13 Tham khảo.
Làm UC nào thì dán thêm mục T6 của UC đó; đang dựng lab thì dán thêm T7.