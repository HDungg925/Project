# NT132 · Project 2 (Ceph: block · file · object) · FILE NGỮ CẢNH CHO AI

> **v3.1 · 04/11/2026** (v3.1: đổi tên B1, sửa mốc MinIO; v3.0 cập nhật theo D15–D20: bỏ TrueNAS, chỉ dùng Ceph cho cả block, file, object; bản trước: v2.1 · 21/10/2026). Dán file này ở đầu mỗi phiên AI mới, kèm một câu:
> "Người đang chat: <tên>, dùng máy <LAP-24 / LAP-16A / LAP-16B>. Hôm nay cần: …"
> Chi tiết (đặc tả UC, runbook, kiến trúc, nhật ký): `NT132-P2-TaiLieuNhom.md`. Chỉ dán phần cần cho phiên đó.
> **Nhãn trạng thái:** CHỐT (nhóm đã quyết) · ĐỀ XUẤT (có phương án, chờ nhóm duyệt) · CHƯA CHỐT (chưa có câu trả lời; không tự điền).
> **Nhãn độ tin cậy của lệnh/dữ kiện:** [đã chạy dd/mm] · [docs] (tài liệu chính thức) · [dự kiến] (chưa chạy) · [?] (chưa kiểm chứng).

---

## 0. Cách làm việc với nhóm

- **Vai trò:** trợ lý kỹ thuật cho nhóm 3 sinh viên năm 3 ngành An toàn thông tin (UIT), làm đồ án môn NT132.
- **Ngôn ngữ:** tiếng Việt. Giữ thuật ngữ tiếng Anh (OSD, bucket, quorum…), giải thích nghĩa ở lần xuất hiện đầu.
- **Trình độ:** nhóm lần đầu dùng Ceph (RADOS, RBD, CephFS, RGW) và S3. Kiến thức mạng/HĐH/CSDL học cách đây hơn 1 năm → nhắc khái niệm nền trước khi đi sâu; luôn có ví dụ cụ thể và lý do.
- **Mục tiêu học:** hiểu bản chất để demo trực tiếp và trả lời vấn đáp. Ai trong nhóm cũng có thể bị hỏi bất kỳ phần nào.
- **Lệnh:** đặt trong code block, ghi rõ **chạy ở đâu** (host Kali / ceph1 / ceph2 / ceph3 / ceph4 / trong `cephadm shell` / Ceph dashboard), giải thích từng lệnh.
- **Phản biện:** thấy lỗi kỹ thuật, mâu thuẫn trong file này, hoặc dữ kiện mới làm lý do của một quyết định CHỐT không còn đúng → nói ra **một lần**, nêu ảnh hưởng và cách kiểm. Ưu tiên chỉ ra sai sót hơn là đồng ý. Không nhắc lại khi không có gì mới.
- **Không bịa:** không khẳng định đường dẫn menu, tên flag, giá trị mặc định, mã lỗi nếu chưa có nhãn [đã chạy] hoặc chưa tra docs đúng phiên bản. Khi đó nói "cần kiểm" và chỉ cách kiểm.
- **ĐỀ XUẤT / CHƯA CHỐT:** không coi là quyết định. Được góp ý, nhưng ghi rõ là đề xuất.
- **Dữ kiện dễ lỗi thời (mục 7):** dòng nào kiểm cách đây hơn 30 ngày thì tra lại nếu có công cụ tìm kiếm. **Trạng thái (mục 4)** cũ hơn 7 ngày → hỏi xác nhận trước khi dựa vào.
- **Cập nhật file:** khi nhóm báo tiến độ hoặc ra quyết định, cuối câu trả lời đề xuất **bản thay thế** cho mục 4 (tối đa 15 dòng) và, nếu có, dòng thêm/sửa cho mục 2 hoặc mục 5. Không đề xuất nối thêm nhật ký (nhật ký nằm ở tài liệu nhóm, T12).
- **Bí mật:** không xin, không ghi lại access key, secret key, mật khẩu thật. Dùng placeholder `<ACCESS_KEY>`, `<SECRET_KEY>`.

---

## 1. Đề tài & phạm vi

- **Môn:** NT132 – Quản trị mạng và hệ thống (UIT). **Đề tài:** Project 2 – Building up Storage Services and Storage Gateway.
- **Đề gốc** dùng TrueNAS (NAS) + MinIO gateway. MinIO đã gỡ gateway (2022) → thầy đề xuất thay bằng Ceph (07/10). **28/10 thầy xác nhận: chỉ dùng Ceph, không cần TrueNAS** (D15). Ceph tự cung cấp cả 3 kiểu lưu trữ trên một cụm: **block** (RBD), **file** (CephFS, chia sẻ qua SMB), **object** (RGW, S3). Thầy góp ý thêm cho demo: block / file / object; thêm 1 node; xóa 1 node.
- **Trọng tâm:** một cụm Ceph phục vụ đủ block, file, object (unified storage); object (RGW), IAM, erasure coding; mở rộng cụm bằng cách thêm máy (scale-out). Project 1 (nhóm khác) dùng Ceph RBD làm ổ đĩa cho VM (Proxmox) → UC2 (block) của nhóm chỉ làm **mức ngắn**, đủ chứng minh "một cụm, ba kiểu".
- **Hình thức:** thuyết trình 30–35 phút, không đọc slide. Slide 6–10 là kiến trúc (Deployment ở slide 10), slide 11–16 là 6 use case. **Tổng 17 slide** (Q9 CHỐT 04/11: giữ 17, không gộp). Mỗi UC phải chứng minh objective/benefit; **ít nhất 3 benefit phải xuất hiện trong demo**. Demo trực tiếp trên 1 laptop. Chưa có rubric chính thức. Hạn nộp / ngày báo cáo: **CHƯA CHỐT**.
- **"Storage gateway" trong đề tài (ĐỀ XUẤT, D17; thay D6 đã hủy):** gateway = thành phần dịch giao thức mà client dùng sang thao tác trên RADOS.
  - (a) **RGW** (RADOS Gateway): S3/HTTP → RADOS. Dùng ở UC3–UC6.
  - (b) **Samba do module `smb` của Ceph triển khai:** SMB → CephFS → RADOS. Dùng ở UC1.
  - RBD (UC2) **không có daemon gateway**: thư viện phía client (librbd, hoặc krbd trong kernel) tự chia ổ đĩa thành các object RADOS. Gateway cho block (NVMe-oF, iSCSI): ngoài phạm vi.
  - Không còn: Cloud Sync (bỏ cùng TrueNAS, D15); NFS export bucket RGW (D9 đã hủy, UC1 bỏ phần file↔object).
- **Scope / Objectives / Benefits:** bản nháp ở tài liệu nhóm T1, **CHƯA CHỐT**. Bản nháp đã viết lại theo D15 (04/11); nhóm duyệt ở Q8.
- **Đổi số UC (03/11, D16):** UC2 cũ → **UC3** · UC3 cũ → **UC4** · UC4 cũ → **UC5** · UC5 cũ → **UC6**. UC1 viết lại (SMB trên CephFS thay cho Cloud Sync). UC2 mới = block. Đối chiếu đề gốc: UC1 ↔ UC1 gốc (SMB/CIFS) · UC3 ↔ UC2 gốc (S3) · UC4–UC6 ↔ UC3–UC5 gốc · UC2 (block) là phần thêm theo góp ý 28/10. Các dòng nhật ký (tài liệu nhóm T12) trước 03/11 dùng **số cũ**.

