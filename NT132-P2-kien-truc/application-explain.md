> Bản 04/11/2026: viết lại theo thiết kế một cụm Ceph (D15) và bố cục mới của slide 7 (sửa 04/11). Sơ đồ: `slide07-application.mmd`. Bản 21/10 (có TrueNAS, Cloud Sync) không còn dùng. Bản cũ thiếu lời nói mẫu, câu hỏi vấn đáp, "Không được nói"; bản này đã bổ sung.

#### **Cách đọc sơ đồ**

Đọc **từ trên xuống, qua 3 khung**:

1. **Khung xanh lá (trên cùng):** client, tức chương trình chủ động gửi yêu cầu.
2. **Khung cam (giữa):** **gateway**, tức "người phiên dịch" đổi giao thức của client sang Ceph: Samba (SMB) và RGW (S3).
3. **Khung xanh dương (dưới cùng):** **lõi Ceph**: MGR, RADOS (MON + OSD), MDS.

3 khung **không ghi tiêu đề là cố ý**: Mermaid luôn đặt tiêu đề ở giữa khung, mà đường nối đi qua đúng chỗ đó và đè lên chữ. Nghĩa của từng khung ghi ở dòng đầu chú thích (dưới sơ đồ). Khi dựng slide, có thể gõ thêm nhãn "Client", "Gateway", "Lõi Ceph" bên cạnh khung bằng công cụ làm slide.

Ký hiệu:

* **Mũi tên** đi từ bên gọi tới bên được gọi.
* **Chữ trên mũi tên** là *giao diện* (interface).
* **Nét đứt xám** là phần Should (IAM API, UC4 tầng 2).

Sau đó lần theo **4 con đường**:

* **File (UC1):** smbclient → Samba → MDS (metadata) và OSD (dữ liệu).
* **Block (UC2):** rbd/krbd trên Kali → **thẳng** OSD, không qua gateway.
* **Object (UC3–UC6):** client S3 → RGW → OSD.
* **Quản trị:** trình duyệt → MGR (dashboard); ceph CLI, radosgw-admin → MON.

#### **Ôn lại khái niệm nền trước khi vào từng khối**

* **Giao diện (interface):** thỏa thuận về cách hai phần mềm nói chuyện. Có 2 kiểu:
  * *Giao thức mạng*: SMB, HTTP. Hai bên có thể nằm ở hai máy khác nhau.
  * *Thư viện (library)*: librados, libcephfs. Chương trình gọi hàm trong thư viện, thư viện lo phần mạng.
* **REST/HTTP:** mỗi thao tác là một request HTTP tới một URL. Ví dụ: `PUT /nhom2-data/baocao.pdf` là ghi, `GET` cùng URL là đọc, `DELETE` là xóa. Bucket và key nằm ngay trong đường dẫn (path-style, gotcha S4).
* **SigV4:** client dùng secret key tính một chữ ký HMAC trên nội dung request và thời điểm gửi. Server tính lại rồi so sánh. **Secret key không bao giờ đi qua mạng.** Nhưng chú ý: SigV4 là *ký*, **không phải mã hóa**. Nội dung file vẫn đi dạng rõ qua HTTP.
* **SMB2/3:** giao thức chia sẻ file kiểu Windows (cái bạn dùng khi mở `\\server\share`). Samba do Ceph quản lý chỉ hỗ trợ SMB2 và SMB3; không có SMB1, bản có lỗ hổng EternalBlue/WannaCry.
* **Ổ khối (block device):** hệ điều hành thấy một ổ trống, ví dụ `/dev/rbd0`, rồi tự format (ví dụ ext4) và mount như ổ thật. **krbd** là driver RBD có sẵn trong kernel Linux; lệnh `rbd map` dùng nó để gắn một image RBD thành ổ.
* **Metadata và dữ liệu:** metadata là thông tin *về* file (tên, thư mục, quyền, kích thước); dữ liệu là *nội dung* file. CephFS tách hai thứ này: MDS giữ metadata, OSD giữ dữ liệu.

#### **Giải thích từng khối**

