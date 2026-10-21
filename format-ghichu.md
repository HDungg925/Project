# Hướng dẫn Xây dựng Bài Thuyết trình Báo cáo Đồ án

## 1. Tổng quan về Kỹ năng Thuyết trình

### 1.1. Tầm quan trọng của kỹ năng trình bày

Kỹ năng trình bày (presentation skill) là một **soft skill** quan trọng giúp phát triển nhanh nhất trong công việc. Nhiều người chỉ tập trung vào kỹ thuật (technical) mà bỏ qua kỹ năng này, dẫn đến khi báo cáo không tốt. Cần thay đổi quan điểm ngay lập tức.

### 1.2. Phân bổ thời gian làm báo cáo

Thời gian và công sức dành cho việc làm báo cáo nên chiếm **30%** tổng thời gian thực hiện đồ án. Điều này cho thấy tầm quan trọng của khâu chuẩn bị báo cáo.

### 1.3. Đối tượng người nghe

- Nếu đối tượng nghe là bạn bè cùng lớp thì dễ dàng.
- Nếu đối tượng nghe là **sếp** thì rất nguy hiểm. Trình bày sai một cái là có vấn đề, trình bày mà không nhăn mặt (thiếu tự tin) là run.

## 2. Cấu trúc Bài Thuyết trình (15 Slide)

### 2.1. Slide 1: Xác định Ngữ cảnh và Phạm vi (Scope)

Khi trình bày bất kỳ điều gì, phải đưa người nghe vào **ngữ cảnh** (context). Trong ngữ cảnh đó, phải **có phạm vi công việc** (scope of work).

> **Ví dụ:** Mô tả "chiếc xe máy". Nếu không bó scope, người thì hình dung xe dream, người hình dung xe Wave, người hình dung Vision → ba người cãi lộn. Phải đưa ra thông điệp rõ ràng: "Tôi đang trình bày chiếc xe máy là chiếc xe dream" – tất cả các xe khác không quan tâm.

(phần này giống như bối cảnh thực tế đúng không?)
### 2.2. Slide 2: Mục tiêu (Objectives)

#### 2.2.1. Yêu cầu về Objectives

- Liệt kê **tối thiểu 3 objectives**.
- Mỗi objective là một **phát biểu ngắn gọn, không dài dòng**.
- Ba objectives này phải giải quyết được ba vấn đề đang nan giải (pain points).

#### 2.2.2. Cách viết Objectives

- Objectives phải trả lời câu hỏi: "Tại sao tôi lại làm việc này?".
- Mỗi objective phải chỉ rõ vấn đề cốt lõi mà nó giải quyết.
- Các objectives phải **liền mạch** với nhau.

> **Ví dụ:** "Tôi sẽ đi mô tả chiếc xe dream" – tại sao không phải là mô tả chiếc xe Wave? Có vấn đề gì ở đây? Đó là lý do cần objectives.

### 2.3. Slide 3: Lợi ích (Benefits)

Sau khi trình bày mục tiêu và giải quyết bài toán, cần chỉ ra: nếu triển khai thì sẽ được những lợi ích gì.

> **Cảnh báo:** Nếu không nói được benefits, không bao giờ người ta đầu tư tiền cho làm. Dù có là siêu nhân, nếu không nói được benefits thì suốt đời chỉ làm engineer, không làm sếp được.

### 2.4. Slide 4: Tổng quan Công nghệ (Overview - What)

- Giới thiệu công nghệ sử dụng là gì.
- Có thể copy khái niệm từ trên mạng nhưng **phải giải thích được**.
- Slide này trả lời câu hỏi: **"Nó là cái gì?"** (What).

### 2.5. Slide 5: Cách thức Hoạt động (How it Works)

Slide này trả lời câu hỏi: **"Nó hoạt động như thế nào?"** (How).

### 2.6. Slide 6-9: Kiến trúc (Architecture)

#### 2.6.1. Business Architecture

- Nói về **business capability** – nền tảng có khả năng làm được những chức năng (functional) nào.

#### 2.6.2. Application Architecture

#### 2.6.3. Integration Architecture

#### 2.6.4. Infrastructure Architecture

> **Lưu ý:** Có 6 kiến trúc trong enterprise architecture, nhưng trong môn này chỉ làm 4 kiến trúc trên (bỏ qua data architecture và skill architecture).

### 2.7. Slide 10-14: Business Use Case (5 Slide)

- Lấy ra từ các chức năng (functional) đã nêu ở phần business capability.
- **Mỗi use case phải chứng minh** được objectives và benefits đã nêu.
- **Tính liên kết:** Ít nhất 3 benefits phải nằm trong các use case này, và trong phần demo phải có 3 cái đó.

> **Cảnh báo:** Nếu benefits nói giải quyết bài toán data loss prevention mà trong use case không có, thì bài thuyết trình không thuyết phục.

## 3. Quy trình Phát triển Bài Thuyết trình

### 3.1. Vòng lặp cập nhật

- Ban đầu viết objectives, benefits, công nghệ.
- Trong quá trình làm, nếu phát hiện tính năng mới hay hơn, **quay lại update** các phần trước.
- Không được viết xong phần này rồi bỏ luôn, qua phần khác làm mà không quay lại cập nhật.

> **Ví dụ:** Một người nghiên cứu thấy có 2-3 tính năng đơn giản quá, viết bài bình thường. Tự nhiên làm thêm 2-3 tính năng hay quá, quay lại update. Đó là vòng lặp.

### 3.2. Sai sót và điều chỉnh

- Hướng đi có thể không rõ ràng, có thể fail.
- Khi phát hiện hiểu sai, quay ngược lại update.
- Khi phát hiện "chém gió" quá, quay lại update.

### 3.3. Nguyên tắc tổng thể

Tất cả các phần phải **nằm trong đầu** để khi làm mọi thứ đều **link lại với nhau**, biết lúc nào cần cập nhật.

## 4. Yêu cầu về Slide và Thời lượng

### 4.1. Số lượng slide

- **Khoảng 15 slide** là đẹp.
- Không nên làm quá 15 slide.
- Phải **chọn lọc** kiến thức: cái nào cần nói, cái nào không nên nói.

### 4.2. Thời lượng trình bày

- Mỗi slide tối thiểu 2 phút.
- 15 slide = 30 phút.
- Thêm 5 phút run, nói tào lao → tổng cộng 35 phút.


## 5. Lưu ý Quan trọng

### 5.1. Về cách báo cáo

- **Không đọc slide** khi báo cáo. Nếu đứng lên mở slide và đọc, thầy sẽ đuổi về.
- Báo cáo không phải cho thầy mà là cho **tất cả các bạn trong lớp**.
- Thông qua kết quả các bạn nhận được và kết quả làm được, thầy đánh giá pass hay không pass.

### 5.2. Về nghiên cứu khoa học

- Người ta đọc **abstract** trước. Nếu abstract ok mới đọc tiếp.
- Nếu abstract không ok, người ta không đọc tới conclusion, bỏ bài luôn.
- Khi trình bày cho sếp cũng vậy: sếp nhìn phần đầu tiên, sau đó nhìn kết luận.

### 5.3. Về viết objectives và benefits

- Phần này **rất khó viết** nhưng phải tập viết.
- Càng làm hùng hồn, càng làm hay, đề tài càng hay.
- **Phải tốn 20-40% thời gian** ngồi viết cho thật tốt.
- Phải **nghiên cứu thực tế** (real), không được hư cấu.
- Ban đầu viết xong đừng hy vọng là xong – phải liên tục cập nhật.
