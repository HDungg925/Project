#### **Cách đọc sơ đồ**

Đọc **từ trên xuống, theo 3 tầng**:

1. **Trên cùng (xanh lá):** các client, tức chương trình chủ động gửi yêu cầu.  
2. **Giữa:** các dịch vụ nhận yêu cầu. TrueNAS bên trái (cam), Ceph bên phải (xanh dương).  
3. **Dưới cùng:** nơi dữ liệu thật sự nằm: ZFS (TrueNAS) và RADOS (Ceph).

Ký hiệu:

* **Mũi tên** đi từ bên gọi tới bên được gọi.  
* **Chữ trên mũi tên** là *giao diện* (interface).  
* **Khung lồng nhau** nghĩa là “nằm bên trong” (module dashboard nằm trong MGR).  
* **Nét đứt xám** là phần Should.

Sau đó lần theo **4 con đường**:

* **File:** smbclient → Samba → ZFS.  
* **Object:** client S3 → RGW → RADOS.  
* **Cầu nối:** Cloud Sync đọc/ghi ZFS và gọi RGW.  
* **Quản trị:** trình duyệt, ceph CLI.

#### **Ôn lại khái niệm nền trước khi vào từng khối**

* **Giao diện (interface):** thỏa thuận về cách hai phần mềm nói chuyện. Có 2 kiểu:  
  * *Giao thức mạng*: SMB, HTTP. Hai bên có thể nằm ở hai máy khác nhau.  
  * *Thư viện (library)*: librados. Chương trình gọi hàm trong thư viện, thư viện lo phần mạng.  
* **REST/HTTP:** mỗi thao tác là một request HTTP tới một URL. Ví dụ: `PUT /nas-backup/from-nas/baocao.pdf` là ghi, `GET` cùng URL là đọc, `DELETE` là xóa. Bucket và key nằm ngay trong đường dẫn (path-style, gotcha S4).  
* **SigV4:** client dùng secret key tính một chữ ký HMAC trên nội dung request và thời điểm gửi. Server tính lại rồi so sánh. **Secret key không bao giờ đi qua mạng.** Nhưng chú ý: SigV4 là *ký*, **không phải mã hóa**. Nội dung file vẫn đi dạng rõ qua HTTP.  
* **SMB2/3:** giao thức chia sẻ file kiểu Windows (cái bạn dùng khi mở `\\server\share`). SMB1 tắt vì lỗ hổng EternalBlue (gotcha CS6).

#### **Giải thích từng khối**

| Khối | Là gì | Ví dụ đời thường |
| ----- | ----- | ----- |
| **smbclient** | Công cụ dòng lệnh nói SMB, dùng thay cho File Explorer khi demo. | File Explorer bản gõ lệnh. |
| **Client S3** (aws cli, rclone, boto3/curl) | Ba công cụ khác nhau, cùng nói S3 với cùng một RGW (chứng minh B2). Lưu ý: **boto3 chỉ tính ra presigned URL ngay trên máy client, không gửi gì tới RGW**; curl mới là bên gửi request GET. | Ba hãng điện thoại khác nhau cùng gọi vào một tổng đài. |
| **Trình duyệt (quản trị)** | Mở 2 giao diện web: Web UI của TrueNAS và dashboard của Ceph. | — |
| **ceph CLI / radosgw-admin** | Công cụ quản trị. Đi **thẳng vào RADOS qua librados**, không qua S3, nên **không bị kiểm SigV4 hay bucket policy**. Chỉ dùng được trên máy có keyring admin (tệp chứa khóa quản trị). Ví dụ: `radosgw-admin user create`, `quota set`, `ceph -s`. | Chìa khóa vạn năng của quản lý tòa nhà, không cần quẹt thẻ ở cửa. |
| **Samba** | Phần mềm trên TrueNAS nói giao thức SMB: nhận lệnh mở/ghi file rồi ghi xuống ZFS. | Quầy tiếp nhận hồ sơ giấy. |
| **Web UI (TrueNAS)** | Bảng điều khiển để cấu hình pool, share, user, Cloud Sync task. Đằng sau là một tiến trình middleware cấu hình các dịch vụ khác; tôi không vẽ các đường “cấu hình” để sơ đồ đỡ rối. | Bảng điều khiển của máy giặt. |
| **Cloud Sync (rclone)** | Đọc thư mục trên ZFS, rồi đóng vai **một client S3** gửi request tới RGW (và chiều ngược lại cho task PULL). rclone xuất hiện 2 lần trên sơ đồ: một lần là công cụ trên máy client (UC2), một lần nằm bên trong Cloud Sync. Cùng phần mềm, hai vai. | Xe tải chở hàng theo chuyến giữa 2 kho. |
| **ZFS** | Vừa là filesystem vừa là RAID: pool mirror, dataset, snapshot, checksum. | Tủ hồ sơ có 2 ngăn giống hệt nhau và tự phát hiện giấy bị hỏng. |
| **RGW (RADOS Gateway)** | Nhận request S3/IAM qua HTTP, kiểm chữ ký và quyền, rồi chuyển thành thao tác trên RADOS. Đây là “gateway” nghĩa (a) ở D6. | Người phiên dịch / quầy giao dịch: khách nói “S3”, kho chỉ hiểu “RADOS”. |
| **MGR (manager)** | Phòng điều hành, **không nằm trên đường đi của dữ liệu**. Module dashboard là màn hình giám sát; module cephadm là bộ phận triển khai, đưa daemon lên từng máy (slide 10). | Phòng điều hành có màn hình camera và bộ phận điều phối nhân sự. |
| **RADOS** | Lõi lưu trữ của Ceph, gồm MON và OSD. | Nhà kho. |
| **MON** | Giữ **bản đồ cụm** (cluster map: có những OSD nào, đang sống hay chết, dữ liệu nào nằm ở đâu theo CRUSH). Cần đa số (2/3) đồng ý mới cập nhật được bản đồ. | Ban quản lý giữ sơ đồ kho. |
| **OSD** | Mỗi OSD quản lý một ổ đĩa: lưu dữ liệu, chia mảnh EC, phục hồi khi có máy hỏng. | Thủ kho của từng kệ. |
| **librados** | Thư viện mà mọi chương trình muốn dùng RADOS đều phải đi qua. Nó hỏi MON lấy bản đồ, rồi **tự tính vị trí và nói chuyện thẳng với OSD**. MON không nằm trên đường đi của dữ liệu. | Có sơ đồ kho trong tay thì tự đi thẳng tới kệ, không phải hỏi ban quản lý mỗi lần lấy hàng. |