| UC | Mục tiêu | Đạt khi (tóm tắt) | Hạng (ĐỀ XUẤT, giả định hạn gần) |
|---|---|---|---|
| UC1 | **File:** share SMB do Ceph cung cấp (CephFS + module `smb`), thay vai trò NAS của TrueNAS | Từ host Kali, `smbclient` (SMB2/3) ghi rồi đọc lại file trên share, md5 khớp; `ceph fs status` thấy MDS active; dung lượng pool data của CephFS tăng (`ceph df`) — ĐỀ XUẤT, chi tiết ở T6 | Must |
| UC2 | **Block:** 1 image RBD trên cùng cụm (mức ngắn) | Tạo image, map vào host Kali (client), format ext4, mount, ghi rồi đọc file; `rbd ls`, `rbd info` thấy image — ĐỀ XUẤT, chi tiết ở T6 | Must (mức ngắn) |
| UC3 | **Object:** RGW tương thích S3 với nhiều client (UC2 cũ) | Cùng endpoint dùng được bằng aws cli và rclone; presigned URL tải được rồi hết hạn | Must |
| UC4 | IAM theo least privilege (UC3 cũ) | User chỉ-đọc bị 403 khi ghi, vẫn đọc được | Must (user + bucket policy); Should (IAM Accounts, group); Could (AssumeRole) |
| UC5 | Erasure coding 2+1, node chết, thêm node (UC4 cũ, mở rộng theo góp ý 28/10) | Tắt 1 node ("xóa 1 node" của thầy, D18): vẫn đọc/ghi, md5 khớp; bật lại: tự recovery về HEALTH_OK. Thêm `ceph4`: dữ liệu tự dàn sang OSD mới (backfill), tổng dung lượng tăng (`ceph osd df`, `ceph df`) | Must (tắt/bật node; thêm node, phần backfill lâu thì quay video); `min_size 3` là Should (video) |
| UC6 | Thresholds & Limits (UC5 cũ) | Quota chặn ghi; rate limit trả 503; ngưỡng đầy chặn ghi rồi khôi phục | Must (quota, rate limit, ngưỡng đầy bằng video); Should (`rgw_max_put_size`) |

Chi tiết từng UC: tài liệu nhóm T6. Hạng ưu tiên và giả định về hạn: T10.

---

## 2. Quyết định hiện hành

| # | Nội dung | Trạng thái | Lý do | Xem lại khi |
|---|---|---|---|---|
| D1 | Ceph thay MinIO hoàn toàn | CHỐT 12/10 | MinIO gỡ gateway 2022; bản cộng đồng ngừng bảo trì; thầy đề xuất Ceph | Thầy đổi yêu cầu |
| D2 | Object storage = Ceph RGW, không dùng cloud công cộng | CHỐT 12/10 | Demo không phụ thuộc Internet; tránh phí; chưa chắc tạo được tài khoản AWS không cần thẻ ở VN | Thầy yêu cầu UC3 (object) dùng S3 thật (Q2) |
| D3 | 1 laptop, host Kali + KVM/libvirt, mạng NAT `192.168.122.0/24` | CHỐT 12/10 | Thầy yêu cầu demo trên 1 máy; Linux host nhẹ hơn Windows | — |
| D4 | Dòng phiên bản: Ceph Tentacle 20.2.x (phần TrueNAS CE 25.10.x bỏ theo D15). **Bản vá ghim: 20.2.5** | CHỐT 12/10; bản vá CHỐT 04/11 | Squid hết hỗ trợ dự kiến 31/10/2026. Tối thiểu phải là 20.2.4 (vá CVE-2025-30156 của CephX và 3 lỗi RGW, MON); 20.2.5 là bản mới nhất, Ceph khuyến nghị mọi người nâng lên [release notes, đọc 04/11] | Có bản vá mới; 20.2.5 gặp lỗi thì lùi về 20.2.4 |
| D5 | ~~UC1 = 2 task Cloud Sync, COPY, đường dẫn tách~~ | HỦY 28/10 → D15, D16 | Không còn TrueNAS, không còn Cloud Sync | — |
| D6 | ~~Định nghĩa gateway: (a) RGW, (b) Cloud Sync~~ | HỦY 28/10 → D17 | Vế (b) Cloud Sync không còn (D15) | — |
| D7 | Kiến trúc 5 loại (mục 3.6); Deployment là slide riêng (slide 10) | CHỐT 20/10 | 4 loại có trong ghi chú của thầy; thầy xác nhận Deployment bắt buộc (Q3) | — |
| D8 | Máy demo = LAP-24 (của Dũng) | CHỐT 20/10 | Máy duy nhất có 24 GB RAM; nhóm chốt (Q7) | — |
| D9 | ~~NFS export bucket RGW~~ | HỦY 03/11 | Nhóm chọn UC1 = SMB trên CephFS, bỏ phần file↔object (D16) | — |
| D10 | RGW 2 instance (ceph1 + ceph2) | CHƯA CHỐT (đề xuất Should) | Bỏ SPOF của gateway | Có số đo RAM |
| D11 | Phân bổ RAM phương án A (mục 3.2) | ĐỀ XUẤT, chờ đo (tính lại 04/11: 4 VM Ceph, tổng 16 GB, 8 vCPU; bỏ truenas, thêm MDS, Samba, `ceph4`) | Số đang là ước tính thô; phần cộng thêm cho MDS, Samba chưa có nguồn | Có số đo `free -h`, `ceph orch ps` lúc recovery |
| D12 | ~~Phương án B = 1 VM `ceph-aio`, 3 OSD, EC 2+1 failure-domain osd (RAM 9 GB)~~ | HỦY 04/11 → D20 | Không dựng B; cấu hình B giữ ở mục 3.2, tài liệu nhóm T7.21 để tham khảo | — |
| D13 | OS cho node Ceph | CHƯA CHỐT (đề xuất Ubuntu Server 22.04 LTS) | Bảng OS của Tentacle liệt kê Ubuntu 22.04, không có 24.04 | Ceph thêm 24.04 vào bảng |
| D14 | Versioning bucket (bucket của UC3; `nas-backup` không còn), TLS cho RGW, Object Lock | CHƯA CHỐT (đề xuất: versioning Should, TLS Should, Object Lock Could) | Hướng bảo mật; minh họa giảm rủi ro mất dữ liệu | — |
| D15 | **Bỏ TrueNAS, chỉ dùng Ceph.** Bỏ VM truenas, ZFS, Cloud Sync. Phần file (NAS) do CephFS + module `smb` của Ceph cung cấp | CHỐT 28/10 (thầy); nhóm áp dụng 03/11 | Thầy xác nhận 28/10 (Dũng hỏi riêng): "chỉ cần Ceph thôi, không cần TrueNAS". Lần hỏi 07/10 nhóm hiểu là chỉ thay MinIO. Ceph cung cấp được cả block, file, object | Thầy đổi yêu cầu |
| D16 | **6 UC:** UC1 file (SMB trên CephFS, không có phần file↔object) · UC2 block (RBD, mức ngắn) · UC3 object · UC4 IAM · UC5 EC + node chết + thêm node · UC6 Thresholds & Limits | CHỐT 03/11 | Góp ý demo của thầy 28/10 (block / file / object, thêm / xóa node); giữ đủ 5 UC của đề gốc, thêm UC2 block | Thầy góp ý lại; tổng số slide vượt thời lượng (Q9) |
| D17 | Định nghĩa "storage gateway" mới: (a) RGW, (b) Samba của module `smb`; RBD không qua gateway (mục 1) | ĐỀ XUẤT (thay D6) | Cloud Sync đã bỏ; RGW và Samba đều dịch giao thức client sang RADOS | Nhóm duyệt |
| D18 | **"Thêm 1 node"** = thêm VM `ceph4` vào cụm 3 node, Ceph tự backfill. **"Xóa 1 node"** = kịch bản node chết có sẵn (`virsh destroy`); không làm gỡ có kế hoạch (drain) | CHỐT 03/11. Thứ tự trong UC5: CHỐT 04/11, thêm `ceph4` trước, node chết sau (T6). Vai trò, RAM của `ceph4`: ĐỀ XUẤT (mục 3.2) | EC 2+1 failure domain host và pool replicated size 3 cần ≥ 3 host → thêm phải là host thứ 4. Nhóm chọn (b) cho "xóa 1 node" | Thầy yêu cầu gỡ có kế hoạch (drain); đo thấy không đủ RAM cho `ceph4` |
| D19 | **Không cài monitoring stack** (Prometheus, Grafana, Alertmanager, node-exporter): bootstrap với `--skip-monitoring-stack`. Dashboard của MGR vẫn dùng để quan sát lúc cài (T7) và lúc demo | CHỐT 04/11 | Giữ lề RAM của host (chỉ khoảng 2.5 GiB khi đủ 4 VM); ceph4 chỉ cần image Ceph (N2). Mất các biểu đồ hiệu năng trên dashboard | Đo thấy dư RAM; muốn có biểu đồ backfill / recovery cho UC5 |
| D20 | **Không dựng phương án B.** Thử module `smb` ngay trên cụm A, sau khi có OSD (F1). Dự phòng cho buổi báo cáo là video (M3) và snapshot (tài liệu nhóm T7.19) | CHỐT 04/11 | Giả định còn khoảng 2 tuần (T10.3): thử `smb` trên A cho cùng thông tin mà đỡ khoảng nửa ngày; B không demo được mất hay thêm cả một host | Cụm A hỏng không sửa được mà còn đủ thời gian |

