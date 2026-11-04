> Bản 04/11/2026: viết lại theo thiết kế một cụm Ceph (D15), 6 UC (D16). Sơ đồ: `slide06-business.mmd`. Bản 21/10 (có TrueNAS, 5 UC) không còn dùng.

#### **Cách đọc sơ đồ**

1. Đọc **tiêu đề** trước, rồi tới **khung xám ở giữa**: đó là ranh giới của nền tảng, ghi rõ "một cụm Ceph". Cái gì ở trong khung là nền tảng làm; cái gì ở ngoài là người hoặc hệ thống bên ngoài.
2. Nhìn **bên trái**: 3 nhóm dùng nền tảng (người dùng văn phòng, máy chủ cần ổ đĩa, ứng dụng). Nhìn **bên phải**: người vận hành (quản trị viên).
3. Đọc **từng ô bo tròn từ trên xuống** (UC1 → UC6): dòng 1 là *việc gì*, dòng 2 là *năng lực nghiệp vụ nào* (CAP) và *lợi ích nào* (B1–B3, tra ở ô chú thích).
4. **Đường nối không có mũi tên** nghĩa là "tác nhân này tham gia use case này". Nó **không** phải luồng dữ liệu; luồng dữ liệu là việc của slide 8.
5. **Mọi ô UC cùng một màu xanh**: cả 6 do cùng một cụm Ceph phục vụ. (Bản cũ tô UC1 màu cam vì do TrueNAS làm.)

#### **Giải thích từng khối cho người mới**

**Trước hết, Business Architecture là gì?** Hình dung 5 slide kiến trúc như mô tả một **nhà hàng**:

* *Business* = thực đơn và khách hàng: nhà hàng bán món gì, cho ai.
* *Application* = các bộ phận: bếp, quầy, thu ngân.
* *Integration* = đường đi của một order từ bàn tới bếp rồi quay lại.
* *Infrastructure* = tòa nhà, điện, nước.
* *Deployment* = ai đứng ở bếp nào.

Slide 6 chỉ là "thực đơn": **không có VM, IP, port**.

