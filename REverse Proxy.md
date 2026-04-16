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


đổi port
<img width="923" height="947" alt="image" src="https://github.com/user-attachments/assets/9e0d0871-9c61-47c9-b56a-ffa5a5fe6d61" />


restart dịch vụ và kiểm tra TRẠNG THÁI 
<img width="920" height="339" alt="image" src="https://github.com/user-attachments/assets/a2a0ea54-cc30-45e5-9716-d66e63bf2bbb" />

nội dung file  /etc/apache2/sites-available/wp.phucan.vietnix.tech.conf:
1. Cấu hình cho WordPress (Cổng 8080 và 8443)

<img width="919" height="955" alt="image" src="https://github.com/user-attachments/assets/ff843efa-22ac-4be2-9606-25bf1b257f02" />

Bước 2: Thiết lập Virtual Host cho Laravel

1. Mở file cấu hình mới:
Bash

sudo nano /etc/apache2/sites-available/laravel.phucan.vietnix.tech.conf

2. Dán nội dung cấu hình sau:
Apache

# --- LUỒNG HTTP (Cổng 8080) ---
<VirtualHost *:8080>
    ServerName laravel.phucan.vietnix.tech
    DocumentRoot /var/www/laravel.phucan.vietnix.tech/public
    
    <Directory /var/www/laravel.phucan.vietnix.tech/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# --- LUỒNG HTTPS (Cổng 8443) ---
<VirtualHost *:8443>
    ServerName laravel.phucan.vietnix.tech
    DocumentRoot /var/www/laravel.phucan.vietnix.tech/public
    
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/laravel.phucan.vietnix.tech/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/laravel.phucan.vietnix.tech/privkey.pem
    
    <Directory /var/www/laravel.phucan.vietnix.tech/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>



<img width="919" height="955" alt="image" src="https://github.com/user-attachments/assets/79e0a870-025f-4512-8947-46b01af59ed1" />



