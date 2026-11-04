> Bản 04/11/2026: viết lại theo thiết kế một cụm Ceph (D15) và thêm ceph4 (D18); số liệu LAP-24 đã kiểm ngày 04/11. Sơ đồ: `slide09-infrastructure.mmd`. Bản 21/10 (có VM truenas) không còn dùng.

#### **Ôn lại khái niệm mạng và ảo hóa trước khi đọc**

| Khái niệm | Giải thích | Ví dụ trong lab |
| --- | --- | --- |
| **Hypervisor (KVM)** | Phần mềm chia một máy thật thành nhiều máy ảo, mỗi máy ảo tưởng nó có CPU, RAM, ổ đĩa riêng. KVM nằm trong nhân Linux; **libvirt** là bộ công cụ quản lý nó (`virsh`, virt-manager). | Một căn nhà chia thành 4 phòng trọ, mỗi phòng có khóa riêng. |
| **Luồng CPU (thread)** và **vCPU** | Luồng là số việc CPU chạy song song được. vCPU là CPU ảo cấp cho VM, được xếp lên các luồng thật. Tổng vCPU có thể lớn hơn số luồng thật (overcommit); khi đó các VM phải chờ nhau. | i5-12450H có 12 luồng (`lscpu`, dòng `CPU(s)`); 4 VM × 2 vCPU = 8 vCPU, còn khoảng 4 luồng cho host. |
| **Subnet /24, gateway** | `192.168.122.0/24` nghĩa là 3 số đầu cố định, số cuối chạy từ 1 tới 254. Gateway là "cửa ra" của mạng con. | Gateway là `.1`, chính là host Kali. |
| **Bridge (virbr0)** | Một switch ảo nằm trong host. Các VM cắm vào đây để nói chuyện với nhau và với host. | — |
| **NAT** | VM đi ra Internet "mượn" IP của host. Ngược lại, máy bên ngoài laptop **không tự vào được** VM nếu không cấu hình chuyển port. | Giống mạng Wi-Fi ở nhà: điện thoại ra Internet được, nhưng người ngoài không gọi thẳng vào điện thoại được. |
| **DHCP reservation** | Máy chủ DHCP (của libvirt) luôn cấp đúng một IP cho một địa chỉ MAC, nên IP của VM không đổi sau mỗi lần khởi động (gotcha H5). | ceph1 luôn là `.11`, ceph4 luôn là `.14`. |
| **Port** | Số "cửa" trên một IP, mỗi dịch vụ lắng nghe ở một cửa. | SMB ở 445 (ceph2); S3 của RGW ở 80 (ceph1). |
| **Cấp phát mỏng (thin provisioning)** | File ổ đĩa ảo (qcow2) chỉ chiếm chỗ thật khi VM ghi dữ liệu, và mặc định chỉ phình chứ không tự co. | Cấp 140 GB nhưng dùng thật dự kiến khoảng 50–70 GB. |

#### **Cách đọc sơ đồ**

1. **Khung tím viền đậm** là cái laptop. Mọi thứ đều nằm trong nó; chỉ có Internet ở ngoài.
2. Đọc **từ trên xuống**:
   * host Kali;
   * ba thứ host quản lý: phân vùng `/data` (chứa ổ đĩa ảo), switch ảo `virbr0`, và đường ra Internet;
   * 4 VM Ceph cắm vào `virbr0`.
3. **Chữ trên đường nối VM** là các port VM đó mở, tức là "muốn nói chuyện với VM này thì gõ cửa số mấy".
4. **Nét đứt xám**: Internet chỉ dùng lúc cài đặt (kéo container image), lúc demo không cần.
5. **ceph4 viền đứt xanh**: VM này chỉ tham gia cụm ở UC5 ("thêm 1 node", D18).

#### **Giải thích từng khối**

| Khối | Giải thích | Ví dụ đời thường |
| --- | --- | --- |
| **LAP-24** | Máy thật duy nhất: Acer Nitro 5, **i5-12450H** (12 luồng, có VT-x), 24 GB RAM (Kali thấy 23 GiB), ổ SSD NVMe. Đã kiểm 04/11 (file AI mục 3.1). | Cả tòa nhà. |
| **Host Kali + KVM/libvirt** | Hệ điều hành của laptop. Nó chạy hypervisor, đồng thời là nơi gõ lệnh client khi demo. Sau khi chia RAM cho 4 VM còn khoảng 7 GiB; lúc mở Chrome 6 tab host đã dùng 4,4 GiB, nên chỉ dư khoảng 2,5 GiB. Vì vậy đề xuất để ceph4 tắt tới UC5. | Chủ nhà, vừa ở vừa quản lý phòng trọ. |
| **Phân vùng `/data`** | 98 GB, ext4, có trong `/etc/fstab` nên tự mount khi khởi động (đã kiểm 04/11). Chứa 8 file ổ đĩa ảo của 4 VM. Phải giữ dưới 80% (H2). | Kho chứa đồ của cả 4 phòng. |
| **virbr0 (NAT)** | Switch ảo cho mạng `192.168.122.0/24`; host là `.1`. | Hành lang chung nối các phòng. |
| **ceph1 `.11`** | Node "nặng" nhất (MGR active, RGW, MDS standby) nên được 5 GB. Mở 80 (S3 của RGW), 8443 (dashboard), cùng port MON và OSD. | — |
| **ceph2 `.12`** | 4,5 GB; chạy Samba và MDS active. Mở **445 (SMB)**, cùng port MON và OSD. | — |
| **ceph3 `.13`** | 3,5 GB; chỉ có MON và OSD. Mở port MON (3300, 6789) và OSD (6800–7300). Đây là node nhóm tắt khi demo UC5. | — |
| **ceph4 `.14`** | 3 GB; chỉ có OSD, không có MON. Chỉ mở port OSD. Tham gia cụm ở UC5. | Phòng trọ mới xây thêm, dọn vào sau. |

