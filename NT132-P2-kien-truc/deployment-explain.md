> Bản 04/11/2026: viết lại theo thiết kế một cụm Ceph (D15), thêm MDS, Samba, ceph4 (D18). Sơ đồ: `slide10-deployment.mmd`. Bản 21/10 (có VM truenas) không còn dùng.

#### **Ôn lại khái niệm nền**

| Khái niệm | Giải thích | Ví dụ đời thường |
| --- | --- | --- |
| **Daemon** | Chương trình chạy nền liên tục để phục vụ (MON, OSD, RGW, MDS, Samba…). | Nhân viên trực ca. |
| **VM và container khác nhau thế nào** | VM có *nhân hệ điều hành riêng*, giả lập cả một máy (slide 9). Container *dùng chung nhân* với máy chứa nó, chỉ đóng gói chương trình cùng thư viện của nó. Ở đây có cả hai tầng: daemon Ceph chạy trong container, container chạy trong VM. | VM là căn hộ riêng; container là một phòng có khóa *trong* căn hộ đó, dùng chung điện nước. |
| **podman** | Công cụ chạy container, tương tự Docker. | — |
| **cephadm** | Bộ công cụ cài và quản lý Ceph. Nó tải image Ceph từ quay.io (lúc cài, có Internet), chạy mỗi daemon thành một container, và dùng module trong MGR (slide 7) để đặt daemon lên từng máy qua SSH. | Bộ phận nhân sự: phân ai trực ở phòng nào. |
| **Placement (vị trí đặt daemon)** | Quy định daemon nào chạy trên máy nào. Mặc định cephadm tự chọn; nhóm ghim tường minh cho MGR, MDS, Samba và MON để khớp sơ đồ này (gotcha F4, N1). | Lịch trực ghi rõ tên người, không để ai đến trước thì trực. |
| **Active / standby** | Một bản đang làm việc (active), một bản chờ sẵn (standby). Bản active hỏng thì bản standby nhận thay. | Nhân viên chính và nhân viên dự bị. |
| **qcow2** | Định dạng file ổ đĩa ảo của QEMU/KVM. Mỗi ổ của VM là một file trên `/data`. | — |

#### **Cách đọc sơ đồ**

1. **Trên cùng:** host Kali, gồm công cụ client và nơi chứa file ổ đĩa ảo.
2. **Mũi tên từ `/data` xuống** nghĩa là "các file qcow2 này là ổ đĩa của máy đó": mỗi node Ceph có 2 file (ổ OS + ổ OSD), tổng 8 file. Khớp với cột đĩa ở slide 9.
3. **Mỗi khung là một máy.** Đọc từ trái sang phải: ceph1 → ceph4. Trong mỗi khung đọc từ trên xuống.
4. **Màu và hình dạng ô:**
   * ô bo góc **xanh dương** = daemon lõi Ceph chạy trong container;
   * ô bo góc **cam** = gateway chạy trong container (RGW, Samba);
   * ô **xanh lá** = công cụ client;
   * ô **xám nét đứt** = CHƯA CHỐT (RGW thứ hai, D10).
5. **Hình trụ dưới cùng:** pool nằm trải trên các node. **Khung ceph4 viền đứt** và đường nối pool nét đứt: ceph4 chỉ tham gia từ UC5.

#### **Giải thích từng khối**

| Khối | Giải thích |
| --- | --- |
| **Host Kali** | Nơi chạy mọi công cụ client khi demo (aws cli, rclone, boto3/curl, smbclient, rbd). Đồng thời giữ 8 file qcow2 của 4 VM. |
| **ceph1** | Node "nặng" nhất: MON, MGR đang active kèm dashboard ở port 8443, **RGW duy nhất** ở port 80, MDS standby, OSD.0, và công cụ quản trị. Đây là lý do nó được 5 GB RAM ở slide 9, và cũng là lý do tắt ceph1 thì mất S3 (R1). |
| **ceph2** | MON, MGR standby, **Samba duy nhất** ở port 445, MDS active, OSD.1. Ô xám nét đứt là RGW thứ hai, chỉ có nếu chốt D10. Tắt ceph2 thì mất SMB. |
| **ceph3** | Chỉ MON và OSD.2. Không chạy gateway nào, nên là node an toàn nhất để tắt khi demo UC5. |
| **ceph4** | Chỉ OSD.3. Là node nhóm **thêm vào ở UC5** để cho thấy cụm tự mở rộng. Không có MON. |
| **Vì sao 3 MON, 2 MGR, 2 MDS, 1 RGW, 1 Samba** | MON cần đa số (2 trên 3) nên phải có 3; thêm MON thứ 4 cũng chỉ chịu mất 1 (đa số của 4 là 3), nên không lợi gì. MGR và MDS: một active, một standby. RGW và Samba mới có 1 bản (RGW thứ hai là D10; Samba clustered nằm ngoài phạm vi). Mỗi node có 1 ổ dữ liệu nên có 1 OSD. |
| **Pool** | Pool là khái niệm *logic*, không nằm trên một máy nào. Các PG của nó rải lên các OSD. Pool data của RGW là EC 2+1; các pool khác (metadata của RGW, CephFS, `rbd`) là replicated ×3. Khi thêm ceph4, một phần PG chuyển sang OSD.3 (backfill). |

