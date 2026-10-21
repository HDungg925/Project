#### **Ôn lại khái niệm mạng và ảo hóa trước khi đọc**

| Khái niệm | Giải thích | Ví dụ trong lab |
| ----- | ----- | ----- |
| **Hypervisor (KVM)** | Phần mềm chia một máy thật thành nhiều máy ảo, mỗi máy ảo tưởng nó có CPU, RAM, ổ đĩa riêng. KVM nằm trong nhân Linux; **libvirt** là bộ công cụ quản lý nó (`virsh`, virt-manager). | Một căn nhà chia thành 4 phòng trọ, mỗi phòng có khóa riêng. |
| **vCPU** | CPU ảo cấp cho VM. Tổng vCPU có thể lớn hơn số luồng CPU thật (overcommit); khi đó các VM phải chờ nhau. | 4 VM × 2 vCPU \= 8 vCPU. |
| **Subnet /24, gateway** | `192.168.122.0/24` nghĩa là 3 số đầu cố định, số cuối chạy từ 1 tới 254\. Gateway là “cửa ra” của mạng con. | Gateway là `.1`, chính là host Kali. |
| **Bridge (virbr0)** | Một switch ảo nằm trong host. Các VM cắm vào đây để nói chuyện với nhau và với host. | — |
| **NAT** | VM đi ra Internet “mượn” IP của host. Ngược lại, máy bên ngoài laptop **không tự vào được** VM nếu không cấu hình chuyển port. | Giống mạng Wi-Fi ở nhà: điện thoại ra Internet được, nhưng người ngoài không gọi thẳng vào điện thoại được. |
| **DHCP reservation** | Máy chủ DHCP (của libvirt) luôn cấp đúng một IP cho một địa chỉ MAC, nên IP của VM không đổi sau mỗi lần khởi động (gotcha H5). | ceph1 luôn là `.11`. |
| **Port** | Số “cửa” trên một IP, mỗi dịch vụ lắng nghe ở một cửa. | SMB ở 445; RGW ở 80\. |
| **Cấp phát mỏng (thin provisioning)** | File ổ đĩa ảo (qcow2) chỉ chiếm chỗ thật khi VM ghi dữ liệu, và mặc định chỉ phình chứ không tự co. | Cấp 145 GB nhưng dùng thật dự kiến khoảng 50–70 GB. |

#### **Cách đọc sơ đồ**

1. **Khung tím viền đậm** là cái laptop. Mọi thứ đều nằm trong nó; chỉ có Internet ở ngoài.  
2. Đọc **từ trên xuống**:  
   * host Kali;  
   * ba thứ host quản lý: phân vùng `/data` (chứa ổ đĩa ảo), switch ảo `virbr0`, và đường ra Internet;  
   * 4 VM cắm vào `virbr0`.  
3. **Chữ trên đường nối VM** là các port VM đó mở, tức là “muốn nói chuyện với VM này thì gõ cửa số mấy”.  
4. **Nét đứt xám**: Internet chỉ dùng lúc cài đặt (kéo container image), lúc demo không cần.

#### **Giải thích từng khối**

| Khối | Giải thích | Ví dụ đời thường |
| ----- | ----- | ----- |
| **LAP-24** | Máy thật duy nhất: Acer Nitro 5, i5, 24 GB RAM, một ổ SSD. | Cả tòa nhà. |
| **Host Kali \+ KVM/libvirt** | Hệ điều hành của laptop. Nó chạy hypervisor, đồng thời là nơi gõ lệnh client khi demo. Sau khi chia RAM cho VM, nó còn lại dưới 6,5 GB. | Chủ nhà, vừa ở vừa quản lý phòng trọ. |
| **Phân vùng `/data`** | 100 GB, nơi đặt file ổ đĩa ảo của cả 4 VM. Phải là ext4 và tự mount khi khởi động (H1); phải giữ dưới 80% (H2). | Kho chứa đồ của cả 4 phòng. |
| **virbr0 (NAT)** | Switch ảo cho mạng `192.168.122.0/24`; host là `.1`. | Hành lang chung nối các phòng. |
| **truenas `.10`** | 6 GB RAM (thấp hơn mức khuyến nghị 8 GB, chấp nhận có chủ đích vì pool chỉ 20 GB, CS5); 1 ổ boot 20 GB và 2 ổ 10 GB làm ZFS mirror. Mở 445 (SMB) và 80/443 (Web UI). | — |
| **ceph1 `.11`** | Node “nặng” nhất nên được 4,5 GB. Mở 80 (S3 của RGW), 8443 (dashboard), cộng với port MON và OSD. | — |
| **ceph2, ceph3** | 3,5 GB mỗi máy; mỗi máy có 1 ổ 15 GB làm OSD. Chỉ mở port MON (3300, 6789\) và OSD (6800–7300) để nói chuyện trong cụm. | — |

#### **Lời nói mẫu khi thuyết trình (khoảng 75 giây)**