| Khối | Là gì | Ví dụ đời thường |
| --- | --- | --- |
| **smbclient** | Công cụ dòng lệnh nói SMB, dùng thay cho File Explorer khi demo. | File Explorer bản gõ lệnh. |
| **rbd + krbd** (ổ `/dev/rbd0`) | `rbd` là lệnh quản lý image RBD. `rbd map` nhờ krbd trong kernel Kali gắn image thành ổ `/dev/rbd0`; sau đó Kali dùng như ổ thật (`mkfs.ext4`, `mount`). Nói **thẳng** với OSD, không qua gateway. Kali dùng một user Ceph riêng, chỉ có quyền trên pool `rbd` (gotcha K1). | Cắm ổ cứng USB, nhưng ổ nằm ở kho bên kia mạng. |
| **Client S3** (aws cli, rclone, boto3/curl) | Ba công cụ khác nhau, cùng nói S3 với cùng một RGW (chứng minh B2). Lưu ý: **boto3 chỉ tính ra presigned URL ngay trên máy client, không gửi gì tới RGW**; curl mới là bên gửi request GET. Mũi tên nét đứt IAM API là phần Should (UC4 tầng 2). | Ba hãng điện thoại khác nhau cùng gọi vào một tổng đài. |
| **Trình duyệt (dashboard)** | Mở dashboard của Ceph, một module nằm trong MGR, qua HTTPS (cổng 8443). | Màn hình camera của phòng điều hành. |
| **ceph CLI / radosgw-admin** | Công cụ quản trị. Đi **thẳng vào RADOS qua librados**, không qua S3, nên **không bị kiểm SigV4 hay bucket policy**. Chỉ dùng được trên máy có keyring admin (tệp chứa khóa quản trị). Mũi tên trỏ vào MON vì MON nhận lệnh quản trị cụm; radosgw-admin còn đọc/ghi thẳng các object cấu hình của RGW trên OSD. Ví dụ: `radosgw-admin user create`, `quota set`, `ceph -s`. | Chìa khóa vạn năng của quản lý tòa nhà, không cần quẹt thẻ ở cửa. |
| **Samba (module smb)** | **Gateway file:** nhận lệnh SMB, đổi thành thao tác trên CephFS qua thư viện libcephfs: hỏi MDS về metadata, ghi dữ liệu thẳng tới OSD. Chạy trong container do module `smb` (nằm trong MGR) dựng lên qua cephadm. Đây là gateway nghĩa (b) ở D17. | Quầy tiếp nhận hồ sơ giấy. |
| **RGW (RADOS Gateway)** | **Gateway object:** nhận request S3/IAM qua HTTP, kiểm chữ ký và quyền, rồi đổi thành thao tác trên RADOS qua librados. Đây là gateway nghĩa (a) ở D17. | Người phiên dịch ở quầy giao dịch: khách nói "S3", kho chỉ hiểu "RADOS". |
| **MGR (manager)** | Phòng điều hành, **không nằm trên đường đi của dữ liệu**. Ba module chính: dashboard (màn hình giám sát), smb (lo việc dựng và cấu hình Samba), cephadm (bộ phận triển khai, đưa daemon lên từng máy, slide 10). | Phòng điều hành có màn hình camera và bộ phận điều phối nhân sự. |
| **RADOS · MON** | Giữ **bản đồ cụm** (cluster map: có những OSD nào, đang sống hay chết, dữ liệu nào nằm ở đâu theo CRUSH). Cần đa số (2 trên 3) đồng ý mới cập nhật được bản đồ. | Ban quản lý giữ sơ đồ kho. |
| **RADOS · OSD** | Mỗi OSD quản lý một ổ đĩa: lưu dữ liệu của **cả ba kiểu** (object, file, block), chia mảnh EC, phục hồi khi có máy hỏng. | Thủ kho của từng kệ. |
| **MDS** | Metadata Server của CephFS: giữ "mục lục" (tên file, thư mục, quyền), **không** giữ nội dung file. Chỉ phần file (UC1) cần MDS; block và object không dùng. | Thủ thư giữ mục lục thư viện; sách vẫn nằm trên kệ. |
| **librados / libcephfs** (chữ trên mũi tên) | Thư viện. librados hỏi MON lấy bản đồ, **tự tính vị trí** (CRUSH) và **nói thẳng với OSD**. libcephfs là thư viện của CephFS, dựng trên librados, thêm phần nói chuyện với MDS. | Có sơ đồ kho trong tay thì tự đi thẳng tới kệ, không phải hỏi ban quản lý mỗi lần lấy hàng. |