#### **Lời nói mẫu khi thuyết trình (khoảng 75 giây)**

> "Slide trước cho thấy máy và mạng; slide này cho thấy phần mềm nào được đặt lên máy nào.
>
> Trên cùng là host Kali: mọi công cụ client khi demo chạy ở đây, và ổ đĩa ảo của bốn máy ảo là các file qcow2 trong phân vùng `/data`.
>
> Bên dưới là bốn node Ceph. Mọi daemon, kể cả Samba, chạy trong container podman, do cephadm đặt lên từng máy.
>
> Ba node đầu có MON, vì MON cần đa số hai trên ba. Mỗi node có một OSD giữ một ổ 15 GB.
>
> Hai gateway được tách ra hai máy: RGW cho S3 ở ceph1, Samba cho SMB ở ceph2. MDS chạy một bản active ở ceph2 và một bản dự phòng ở ceph1.
>
> ceph3 chỉ có MON và OSD, nên khi demo chịu lỗi nhóm tắt ceph3: không đụng tới gateway nào. ceph4 vẽ viền đứt: đó là node nhóm thêm vào ở UC5, chỉ có một OSD.
>
> Dưới cùng là pool: không nằm trên máy nào riêng, mà trải trên các OSD, với erasure coding 2+1 cho dữ liệu object."

#### **3 câu hỏi vấn đáp dễ gặp**

1. **"Đã chạy trong VM rồi sao còn container? Có thừa không?"**
   Hai tầng giải quyết hai việc khác nhau. VM *mô phỏng máy chủ* để có 4 "host" trên một laptop. Container là *cách cephadm đóng gói Ceph*: mọi daemon dùng cùng một image cùng phiên bản, nên cài, nâng cấp và gỡ đồng nhất trên mọi máy. Trong triển khai thật, không có tầng VM nhưng vẫn có container.
2. **"Tắt ceph1 thì sao? Tắt ceph2 thì sao?"**
   * **ceph1:** MON còn 2 trên 3, vẫn đủ đa số. MGR chuyển sang bản standby ở ceph2 (chỉ chắc khi đã ghim vị trí, T7.9); dashboard đi theo nó. Dữ liệu vẫn đọc được. Nhưng **RGW mất, nên mất S3**; MDS standby cũng mất nên cụm báo cảnh báo (F3).
   * **ceph2:** **Samba mất, nên mất SMB**; MDS active chuyển sang bản standby ở ceph1 [dự kiến]. S3 vẫn chạy.
   * Cách khắc phục: RGW thứ hai (D10); Samba clustered (ngoài phạm vi). Bảng đầy đủ: tài liệu nhóm T5.6.
3. **"Sao ceph4 không có MON?"**
   4 MON vẫn chỉ chịu mất 1 máy (đa số của 4 là 3), lại thêm một máy phải đồng bộ bản đồ cụm, nên giữ 3 MON. Trước khi thêm ceph4 phải ghim vị trí MON, vì cephadm có thể tự đặt MON lên host mới [?] (gotcha N1).

#### **Không được nói**

* "Mỗi daemon là một máy ảo." Daemon là container trong VM.
* "MGR active (hoặc MDS active) **luôn** ở cùng một máy." Nó có thể chuyển sang node khác khi có sự cố hoặc sau khi khởi động lại.
* "Pool nằm trên ceph1" hoặc "pool là một ổ đĩa." Pool là khái niệm logic, trải trên các OSD.
* "Samba được **cài tay** trên VM." Samba do module `smb` của Ceph dựng lên, chạy trong container qua cephadm.
* "Hệ thống **không có SPOF**." Có: RGW và Samba mới có 1 bản.
* "Thêm ceph4 là có thêm MON, MGR." ceph4 chỉ có OSD (và daemon phụ cephadm tự thêm như `crash`, cần kiểm bằng `ceph orch ps`).
