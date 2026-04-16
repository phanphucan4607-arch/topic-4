Mô hình Hybrid (Nginx + Apache): Tận dụng sức mạnh của cả hai

Đây là mô hình kinh điển giúp tối ưu hiệu năng:

    Nginx (Người gác cổng): Cực kỳ nhanh trong việc xử lý hàng ngàn người truy cập cùng lúc và các file tĩnh (ảnh, video). Nginx sẽ đứng ở "tiền tuyến" (Cổng 80/443).

    Apache (Người thợ lành nghề): Rất giỏi xử lý các logic phức tạp của PHP thông qua các file cấu hình .htaccess. Apache sẽ đứng ở "hậu phương" (Cổng 8080).

    Kết quả: Hệ thống vừa nhanh, vừa ổn định, không lo bị quá tải khi có nhiều khách truy cập.
    Nhằm xây dựng một hạ tầng web server tối ưu về hiệu suất và tính bảo mật, tôi tiến hành triển khai mô hình Hybrid Web Server sử dụng Nginx làm Reverse Proxy đứng trước để điều phối lưu lượng và Apache làm Backend xử lý mã nguồn. Quy trình được thực hiện từ bước đồng bộ hóa dữ liệu qua giao thức SCP, thiết lập VirtualHost đa nhiệm cho WordPress và Laravel, đồng thời cấu hình bảo mật chặn truy cập trực tiếp qua IP máy chủ.

    Bước 1: Đẩy mã nguồn và Database lên VPS

(Chạy trên Terminal máy cậu - Máy An)
Bash

# Đẩy mã nguồn WordPress & Laravel
# Đẩy các file cơ sở dữ liệu
<img width="916" height="63" alt="image" src="https://github.com/user-attachments/assets/14ec1969-ffa2-48cd-a988-02f8fd54b7d7" />

Quy hoạch hạ tầng thư mục (System Architecture)

Sau khi dữ liệu đã được đẩy lên vùng đệm /root/, chúng ta tiến hành đưa chúng về "nhà mới" tại /var/www/. Việc đặt tên thư mục theo tên miền giúp quản trị nhiều dự án mà không bị nhầm lẫn.
Bash

 Đăng nhập vào VPS (Sử dụng IP .141 của bạn)
ssh root@221.132.21.141

 Tạo không gian lưu trữ chuyên nghiệp
mkdir -p /var/www/database_backups

 Di chuyển và định danh lại mã nguồn (Sửa lỗi source_wq -> source_wp)
mv /root/laravel_source /var/www/laravel.phucan.vietnix.tech
mv /root/source_wp /var/www/wp.phucan.vietnix.tech

 Đưa các file SQL vào khu vực dự phòng trước khi nạp
mv /root/*.sql /var/www/database_backups/