---

## 3. Môi trường & kiến trúc (khung tóm tắt)

### 3.1 Máy

| Máy | Chủ máy | RAM | Ghi chú |
|---|---|---|---|
| LAP-24 | Dũng | 24 GB (Kali thấy 23 GiB; swap 3.7 GiB) [đã chạy 04/11] | Acer Nitro 5, **i5-12450H** (Intel thế hệ 12): `lscpu` báo 8 nhân, **12 luồng**, có VT-x [đã chạy 04/11]. Kali Xfce dual boot. `/data` = `/dev/nvme1n1p6`, **ext4, 98 GB, có trong `/etc/fstab`** (theo UUID); hiện có 26 GB (`libvirt` 21 GB, `iso` 4.2 GB) [đã chạy 04/11]; nhóm coi như `/data` trống khi dựng lab (Q10) |
| LAP-16A, LAP-16B | CHƯA CHỐT | khoảng 16 GB | i5 (báo là đời 10, kiểm `lscpu`), Kali dual boot; khoảng 200 GB trống (nằm ở đâu: CHƯA CHỐT) |

Không có server, không có cloud. LAP-24 đã kiểm 04/11; LAP-16A, LAP-16B không cần kiểm nữa vì không dựng phương án B (D20). Lệnh kiểm: tài liệu nhóm T7.1, T7.2. Ghi chú `lscpu` của LAP-24: 8 nhân nhưng 12 luồng (không phải 8 × 2 = 16) vì CPU dạng lai (hybrid): một số nhân chạy 2 luồng, số còn lại 1 luồng [?: theo thông số Intel, chưa tra lại]. Khi tính vCPU, dùng số luồng (dòng `CPU(s)`).

### 3.2 Máy ảo: phương án A trên LAP-24 (ĐỀ XUẤT, chờ đo; tính lại 04/11 theo D15–D18)

| VM | IP | RAM | vCPU | Đĩa (qcow2 cấp phát mỏng, ở `/data`) | Thành phần |
|---|---|---|---|---|---|
| ceph1 | .11 | 5 GB | 2 | 20 GB OS + 15 GB OSD | MON, MGR active + dashboard, OSD.0, **RGW** (gateway object), MDS standby |
| ceph2 | .12 | 4.5 GB | 2 | 20 GB OS + 15 GB OSD | MON, MGR standby, OSD.1, **MDS active** (CephFS), **Samba** (gateway file, do module `smb` triển khai) |
| ceph3 | .13 | 3.5 GB | 2 | 20 GB OS + 15 GB OSD | MON, OSD.2. Giữ ít vai nhất → node để tắt khi demo UC5 |
| ceph4 | .14 | 3 GB | 2 | 20 GB OS + 15 GB OSD | OSD.3. Chỉ tham gia cụm ở UC5 ("thêm 1 node", D18). Không có MON (giữ số MON lẻ) |
| Kali host | .1 | phần còn lại: 23 GiB (Kali thấy) − 16 = khoảng 7 GiB. Lúc đo (đang mở Chrome 6 tab), host dùng 4.4 GiB [đã chạy 04/11] → lề khoảng 2.5 GiB, chưa trừ phần QEMU tốn thêm cho mỗi VM [?] | — | — | client SMB (`smbclient`), S3 (aws cli, rclone, boto3), RBD (`rbd map`, UC2), trình duyệt, virt-manager |

- **Cách ra con số:** lấy số của bản 21/10 rồi cộng phần cho daemon mới: ceph1 +0.5 GB (MDS standby), ceph2 +1 GB (MDS active + Samba), ceph4 3 GB (như ceph3 nhưng không có MON). Phần cộng thêm là ước lượng thô, chưa có nguồn [?]. Đo bằng `ceph orch ps` (cột MEM USE) và `free -h` trong từng VM, nhất là lúc recovery / backfill (E11).
- **Tổng:** 16 GB RAM, 8 vCPU khi đủ 4 VM (bản cũ: 17.5 GB, 8 vCPU, có truenas 6 GB).
  - **CPU:** 8 vCPU trên 12 luồng thật → còn khoảng 4 luồng cho host [đã chạy 04/11, `lscpu`]. Không cần giảm vCPU.
  - **RAM:** lề chỉ khoảng 2.5 GiB khi đủ 4 VM → ĐỀ XUẤT để ceph4 tắt tới UC5 (còn 13 GB, lề khoảng 5.5 GiB). Thứ tự trong UC5 CHỐT 04/11 (T6): thêm ceph4 trước, node chết sau → từ lúc thêm ceph4 tới hết UC6, cả 4 VM cùng chạy (ceph4 đã giữ dữ liệu nên không tắt được). Thứ tự cả buổi: T8.
- `osd_memory_target` = 2 GiB, tắt autotune (giữ như cũ).
- **MDS** (Metadata Server, giữ "mục lục" của CephFS): `mds_cache_memory_limit` mặc định 4 GiB, và docs khuyến nghị ≥ 8 GB RAM cho MDS ở mức đó [docs Squid, tra qua công cụ tìm kiếm 02/11; giá trị mặc định 4 GiB đúng cả trên Tentacle: docs, mã nguồn 04/11] → lab hạ xuống 512 MiB (ĐỀ XUẤT; lab ít file nên MDS dùng ít [dự kiến]). Chạy 2 MDS (1 active, 1 standby): filesystem mặc định muốn có 1 MDS standby, thiếu thì cụm báo `MDS_INSUFFICIENT_STANDBY` và không lên HEALTH_OK (M1) [docs Tentacle: `ceph fs set <fs> standby_count_wanted <số>`, đặt 0 thì tắt cảnh báo này].
- **Samba:** 1 instance, không clustered (không dùng CTDB). RAM của Samba và các daemon phụ của module `smb`: chưa có số [?].
- **Đĩa:** tổng cấp phát 4 × 35 = 140 GB, lớn hơn 98 GB của `/data`. Giả định `/data` trống khi dựng lab (Q10): dùng thật 50–70 GB [dự kiến] ≈ 51–71%, dưới ngưỡng 80% (khoảng 78 GB) của H2. Nếu 26 GB hiện có (`libvirt` 21 GB, `iso` 4.2 GB) vẫn còn thì có thể vượt 80%.
- **Phương án B** (không dựng, D20; D12 đã HỦY; giữ lại để tham khảo nếu cụm A hỏng): 1 VM `ceph-aio` 9 GB (bản cũ 7 GB, cộng MDS + Samba; chờ đo), 4 vCPU, 20 GB OS + 3×10 GB OSD, bootstrap với `--single-host-defaults`, EC 2+1 `crush-failure-domain=osd`, `osd_memory_target` 1.5 GiB. Không còn VM truenas (D15) nên host 16 GB còn khoảng 7 GB. Không demo được mất hay thêm cả một host; muốn mô phỏng "thêm node" thì gắn thêm 1 đĩa 10 GB và tạo OSD thứ 4 (ĐỀ XUẤT).

### 3.3 Phiên bản

- Ceph **20.2.5** Tentacle (bản vá ghim, CHỐT 04/11, D4; phát hành 02/11/2026; bản trước là 20.2.4, phát hành 19/08/2026) [release notes, đọc bản nguồn trên GitHub 04/11]. Từ 20.2.4, cụm cài mới dùng key CephX kiểu `aes256k` (vá CVE-2025-30156), client phải hỗ trợ kiểu key này (K4). Dùng cephadm standalone của Tentacle, **không** `apt install cephadm` (gotcha H8).
- **Module `smb` (mgr) của Tentacle:** dựng share SMB trên CephFS bằng container Samba; bắt buộc dùng cephadm; Samba do Ceph quản lý chỉ hỗ trợ SMB2, SMB3. Docs ghi tính năng SMB "under active development", có thể thiếu hoặc chưa ổn [docs Tentacle, tra qua công cụ tìm kiếm 02/11] → dựng thử sớm.
- OS node Ceph (gồm cả ceph4): CHƯA CHỐT (D13). Không GUI, swap 2 GB.
- Client Ceph trên host Kali (cần cho lệnh `rbd` ở UC2): kho Kali có gói `ceph-common` **20.2.4+ds-2**; kernel **7.1.5**, có module `rbd` [đã chạy 04/11, LAP-24]. Cả hai hỗ trợ key `aes256k` (K4) [docs]. Client 20.2.4 với cụm 20.2.5: cùng dòng Tentacle, dùng được [dự kiến].

