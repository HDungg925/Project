#### **Cách đọc sơ đồ**

1. Đọc **tiêu đề** trước, rồi tới **khung xám ở giữa**: đó là ranh giới của nền tảng. Cái gì ở trong khung là nền tảng làm; cái gì ở ngoài là người hoặc hệ thống bên ngoài.  
2. Nhìn **bên trái**: người dùng chính (văn phòng, ứng dụng). Nhìn **bên phải**: người vận hành (quản trị viên).  
3. Đọc **từng ô bo tròn từ trên xuống** (UC1 → UC5): dòng 1 là *việc gì*, dòng 2 là *năng lực nghiệp vụ nào* và *lợi ích nào* (B1–B3, tra ở ô chú thích).  
4. **Đường nối không có mũi tên**: nghĩa là “tác nhân này tham gia use case này”. Nó **không** phải luồng dữ liệu. Luồng dữ liệu là việc của slide 8\.

#### **Giải thích từng khối cho người mới**

**Trước hết, Business Architecture là gì?** Hình dung 5 slide kiến trúc như mô tả một **nhà hàng**:

* *Business* \= thực đơn và khách hàng: nhà hàng bán món gì, cho ai.  
* *Application* \= các bộ phận: bếp, quầy, thu ngân.  
* *Integration* \= đường đi của một order từ bàn tới bếp rồi quay lại.  
* *Infrastructure* \= tòa nhà, điện, nước.  
* *Deployment* \= ai đứng ở bếp nào.

Slide 6 chỉ là “thực đơn”: **không có VM, IP, port**.

| Khối | Giải thích | Ví dụ đời thường |
| ----- | ----- | ----- |
| **Tác nhân (actor)** | Người hoặc hệ thống **bên ngoài** tương tác với nền tảng. Ứng dụng cũng là tác nhân, dù không phải người. | Khách hàng của nhà hàng. |
| **Khung “Nền tảng lưu trữ”** | Ranh giới hệ thống (system boundary). Bên trong là thứ nhóm xây và chịu trách nhiệm. | Bức tường nhà hàng. |
| **Use case (ô bo tròn)** | Một việc có ý nghĩa mà tác nhân làm được *nhờ* nền tảng, có kết quả kiểm chứng được. | Một món trên thực đơn. |
| **Business capability (CAPn)** | Năng lực lâu dài của nền tảng, không phụ thuộc công nghệ. Use case là *kịch bản dùng* năng lực đó. Một UC có thể dùng nhiều capability (UC1 dùng cả CAP1, CAP2, CAP3). | “Nhà hàng có khả năng nướng” (capability) so với “món gà nướng” (use case). |
| **Benefit (B1–B3)** | Tổ chức được gì khi triển khai. Mỗi benefit phải có UC chứng minh trong demo. | “Khách ăn no, không đau bụng”. |
| **UC1 File SMB ↔ Object S3** | File bỏ vào thư mục `outbox/` được sao lên bucket; object trong `to-nas/` được kéo về `inbox/`. Chạy theo lô, 2 task COPY một chiều (D5). | Xe thư chạy theo chuyến, không phải ống dẫn chạy liên tục. |
| **UC2 Truy cập S3 nhiều client** | Một endpoint S3, nhiều công cụ khác nhau (aws cli, rclone) dùng chung. | Ổ cắm chuẩn: phích của hãng nào cũng cắm được. |
| **UC3 Phân quyền IAM** | Mỗi người chỉ làm đúng việc được cấp (least privilege \= quyền tối thiểu cần thiết). User chỉ-đọc ghi vào thì bị 403\. | Thẻ ra vào chỉ mở được tầng của mình. |
| **UC4 Chịu lỗi bằng Erasure Coding** | Dữ liệu chia 2 mảnh \+ 1 mảnh parity (mảnh kiểm tra) trên 3 máy. Mất 1 máy vẫn dựng lại được. | 2 bạn chép bài, bạn thứ 3 ghi “tổng”; mất một bạn thì suy ra từ 2 người còn lại (ví dụ XOR ở file AI mục 6). |
| **UC5 Ngưỡng và giới hạn** | Quota (hạn mức số lượng/dung lượng), rate limit (tốc độ thao tác), ngưỡng đầy của cụm. | Hạn mức thẻ tín dụng; barie giới hạn số xe qua mỗi phút; đèn báo xăng sắp cạn. |
| **Màu cam / xanh dương** | Cam \= TrueNAS đảm nhận chính (Cloud Sync là tính năng của TrueNAS); xanh \= Ceph. Đây là gợi ý công nghệ *duy nhất* trên slide. | — |