#### **Lời nói mẫu khi thuyết trình (khoảng 75 giây)**

> "Ba slide trước nói về phần mềm; slide này nói phần mềm đó chạy trên cái gì.
>
> Toàn bộ hệ thống nằm trong một cái laptop: Acer Nitro 5, chip i5 thế hệ 12 với 12 luồng, 24 GB RAM, ổ SSD NVMe.
>
> Trên đó là Kali Linux làm host, chạy hypervisor KVM. Host quản lý ba thứ: phân vùng `/data` chứa ổ đĩa ảo, cấp phát mỏng nên chỉ chiếm chỗ khi thật sự ghi; switch ảo virbr0, mạng `192.168.122.0/24`, host là cổng `.1`; và đường ra Internet, chỉ dùng lúc cài nên vẽ nét đứt.
>
> Bốn máy ảo Ceph cắm vào switch ảo, IP cố định từ `.11` tới `.14`. ceph1 mở cổng 80 cho S3, ceph2 mở cổng 445 cho SMB. ceph4 vẽ viền đứt vì nó chỉ được thêm vào cụm ở UC5. Tổng cộng 16 GB RAM và 8 vCPU trên 12 luồng thật; đây là phân bổ đề xuất, nhóm sẽ đo lại khi chạy.
>
> Một điểm phải nói thẳng: bốn 'máy' Ceph nằm trên cùng một ổ SSD. Nên failure domain thật là cái laptop. Lab này mô phỏng cơ chế chịu lỗi, không phải hạ tầng chịu lỗi thật."

#### **3 câu hỏi vấn đáp dễ gặp**

1. **"Mạng NAT thì máy khác trong lớp có truy cập S3 hay SMB được không? Như vậy có an toàn không?"**
   Mặc định là không: NAT của libvirt cho VM đi ra, nhưng không cho máy bên ngoài laptop đi vào nếu không chuyển port [cần kiểm: quét bằng `nmap` từ một máy khác trong mạng]. Điều này tốt cho bảo mật vì SMB 445 và S3 chạy HTTP không lộ ra mạng trường. Nhưng bên trong laptop, mọi tiến trình trên host đều với tới được các port này.
2. **"4 node Ceph trên một SSD thì chịu lỗi có ý nghĩa gì?"**
   Mô phỏng failure domain để học và chứng minh cơ chế: CRUSH đặt mỗi mảnh lên một "host", mất một host vẫn đọc được, thêm một host thì dữ liệu tự dàn sang. Triển khai thật thì phải tách máy, tách ổ, tách nguồn điện (tài liệu nhóm T9, câu 6).
3. **"Laptop 24 GB có đủ RAM cho 4 VM không?"**
   Trên giấy: 16 GB cho VM, host còn khoảng 7 GiB; đo lúc mở Chrome 6 tab thì host đã dùng 4,4 GiB, nên chỉ dư khoảng 2,5 GiB, chưa trừ phần QEMU tốn thêm cho mỗi VM. Vì vậy nhóm để ceph4 tắt cho tới lúc thêm node (khi đó VM chỉ dùng 13 GB). RAM từng VM là đề xuất, sẽ đo bằng `free -h` và `ceph orch ps` lúc chạy thật, nhất là lúc recovery (D11, E11).

#### **Không được nói**

* "Ceph chạy trên **4 máy chủ**." Đó là 4 VM trên một laptop.
* "Hệ thống chịu được hỏng phần cứng." Hỏng SSD hay hỏng laptop là mất tất cả.
* "Mạng NAT nên **không ai** truy cập được." Mọi thứ trên host đều truy cập được, và VM vẫn ra được Internet.
* "Dữ liệu nhân bản qua **mạng riêng** (cluster network)." Public network và cluster network dùng chung `virbr0` (file AI mục 3.4).
* "RAM đã đo, đủ dùng." Mới đo RAM của host; RAM từng VM chưa đo (D11).
* "ceph4 chạy cùng cụm **từ đầu**." ceph4 chỉ tham gia ở UC5.

#### **Phát hiện khi vẽ**

Bảng 6 phát hiện (vCPU, đồng bộ giờ khi không có Internet, SSH 22, dashboard đi theo MGR, SMB nằm ở IP khác S3, lề RAM của host) nằm ở **tài liệu nhóm T5.4**; không chép lại ở đây để chỉ có một nguồn.