### 3.4 Mạng

- Mạng `default` của libvirt, NAT `192.168.122.0/24`. Host Kali là .1 (gateway). IP VM: ceph1–ceph4 = .11–.14, đặt theo MAC (H5); .10 (truenas cũ) để trống. Public network và cluster network của Ceph dùng chung mạng này.
- Port chính: RGW 80 trên ceph1 (xác nhận bằng `ceph orch ls rgw`) · SMB 445 trên node chạy Samba (ĐỀ XUẤT ceph2) · Ceph dashboard 8443 (đi theo MGR active; xem bằng `ceph mgr services`) · MON 3300/6789 · OSD 6800–7300 (MDS, MGR cũng dùng dải này: [?], kiểm bằng `ss -tlnp` trên node) · SSH 22 (cephadm từ ceph1 sang ceph2, ceph3, ceph4).
- Endpoint: S3 `http://192.168.122.11` (chưa có TLS, xem D14) · SMB `//192.168.122.12/<tên share>` (theo vị trí Samba, ĐỀ XUẤT).
- Cần Internet lúc cài (kéo container image từ quay.io). Demo không cần Internet. **Riêng ceph4:** chỉ tham gia cụm ở UC5, lúc không có Internet → image Ceph phải có sẵn trên ceph4 từ trước. Thêm host mới thì cephadm có kéo lại image hay không: [?], kiểm khi tập bằng cách ngắt Internet rồi thêm ceph4.

### 3.5 Lưu trữ: một cụm, ba kiểu

| Kiểu (UC) | Pool | Loại pool | Chứa gì |
|---|---|---|---|
| Object (UC3–UC6) | `default.rgw.buckets.data` | EC profile `ec21` (k=2, m=1, failure-domain host) | Dữ liệu các bucket: `nhom2-data` (UC3, UC4, UC5) và bucket demo khác (T6) |
| Object (metadata) | Các pool RGW còn lại (index, meta, log…) | Replicated size 3 (bắt buộc, E2) | Index, metadata của RGW |
| File (UC1) | 2 pool của CephFS `cephfs`: `cephfs.cephfs.meta`, `cephfs.cephfs.data` (do `ceph fs volume create` tự đặt [docs]) | Replicated size 3 (ĐỀ XUẤT) | 1 share SMB (tên ĐỀ XUẤT `nhom2-share`) trỏ vào một thư mục / subvolume trong CephFS [?] |
| Block (UC2) | `rbd` (ĐỀ XUẤT) | Replicated size 3 (ĐỀ XUẤT) | 1 image `nhom2-disk` 1 GiB (ĐỀ XUẤT; cấp phát mỏng) |

- **Vì sao CephFS và RBD không dùng EC (ĐỀ XUẤT):** pool data mặc định của CephFS nên là replicated, EC chỉ nên làm pool data phụ [docs, chưa kiểm lại trên Tentacle]. RBD trên EC phải bật `allow_ec_overwrites` và vẫn cần một pool replicated cho metadata. Giữ EC ở một chỗ (data của RGW) để UC5 dễ giải thích.
- **Số PG (E8, sửa 04/11):** khoảng 11–12 pool (RGW khoảng 7, CephFS 2, `rbd`, `.mgr`, có thể thêm pool của module `smb` [?]). Tentacle bật autoscaler mặc định; pool tạo không ghi `pg_num` bắt đầu với 1 PG [docs] → không còn nguy cơ vượt giới hạn 250 PG/OSD như ước tính cũ. Ngược lại, 3 pool chứa dữ liệu đặt `pg_num` cố định và tắt autoscaler (EC của RGW 32, `cephfs.cephfs.data` 16, `rbd` 8; nhóm đồng ý 04/11): để UC5 thấy rõ backfill sang ceph4 và để số PG không tự đổi giữa buổi demo. Ước tính khoảng 65 PG mỗi OSD khi có 3 OSD [dự kiến].
- Bucket `nas-backup` (user `truenas-sync`) đã bỏ cùng UC1 cũ (D15). Các tên user / bucket / share / image: xem T6, có thể đổi.

### 3.6 Kiến trúc: 5 sơ đồ (sơ đồ + giải thích ở tài liệu nhóm T5; mã Mermaid, ảnh, file giải thích ở thư mục `NT132-P2-kien-truc/`)

| Loại (slide) | Sơ đồ | Trả lời câu hỏi | Trạng thái |
|---|---|---|---|
| Business (6) | Use case diagram, 6 UC, mỗi UC có tag [CAP · B]; 4 tác nhân (thêm "Máy chủ cần ổ đĩa"); Quản trị viên nối UC1, UC2, UC4, UC5, UC6 | Nền tảng có những business capability nào, phục vụ ai | VẼ LẠI 04/11 |
| Application (7) | Component, 3 tầng: client → giao diện truy cập (RGW, Samba; librbd / krbd nằm ở phía client) → RADOS (MON, OSD) + MDS | Gồm thành phần phần mềm nào, nói chuyện qua giao diện nào | VẼ LẠI 04/11 |
| Integration (8) | Data flow, 3 luồng (CHỐT 04/11): UC1 ghi file qua SMB · UC4 xác thực → phân quyền · UC5 đọc khi ceph3 tắt | Dữ liệu và request đi qua các thành phần thế nào | VẼ LẠI 04/11 (luồng 2, 3 giữ ý cũ, đổi số UC; backfill để ở slide 5) |
| Infrastructure (9) | Mạng và máy chủ: 4 VM Ceph (ceph4 = node thêm ở UC5); số liệu theo mục 3.2, chờ đo | Chạy trên laptop, VM, mạng nào | VẼ LẠI 04/11 |
| Deployment (slide 10) | Thành phần ↔ node (MON×3, MGR×2, OSD×4, RGW×1, MDS×2, Samba×1, container podman/cephadm) | Daemon nào chạy trên VM nào | VẼ LẠI 04/11 |

Bảng màu (CHỐT 04/11): **cam = gateway** (RGW, Samba; thay "cam = TrueNAS" cũ), xanh dương = lõi Ceph (MON, MGR, OSD, MDS, pool), xanh dương viền đứt = ceph4 (thêm ở UC5) · xanh lá = client · tím = hạ tầng · xám nét đứt = Should / CHƯA CHỐT. Ảnh xem trước (PNG) xuất lại 04/11. Mọi file `.mmd` ghi `layout: dagre` trong khối `config:` đầu file, để Mermaid 11 và 12 vẽ giống nhau (Mermaid 12 mặc định vẽ đường gấp khúc, xếp ô khác; tài liệu nhóm T5.0). Sửa sơ đồ thì sửa file `.mmd` trước, rồi chép sang T5.

### 3.7 Giới hạn & SPOF đã biết

- **RGW** chỉ có 1 instance trên ceph1 → tắt ceph1 là mất S3 (ảnh hưởng UC3, UC4, UC6 và phần đọc/ghi qua S3 của UC5). Hướng khắc phục: D10.
- **Samba** chỉ có 1 instance, không clustered (ĐỀ XUẤT đặt ở ceph2) → tắt ceph2 là mất SMB (UC1). MDS active cũng ở ceph2; MDS standby trên ceph1 nhận thay nên CephFS vẫn chạy [dự kiến], nhưng client SMB vẫn mất kết nối vì Samba nằm trên ceph2.
- **Demo UC5 tắt ceph3** (chỉ có MON + OSD): mất 1 trong 3 MON vẫn đủ quorum 2/3; không đụng tới RGW, Samba, MDS.
- **EC 2+1 khi mất 1 node:** dữ liệu còn nhưng không còn dự phòng. Khi cụm đã có ceph4 (4 host) và node chết đủ lâu để OSD bị đánh dấu out (`mon_osd_down_out_interval`, mặc định 10 phút [docs Tentacle]), Ceph có chỗ dựng lại mảnh bị mất trên host còn lại → có lại dự phòng mà không cần node chết quay lại [dự kiến]. Khi chỉ có 3 host: degraded cho tới khi node đó quay lại.
- **Failure domain thật là cái laptop:** 4 "host" Ceph cùng nằm trên 1 SSD. Lab mô phỏng *cơ chế* chịu lỗi, không phải *hạ tầng* chịu lỗi.