#### **Lời nói mẫu khi thuyết trình (khoảng 75 giây)**

> “Slide này trả lời một câu: nền tảng của nhóm phục vụ ai, và làm được gì cho họ. Ở đây chưa có máy móc gì cả.  
>  Bên trái là hai nhóm người dùng. Một là nhân viên văn phòng, quen làm việc với thư mục chia sẻ. Hai là ứng dụng hoặc lập trình viên, làm việc qua S3. Bên phải là quản trị viên, người cấu hình và vận hành.  
>  Khung ở giữa là nền tảng, gồm 5 use case. Mỗi ô ghi năng lực nghiệp vụ và lợi ích mà nó chứng minh.  
>  UC1: file nhân viên bỏ vào thư mục chia sẻ sẽ được sao sang kho object. Đó là B1, chống mất dữ liệu.  
>  UC2: cùng một cổng S3, nhiều công cụ khác nhau đều dùng được. Đó là B2, không bị trói vào một nhà cung cấp.  
>  UC3 và UC5 là kiểm soát: ai được làm gì, và mỗi người được dùng bao nhiêu. Đó là B3.  
>  UC4: hỏng một máy, dữ liệu vẫn đọc được. Lại là B1.  
>  Như vậy cả ba lợi ích ở slide 3 đều có use case chứng minh, và cả năm use case đều có demo ở phần sau.  
>  Màu cam là phần TrueNAS làm chính, màu xanh là phần Ceph. Slide tiếp theo sẽ mở chiếc hộp này ra xem bên trong có những phần mềm nào.”

#### **3 câu hỏi vấn đáp dễ gặp**

1. **“Capability khác use case thế nào?”**  
    Capability là *năng lực* của nền tảng, ổn định, không gắn công nghệ. Use case là *một kịch bản* tác nhân dùng các năng lực đó để đạt mục tiêu. Ví dụ UC1 dùng tới 3 capability: chia sẻ file (CAP1), lưu object (CAP2), sao lưu file ↔ object (CAP3). Trên slide chỉ ghi CAP3 vì giới hạn chữ; bảng đầy đủ ở T2.1.  
2. **“Sao quản trị viên nối với UC1? Người dùng văn phòng tự đồng bộ được không?”**  
    Không. Người dùng chỉ thả file vào `outbox/`. Quản trị viên tạo credential và 2 task Cloud Sync, chạy theo lịch hoặc bấm Run Now. Người dùng văn phòng không bao giờ chạm vào S3.  
3. **“UC3 gọi là IAM, nhưng phần Must chỉ là bucket policy. Vậy có phải IAM thật không?”**  
    Bucket policy dùng đúng ngôn ngữ policy của IAM (Effect, Principal, Action, Resource), nên vẫn là phân quyền kiểu IAM. Phần IAM đầy đủ (account, group, identity policy) là Should (UC3 tầng 2). Nói rõ phần nào sẽ demo live.

#### **Không được nói**

* “Nền tảng **không bao giờ** mất dữ liệu.” Cả cụm nằm trên 1 laptop; EC 2+1 mất 2 node là mất dữ liệu (T1.5, E1).  
* “Tương thích **100%** với AWS S3.” Chỉ nói về những API đã demo.  
* “UC1 là **đồng bộ hai chiều**” hoặc “Cloud Sync là gateway.” Đây là 2 task COPY một chiều, chạy theo lô (D5, D6).  
* “Rate limit **chống được DoS**.” Nó chỉ giới hạn theo user/bucket, trên từng RGW.  
* Đọc đường nối là “dữ liệu chạy từ A sang B”. Đường nối ở đây chỉ là quan hệ tham gia.

