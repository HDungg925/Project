> Bản 04/11/2026: viết lại theo thiết kế một cụm Ceph (D15), 6 UC (D16). Sơ đồ: `slide08-integration.mmd`. Luồng 1 viết mới (ghi file qua SMB, thay cho Cloud Sync cũ); Luồng 2, 3 giữ ý cũ, đổi số UC (UC3 cũ → UC4, UC4 cũ → UC5).

#### **Cách đọc sơ đồ**

1. Đọc tiêu đề, rồi đi **từ trái sang phải qua 3 làn**. **Trong mỗi làn, đọc từ trên xuống.**
2. Mũi tên là **chiều request hoặc dữ liệu đi**. Lưu ý: khác slide 7, nơi mũi tên là "ai gọi ai".
3. Chữ trên mũi tên là giao diện, **thứ tự bước** (1, 2) hoặc điều kiện rẽ nhánh (sai/đúng, có quyền/không có quyền).
4. Màu ô giữ nguyên quy ước: xanh lá = client, **cam = gateway** (Samba, RGW), xanh dương = lõi Ceph.
5. **Ô trắng** không phải thành phần đang chạy. Nó là kết quả (403, được phép) hoặc máy đã tắt.
6. Mũi tên có **đầu ✕** nghĩa là thành phần đó không được dùng tới. Dấu ✕ khá nhỏ khi chiếu, nên nhãn "bỏ qua (down)" và chữ "ĐÃ TẮT" mới là thứ truyền nghĩa chính.

#### **Ôn lại khái niệm nền trước khi vào từng luồng**

| Khái niệm | Giải thích | Ví dụ đời thường |
| --- | --- | --- |
| **PG (placement group)** | Ceph không nhớ vị trí từng object. Nó băm (hash) tên object để ra số PG, rồi dùng CRUSH tính ra danh sách OSD giữ PG đó. Danh sách này gọi là **acting set**. | Kho chia hàng vào các "thùng" theo mã số; mỗi thùng có sẵn danh sách thủ kho phụ trách. |
| **OSD chính (primary)** | OSD đứng đầu acting set. **Mọi đọc/ghi của PG đều đi qua nó.** Client chỉ nói chuyện với OSD chính, không nói với các OSD còn lại. | Trưởng nhóm nhận việc rồi tự chia cho các thành viên. |
| **Bản sao (replica)** | Pool replicated ×3: OSD chính ghi 1 bản và gửi 2 bản y hệt cho 2 OSD khác. Dùng được 1/3 dung lượng. Luồng 1 (file) dùng kiểu này. | Photo thêm 2 bản cất ở 2 tủ khác. |
| **Mảnh (shard)** | EC 2+1: OSD chính cắt dữ liệu thành 2 mảnh data, tính 1 mảnh parity (XOR, xem ví dụ ở file AI mục 6). Nó giữ 1 mảnh và gửi 2 mảnh cho 2 OSD còn lại. Dùng được 2/3 dung lượng. Luồng 3 (object) dùng kiểu này. | 2 bạn mỗi người chép nửa bài, bạn thứ 3 ghi "tổng" để dựng lại khi mất một nửa. |
| **Metadata và MDS** | Metadata là thông tin *về* file (tên, thư mục, quyền). Trong CephFS, MDS giữ metadata và lưu nó vào pool metadata; nội dung file nằm ở pool data. | Thủ thư ghi mục lục; sách vẫn đặt trên kệ. |
| **"down"** | OSD không gửi tín hiệu sống (heartbeat) trong khoảng 20–30 giây thì MON đánh dấu nó là down, và PG loại nó khỏi acting set (gotcha E5). | Thủ kho nghỉ không báo; sau một lúc ban quản lý gạch tên khỏi danh sách trực. |

#### **Giải thích từng luồng**

**Luồng 1 · UC1: ghi file qua SMB**