---

## 4. Trạng thái & câu hỏi mở (cập nhật 04/11/2026; khi cập nhật thì thay cả mục này)

- **Giai đoạn:** 1 – Nền tảng. Tiến độ dựng lab giữ như ngày 21/10 (lần sửa 03/11 không cập nhật phần này): chưa dựng lab, mọi lệnh trong tài liệu đều [dự kiến].
- **Đã có:** LAP-24 đã nâng 24 GB RAM, có `/data` (18/10). 2 GB/VM chỉ là gợi ý; chấm bằng demo trực tiếp (13/10). Ghi chú thuyết trình của thầy (20/10). **Q3 CHỐT 20/10:** Deployment bắt buộc, slide riêng (10), kiến trúc 6–10. Máy demo = LAP-24 (Q7). 5 sơ đồ kiến trúc + file giải thích, vẽ cho thiết kế cũ (21/10). **28/10:** thầy xác nhận bỏ TrueNAS, chỉ dùng Ceph; góp ý demo block / file / object, thêm / xóa node (D15). **03/11:** nhóm chốt 6 UC (D16) và cách làm thêm / xóa node (D18). **04/11, kiểm LAP-24:** i5-12450H, 12 luồng, có VT-x, Kali thấy 23 GiB RAM; **Q5 CHỐT:** `/data` là ext4 98 GB, có trong fstab; hiện có 26 GB, nhóm coi như trống khi dựng lab (Q10). **04/11:** chốt thứ tự trong UC5: thêm ceph4 trước, node chết sau (D18); ghim bản 20.2.5 (D4); không cài monitoring stack (D19); Kali của LAP-24 dùng được key mới (kernel 7.1.5, `ceph-common` 20.2.4; K4). Lộ trình T10 giả định báo cáo khoảng 16/11 (Q1); không dựng phương án B (D20).
- **Việc tiếp theo** (lộ trình ở tài liệu nhóm T10.4, giả định báo cáo khoảng 16/11): dọn `/data` (Q10) → dựng cụm A theo T7, thử module `smb` ngay sau khi có OSD (F1, mốc kiểm 07/11) → đo RAM theo mục 3.2 (D11) → duyệt T1 bản viết lại (Q8) → dựng slide.

| # | Câu hỏi | Hỏi ai | Trạng thái |
|---|---|---|---|
| Q1 | Hạn nộp / ngày báo cáo? | Thầy | CHƯA CHỐT (T10 tạm giả định khoảng 16/11, nhóm chọn 04/11) |
| Q2 | UC3 (object; UC2 của đề gốc) có bắt buộc kết nối AWS S3 thật không? | Thầy | CHƯA CHỐT |
| Q4 | Đời CPU (`lscpu`), số luồng, VT-x | Nhóm | LAP-24 xong 04/11 (mục 3.1); LAP-16A, LAP-16B không cần nữa (D20) |
| Q6 | Khoảng 200 GB trống của 2 máy 16 GB nằm ở đâu? | Nhóm | BỎ 04/11: không dựng phương án B (D20) |
| Q7 | Ai là TV1/TV2/TV3; chủ của LAP-16A, LAP-16B | Nhóm | Một phần CHỐT: máy demo = LAP-24 (Dũng, 20/10); Phan Hoàng Dũng (chủ LAP-24) nhận TV1 (04/11); TV2, TV3 và chủ LAP-16A, LAP-16B CHƯA CHỐT |
| Q8 | Duyệt Scope/Objectives/Benefits và hạng Must/Should/Could (T1 phải viết lại theo D15) | Nhóm | CHƯA CHỐT |
| Q9 | Có 6 UC thì giữ 17 slide hay gộp để còn 16? | Nhóm | CHỐT 04/11: giữ 17 slide (slide 11–16 là 6 UC, slide 17 kết luận) |
| Q10 | 26 GB đang có trên `/data` của LAP-24 (`libvirt` 21 GB, `iso` 4.2 GB) tính thế nào? | Chủ LAP-24 | CHỐT 04/11: coi như `/data` trống khi dựng lab (nhóm tự dọn) |

---

## 5. Gotchas: quy tắc · lý do · cách kiểm

Cột "Nguồn": docs = tài liệu chính thức; nguồn ngoài = blog/spec không chính thức; kinh nghiệm = ý kiến chuyên môn, chưa đối chiếu docs; ? = chưa kiểm chứng. Khi đã chạy thử, ghi thêm [đã chạy dd/mm].

Tiền tố mã: F = file (CephFS, MDS, SMB) · S = S3 client · R = RGW · E = RADOS / erasure coding · H = host Kali, KVM · M = demo · K = block (RBD) · N = thêm node. **CS1–CS6 (TrueNAS / Cloud Sync) đã bỏ cùng TrueNAS (D15); mã CS không dùng lại.** Mục 5.7, 5.8 là phần mới; số mục 5.2–5.6 giữ như cũ.

### 5.1 CephFS, MDS, SMB (UC1)

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| F1 | Dựng thử module `smb` thật sớm: ngay trên cụm A, sau khi có OSD (T7.10 → T7.14 → T7.15 → UC1-P2..P6), trước cả RGW (D20, không dựng phương án B). Nếu không chạy được, phương án thay (ĐỀ XUẤT): chia sẻ CephFS qua NFS (module `nfs`), hoặc mount CephFS thẳng trên host Kali. Đổi sang phương án thay thì phải chốt lại D16, vì UC1 gốc là SMB/CIFS. | Docs Tentacle ghi SMB "under active development"; module bắt buộc dùng cephadm. | `ceph mgr module ls` (có `smb`); `ceph orch ls` (có service smb); từ Kali: `smbclient -L //<ip> -U <user>` liệt kê được share | docs Tentacle (đọc bản nguồn trên GitHub 04/11: `mgr/smb`, `cephadm/services/smb`); phương án thay: kinh nghiệm |
| F2 | Hạ `mds_cache_memory_limit` xuống 512 MiB = 536870912 byte (ĐỀ XUẤT). | Mặc định 4 GiB và docs khuyến nghị ≥ 8 GB RAM cho MDS ở mức đó, trong khi ceph2 chỉ có 4.5 GB. | `ceph config get mds mds_cache_memory_limit`; `ceph orch ps --daemon-type mds` (cột MEM USE) | docs Squid (khuyến nghị RAM); docs Tentacle (mặc định 4 GiB, mã nguồn 04/11) |
| F3 | Giữ 2 MDS (1 active, 1 standby). Nếu chỉ chạy 1 MDS thì phải đặt `standby_count_wanted 0` cho filesystem. | Filesystem mặc định muốn có 1 standby; thiếu thì cảnh báo `MDS_INSUFFICIENT_STANDBY`, cụm không lên HEALTH_OK (M1). | `ceph fs status`; `ceph health detail` | docs Tentacle (`cephfs/standby`, `cephfs/health-messages`, 04/11) |
| F4 | Đặt vị trí MDS (ceph1, ceph2) và Samba (ceph2) bằng placement tường minh, không để cephadm tự chọn host. | Daemon rơi vào ceph3 thì lúc tắt ceph3 ở UC5 sẽ mất CephFS hoặc SMB, lệch kịch bản và sơ đồ. | `ceph orch ps --daemon-type mds`; `ceph orch ps ceph2`; `ceph orch ls` (cột PLACEMENT) | kinh nghiệm |
| F5 | Không mount CephFS và không `rbd map` bằng kernel client trên chính node chạy OSD; client luôn là host Kali. | Kernel client chạy chung máy với OSD có thể treo (deadlock) khi máy thiếu RAM. | — | kinh nghiệm (nhớ docs có cảnh báo, chưa tra lại) → ? |

