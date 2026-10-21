#### **Cách đọc sơ đồ**

1. Đọc tiêu đề, rồi đi **từ trái sang phải qua 3 làn**. **Trong mỗi làn, đọc từ trên xuống.**  
2. Mũi tên là **chiều request hoặc dữ liệu đi**. Lưu ý: khác slide 7, nơi mũi tên là “ai gọi ai”.  
3. Chữ trên mũi tên là giao diện hoặc điều kiện rẽ nhánh (sai/đúng, có quyền/không có quyền).  
4. Màu ô giữ nguyên quy ước: xanh lá \= client, cam \= TrueNAS, xanh dương \= Ceph.  
5. **Ô trắng** không phải thành phần đang chạy. Nó là kết quả (403, được phép), khoảng chờ (ô lục giác), hoặc máy đã tắt.  
6. Mũi tên có **đầu ✕** nghĩa là thành phần đó không được dùng tới. Dấu ✕ khá nhỏ khi chiếu, nên nhãn “bỏ qua (down)” và chữ “ĐÃ TẮT” mới là thứ truyền nghĩa chính.

#### **Ôn lại khái niệm nền trước khi vào từng luồng**

| Khái niệm | Giải thích | Ví dụ đời thường |
| ----- | ----- | ----- |
| **PG (placement group)** | Ceph không nhớ vị trí từng object. Nó băm (hash) tên object để ra số PG, rồi dùng CRUSH tính ra danh sách OSD giữ PG đó. Danh sách này gọi là **acting set**. | Kho chia hàng vào các “thùng” theo mã số; mỗi thùng có sẵn danh sách thủ kho phụ trách. |
| **OSD chính (primary)** | OSD đứng đầu acting set. **Mọi đọc/ghi của PG đều đi qua nó.** Client chỉ nói chuyện với OSD chính, không nói với các OSD còn lại. | Trưởng nhóm nhận việc rồi tự chia cho các thành viên. |
| **Mảnh (shard)** | EC 2+1: OSD chính cắt dữ liệu thành 2 mảnh data, tính 1 mảnh parity (XOR, xem ví dụ ở file AI mục 6). Nó giữ 1 mảnh và gửi 2 mảnh cho 2 OSD còn lại. | 2 bạn mỗi người chép nửa bài, bạn thứ 3 ghi “tổng” để dựng lại khi mất một nửa. |
| **Bucket index / omap** | Bucket index là **mục lục** của bucket (có object nào, kích thước, ETag), giúp `aws s3 ls` chạy nhanh. Nó nằm trong **omap**, một kho key-value nhỏ gắn với RADOS object. **Pool EC không hỗ trợ omap**, nên pool index phải là replicated (gotcha E2). | Mục lục cuốn sách, tách riêng khỏi nội dung. |
| **“down”** | OSD không gửi tín hiệu sống (heartbeat) trong khoảng 20–30 giây thì MON đánh dấu nó là down, và PG loại nó khỏi acting set (gotcha E5). | Thủ kho nghỉ không báo; sau một lúc ban quản lý gạch tên khỏi danh sách trực. |

#### **Giải thích từng luồng**

**Luồng 1 · UC1: file thành object**

| Bước | Chuyện gì xảy ra |
| ----- | ----- |
| smbclient → Samba → ZFS | Người dùng chép `baocao.pdf` vào `outbox/` qua SMB2/3. Samba ghi file xuống ZFS. **Lúc này file chưa hề có trong Ceph.** |
| Ô lục giác “chờ” | Không có gì chạy cho tới khi bấm Run Now hoặc tới lịch. Đây là bằng chứng cho D6: Cloud Sync không nằm trên đường đi của dữ liệu. Nếu RGW đang hỏng, chép file qua SMB vẫn thành công. |
| Cloud Sync đọc `outbox/` → PUT S3 | rclone liệt kê thư mục, so với bucket rồi chép file mới. Mỗi request được ký bằng SigV4. File lớn thì client chia thành multipart (gotcha S2). |
| RGW “kiểm như Luồng 2” | Xác thực rồi phân quyền. `truenas-sync` là chủ bucket `nas-backup` nên được phép. |
| RGW → pool index / OSD chính | RGW cập nhật mục lục (pool replicated ×3) và ghi dữ liệu. Với cả hai, RGW chỉ gửi tới **OSD chính** của PG tương ứng. |
| OSD chính → 2 OSD còn lại | OSD chính chia 2 data \+ 1 parity, giữ 1 mảnh, gửi 2 mảnh. Nó chỉ báo xong cho RGW khi các mảnh đã ghi; sau đó RGW mới trả 200 cho Cloud Sync. |

**Luồng 2 · UC3: xác thực trước, phân quyền sau**

| Bước | Chuyện gì xảy ra |
| ----- | ----- |
| Xác thực (authentication, “bạn là ai”) | RGW tìm user theo access key, lấy secret key đã lưu, tự tính lại chữ ký và so với chữ ký client gửi. Sai thì trả **403** trước khi xét quyền. Mã lỗi thường gặp: `SignatureDoesNotMatch` (sai chữ ký), `InvalidAccessKeyId` (không có key này), `RequestTimeTooSkewed` (lệch giờ, gotcha S7). |
| Phân quyền (authorization, “bạn được làm gì”) | RGW xét bucket policy: có **Deny tường minh thì luôn bị cấm**; có Allow thì được phép. Không có Allow thì user thường còn được xét theo ACL, nên chủ bucket vẫn có quyền. Vì vậy nhãn trên sơ đồ là “có quyền / không có quyền”, không ghi “Allow / không Allow”. Với `alice-ro` PUT: policy chỉ cho Get và List, cô ấy không phải chủ bucket, nên nhận **403 AccessDenied**. |

