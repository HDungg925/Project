#### **Ôn lại khái niệm nền**

| Khái niệm | Giải thích | Ví dụ đời thường |
| ----- | ----- | ----- |
| **Daemon** | Chương trình chạy nền liên tục để phục vụ (MON, OSD, RGW…). | Nhân viên trực ca. |
| **VM và container khác nhau thế nào** | VM có *nhân hệ điều hành riêng*, giả lập cả một máy (slide 9). Container *dùng chung nhân* với máy chứa nó, chỉ đóng gói chương trình cùng thư viện của nó. Ở đây có cả hai tầng: daemon Ceph chạy trong container, container chạy trong VM. | VM là căn hộ riêng; container là một phòng có khóa *trong* căn hộ đó, dùng chung điện nước. |
| **podman** | Công cụ chạy container, tương tự Docker. | — |
| **cephadm** | Bộ công cụ cài và quản lý Ceph. Nó tải image Ceph từ quay.io (lúc cài, có Internet), chạy mỗi daemon thành một container, và dùng module trong MGR (đã thấy ở slide 7\) để đặt daemon lên từng máy qua SSH. | Bộ phận nhân sự: phân ai trực ở phòng nào. |
| **qcow2** | Định dạng file ổ đĩa ảo của QEMU/KVM. Mỗi ổ của VM là một file trên `/data`. | — |

#### **Cách đọc sơ đồ**

1. **Trên cùng:** host Kali, gồm công cụ client và nơi chứa file ổ đĩa ảo.  
2. **Mũi tên từ `/data` xuống** nghĩa là “các file qcow2 này là ổ đĩa của máy đó”: truenas có 3 file (boot \+ 2 ổ ZFS), mỗi node Ceph có 2 file (ổ OS \+ ổ OSD). Khớp với cột đĩa ở slide 9\.  
3. **Mỗi khung là một máy.** Đọc từ trái sang phải: truenas, ceph1, ceph2, ceph3. Trong mỗi khung đọc từ trên xuống.  
4. **Hình dạng ô cho biết loại phần mềm:**  
   * ô bo góc xanh dương \= daemon Ceph chạy trong container;  
   * ô vuông cam \= dịch vụ có sẵn của TrueNAS (không dùng container của cephadm);  
   * ô xanh lá \= công cụ client.  
5. **Hình trụ dưới cùng:** pool nằm trải trên cả 3 node Ceph.

#### **Giải thích từng khối**

| Khối | Giải thích |
| ----- | ----- |
| **Host Kali** | Nơi chạy mọi công cụ client khi demo (T6 ghi “Trên host Kali”). Đồng thời giữ 9 file qcow2 của 4 VM. |
| **truenas** | 4 dịch vụ cài sẵn trong hệ điều hành TrueNAS: Web UI, Samba, Cloud Sync (chỉ chạy rclone khi task chạy), ZFS pool mirror. |
| **ceph1** | Node “nặng” nhất: MON, MGR đang active kèm dashboard ở port 8443, RGW duy nhất ở port 80, OSD.0, và công cụ quản trị. Đây là lý do nó được 4,5 GB RAM ở slide 9, và cũng là lý do tắt ceph1 thì mất S3 (R1). |
| **ceph2** | MON, MGR standby (dự phòng), OSD.1. Ô xám nét đứt là RGW thứ hai, chỉ có nếu chốt D10. |
| **ceph3** | Chỉ MON và OSD.2. Đây là node an toàn nhất để tắt khi demo UC4. |
| **Vì sao 3 MON, 2 MGR, 1 RGW** | MON cần đa số (2 trên 3\) nên phải có 3\. MGR chỉ cần một active và một dự phòng. RGW mới có 1 (D10 chưa chốt). Mỗi node có 1 ổ dữ liệu nên có 1 OSD. |
| **Pool** | Pool là khái niệm *logic*, không nằm trên một máy nào. Các PG của nó rải lên cả 3 OSD. Pool data của RGW là EC 2+1; các pool khác (index, metadata) là replicated ×3. |

#### **Lời nói mẫu khi thuyết trình (khoảng 75 giây)**

> “Slide trước cho thấy máy và mạng; slide này cho thấy phần mềm nào được đặt lên máy nào.  
>  Trên cùng là host Kali: mọi công cụ client khi demo chạy ở đây, và ổ đĩa ảo của 4 máy ảo là các file qcow2 trong phân vùng `/data`.  
>  Bên dưới là 4 máy ảo. TrueNAS chạy các dịch vụ có sẵn của nó: Web UI, Samba, Cloud Sync và ZFS.  
>  Ba node Ceph thì khác: mọi daemon chạy trong container podman, do cephadm đặt lên từng máy.  
>  Cả ba node đều có MON, vì MON cần đa số hai trên ba. Mỗi node có một OSD giữ một ổ 15 GB.  
>  ceph1 gánh nhiều nhất: MGR đang active với dashboard, RGW duy nhất, và công cụ quản trị. Vì vậy khi demo chịu lỗi, nhóm tắt ceph3 chứ không tắt ceph1. Tắt ceph1 là mất cổng S3; đó là điểm yếu nhóm đã biết, và cách sửa là RGW thứ hai trên ceph2, ô nét đứt, chưa chốt.  
>  Dưới cùng là pool: không nằm trên máy nào riêng, mà trải trên cả ba OSD, với erasure coding 2+1 cho dữ liệu.”

#### **3 câu hỏi vấn đáp dễ gặp**

1. **“Đã chạy trong VM rồi sao còn container? Có thừa không?”**  
    Hai tầng giải quyết hai việc khác nhau. VM *mô phỏng máy chủ* để có 3 “host” trên một laptop. Container là *cách cephadm đóng gói Ceph*: mọi daemon dùng cùng một image cùng phiên bản, nên cài, nâng cấp và gỡ đồng nhất trên mọi máy. Trong triển khai thật, không có tầng VM nhưng vẫn có container.  
2. **“Tắt ceph1 thì sao?”**  
   * MON còn 2 trên 3, vẫn đủ đa số.  
   * MGR chuyển sang bản dự phòng; dashboard đi theo nó, và chỉ là ceph2 nếu đã ghim vị trí.  
   * OSD.1 và OSD.2 vẫn giữ đủ 2 mảnh nên dữ liệu đọc được.  
   * Nhưng **RGW mất, nên mất S3**. Muốn tránh thì cần D10 (RGW thứ hai, kèm cơ chế chia tải).  
3. **“Sao ceph3 chỉ có 2 daemon? Như vậy có lệch tải không?”**  
    Có lệch, và nhóm chấp nhận có chủ đích: ceph1 nhận nhiều vai hơn nên được nhiều RAM hơn (4,5 GB so với 3,5 GB, slide 9). Đổi lại, ceph3 là node “an toàn để tắt” khi demo UC4.

#### **Không được nói**

* “Mỗi daemon là một máy ảo.” Daemon là container trong VM.  
* “MGR active **luôn** ở ceph1.” Nó có thể chuyển sang node khác khi có sự cố hoặc sau khi khởi động lại.  
* “Pool nằm trên ceph1” hoặc “pool là một ổ đĩa.” Pool là khái niệm logic, trải trên 3 OSD.  
* “TrueNAS cũng chạy bằng cephadm/container.” Không: đó là dịch vụ có sẵn của TrueNAS.  
* “Hệ thống không có SPOF.” Có: RGW đang chỉ có 1 bản.