### 5.2 S3 client (aws cli, boto3, rclone)

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| S1 | Nếu PutObject lỗi `MissingContentLength` hoặc 400 lạ: đặt `request_checksum_calculation` và `response_checksum_validation` = `when_required` (trong profile, biến môi trường, hoặc config của boto3). | aws cli/boto3 bản mới (từ boto3 1.36) bật checksum mặc định; RGW có thể chưa hỗ trợ. | Chạy lại lệnh sau khi đặt; `aws --version`. | nguồn ngoài (spec OpenStack 2026.1); Tentacle đã hỗ trợ chưa: ? |
| S2 | Muốn PUT đơn lẻ (không multipart) thì dùng `aws s3api put-object`, không dùng `aws s3 cp`. | `aws s3 cp` tự chuyển sang multipart khi file lớn hơn ngưỡng (mặc định 8 MB). | `aws s3 cp … --debug`: có `CreateMultipartUpload` là multipart. | kinh nghiệm |
| S3 | Kiểm toàn vẹn bằng `md5sum` file gốc so với file tải về, không so ETag. | ETag của object multipart không phải md5 (có hậu tố `-N`). | `aws s3api head-object …` xem ETag có dấu `-`. | kinh nghiệm |
| S4 | Profile dùng region `us-east-1` (hoặc đúng `api_name` của zonegroup) và path-style. | Region khác có thể làm tạo bucket lỗi `InvalidLocationConstraint`; endpoint là IP nên không dùng virtual-host style được. | `radosgw-admin zonegroup get` xem `api_name`. | kinh nghiệm |
| S5 | Presigned URL: ai có URL đều dùng được cho tới khi hết hạn; đặt hạn ngắn khi demo. | URL mang sẵn chữ ký, không cần key. | Tạo URL hạn 60 s; mở lại sau 60 s → 403. | kinh nghiệm |
| S6 | Khi demo rate limit, đặt `AWS_MAX_ATTEMPTS=1`. | aws cli tự retry khi nhận 503 → demo chậm và che mất lỗi. | `--debug` thấy các lần retry. | kinh nghiệm |
| S7 | Mọi máy (host Kali, 4 node Ceph) phải đồng bộ giờ. | SigV4 từ chối request lệch giờ quá khoảng 15 phút (403 `RequestTimeTooSkewed`). | `date` trên host và từng VM; `chronyc tracking`. | kinh nghiệm |

### 5.3 RGW

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| R1 | Khi demo UC5 chỉ tắt ceph3. Không tắt ceph1 (RGW; trừ khi D10 đã xong) và không tắt ceph2 (Samba, MDS active). | RGW chỉ có 1 instance trên ceph1; Samba chỉ có 1 instance trên ceph2 (mục 3.7). | `ceph orch ps --daemon-type rgw`; `ceph orch ps ceph2` | kinh nghiệm |
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
| E2 | Trong các pool của RGW, chỉ data pool dùng EC; mọi pool khác (index, meta, log, `buckets.non-ec`) phải replicated. Pool của CephFS và `rbd` cũng để replicated (ĐỀ XUẤT, mục 3.5). | Pool EC không hỗ trợ omap; index và metadata của multipart dùng omap. | `ceph osd pool ls detail` (xem replicated hay erasure) | docs Ceph |
| E3 | Tạo EC data pool trước khi RGW ghi object đầu tiên (an toàn nhất: trước `ceph orch apply rgw`). Tên pool phải khớp zone. | Chưa có pool thì RGW tự tạo pool replicated; loại pool không đổi được sau khi tạo. | `radosgw-admin zone get --rgw-zone=default` xem `data_pool`; `ceph osd pool ls detail`. | kinh nghiệm |
| E4 | Đặt `min_size 3` khi đang thiếu 1 node → I/O **treo**, không báo lỗi. Phần này quay video, hoặc chỉ chiếu `ceph pg stat`; client đặt `--cli-read-timeout`. | PG không đủ mảnh để nhận I/O nên request chờ mãi. | PG không còn trạng thái `active`. | kinh nghiệm |
| E5 | Tắt node xong, chờ khoảng 20–30 s rồi mới chạy `ceph -s`. | OSD chỉ bị đánh dấu down sau thời gian heartbeat. | `ceph osd tree` | kinh nghiệm |
| E6 | Tên object đưa vào `ceph osd map` phải lấy từ `rados -p <pool> ls`. | `ceph osd map` tính vị trí cho cả tên không tồn tại. | So kết quả với `rados -p <pool> stat <tên>`. | kinh nghiệm |
| E7 | Không trộn số liệu và kịch bản của phương án A với B. B (1 VM) dùng failure-domain osd; nếu B dùng EC 2+2 thì min_size = 3, hành vi khác A. | Hai phương án tạo ra trạng thái cụm khác nhau. | `ceph osd pool get <pool> min_size`; `ceph osd erasure-code-profile get <profile>` | docs Ceph |
| E8 | (Sửa 04/11.) Đặt `pg_num` cố định và tắt autoscaler cho 3 pool chứa dữ liệu: EC của RGW 32, `cephfs.cephfs.data` 16, `rbd` 8 (T7.11, T7.14, T7.16). Các pool khác (metadata) để autoscaler. Kiểm tổng PG mỗi OSD sau khi tạo đủ pool. | Tentacle bật autoscaler mặc định; pool tạo không ghi `pg_num` bắt đầu với 1 PG. Pool EC chỉ 1 PG thì khi thêm ceph4 có thể không dời dữ liệu sang OSD mới (UC5 yếu); autoscaler có thể đổi số PG giữa buổi demo (dữ liệu dời bất ngờ). Giới hạn `mon_max_pg_per_osd` = 250, mục tiêu `mon_target_pg_per_osd` = 100; ước tính khoảng 65 PG/OSD với 3 OSD (mục 3.5). | `ceph osd pool autoscale-status` (cột PG_NUM, AUTOSCALE); `ceph osd df` (cột PGS); `ceph health detail` | docs Tentacle (giá trị mặc định đọc trong mã nguồn nhánh `tentacle`, 04/11) |
| E9 | Ngưỡng đầy: đặt theo thứ tự nearfull < backfillfull < full, ngay trên %USE hiện tại; khôi phục ngay sau demo; làm sau cùng; không làm khi recovery hoặc backfill (thêm ceph4, UC5) chưa xong. | Sai thứ tự sinh lỗi health; quên khôi phục thì mọi UC sau bị chặn ghi; backfillfull thấp chặn cả recovery lẫn backfill. | `ceph osd df`; `ceph osd dump` (các dòng `*_ratio`); `ceph health detail` | kinh nghiệm |
| E10 | Tắt `osd_memory_target_autotune` **trước khi** tạo OSD, rồi đặt `osd_memory_target` bằng số byte. | Autotune bật mặc định và có thể tính ra giá trị quá thấp trên VM nhỏ. | `ceph config show osd.0 osd_memory_target` (làm cho từng OSD) | docs Red Hat (autotune); ? (giá trị theo host đè lên) |
| E11 | Coi `osd_memory_target` là mục tiêu, không phải trần; đo RAM cả lúc recovery và backfill. | OSD dùng vượt mục tiêu khi recovery / backfill, đúng lúc demo UC5. | `ceph orch ps` (cột MEM USE) và `free -h` trong VM khi đang recovery / backfill. | kinh nghiệm |

