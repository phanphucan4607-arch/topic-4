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


Sau khi Nginx đã sẵn sàng ở tiền tuyến, chúng ta tiến hành cấu hình Apache đóng vai trò "máy chủ xử lý mã nguồn" tại các cổng nội bộ 8080 (HTTP) và 8443 (HTTPS).
Bước 1: Khởi tạo Virtual Host cho phân vùng WordPress

Chúng ta tách biệt hai luồng truy cập để đảm bảo tính toàn vẹn của dữ liệu khi đi qua Proxy.''
sudo nano /etc/apache2/sites-available/wp.phucan.vietnix.tech.conf

# --- LUỒNG XỬ LÝ HTTP (PORT 8080) ---
<VirtualHost *:8080>
    ServerName wp.phucan.vietnix.tech
    DocumentRoot /var/www/wp.phucan.vietnix.tech
    
    <Directory /var/www/wp.phucan.vietnix.tech>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# --- LUỒNG XỬ LÝ HTTPS (PORT 8443) ---
<VirtualHost *:8443>
    ServerName wp.phucan.vietnix.tech
    DocumentRoot /var/www/wp.phucan.vietnix.tech

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/wp.phucan.vietnix.tech/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/wp.phucan.vietnix.tech/privkey.pem
    
    <Directory /var/www/wp.phucan.vietnix.tech>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

Bước 2: Khởi tạo Virtual Host cho phân vùng Laravel

Quy tắc đặc biệt: Đối với Laravel, cổng vào duy nhất phải là thư mục /public để bảo vệ các tập tin cấu hình hệ thống (như .env).

sudo nano /etc/apache2/sites-available/laravel.phucan.vietnix.tech.conf
# --- LUỒNG XỬ LÝ HTTP (PORT 8080) ---
<VirtualHost *:8080>
    ServerName laravel.phucan.vietnix.tech
    DocumentRoot /var/www/laravel.phucan.vietnix.tech/public
    
    <Directory /var/www/laravel.phucan.vietnix.tech/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# --- LUỒNG XỬ LÝ HTTPS (PORT 8443) ---
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
KÍCH HOẠT VÀ KIỂM ĐỊNH HỆ THỐNG

Sau khi "xây dựng" xong các bản thiết kế, chúng ta tiến hành đưa chúng vào vận hành thực tế.
Bash

# 1. Loại bỏ cấu hình mặc định để tối ưu tài nguyên
sudo a2dissite 000-default.conf

# 2. Kích hoạt đồng thời các phân vùng dự án
sudo a2ensite wp.phucan.vietnix.tech.conf
sudo a2ensite laravel.phucan.vietnix.tech.conf

# 3. Kiểm định tính toàn vẹn của cú pháp (Bắt buộc)
sudo apache2ctl configtest



<img width="919" height="955" alt="image" src="https://github.com/user-attachments/assets/79e0a870-025f-4512-8947-46b01af59ed1" />