| Bước | Chuyện gì xảy ra |
| --- | --- |
| smbclient → Samba | Người dùng chép `baocao.pdf` vào thư mục chia sẻ qua SMB2/3. Samba (chạy trên ceph2) xác thực người dùng SMB rồi nhận lệnh. Samba là **gateway**: đổi lệnh SMB thành thao tác trên CephFS. |
| Bước 1: Samba → MDS | Samba (qua thư viện libcephfs) xin MDS tạo file. MDS ghi mục mới vào "mục lục" (tên file, thư mục, quyền), cho phép Samba ghi, và lưu metadata này vào **pool metadata** (replicated ×3). |
| Bước 2: Samba → OSD chính | Samba chia nội dung file thành các object, tự tính vị trí bằng CRUSH rồi gửi **thẳng tới OSD chính** của từng PG trong **pool data** của CephFS. Dữ liệu **không đi qua MDS**. |
| OSD chính → 2 OSD còn lại | Pool data là replicated ×3 (ĐỀ XUẤT, file AI mục 3.5): OSD chính ghi 1 bản, gửi 2 bản sao. Chỉ khi đủ 3 bản mới báo xong; sau đó Samba mới báo thành công cho smbclient. |

Điểm chính: **Samba nằm trên đường đi của dữ liệu**. Chép file xong là dữ liệu đã nằm trong Ceph, đủ 3 bản. Thiết kế cũ (Cloud Sync) khác hẳn: file nằm trên NAS, phải chờ tới lúc bấm Run Now hoặc tới lịch thì mới được chép sang kho object.

**Luồng 2 · UC4: xác thực trước, phân quyền sau**

| Bước | Chuyện gì xảy ra |
| --- | --- |
| Xác thực (authentication, "bạn là ai") | RGW tìm user theo access key, lấy secret key đã lưu, tự tính lại chữ ký và so với chữ ký client gửi. Sai thì trả **403** trước khi xét quyền. Mã lỗi thường gặp: `SignatureDoesNotMatch` (sai chữ ký), `InvalidAccessKeyId` (không có key này), `RequestTimeTooSkewed` (lệch giờ, gotcha S7). |
| Phân quyền (authorization, "bạn được làm gì") | RGW xét bucket policy: có **Deny tường minh thì luôn bị cấm**; có Allow thì được phép. Không có Allow thì user thường còn được xét theo ACL, nên chủ bucket vẫn có quyền. Vì vậy nhãn trên sơ đồ là "có quyền / không có quyền", không ghi "Allow / không Allow". Với `alice-ro` PUT: policy chỉ cho Get và List, cô ấy không phải chủ bucket, nên nhận **403 AccessDenied**. |

Ví dụ đời thường: bảo vệ tòa nhà kiểm **thẻ** trước (thẻ thật hay giả), rồi mới xem thẻ đó **được vào tầng nào**. Hai lần từ chối đều là "403", nhưng vì hai lý do khác nhau.

**Luồng 3 · UC5: đọc khi ceph3 tắt**

| Bước | Chuyện gì xảy ra |
| --- | --- |
| aws cli GET → RGW → OSD chính | Giống đọc bình thường. Nếu OSD chính cũ nằm trên ceph3, PG đã chọn OSD chính mới trong số OSD còn sống. |
| "bỏ qua (down)" | OSD trên ceph3 đã bị đánh dấu down và loại khỏi acting set. OSD chính **không gửi yêu cầu tới nó rồi ngồi chờ**; nó biết trước là không dùng được. |
| Gom 2 mảnh, dựng lại | Cần đủ k = 2 mảnh. Nếu mảnh mất là parity, 2 mảnh data còn nguyên, không phải tính gì. Nếu mảnh mất là data, OSD chính dùng mảnh còn lại và parity để tính lại (XOR). |
| Client nhận file, md5 khớp | Bằng chứng cho B1 trong demo UC5 (bước UC5-D5). |

#### **Lời nói mẫu khi thuyết trình (khoảng 90 giây)**