**Vì sao sơ đồ không vẽ mũi tên MDS → RADOS và MGR → RADOS?** MDS và MGR cũng lưu trạng thái của mình vào RADOS, nhưng vẽ thêm 2 mũi tên này làm lõi Ceph bị đẩy xuống một tầng, đường nối dài và rối hơn (bố cục sửa 04/11). Ý này nói miệng khi cần.

#### **Lời nói mẫu khi thuyết trình (khoảng 75 giây)**

> "Slide trước là thực đơn; slide này mở chiếc hộp ra xem bên trong có những phần mềm nào. Đọc từ trên xuống qua ba khung.
>
> Khung xanh lá trên cùng là client: smbclient cho file, lệnh rbd cho ổ đĩa, aws cli và rclone cho S3, cùng các công cụ quản trị.
>
> Khung cam ở giữa là gateway, tức người phiên dịch. Samba nhận giao thức SMB và dịch sang CephFS. RGW nhận S3 và dịch sang RADOS. Đây chính là chữ *storage gateway* trong tên đề tài.
>
> Khung xanh dương dưới cùng là lõi Ceph. RADOS gồm MON giữ bản đồ cụm và OSD giữ dữ liệu. MDS giữ mục lục cho file. MGR là phòng điều hành.
>
> Chú ý ba điều. Một: file, block và object cuối cùng đều ghi vào cùng các OSD, một kho cho ba kiểu. Hai: block không cần gateway, lệnh rbd trên Kali nói thẳng với OSD. Ba: công cụ quản trị đi thẳng vào RADOS, không qua S3, nên chỉ máy có khóa admin mới dùng được.
>
> Slide sau sẽ cho thấy một request cụ thể đi qua các khối này như thế nào."

#### **3 câu hỏi vấn đáp dễ gặp**

1. **"Block không qua gateway thì ai đổi ổ đĩa thành object?"**
   Thư viện phía client: krbd trong kernel Kali chia ổ thành nhiều object RADOS (mặc định mỗi object 4 MiB [docs; kiểm bằng `rbd info`]), tự tính vị trí bằng CRUSH rồi gửi thẳng tới OSD. Không có daemon trung gian nên không có điểm nghẽn ở giữa. Đổi lại, máy client phải có khóa Ceph; vì vậy Kali chỉ được cấp một user riêng có quyền trên pool `rbd` (K1).
2. **"Dữ liệu file có đi qua MDS không? MDS có phải điểm nghẽn?"**
   Không. MDS chỉ trả lời câu hỏi về metadata (file tên gì, nằm trong thư mục nào, ai có quyền); nội dung file đi thẳng từ Samba tới OSD. Lab chạy 2 MDS (một active, một standby) để MDS active hỏng thì bản standby nhận thay (gotcha F3).
3. **"ceph CLI / radosgw-admin có bị bucket policy chặn không?"**
   Không: chúng đi thẳng librados, không qua S3/SigV4. Chỉ chạy được trên máy có keyring admin. Vì vậy keyring admin phải giữ kỹ, không chép sang máy client.

#### **Không được nói**

* "Samba (hoặc RGW) **lưu** dữ liệu." Gateway chỉ dịch giao thức; dữ liệu nằm ở OSD.
* "Dữ liệu file đi **qua MDS**." MDS chỉ giữ metadata.
* "Block cũng **đi qua gateway**." RBD nói thẳng với OSD.
* "MON hoặc MGR **tham gia đọc/ghi dữ liệu**." MON chỉ đưa bản đồ cụm; MGR là phòng điều hành.
* "SigV4 **mã hóa** nội dung." SigV4 chỉ ký; muốn mã hóa đường truyền phải có TLS (D14).
* "Module smb **chính là** Samba." Module `smb` nằm trong MGR, chỉ dựng và cấu hình container Samba.
