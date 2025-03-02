# THIẾT KẾ HỆ THỐNG CỔNG THANH TOÁN TRUNG GIAN

Chào mọi người! Hôm nay mình muốn chia sẻ về một thiết kế hệ thống cổng thanh toán trung gian với ngân hàng.

Khi lướt GitHub, mình tình cờ tìm thấy một dự án chia sẻ cách lấy API ngân hàng: [MBBank API](https://github.com/CookieGMVN/MBBank). Từ đó, mình nảy ra ý tưởng thiết kế một cổng thanh toán trung gian tương tự như PayOS, SePay, MoMo và các nền tảng thanh toán khác.

Dưới đây là bản thiết kế sơ bộ về core của hệ thống. Trên thực tế, khi triển khai, hệ thống có thể cần bổ sung thêm nhiều service khác để đảm bảo hiệu suất và tính mở rộng.

## CÁC THÀNH PHẦN CHÍNH CỦA HỆ THỐNG

### 1. Auth Service

- Cung cấp và xác thực token cho user khi đăng nhập.

### 2. User Service

- Quản lý thông tin tài khoản người dùng trong hệ thống.

### 3. Transaction Service

- Tạo và quản lý giao dịch (transaction).

### 4. App Management Service

- Cho phép người dùng tạo nhiều dự án khác nhau để quản lý riêng biệt các transaction và tài khoản ngân hàng theo từng dự án.

### 5. Bank Link Service

- Quản lý kết nối tài khoản ngân hàng và token của từng user.

### 6. Scheduler Service

- Quản lý tất cả các cron job của hệ thống, đảm bảo rằng chỉ một instance chạy duy nhất để tránh lỗi khi triển khai multi-instance.

### 7. Reconciliation Service

- Đối soát giao dịch: so sánh kết quả transaction với lịch sử giao dịch nhận được từ ngân hàng.

### 8. Bank Connector Service

- Quản lý kết nối với API ngân hàng, thực hiện các tác vụ liên quan đến truy vấn lịch sử giao dịch.

### 9. Notification Service

- Hiện tại chủ yếu được sử dụng để gửi webhook đến bên thứ ba.
- Nếu hệ thống phát triển và cần nhiều loại thông báo hơn (FCM, Email, SMS...), nên chuyển sang mô hình fanout, trong đó Notification Service chỉ đóng vai trò message queue để phân phối đến các service gửi thông báo.

## LUỒNG XỬ LÝ CHÍNH

### 1. Tạo Giao Dịch (Transaction)

- **Bước 1**: Client gọi API để tạo một transaction.
- **Bước 2**: Transaction Service xác thực tính hợp lệ của client thông qua App Management Service. Nếu hợp lệ, hệ thống sẽ tạo transaction thuộc về project đó với trạng thái _chờ xác nhận_.

### 2. Quy Trình Xác Nhận Giao Dịch

- **Bước 1**: Scheduler Service kích hoạt cron job gọi đến Reconciliation Service để bắt đầu quá trình xác nhận giao dịch.
- **Bước 2**: Reconciliation Service truy vấn Transaction Service để lấy danh sách các transaction cần xử lý, được nhóm theo project.
  - Nếu số lượng giao dịch lớn, Reconciliation Service phát sự kiện lên message queue để các instance khác cùng xử lý song song, tăng tốc độ xác nhận.
  - Để tránh xử lý trùng lặp, Transaction Service sử dụng distributed lock trên các transaction đang được xử lý.
- **Bước 3**: Reconciliation Service gọi Bank Connector Service để truy vấn lịch sử giao dịch của từng tài khoản ngân hàng.
- **Bước 4**: Bank Connector Service gọi Bank Link Service để lấy token của các tài khoản ngân hàng cần kiểm tra.
- **Bước 5**: Bank Connector Service sử dụng API ngân hàng để lấy lịch sử giao dịch và gửi kết quả về Reconciliation Service.
- **Bước 6**: Reconciliation Service đối chiếu dữ liệu nhận được từ ngân hàng với danh sách transaction và xác định giao dịch nào thành công. Các giao dịch thành công sẽ được gửi sự kiện đến Transaction Service để đánh dấu là _hoàn thành_.
- **Bước 7**: Sau khi transaction được đánh dấu hoàn thành, Transaction Service gửi sự kiện đến Notification Service để webhook thông báo kết quả về ứng dụng của khách hàng.

## TỔNG KẾT

Hệ thống trên chỉ là thiết kế sơ bộ và có thể mở rộng thêm nhiều service tùy theo nhu cầu thực tế. Điểm quan trọng nhất là đảm bảo tính nhất quán dữ liệu, khả năng mở rộng và hiệu suất xử lý giao dịch khi hệ thống phát triển.

Mình rất mong nhận được góp ý và thảo luận từ mọi người để hoàn thiện mô hình này hơn nữa!