> "Slide trước cho biết có những thành phần nào; slide này cho thấy chúng phối hợp ra sao, qua ba tình huống. Mỗi làn đọc từ trên xuống.
>
> Làn thứ nhất: nhân viên chép một file vào thư mục chia sẻ. Lệnh SMB tới Samba, chính là gateway. Samba làm hai bước. Bước một: hỏi MDS để tạo tên file và quyền trong mục lục. Bước hai: gửi nội dung file thẳng tới OSD chính, không đi qua MDS. OSD chính ghi một bản và gửi hai bản sao cho hai OSD khác, ghi xong mới báo thành công. Nghĩa là chép file xong thì dữ liệu đã nằm trong cụm Ceph, đủ ba bản.
>
> Làn thứ hai là bước kiểm tra khi gọi S3. Đầu tiên là xác thực: RGW tính lại chữ ký, sai là 403 ngay. Đúng thì mới tới phân quyền: xét policy, không có quyền là 403 AccessDenied. Đây là cảnh demo ở UC4.
>
> Làn thứ ba: ceph3 đã tắt. OSD chính biết ceph3 đã bị đánh dấu down nên bỏ qua, lấy mảnh từ OSD còn sống, đủ hai mảnh là dựng lại được file. Client nhận đúng file, md5 khớp. Đây là cảnh demo ở UC5."

#### **3 câu hỏi vấn đáp dễ gặp**

1. **"Samba có phải gateway thật không? Khác Cloud Sync trong thiết kế cũ chỗ nào?"**
   Có. Gateway nằm *trên đường đi* của mỗi request: mỗi lệnh ghi qua SMB được Samba đổi ngay thành thao tác trên CephFS. Cloud Sync cũ thì chạy theo lô, file nằm trên NAS cho tới khi task chạy. Hệ quả: Samba (trên ceph2) tắt là mất SMB ngay. Đó là SPOF nhóm đã biết (tài liệu nhóm T5.6).
2. **"Khi tắt ceph3, client có bị gián đoạn không?"**
   Có thể có một khoảng ngắn: trong khoảng 20–30 giây trước khi OSD bị đánh dấu down, request chạm vào các PG có mảnh trên ceph3 có thể phải chờ. Sau đó I/O chạy tiếp ở trạng thái degraded (thiếu dự phòng). Con số chính xác phải đo khi chạy thử (E5); đó cũng là lý do demo chờ khoảng 30 giây rồi mới chạy `ceph -s`.
3. **"Sao file dùng 3 bản sao, còn object lại dùng erasure coding?"**
   Pool data mặc định của CephFS nên là replicated; EC chỉ nên làm pool data phụ [docs, chưa kiểm lại trên Tentacle]. Nhóm giữ EC ở một chỗ (dữ liệu của RGW) để UC5 dễ giải thích. Đổi lại, file tốn gấp 3 dung lượng, còn object chỉ gấp 1,5.

#### **Không được nói**

* "Dữ liệu file đi **qua MDS**." MDS chỉ giữ metadata.
* "Samba (hoặc RGW) gửi 3 bản (3 mảnh) tới 3 OSD." Sai: gateway chỉ gửi tới OSD chính; OSD chính tự chép bản sao hoặc chia mảnh rồi phân phối.
* "OSD chính hỏi OSD đã tắt rồi chờ hết thời gian." Sai: OSD đã bị đánh dấu down thì bị loại khỏi acting set.
* "Đọc khi thiếu node **luôn** phải giải mã." Chỉ khi mảnh mất là mảnh data.
* "Tắt node mà **không có gián đoạn nào**." Chưa đo; có thể có khoảng chờ ngắn trước khi OSD bị đánh dấu down.
* "403 nghĩa là sai mật khẩu." Có hai loại 403: xác thực thất bại và không có quyền.
* "File trên thư mục chia sẻ cũng được bảo vệ bằng **erasure coding**." Pool data của CephFS là replicated ×3 (ĐỀ XUẤT).
* "MON tham gia vào việc đọc/ghi."