| Khối | Giải thích | Ví dụ đời thường |
| --- | --- | --- |
| **Tác nhân (actor)** | Người hoặc hệ thống **bên ngoài** tương tác với nền tảng. Ứng dụng hay máy chủ cũng là tác nhân, dù không phải người. | Khách hàng của nhà hàng. |
| **Khung "Nền tảng lưu trữ (một cụm Ceph)"** | Ranh giới hệ thống (system boundary). Bên trong là thứ nhóm xây và chịu trách nhiệm. Chữ "một cụm" là ý chính: cả 6 UC dùng chung một kho. | Một căn bếp nấu cho cả quán. |
| **Use case (ô bo tròn)** | Một việc có ý nghĩa mà tác nhân làm được *nhờ* nền tảng, có kết quả kiểm chứng được. | Một món trên thực đơn. |
| **Business capability (CAPn)** | Năng lực lâu dài của nền tảng, không phụ thuộc công nghệ. Use case là *kịch bản dùng* năng lực đó. Một UC có thể dùng nhiều capability: UC5 dùng CAP5 (bảo vệ dữ liệu) và CAP6 (mở rộng). Bảng đầy đủ: tài liệu nhóm T2.1. | "Nhà hàng có khả năng nướng" (capability) so với "món gà nướng" (use case). |
| **Benefit (B1–B3)** | Tổ chức được gì khi triển khai. Mỗi benefit phải có UC chứng minh trong demo. | "Khách ăn no, không đau bụng". |
| **Máy chủ cần ổ đĩa** (tác nhân mới) | Một hệ thống cần một ổ đĩa riêng qua mạng, ví dụ máy chủ cơ sở dữ liệu. Trong demo, host Kali đóng vai này. | Thuê thêm một ổ cứng, nhưng ổ nằm ở kho bên kia mạng. |
| **UC1 Chia sẻ file qua SMB** | Người dùng mở thư mục chia sẻ (như ổ `Z:\` ở công ty) và chép file. File nằm trong CephFS của cụm. UC này thay vai trò NAS của TrueNAS. | Tủ hồ sơ chung của cả phòng. |
| **UC2 Cấp ổ đĩa khối** | Cụm cấp một "ổ đĩa ảo" (image RBD); máy chủ gắn vào rồi tự format như ổ thật. Nhóm làm ở mức ngắn, vì Project 1 làm sâu phần này. | Cửa hàng cho thuê ổ cứng. |
| **UC3 Truy cập S3 nhiều client** | Một endpoint S3, nhiều công cụ khác nhau (aws cli, rclone) dùng chung. | Ổ cắm chuẩn: phích của hãng nào cũng cắm được. |
| **UC4 Phân quyền IAM** | Mỗi người chỉ làm đúng việc được cấp (least privilege = quyền tối thiểu cần thiết). User chỉ-đọc ghi vào thì bị 403. | Thẻ ra vào chỉ mở được tầng của mình. |
| **UC5 Chịu lỗi EC, thêm node** | Dữ liệu chia 2 mảnh + 1 mảnh parity (mảnh kiểm tra) trên 3 máy; mất 1 máy vẫn dựng lại được. Thêm 1 máy thì cụm tự dời bớt dữ liệu sang, tổng dung lượng tăng mà không phải dừng dịch vụ. | 2 bạn chép bài, bạn thứ 3 ghi "tổng"; mất một bạn thì suy ra từ 2 người còn lại (ví dụ XOR ở file AI mục 6). Kho thêm kệ mới, thủ kho tự dời bớt hàng sang. |
| **UC6 Ngưỡng và giới hạn** | Quota (hạn mức số lượng/dung lượng), rate limit (tốc độ thao tác), ngưỡng đầy của cụm. | Hạn mức thẻ tín dụng; barie giới hạn số xe qua mỗi phút; đèn báo xăng sắp cạn. |
| **Màu xanh của các ô UC** | Tất cả cùng màu: một cụm Ceph phục vụ cả 6 việc. Slide này không gợi ý công nghệ nào khác. | — |

#### **Lời nói mẫu khi thuyết trình (khoảng 75 giây)**

> "Slide này trả lời một câu: nền tảng của nhóm phục vụ ai, và làm được gì cho họ. Ở đây chưa có máy móc gì cả.
>
> Bên trái là ba nhóm người dùng. Nhân viên văn phòng quen làm việc với thư mục chia sẻ. Máy chủ ứng dụng cần một ổ đĩa riêng. Và ứng dụng hoặc lập trình viên làm việc qua S3. Bên phải là quản trị viên, người cấu hình và vận hành.
>
> Khung ở giữa là nền tảng: một cụm Ceph duy nhất, gồm sáu use case. Ba use case đầu là ba kiểu lưu trữ: file, block và object, cùng một cụm phục vụ cả ba. Đó là B2: một nền tảng mở cho mọi kiểu dữ liệu.
>
> UC4 và UC6 là kiểm soát: ai được làm gì, và mỗi người được dùng bao nhiêu. Đó là B3.
>
> UC5: hỏng một máy, dữ liệu vẫn đọc được; thêm một máy, cụm tự mở rộng. Đó là B1, chống mất dữ liệu, và cũng là B2.
>
> Như vậy cả ba lợi ích ở slide 3 đều có use case chứng minh, và cả sáu use case đều có demo ở phần sau. Slide tiếp theo sẽ mở chiếc hộp này ra xem bên trong có những phần mềm nào."

#### **3 câu hỏi vấn đáp dễ gặp**

1. **"Capability khác use case thế nào?"**
   Capability là *năng lực* của nền tảng, ổn định, không gắn công nghệ. Use case là *một kịch bản* tác nhân dùng các năng lực đó để đạt mục tiêu. Ví dụ UC5 dùng 2 capability: bảo vệ dữ liệu khi hỏng phần cứng (CAP5) và mở rộng bằng cách thêm máy (CAP6). Bảng đầy đủ ở tài liệu nhóm T2.1.
2. **"Đề có TrueNAS, sao sơ đồ không còn? Ai làm phần thư mục chia sẻ?"**
   Thầy xác nhận ngày 28/10 chỉ cần Ceph. Phần NAS do Ceph tự làm: CephFS là filesystem của Ceph, còn Samba (do module `smb` của Ceph dựng lên) là gateway cho người dùng truy cập qua SMB. Nhờ vậy một cụm phục vụ cả file, block và object, thay vì hai hệ thống riêng.
3. **"UC2 block có gì khác Project 1?"**
   Project 1 dùng RBD làm ổ đĩa cho máy ảo (Proxmox) và đi sâu vào phần đó. Nhóm chỉ demo ngắn (tạo một ổ, gắn vào máy, ghi file) để chứng minh một cụm đủ cả ba kiểu lưu trữ. Trọng tâm của nhóm là object (S3), phân quyền IAM, erasure coding và mở rộng cụm.

#### **Không được nói**

* "Nền tảng **không bao giờ** mất dữ liệu." Cả cụm nằm trên 1 laptop; EC 2+1 mất 2 node là mất dữ liệu (tài liệu nhóm T1.5; file AI gotcha E1).
* "Tương thích **100%** với AWS S3." Chỉ nói về những API đã demo.
* "Ceph thay được NAS thương mại **về mọi mặt**." Module `smb` của Ceph Tentacle còn đang phát triển (gotcha F1); chỉ nói những gì đã demo.
* "Thêm máy là **nhanh hơn ngay**" hoặc "mở rộng **vô hạn**." Demo chỉ cho thấy dung lượng tăng và dữ liệu tự dàn đều; hiệu năng không đo (ngoài phạm vi).
* "Rate limit **chống được DoS**." Nó chỉ giới hạn theo user/bucket, trên từng RGW.
* Đọc đường nối là "dữ liệu chạy từ A sang B". Đường nối ở đây chỉ là quan hệ tham gia.