> “Ba slide trước nói về phần mềm; slide này nói phần mềm đó chạy trên cái gì.  
>  Toàn bộ hệ thống nằm trong một cái laptop: Acer Nitro 5, 24 GB RAM, một ổ SSD.  
>  Trên đó là Kali Linux làm host, chạy hypervisor KVM. Host quản lý ba thứ. Thứ nhất là phân vùng `/data` 100 GB, chứa ổ đĩa ảo; các ổ cấp phát mỏng, cấp tổng 145 GB nhưng chỉ chiếm chỗ khi thật sự ghi. Thứ hai là switch ảo virbr0, mạng `192.168.122.0/24`, host là cổng `.1`. Thứ ba là đường ra Internet qua NAT, chỉ dùng lúc cài đặt, nên vẽ nét đứt; lúc demo không cần mạng ngoài.  
>  Bốn máy ảo cắm vào switch ảo, IP cố định từ `.10` tới `.13`. Tổng cộng 17,5 GB RAM và 8 vCPU. Đây là phân bổ đề xuất, nhóm sẽ đo lại khi chạy thật. TrueNAS mở port SMB và Web UI. ceph1 mở port 80 cho S3 và 8443 cho dashboard. Cả ba node Ceph mở port MON và OSD để nói chuyện với nhau.  
>  Một điểm phải nói thẳng: ba ‘máy’ Ceph thật ra nằm trên cùng một ổ SSD. Nên failure domain thật là cái laptop. Lab này mô phỏng cơ chế chịu lỗi, không phải hạ tầng chịu lỗi thật.”

#### **3 câu hỏi vấn đáp dễ gặp**

1. **“Mạng NAT thì máy khác trong lớp có truy cập S3 hay SMB được không? Như vậy có an toàn không?”**  
    Mặc định là không: NAT của libvirt cho VM đi ra, nhưng không cho máy bên ngoài laptop đi vào nếu không chuyển port \[cần kiểm: quét bằng `nmap` từ một máy khác trong mạng\]. Điều này tốt cho bảo mật vì SMB 445 và S3 chạy HTTP không lộ ra mạng trường. Nhưng bên trong laptop, mọi tiến trình trên host đều với tới được các port này.  
2. **“3 node Ceph trên một SSD thì chịu lỗi có ý nghĩa gì?”**  
    Mô phỏng failure domain để học và chứng minh cơ chế: CRUSH đặt mỗi mảnh lên một “host”, mất một host vẫn đọc được. Triển khai thật thì phải tách máy, tách ổ, tách nguồn điện (T9 câu 6).  
3. **“Cấp 145 GB mà `/data` chỉ có 100 GB, lỡ đầy thì sao?”**  
    Ổ cấp phát mỏng chỉ chiếm chỗ khi ghi; dùng thật dự kiến khoảng 50–70 GB. Nếu `/data` đầy, QEMU tạm dừng VM và cả cụm Ceph đứng (H2), nên nhóm giữ dưới 80% và theo dõi bằng `df -h /data`.

#### **Không được nói**

* “Ceph chạy trên **3 máy chủ**.” Đó là 3 VM trên một laptop.  
* “Hệ thống chịu được hỏng phần cứng.” Hỏng SSD hay hỏng laptop là mất tất cả.  
* “Mạng NAT nên **không ai** truy cập được.” Mọi thứ trên host đều truy cập được, và VM vẫn ra được Internet.  
* “Dữ liệu nhân bản qua **mạng riêng** (cluster network).” Public network và cluster network dùng chung `virbr0` (file AI mục 3.4).  
* “RAM đã đo, đủ dùng.” Chưa đo (D11).

#### **Phát hiện thêm khi vẽ (nói một lần)**

| \# | Vấn đề | Ảnh hưởng | Cách kiểm / đề xuất |
| ----- | ----- | ----- | ----- |
| 1 | **Tổng 8 vCPU so với số luồng CPU thật chưa biết** (Q4). Nhiều i5 dòng H có 4 nhân 8 luồng; khi đó riêng các VM đã chiếm hết, chưa tính host, trình duyệt và virt-manager. | Lúc recovery (UC4) cần CPU để tính lại mảnh EC; nếu CPU bị tranh nhau thì demo chậm, thậm chí OSD báo chậm. | Trên host Kali: `lscpu`, xem `Core(s) per socket` và `Thread(s) per core`. Nếu chỉ có 8 luồng, cân nhắc giảm truenas xuống 1 vCPU (ĐỀ XUẤT). |
| 2 | **Đồng bộ giờ khi demo không có Internet.** chrony trong VM không còn nguồn giờ nào. Sau khi laptop sleep/resume, giờ giữa các VM có thể lệch (H4). MON cảnh báo lệch giờ khi vượt một ngưỡng rất nhỏ (mặc định khoảng 0,05 giây \[cần kiểm: `ceph config help mon_clock_drift_allowed`\]); SigV4 từ chối request khi lệch khoảng 15 phút. | Health WARN khi demo; trường hợp xấu là 403 hàng loạt. | ĐỀ XUẤT: cho host Kali làm máy chủ giờ cho mạng `192.168.122.0/24`, các VM lấy giờ từ `192.168.122.1` \[cần kiểm cấu hình chrony\]. |
| 3 | **SSH 22 không có trong danh sách port ở mục 3.4**, dù cephadm cần SSH từ ceph1 sang ceph2 và ceph3 (T7.9). | Thiếu khi viết tường lửa hoặc khi bị hỏi “các máy nói chuyện qua port nào”. | Thêm “SSH 22 (cephadm)” vào mục 3.4. Tôi không vẽ lên sơ đồ để giữ nó gọn. |
| 4 | **Dashboard 8443 vẽ trên ceph1** vì MGR active đang ở đó. MGR chuyển sang node khác thì dashboard đi theo (xem điểm 5 ở Phần 0). | Mở `https://192.168.122.11:8443` lúc demo có thể không vào được. | `ceph mgr services` cho biết địa chỉ dashboard hiện tại \[cần kiểm\]. |