Ví dụ đời thường: bảo vệ tòa nhà kiểm **thẻ** trước (thẻ thật hay giả), rồi mới xem thẻ đó **được vào tầng nào**. Hai lần từ chối đều là “403”, nhưng vì hai lý do khác nhau.

**Luồng 3 · UC4: đọc khi ceph3 tắt**

| Bước | Chuyện gì xảy ra |
| ----- | ----- |
| aws cli GET → RGW → OSD chính | Giống đọc bình thường. Nếu OSD chính cũ nằm trên ceph3, PG đã chọn OSD chính mới trong số OSD còn sống. |
| “bỏ qua (down)” | OSD trên ceph3 đã bị đánh dấu down và loại khỏi acting set. OSD chính **không gửi yêu cầu tới nó rồi ngồi chờ**; nó biết trước là không dùng được. |
| Gom 2 mảnh, dựng lại | Cần đủ k \= 2 mảnh. Nếu mảnh mất là parity, 2 mảnh data còn nguyên, không phải tính gì. Nếu mảnh mất là data, OSD chính dùng mảnh còn lại và parity để tính lại (XOR). |
| Client nhận file, md5 khớp | Bằng chứng cho B1 trong demo UC4-D5. |

#### **Lời nói mẫu khi thuyết trình (khoảng 90 giây)**

> “Slide trước cho biết có những thành phần nào; slide này cho thấy chúng phối hợp ra sao, qua ba tình huống. Mỗi làn đọc từ trên xuống.  
>  Làn thứ nhất: nhân viên chép file vào thư mục outbox qua SMB, Samba ghi xuống ZFS. Chú ý ô trắng ở giữa: tới đây không có gì chạy tiếp, cho tới khi quản trị viên bấm Run Now hoặc tới lịch. Đó là lý do nhóm nói Cloud Sync là cầu nối theo lô, không phải gateway nằm trên đường dữ liệu. Khi task chạy, rclone đọc file và gửi lệnh PUT S3 có chữ ký SigV4 tới RGW. RGW làm hai việc: cập nhật mục lục của bucket trong pool replicated, và gửi dữ liệu tới OSD chính của PG. OSD chính tự chia ra 2 mảnh dữ liệu và 1 mảnh parity, giữ một, gửi hai mảnh còn lại cho 2 OSD kia. RGW không gửi thẳng 3 mảnh.  
>  Làn thứ hai là bước kiểm tra mà làn một đi qua. Đầu tiên là xác thực: RGW tính lại chữ ký, sai là 403 ngay. Đúng thì mới tới phân quyền: xét policy, không có quyền là 403 AccessDenied. Đây là cảnh demo ở UC3.  
>  Làn thứ ba: ceph3 đã tắt. OSD chính biết ceph3 đã bị đánh dấu down nên bỏ qua, lấy mảnh từ OSD còn sống, đủ hai mảnh là dựng lại được file. Client nhận đúng file, md5 khớp. Đây là cảnh demo ở UC4.”

#### **3 câu hỏi vấn đáp dễ gặp**

1. **“Sao nói Cloud Sync không phải gateway? Nó cũng chuyển file thành object mà.”**  
    Gateway nằm *trên đường đi* của mỗi request. Ở đây, chép file qua SMB xong là xong, không có gì đi qua Ceph cho tới khi task chạy (ô “chờ”). Tắt RGW thì SMB vẫn chạy, chỉ việc sao lưu bị trễ. Gateway thật sự là RGW (phiên dịch S3 → RADOS). Gateway file→object trực tuyến là NFS export bucket (D9), hiện CHƯA CHỐT.  
2. **“Khi tắt ceph3, client có bị gián đoạn không?”**  
    Có thể có một khoảng ngắn: trong khoảng 20–30 giây trước khi OSD bị đánh dấu down, request chạm vào các PG có mảnh trên ceph3 có thể bị chờ. Sau đó I/O chạy tiếp ở trạng thái degraded (thiếu dự phòng). Con số chính xác phải đo khi chạy thử (E5); đó cũng là lý do demo chờ khoảng 30 giây rồi mới `ceph -s`.  
3. **“Sao không để bucket index trong pool EC cho tiết kiệm?”**  
    Index nằm trong omap, mà pool EC không hỗ trợ omap (E2). Index lại nhỏ, nên replicated ×3 gần như không tốn thêm bao nhiêu. Chỉ phần dữ liệu lớn mới đáng dùng EC.

#### **Không được nói**

* “Cloud Sync đồng bộ **ngay** khi file vào thư mục.” Nó chạy theo lô (Run Now hoặc lịch).  
* “RGW gửi 3 mảnh tới 3 OSD.” Sai: RGW chỉ gửi tới OSD chính; OSD chính chia và phân phối.  
* “OSD chính hỏi OSD đã tắt rồi chờ hết thời gian.” Sai: OSD đã bị đánh dấu down thì bị loại khỏi acting set.  
* “Đọc khi thiếu node **luôn** phải giải mã.” Chỉ khi mảnh mất là mảnh data.  
* “Tắt node mà **không có gián đoạn nào**.” Chưa đo; có thể có khoảng chờ ngắn trước khi OSD bị đánh dấu down.  
* “403 nghĩa là sai mật khẩu.” Có hai loại 403: xác thực thất bại và không có quyền.  
* “MON tham gia vào việc đọc/ghi.”