### 5.5 KVM / host Kali

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| H1 | File qcow2 chỉ đặt trên ext4 (`/data`), không đặt trên NTFS; `/data` phải có trong `/etc/fstab`. | NTFS chậm, lỗi quyền; không tự mount thì VM không bật được. | `df -hT /data`; `findmnt /data`; `grep /data /etc/fstab` | kinh nghiệm |
| H2 | Giữ `/data` dưới 80% (khoảng 78 GB trên 98 GB). Giả định `/data` trống khi dựng lab (Q10); 26 GB hiện có phải dọn nếu không dùng. | qcow2 chỉ phình chứ không tự co (trừ khi bật discard); đĩa đầy thì QEMU tạm dừng VM → cả cụm Ceph đứng. | `df -h /data`; `virsh domstate <vm> --reason` | kinh nghiệm |
| H3 | Chỉ snapshot khi VM đã tắt. Với cụm Ceph: `ceph osd set noout`, tắt, chụp và revert **mọi node Ceph cùng lúc** (cả ceph4 nếu đã vào cụm). Trước đó umount, `rbd unmap` trên Kali (K2). | Snapshot có RAM tốn dung lượng bằng RAM của VM và làm giờ trong VM lùi lại; revert lẻ 1 node làm lệch trạng thái cụm. | `virsh snapshot-list <vm>`; sau khi bật: `ceph -s`, `ceph time-sync-status` | kinh nghiệm |
| H4 | Sau khi laptop sleep/resume hoặc revert snapshot: chỉnh giờ trước khi demo. | chrony mặc định chỉ nhảy giờ lúc khởi động, sau đó chỉnh rất chậm. | `chronyc tracking`; sửa bằng `sudo chronyc makestep` | kinh nghiệm |
| H5 | IP .11–.14 phải được đặt DHCP reservation theo MAC (hoặc thu hẹp dải DHCP). | Dải DHCP mặc định của mạng `default` là .2–.254, có thể cấp trùng IP. | `virsh net-dumpxml default`; `virsh net-dhcp-leases default` | kinh nghiệm |
| H6 | Dùng `virsh -c qemu:///system` (hoặc đặt `LIBVIRT_DEFAULT_URI`). | Chạy không sudo sẽ nối vào `qemu:///session`, không thấy VM nào. | `virsh uri` | kinh nghiệm |
| H7 | Không `apt full-upgrade` Kali trong 1–2 tuần trước demo. | Kali là bản rolling; cập nhật có thể làm hỏng libvirt/QEMU. | — | kinh nghiệm |
| H8 | Không cài cephadm bằng `apt` của Ubuntu; dùng cephadm standalone bản 20.2.x. | Kho Ubuntu 24.04 cung cấp Ceph Squid 19.2 (22.04 còn cũ hơn) → bootstrap ra bản cũ. | `./cephadm version`; sau bootstrap chạy `ceph versions` | docs Ceph + Ubuntu |
| H9 | Node Ceph (cả ceph4) cần Python 3, systemd, Podman hoặc Docker, chrony, LVM2; cần Internet lúc cài. | Yêu cầu của cephadm. | `cephadm check-host` | docs Ceph |

### 5.6 Demo

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| M1 | Bật VM ít nhất 10–15 phút trước giờ demo; chỉ bắt đầu khi cụm HEALTH_OK (hoặc chỉ còn WARN đã biết). | Cụm cần thời gian để peering và đồng bộ giờ. | `ceph -s` | kinh nghiệm |
| M2 | Chạy UC theo thứ tự UC1 → UC6; chỉ bật ceph4 khi tới phần thêm node của UC5 (ĐỀ XUẤT, mục 3.2); phần ngưỡng đầy của UC6 quay video. | Giữ lề RAM của host (chỉ khoảng 2.5 GiB khi đủ 4 VM); ngưỡng đầy làm hỏng các bước sau và chặn recovery / backfill của UC5. | `free -h` trên host | kinh nghiệm |
| M3 | Mỗi UC có video dự phòng quay từ lab thật. | Không có phương án B (D20); video là dự phòng cho buổi báo cáo. | Video có trong máy demo trước buổi báo cáo | kinh nghiệm |
| M4 | Tắt chế độ sleep khi gập máy, cắm sạc khi demo. | Laptop sleep làm VM đứng và lệch giờ. | Cài đặt nguồn của Kali | kinh nghiệm |

### 5.7 RBD (UC2)

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| K1 | Host Kali dùng một user Ceph riêng, chỉ có quyền trên pool `rbd` (ĐỀ XUẤT: `client.kali-rbd`, quyền `profile rbd`). Không chép keyring `client.admin` sang Kali. | Least privilege: keyring admin là toàn quyền cả cụm (nối với chủ đề UC4). | Trên ceph1: `ceph auth get client.kali-rbd`. Trên Kali: `rbd ls --id kali-rbd` chạy được, lệnh quản trị cụm thì không | kinh nghiệm |
| K2 | Trên Kali: umount rồi `rbd unmap` xong mới tắt cụm (T7.18) hoặc revert snapshot. | Ổ RBD còn map khi cụm đã tắt → lệnh trên host (umount, tắt máy) có thể treo vì I/O chờ mãi. | `rbd showmapped`; `findmnt` | kinh nghiệm |
| K3 | Nếu `rbd map` báo lỗi liên quan feature của image: tạo lại image chỉ với `--image-feature layering`. | krbd (driver RBD trong kernel) có thể không hỗ trợ mọi feature mặc định của image. | `dmesg` (xem các dòng cuối ngay sau lệnh lỗi); `rbd info <image>` | kinh nghiệm; kernel của Kali hỗ trợ tới đâu: ? |
| K4 | Client RBD phải hỗ trợ key CephX kiểu `aes256k`: kernel ≥ 7.0 (cho `rbd device map`), `ceph-common` ≥ 20.2.4 (Tentacle) hoặc ≥ 19.2.6 (Squid) cho lệnh `rbd`. | Cụm cài mới từ 20.2.4 tạo key kiểu `aes256k` (bản vá CVE-2025-30156: key cũ `aes` dùng AES-CBC không có HMAC). Client không hỗ trợ thì không xác thực được. | `uname -r`; `apt policy ceph-common`. LAP-24 đạt [đã chạy 04/11: kernel 7.1.5, `ceph-common` 20.2.4+ds-2]; LAP-16A, LAP-16B: không cần (D20) | docs (CVE-2025-30156, auth-config-ref) |

### 5.8 Thêm node (UC5)

| ID | Quy tắc | Lý do | Cách kiểm | Nguồn |
|---|---|---|---|---|
| N1 | Trước khi thêm ceph4, ghim placement của các service đang đặt theo số lượng, nhất là MON: `ceph orch apply mon --placement="ceph1 ceph2 ceph3"`. Không bật `ceph orch apply osd --all-available-devices` nếu muốn demo thêm OSD từng bước. | Mặc định cephadm tự thêm MON lên host mới (tối đa 5 MON) [docs] → ceph4 có MON, số MON thành chẵn, khác sơ đồ Deployment. Có `--all-available-devices` thì OSD trên ceph4 tự tạo ngay khi thêm host. | `ceph orch ls` (cột PLACEMENT); sau khi thêm: `ceph orch ps ceph4` | docs Tentacle (`cephadm/services/mon`, 04/11) |
| N2 | Chuẩn bị ceph4 xong từ trước như các node khác (H9, `/etc/hosts`, khóa SSH của cụm, image Ceph kéo sẵn), và tập "thêm node" lúc đã ngắt Internet. | Demo không có Internet; thiếu image thì không tạo được daemon trên ceph4. | Ngắt Internet → `ceph orch host add ceph4 192.168.122.14` → thêm OSD → `ceph orch ps ceph4` | kinh nghiệm; cephadm có kéo lại image không: ? |
| N3 | Chỉ thêm node khi cụm HEALTH_OK, không chồng lên recovery của phần node chết. Đo thời gian backfill khi tập; lâu thì quay video (D18). | Backfill và recovery chạy cùng lúc thì khó giải thích và tốn RAM, CPU (E11). | `ceph -s` (số PG backfilling, misplaced giảm dần); `ceph osd df` (%USE của OSD mới tăng) | kinh nghiệm |
| N4 | Khi cụm đã có 4 host: node tắt lâu hơn `mon_osd_down_out_interval` thì Ceph tự dựng lại mảnh bị mất lên host còn lại. Muốn giữ kịch bản "bật lại → recovery" thì bật lại sớm hơn mốc đó, hoặc `ceph osd set noout` trước. | Có host thứ 4 làm chỗ trống; giá trị mặc định 10 phút (600 s) [docs Tentacle]. | `ceph config get mon mon_osd_down_out_interval`; `ceph osd tree` (OSD chuyển sang out) | kinh nghiệm; giá trị: docs Tentacle (mã nguồn 04/11) |

---

## 6. Cách giải thích đã thống nhất

| Khái niệm | Cách giải thích |
|---|---|
| 3 kiểu lưu trữ | Block = ổ trống gắn vào máy; File = cây thư mục + phân quyền (SMB/NFS); Object = "quầy gửi đồ": dữ liệu + metadata + key, nằm trong bucket, gọi qua HTTP. Ceph làm cả 3 trên **một cụm**, chung RADOS bên dưới (unified storage): block = RBD, file = CephFS (+ MDS), object = RGW |
| S3 | Bucket, object, key. Access key = định danh; secret key = bí mật dùng để **ký** request (HMAC, SigV4), **không bao giờ được gửi qua mạng** |
| Storage gateway | "Người phiên dịch" giữa giao thức client dùng và giao thức của kho. Trong đề tài (mục 1, D17, ĐỀ XUẤT): (a) RGW dịch S3 → RADOS; (b) Samba dịch SMB → CephFS → RADOS. RBD không có gateway: thư viện phía client tự nói với RADOS |
| SDS | Phần mềm gom ổ đĩa của nhiều máy thường thành một kho, tự phân tán, nhân bản, sửa lỗi. Scale-out: thêm máy là thêm dung lượng; khác NAS truyền thống scale-up (thêm ổ vào cùng một máy, chạm giới hạn của máy là hết) |
| Ceph | RADOS = nhà kho; OSD = thủ kho từng kệ (1 OSD/ổ); MON = ban quản lý giữ sơ đồ cụm, cần **đa số** (2 trên 3) đồng ý để tránh split-brain; MGR = phòng điều hành/dashboard; RGW = quầy giao dịch S3; MDS = thủ thư giữ mục lục của CephFS (tên file, thư mục, quyền), còn dữ liệu file vẫn nằm ở OSD; Samba = quầy tiếp nhận hồ sơ kiểu SMB; cephadm = công cụ cài đặt, chạy daemon trong container |
| Pool / PG / CRUSH | object → hash → PG → CRUSH → danh sách OSD; giống hash table, không cần bảng tra trung tâm; failure domain = host |
| Thêm node (scale-out) | Thêm ceph4 → CRUSH tính lại → một phần PG chuyển sang OSD mới (backfill) trong khi client vẫn đọc/ghi. Như kho thêm kệ mới: thủ kho tự dời bớt hàng sang cho đều, không phải đóng cửa kho |
| Node chết và gỡ node | Chết đột ngột (UC5) = degraded, tạm hết dự phòng, chờ node quay lại (hoặc tự dựng lại lên host thứ 4, N4). Gỡ có kế hoạch (drain) = chép dữ liệu đi trước rồi mới gỡ, lúc nào cũng còn dự phòng. Nhóm chỉ demo trường hợp đầu (D18) |
| Erasure coding | Ví dụ XOR: A=1010, B=0110, P=A⊕B=1100; mất B thì B=A⊕P. Giống RAID 5 nhưng các mảnh nằm trên nhiều máy |
| Replication vs EC | Dung lượng dùng được: replica size n → 1/n (size 3 → 1/3); EC → k/(k+m) (2+1 → 2/3; 2+2 → 1/2) |
| min_size | Số mảnh tối thiểu để PG còn nhận I/O; đánh đổi giữa tính sẵn sàng và an toàn dữ liệu |
| Vì sao bỏ TrueNAS | Thầy xác nhận 28/10 chỉ dùng Ceph (D15). Ceph tự làm được phần NAS (CephFS + SMB), nên một cụm phục vụ được cả block, file, object |
| Vì sao bỏ MinIO | Gateway bị gỡ 2022 + bản cộng đồng ngừng bảo trì + Ceph khớp ghi chú "software define storage" của thầy |

**Chưa giải thích sâu:** Paxos, BlueStore, PG autoscaler, SigV4 từng bước, logic đánh giá policy (Deny/Allow), các lệnh cephadm từng bước, cấu hình module `smb` (user, share), subvolume của CephFS, krbd khác librbd thế nào, placement của cephadm.

---

## 7. Dữ kiện dễ lỗi thời

| Dữ kiện | Nguồn | Kiểm ngày |
|---|---|---|
| Ceph Tentacle mới nhất 20.2.5 (02/11/2026; trước đó 20.2.4, 19/08/2026), hỗ trợ dự kiến tới 01/06/2027; Squid 19.2.x hết hỗ trợ dự kiến 31/10/2026 (chưa kiểm lại sau mốc này) | docs.ceph.com/en/latest/releases (04/11: qua kết quả tìm kiếm); release notes: github.com/ceph/ceph, `doc/releases/tentacle.rst` (04/11) | 04/11/2026 |
| Key CephX kiểu `aes256k` có từ Tentacle 20.2.4 / Squid 19.2.6, cụm cài mới dùng mặc định; kernel Linux hỗ trợ từ 7.0 (K4) | github.com/ceph/ceph, `doc/security/CVE-2025-30156.rst` | 04/11/2026 |
| Tentacle: autoscaler bật mặc định, pool mới không ghi `pg_num` bắt đầu với 1 PG; `mon_max_pg_per_osd` 250; `mon_target_pg_per_osd` 100 (E8) | github.com/ceph/ceph, nhánh `tentacle`, `src/common/options/*.yaml.in` | 04/11/2026 |
| Bảng OS của Tentacle: Ubuntu 22.04 (gói hạng A, container host H), CentOS 9 (A, H), Debian 12 (C); không có Ubuntu 24.04 | docs.ceph.com (tentacle) start/os-recommendations | 20/10/2026 |
| Kho Ubuntu 24.04 cung cấp Ceph Squid 19.2.x | ubuntu.com/ceph/docs/supported-ceph-versions | 20/10/2026 |
| Module `smb` (mgr) của Tentacle: Samba trên CephFS, bắt buộc cephadm, chỉ SMB2/SMB3; docs ghi SMB "under active development"; Samba chỉ triển khai khi có share đầu tiên; image Samba lấy từ dự án samba-container | docs.ceph.com/en/tentacle/mgr/smb; docs.ceph.com/en/tentacle/cephadm/services/smb (02/11 qua kết quả tìm kiếm; 04/11 đọc bản nguồn trên GitHub, nhánh `tentacle`) | 04/11/2026 |
| MDS: `mds_cache_memory_limit` mặc định 4 GiB (đúng cả trên Tentacle); khuyến nghị ≥ 8 GB RAM cho MDS ở mức cache đó | docs.ceph.com/en/squid/start/hardware-recommendations (qua tìm kiếm, 02/11); mã nguồn nhánh `tentacle`, `src/common/options/mds.yaml.in` (04/11) | 04/11/2026 |
| iSCSI gateway của Ceph chỉ còn bảo trì từ 11/2022; hướng thay thế cho block qua mạng là NVMe-oF gateway | croit.io blog (block storage options); Red Hat Ceph 6.0 release notes | 02/11/2026 |
| RGW Cloud Transition (lifecycle → storage class `cloud-s3`); Tentacle có Cloud Restore (cần `retain_head_object=true`), nên không còn "chỉ một chiều" | docs.ceph.com radosgw/cloud-transition, cloud-restore | 19/10/2026 |
| IAM Accounts có từ Squid 19.2.0; account ID = "RGW" + 17 chữ số; user của account bắt đầu không có quyền nào | docs.ceph.com/en/squid/radosgw/account | 20/10/2026 |
| boto3 từ 1.36 bật checksum mặc định; RGW có thể lỗi; cách tránh: `when_required` | specs.openstack.org (glance, 2026.1) | 19/10/2026 |
| Rate limit RGW tính theo từng instance; vượt ngưỡng trả HTTP 503 | ceph.io blog (2025) rgw-rate-limiting | 19/10/2026 |
| MinIO gỡ gateway từ `RELEASE.2022-10-29T06-21-33Z`; bản cộng đồng chuyển maintenance 12/2025; README kho `minio/minio` ghi "no longer maintained" (đổi khoảng 02/2026); kho archive 25/04/2026 (sửa 04/11, bản cũ ghi 02/2026) | min.io blog; vonng.com; itsfoss.com; README trên GitHub (đọc 04/11); ngày archive: banner GitHub qua nguồn trích lại (pinggy.io, devtake.pages.dev), chưa tự xem [?] | 04/11/2026 |
| AWS từ 16/09/2026: "phần lớn khách mới không cần thẻ", 100 USD credit, triển khai dần | aws.amazon.com blog (getting-started) | 19/10/2026 |

---

## 8. Tài liệu nhóm chứa gì (`NT132-P2-TaiLieuNhom.md`)

T1 Ngữ cảnh, Scope, Objectives, Benefits (bản nháp) · T2 Business capabilities & bảng truy vết · T3 Khung 17 slide · T4 Bối cảnh môn & đề gốc · T5 Kiến trúc (5 sơ đồ Mermaid + cách đọc; 5 file giải thích nằm ở `NT132-P2-kien-truc/`) · T6 Đặc tả UC1–UC6 · T7 Runbook dựng lab · T8 Kịch bản demo & đánh giá thời lượng · T9 Câu hỏi vấn đáp · T10 Nhóm, phân công, hạng ưu tiên, lộ trình · T11 Rủi ro · T12 Nhật ký · T13 Tham khảo.
Làm UC nào thì dán thêm mục T6 của UC đó; đang dựng lab thì dán thêm T7.