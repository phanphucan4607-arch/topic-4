 **Mô hình Hybrid (Nginx + Apache): Tận dụng sức mạnh của cả hai**
#### Giai đoạn 1 cài đặt môi trường và đổi cổng Apache 
-- Cập nhật hệ thống và cài đặt LEMP/LAMP kết hợp

```
apt update && apt upgrade -y
# Cài Nginx, Apache và MySQL
apt install nginx apache2 mysql-server php8.1-fpm php8.1 libapache2-mod-php8.1 php8.1-mysql php8.1-curl php8.1-gd php8.1-

mbstring php8.1-xml php8.1-zip -y
```
- đổi cổng: Mặc định cả 2 điều muốn giành cổng 80. chúng ta sẽ bắt Apache lùi về phí sau, chạy cổng 8080(http) 8443(hhtps)
```
nano /etc/apache2/ports.cconf

# sửa nội dung thành
Listen 8080
<IfModule ssl_module>
	Listen 8443
</IfModule>

# Sửa file vhost mặc định của Apache
nano /etc/apache2/sites-available/000-default.conf
Sửa dòng đầu tiên thành: <VirtualHost *:8080>

# khởi động lại Apache
systemctl restart apache2
```
Cài đặt phpMyAdmin (Để quản lý DB cho dễ)
```
apt install phpmyadmin -y
# Trong lúc cài, nếu nó hỏi chọn webserver, hãy chọn Apache2.
# Để truy cập được phpMyAdmin qua Nginx sau này, ta tạo một liên kết:
ln -s /usr/share/phpmyadmin /var/www/html/phpmyadmin
```

#### Giai đoạn 2: đẩy souce code và SSL lên vps 
```
# Tạo thư mục cho 2 website
mkdir -p /var/www/laravel.phucan.vietnix.tech
mkdir -p /var/www/wp.phucan.vietnix.tech

# Tạo thư mục chứa chứng chỉ SSL cho Nginx
mkdir -p /etc/nginx/ssl
```

```
A. Đẩy toàn bộ chứng chỉ SSL:
scp * root@221.132.21.144:/etc/nginx/ssl/
```

```
B. Đẩy Source Code (giả sử bạn đã nén thành file .zip):
scp laravel_source.zip root@221.132.21.144:/var/www/laravel.phucan.vietnix.tech/
scp wp_source.zip root@221.132.21.144:/var/www/wp.phucan.vietnix.tech/
```
3. Giải nén và Cấp quyền (Cửa sổ SSH trên VPS)
```
A. Giải nén Laravel:

cd /var/www/laravel.phucan.vietnix.tech
unzip laravel_source.zip

B. Giải nén WordPress:

cd /var/www/wp.phucan.vietnix.tech
unzip wp_source.zip
```

C. Cấp quyền cho Apache và Nginx có thể đọc file:

Đây là bước cực kỳ quan trọng trong mô hình chạy tay để tránh lỗi 403/500:
```
# Ubuntu/Debian dùng user www-data cho cả Nginx và Apache
chown -R www-data:www-data /var/www/laravel.phucan.vietnix.tech
chown -R www-data:www-data /var/www/wp.phucan.vietnix.tech
chmod -R 755 /var/www/
```
#### Giai đoạn 3 viết cấu hình Viruahost 
```
Laravel có đặc thù là thư mục chạy phải trỏ vào /public.
Tạo file cấu hình:

nano /etc/apache2/sites-available/laravel.phucan.vietnix.tech.conf
```
```
<VirtualHost *:8080>
    ServerName laravel.phucan.vietnix.tech
    DocumentRoot /var/www/laravel.phucan.vietnix.tech/public

    <Directory /var/www/laravel.phucan.vietnix.tech/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/laravel_error.log
    CustomLog ${APACHE_LOG_DIR}/laravel_access.log combined
</VirtualHost>
```
```
2. Cấu hình VirtualHost cho WordPress
WordPress thì chạy ngay tại thư mục gốc của nó.
Tạo file cấu hình:

nano /etc/apache2/sites-available/wp.phucan.vietnix.tech.conf
```
```
<VirtualHost *:8080>
    ServerName wp.phucan.vietnix.tech
    DocumentRoot /var/www/wp.phucan.vietnix.tech

    <Directory /var/www/wp.phucan.vietnix.tech>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/wp_error.log
    CustomLog ${APACHE_LOG_DIR}/wp_access.log combined
</VirtualHost>
```
```
3. kích hoạt cấu hình module Rewrite
Apache cần module rewrite để chạy các đường dẫn đẹp của worpress và laravel
# Kích hoạt 2 website mới
a2ensite laravel.phucan.vietnix.tech.conf
a2ensite wp.phucan.vietnix.tech.conf

# Kích hoạt module rewrite (Rất quan trọng)
a2enmod rewrite

# Tắt site mặc định (để tránh xung đột)
a2dissite 000-default.conf

# Khởi động lại Apache để áp dụng
systemctl restart apache2
```
4. kiểm tra kết quả
```
curl -I http://localhost:8080 -H "Host: laravel.phucan.vietnix.tech"
curl -I http://localhost:8080 -H "Host: wp.phucan.vietnix.tech"
```
#### Giai đoạn 4 cấu hình Nginx Reverse Proxy và SSL
Ở giai đoạn này, chúng ta sẽ tạo 3 file cấu hình: một file cho Default Vhost (chặn truy cập lạ), một cho Laravel, và một cho WordPress.
```
1. Cấu hình Default Vhost (Chặn truy cập qua IP)
Tạo file: nano /etc/nginx/sites-available/default
Dán nội dung (chỉ để hiện trang trắng hoặc thông báo lỗi):

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444; # Đóng kết nối ngay lập tức nếu truy cập qua IP
}
```
2. Cấu hình Nginx cho Laravel (HTTPS + Proxy)
```
nano /etc/nginx/sites-available/laravel.phucan.vietnix.tech
```
```
server {
    listen 80;
    server_name laravel.phucan.vietnix.tech;
    return 301 https://$host$request_uri; # Tự động chuyển HTTP sang HTTPS
}

server {
    listen 443 ssl http2;
    server_name laravel.phucan.vietnix.tech;

    # Đường dẫn SSL (Thay đúng tên file bạn đã đẩy lên ở GĐ 2)
    ssl_certificate     /etc/nginx/ssl/laravel.fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/ssl.laravel.phucan.vietnix.tech.key;

    # Tối ưu SSL
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8080; # Đẩy sang Apache
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
3. Cấu hình Nginx cho WordPress (HTTPS + Proxy)
```
nano /etc/nginx/sites-available/wp.phucan.vietnix.tech
```
```
server {
    listen 80;
    server_name wp.phucan.vietnix.tech;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name wp.phucan.vietnix.tech;

    # Đường dẫn SSL (Thay đúng tên file bạn đã đẩy lên ở GĐ 2)
    ssl_certificate     /etc/nginx/ssl/wp.fullchain.crt;
    ssl_certificate_key /etc/nginx/ssl/ssl.wp.phucan.vietnix.tech.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8080; # Đẩy sang Apache
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WordPress cần thêm dòng này để không bị lỗi CSS khi chạy HTTPS qua Proxy
        proxy_set_header X-Forwarded-Port 443;
    }
}
```
4. Kiểm tra kích hoạt
```
# Tạo liên kết từ sites-available sang sites-enabled
ln -s /etc/nginx/sites-available/laravel.phucan.vietnix.tech /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/wp.phucan.vietnix.tech /etc/nginx/sites-enabled/

# Kiểm tra cú pháp (Cực kỳ quan trọng, nếu báo OK mới được restart)
nginx -t

# Nếu báo OK, khởi động lại Nginx
systemctl restart nginx
```

#### Giai đoạn 5 thiết lập Database và kết nối code 
1. Tạo Database và user trong MySQL
```
mysql -u root -p
# Nhập mật khẩu root MySQL của bạn (Nếu mới cài chưa có pass thì cứ nhấn Enter)
```
- cho Laravel
```
CREATE DATABASE lara_admin;
CREATE USER 'lara_admin'@'localhost' IDENTIFIED BY 'Admin@123';
GRANT ALL PRIVILEGES ON lara_admin.* TO 'lara_admin'@'localhost';
```
- cho Wordpress
```
CREATE DATABASE wp_admin;
CREATE USER 'wp_admin'@'localhost' IDENTIFIED BY 'Admin@123';
GRANT ALL PRIVILEGES ON wp_admin.* TO 'wp_admin'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
```
# Import cho Laravel
mysql -u lara_admin -pAdmin@123 lara_admin < /path/to/your/laravel_database.sql

# Import cho WordPress
mysql -u wp_admin -pAdmin@123 wp_admin < /path/to/your/wordpress_database.sql

(Thay /path/to/your/... bằng đường dẫn thực tế đến file SQL của bạn).
```
3. Kết nối Code Laravel
mở file .env
```
nano /var/www/laravel.phucan.vietnix.tech/.env
```
PP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:krHS59fq5p5L06begya3XRa7qsXuFchiZqJmmhzu6lg=
APP_DEBUG=true
APP_URL=https://laravel.phucan.vietnix.tech

LOG_CHANNEL=stack
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=lara_admin
DB_USERNAME=lara_admin
DB_PASSWORD=Admin@123

BROADCAST_DRIVER=log
CACHE_DRIVER=file
FILESYSTEM_DISK=local
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120
```
```
```
# lênh dọn dẹp
cd /var/www/laravel.phucan.vietnix.tech
php artisan config:clear
php artisan cache:clear
```
4. Kết nối với Wordpress
```
nano /var/www/wp.phucan.vietnix.tech/wp-config.php

# sửa lại các chỗ sau
define( 'DB_NAME', 'wp_admin' );
define( 'DB_USER', 'wp_admin' );
define( 'DB_PASSWORD', 'Admin@123' );
define( 'DB_HOST', 'localhost' );
````
```
# CỰC KỲ QUAN TRỌNG (Để WordPress hiểu đang chạy qua Proxy HTTPS):
Dán đoạn này lên ngay trên dòng /* That's all, stop editing! Happy publishing. */:

if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}
define('WP_HOME', 'https://wp.phucan.vietnix.tech');
define('WP_SITEURL', 'https://wp.phucan.vietnix.tech');
```
5. Chốt hạ: Cấp quyền sở hữu lần cuối
Vì Apache (Backend) sẽ là ông trực tiếp cầm file, bạn phải cho ông ấy quyền:
```
chown -R www-data:www-data /var/www/laravel.phucan.vietnix.tech
chown -R www-data:www-data /var/www/wp.phucan.vietnix.tech
chmod -R 775 /var/www/laravel.phucan.vietnix.tech/storage
chmod -R 775 /var/www/laravel.phucan.vietnix.tech/bootstrap/cache
```
Đây là mô hình kinh điển giúp tối ưu hiệu năng:

    Nginx (Người gác cổng): Cực kỳ nhanh trong việc xử lý hàng ngàn người truy cập cùng lúc và các file tĩnh (ảnh, video). Nginx sẽ đứng ở "tiền tuyến" (Cổng 80/443).

    Apache (Người thợ lành nghề): Rất giỏi xử lý các logic phức tạp của PHP thông qua các file cấu hình .htaccess. Apache sẽ đứng ở "hậu phương" (Cổng 8080).

    Kết quả: Hệ thống vừa nhanh, vừa ổn định, không lo bị quá tải khi có nhiều khách truy cập.
    Nhằm xây dựng một hạ tầng web server tối ưu về hiệu suất và tính bảo mật, tôi tiến hành triển khai mô hình Hybrid Web Server sử dụng Nginx làm Reverse Proxy đứng trước để điều phối lưu lượng và Apache làm Backend xử lý mã nguồn. Quy trình được thực hiện từ bước đồng bộ hóa dữ liệu qua giao thức SCP, thiết lập VirtualHost đa nhiệm cho WordPress và Laravel, đồng thời cấu hình bảo mật chặn truy cập trực tiếp qua IP máy chủ.

    Bước 1: Đẩy mã nguồn và Database lên VPS
sử data đã cung cấp sẵn ta chỉ dùng scp base qua vps 


Đẩy mã nguồn WordPress & Laravel
 Đẩy các file cơ sở dữ liệu
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
```
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
```

Sau khi Nginx đã sẵn sàng ở tiền tuyến, chúng ta tiến hành cấu hình Apache đóng vai trò "máy chủ xử lý mã nguồn" tại các cổng nội bộ 8080 (HTTP) và 8443 (HTTPS).
Bước 1: Khởi tạo Virtual Host cho phân vùng WordPress

Chúng ta tách biệt hai luồng truy cập để đảm bảo tính toàn vẹn của dữ liệu khi đi qua Proxy.''
sudo nano /etc/apache2/sites-available/wp.phucan.vietnix.tech.conf
```
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
```
Bước 2: Khởi tạo Virtual Host cho phân vùng Laravel

Quy tắc đặc biệt: Đối với Laravel, cổng vào duy nhất phải là thư mục /public để bảo vệ các tập tin cấu hình hệ thống (như .env).

sudo nano /etc/apache2/sites-available/laravel.phucan.vietnix.tech.conf
```
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
```
KÍCH HOẠT VÀ KIỂM ĐỊNH HỆ THỐNG
Sau khi "xây dựng" xong các bản thiết kế, chúng ta tiến hành đưa chúng vào vận hành thực tế.

**1. Loại bỏ cấu hình mặc định để tối ưu tài nguyên**
sudo a2dissite 000-default.conf

 **2. Kích hoạt đồng thời các phân vùng dự án**
sudo a2ensite wp.phucan.vietnix.tech.conf
sudo a2ensite laravel.phucan.vietnix.tech.conf

**3. Kiểm định tính toàn vẹn của cú pháp (Bắt buộc)**
sudo apache2ctl configtest

CÁCH THIẾT LẬP "TẤM KHIÊN" TRÊN NGINX

Ép Nginx nhận cấu hình mới (Cực mạnh)
**Sửa lại file default cho chắc (Copy y xì đoạn dưới này)**
sudo nano /etc/nginx/sites-available/default

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 403;
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;

    # Ông mượn tạm SSL của thằng WP để nó không báo lỗi khi khởi động
    ssl_certificate /etc/letsencrypt/live/wp.phucan.vietnix.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wp.phucan.vietnix.tech/privkey.pem;

    return 403;
}
```

```
sudo nginx -t
sudo systemctl reload nginx
```

<img width="884" height="797" alt="image" src="https://github.com/user-attachments/assets/d626e172-07fb-4741-969d-f9423a8014f8" />


Chúng ta sẽ thiết lập Nginx làm lớp bảo vệ phía trước, xử lý SSL và file tĩnh, sau đó mới đẩy yêu cầu "khó" (PHP) xuống cho Apache.
1. Cấu hình Virtual Host cho WordPress (Century Auto)

Thao tác:sudo nano /etc/nginx/sites-available/wp.phucan.vietnix.tech
```
server {
    listen 80;
    server_name wp.phucan.vietnix.tech;
    root /var/www/wordpress;

    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg)$ {
        expires 30d;
        try_files $uri =404;
        access_log off;
        add_header X-Static-Handler "Nginx-Handled-This-File";
    }

    location / {
        proxy_pass http://127.0.0.1:8080/;
        include proxy_params;
    }

    set $redirect_https 0;
    if ($redirect_https = 1) {
        return 301 https://$host$request_uri/;
    }
}

server {
    listen 443 ssl;
    server_name wp.phucan.vietnix.tech;
    root /var/www/wordpress;

    ssl_certificate /etc/letsencrypt/live/wp.phucan.vietnix.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wp.phucan.vietnix.tech/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg)$ {
        expires 30d;
        try_files $uri =404;
        add_header X-Static-Handler "Nginx-Handled-This-File";
    }

    location / {
        proxy_pass https://127.0.0.1:8443/;
        proxy_http_version 1.1;
        proxy_ssl_server_name on;
        proxy_ssl_name $host;
	proxy_set_header Accept-Encoding "";
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

**Cấu hình Virtual Host cho Laravel (Coza Store)**
sudo nano /etc/nginx/sites-available/laravel.phucan.vietnix.tech
```
server {
    listen 80;
    server_name laravel.phucan.vietnix.tech;
    root /var/www/laravel.phucan.vietnix.tech/public;

    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg)$ {
        expires 30d;
        try_files $uri =404;
        access_log off;
    }

    location / {
        proxy_pass http://127.0.0.1:8080/;
        include proxy_params;
    }
}

server {
    listen 443 ssl;
    server_name laravel.phucan.vietnix.tech;
    root /var/www/laravel.phucan.vietnix.tech/public;

    ssl_certificate /etc/letsencrypt/live/laravel.phucan.vietnix.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/laravel.phucan.vietnix.tech/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass https://127.0.0.1:8443/;
        proxy_http_version 1.1;
        proxy_ssl_server_name on;
        proxy_ssl_name $host;
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**cấu hình đồng bộ biến môi trường**
```
sudo nano /var/www/wordpress/wp-config.php
```

<img width="814" height="392" alt="image" src="https://github.com/user-attachments/assets/589bda3b-fff4-4968-ade2-e07b1eefc81e" />
<img width="1859" height="166" alt="image" src="https://github.com/user-attachments/assets/3a4f9be8-84a1-454e-be50-8e27234e110f" />


Kích hoạt và Kiểm tra

Sau khi lưu file, cậu cần tạo liên kết (symlink) và khởi động lại Nginx:

kích hoạt cấu hình

sudo ln -s /etc/nginx/sites-available/wp.phucan.vietnix.tech /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/laravel.phucan.vietnix.tech /etc/nginx/sites-enabled/

Kiểm tra lỗi cú pháp (Bắt buộc)
sudo nginx -t

Nếu báo "syntax is ok", hãy restart Nginx
sudo systemctl restart nginx

<img width="1817" height="957" alt="image" src="https://github.com/user-attachments/assets/819f5936-b620-4adc-9f2a-dc8ad8718b05" />

1. Nginx "ôm" hết việc nhẹ (Xử lý file tĩnh)

    Cách làm: Trong file cấu hình, ông thấy đoạn location ~* \.(jpg|jpeg|png...)$.

    Giải thích: Khi khách vào xem web, nếu họ chỉ cần xem cái ảnh hay file CSS, Nginx nó tự vào thư mục code (root /var/www/...) lấy đưa cho khách luôn. Nó không thèm hỏi thằng Apache làm gì cho mất công.

    Lợi ích: Server chạy cực nhanh vì Apache không phải lo mấy cái việc vặt vãnh này, chỉ tập trung "nấu" PHP thôi.

2. Luồng dữ liệu "ai đi đường nấy" (Song song HTTP/HTTPS)

    Cách làm: Ông viết 2 khối server {} riêng biệt. Một cái nghe cổng 80 (HTTP), một cái nghe cổng 443 (HTTPS).

    Giải thích: * Nếu khách vào bằng HTTP (không bảo mật), Nginx sẽ "dẫn" họ xuống Apache cổng 8080.

        Nếu khách vào bằng HTTPS (có ổ khóa xanh), Nginx sẽ "dẫn" họ xuống Apache cổng 8443.

    Tại sao không dùng 301? Thường người ta hay ép khách từ 80 sang 443 (return 301), nhưng sếp ông yêu cầu không chuyển hướng cưỡng bức. Nghĩa là khách muốn vào kiểu gì thì ông chiều kiểu đó, giữ nguyên giao thức để test hoặc theo yêu cầu riêng của dự án.
   
------
🛡️ THIẾT LẬP "TẤM KHIÊN" DEFAULT VHOST (CHẶN IP & DOMAIN RÁC)
1. Tại sao ông bắt buộc phải làm bước này? (Lý thuyết)

    Tình huống: Nếu ông không làm, khi ai đó trỏ một cái tên miền vớ vẩn (ví dụ: web-rac.com) về IP của ông, Nginx sẽ "lú lẫn" và hiển thị đại một trong hai trang của ông lên.

    Hậu quả: Google sẽ phạt lỗi Duplicate Content (trùng lặp nội dung), làm tụt hạng SEO. Tệ hơn, hacker có thể quét IP để tìm lỗ hổng của server nếu ông để lộ giao diện mặc định.

    Giải pháp: Xây một cái "phòng bảo vệ" để bắt hết những thằng đi lạc và đuổi thẳng cổ bằng lỗi 403 Forbidden

    sudo nano /etc/nginx/sites-available/default

   xóa hết đống cũ đi, dán đúng đoạn này vào cho tôi. Tôi đã dùng chứng chỉ SSL của trang wp.phucan để làm "giấy thông hành" tạm thời cho khối HTTPS:

```
# --- KHỐI 1: CHẶN TRUY CẬP HTTP (CỔNG 80) ---
server {
    listen 80 default_server;
    server_name _; # Dấu gạch dưới nghĩa là "bất cứ ai không có tên trong danh sách"
    
    # Đuổi thẳng cổ
    return 403; 
}

# --- KHỐI 2: CHẶN TRUY CẬP HTTPS (CỔNG 443) ---
server {
    listen 443 ssl default_server;
    server_name _;

    # Nginx cần chứng chỉ để "nói chuyện" với khách trước khi đuổi khách đi.
    # Ta dùng tạm chứng chỉ của WordPress đã có sẵn trên máy ông.
    ssl_certificate /etc/letsencrypt/live/wp.phucan.vietnix.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wp.phucan.vietnix.tech/privkey.pem;

    return 403;
}
```
default_server: Đây là lệnh ưu tiên cao nhất. Ông đang dặn Nginx: "Nếu có thằng nào vào mà tên miền không khớp với file wp.phucan hay laravel.phucan, thì cứ ném hết nó vào đây cho tôi xử lý".

server_name _: Cái dấu gạch dưới này là một "biến ảo". Nó đại diện cho việc truy cập bằng IP trực tiếp hoặc những tên miền "râu ông nọ cắm cằm bà kia".

return 403: Đây là mã lỗi "Cấm vào". Nó nhẹ hơn lỗi 500 nhưng cực kỳ uy lực để chặn đứng các cuộc dò quét tự động.

```
sudo nginx -t  # Kiểm tra xem có gõ nhầm dấu nào không
sudo systemctl restart nginx
```
```
-- 1. Cập nhật tên miền chính cho Website
UPDATE Sa3QIZ_options SET option_value = 'https://wp.phucan.vietnix.tech' WHERE option_name = 'home' OR option_name = 'siteurl';

-- 2. Thay link cũ trong toàn bộ nội dung bài viết (giúp hiện ảnh và link đúng)
UPDATE Sa3QIZ_posts SET post_content = REPLACE(post_content, 'https://linhlt.id.vn', 'https://wp.phucan.vietnix.tech');

-- 3. Thay link cũ trong các đường dẫn tĩnh (GUID)
UPDATE Sa3QIZ_posts SET guid = REPLACE(guid, 'https://linhlt.id.vn', 'https://wp.phucan.vietnix.tech');

-- 4. Thay link cũ trong các trường dữ liệu mở rộng (Metadata)
UPDATE Sa3QIZ_postmeta SET meta_value = REPLACE(meta_value, 'https://linhlt.id.vn', 'https://wp.phucan.vietnix.tech');
```

Đây là bộ lệnh Search & Replace (Tìm và Thay thế) đồng loạt để "thay máu" toàn bộ dữ liệu. Vì WordPress lưu link tuyệt đối (có kèm tên miền) ở khắp mọi nơi, nên khi ông "chuyển nhà" sang server mới, ông phải dùng SQL để ép hệ thống đổi từ tên miền cũ sang tên miền mới.

Cụ thể, bộ lệnh này xử lý 3 tầng dữ liệu:

    Tầng cấu hình (Bảng options): Sửa lại "hộ khẩu" để WordPress biết nó đang chạy trên tên miền mới, tránh bị tự động đá về web cũ.

    Tầng nội dung (Bảng posts & guid): Sửa lại toàn bộ link ảnh và link bài viết trong các bài đăng. Nếu không có bước này, ảnh sẽ bị lỗi (không hiện) và khi bấm vào menu nó sẽ nhảy sang trang của Leader.

    Tầng giao diện (Bảng postmeta): Quét sạch các link cũ còn sót lại trong cài đặt của Theme (như ảnh Logo, Banner, Slider).

Mục đích cuối cùng: Làm cho website hoạt động nhất quán trên tên miền mới phucan.vietnix.tech mà không cần phải ngồi sửa tay hàng nghìn bài viết.

**Cấp quyền sở hữu (Permissions) cho thư mục Web**

thực hiện lệnh mv dữ liệu từ /root/ sang /var/www/. Lúc này, toàn bộ file vẫn thuộc quyền sở hữu của user root.

    Vấn đề: WordPress sẽ không thể upload ảnh, Laravel sẽ không thể ghi Log hoặc Cache (lỗi 500).

và bây giờ có thể truy cập http và https được rồi 

```
sudo chown -R www-data:www-data /var/www/wp.phucan.vietnix.tech
sudo chown -R www-data:www-data /var/www/laravel.phucan.vietnix.tech
sudo chmod -R 755 /var/www/wp.phucan.vietnix.tech
```

 1. Di chuyển vào thư mục dự án
cd /var/www/laravel.phucan.vietnix.tech
 2. Khởi tạo file cấu hình từ file mẫu
cp .env.example .env
 3. Mở file để sửa thông số Database (Sửa tay xong Ctrl+O, Enter, Ctrl+X)
```
nano .env
```


<img width="740" height="379" alt="image" src="https://github.com/user-attachments/assets/159aa29d-9e53-406c-a61a-2e819a4a0f79" />

```
php artisan key:generate
php artisan config:clear
php artisan migrate:status
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache

```
Sau khi làm xong 4 bước trên, để biết web đã thực sự "thông" chưa, ông dùng lệnh này để xem Laravel có đọc được các bảng dữ liệu không:
php artisan migrate:status

<img width="876" height="383" alt="image" src="https://github.com/user-attachments/assets/51ff7f70-f5d0-4b73-a038-b9f4aa83c842" />

Dòng APP_KEY trong Laravel cực kỳ quan trọng. Nó dùng để mã hóa mật khẩu người dùng, mã hóa Session (đăng nhập) và Cookie. Không có chìa khóa này, Laravel sẽ từ chối hoạt động vì lý do an toàn.

<img width="1808" height="510" alt="image" src="https://github.com/user-attachments/assets/5be38254-e505-4fdf-9546-03e698160d88" />

<img width="1842" height="998" alt="image" src="https://github.com/user-attachments/assets/9bc9b12f-04eb-4745-80a0-7f24380f0ed4" />

**cấu hình wp chạy ssong song http và https**
## Chạy lệnh này để vô hiệu hóa plugin mà không cần vào Admin

mv /var/www/wp.phucan.vietnix.tech/wp-content/plugins/really-simple-ssl /var/www/wp.phucan.vietnix.tech/wp-content/plugins/really-simple-ssl-OFF

Ép WordPress chạy theo giao thức của trình duyệt
Bạn hãy mở lại file wp-config.php và thay thế đoạn cấu hình WP_HOME và WP_SITEURL cũ bằng đoạn code "linh hoạt" này. Nó sẽ tự nhìn vào thanh địa chỉ của bạn để quyết định:

## Mở file: 
nano /var/www/wp.phucan.vietnix.tech/wp-config.php

```
// Đoạn code giúp chạy song song HTTP và HTTPS
$protocol = (isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on') ? 'https://' : 'http://';

define('WP_HOME', $protocol . $_SERVER['HTTP_HOST']);
define('WP_SITEURL', $protocol . $_SERVER['HTTP_HOST']);

// Ngăn WordPress tự động chuyển hướng link
define('RELOCATE', true);
```

Vào MySQL: mysql -u root -p
Chọn DB: USE phucan_wp_db;
Chạy lệnh:

```
UPDATE Sa3QIZ_options SET option_value = 'http://wp.phucan.vietnix.tech' WHERE option_name IN ('siteurl', 'home');

```

**trỏ domain về đúng vị trí**

<img width="1920" height="1080" alt="Screenshot from 2026-04-17 09-41-14" src="https://github.com/user-attachments/assets/7e0c8e1c-f7c5-41cb-98e7-4712aebb95fe" />

Bước 1: Cài đặt công cụ WP-CLI
WP-CLI là công cụ quản trị WordPress bằng dòng lệnh, giúp thực hiện việc thay đổi dữ liệu trực tiếp trong Database một cách an toàn và nhanh chóng.


**Tải tệp thực thi WP-CLI**

curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar

**Cấp quyền thực thi**

chmod +x wp-cli.phar

**Di chuyển vào hệ thống để sử dụng lệnh 'wp' toàn cục**

sudo mv wp-cli.phar /usr/local/bin/wp


Bước 2: Thực hiện thay đổi tên miền (Search & Replace)
Di chuyển vào thư mục gốc của mã nguồn và thực hiện lệnh thay thế chuỗi ký tự trên toàn bộ Database.

**Di chuyển vào thư mục mã nguồn**

cd /var/www/wp.phucan.vietnix.tech/

**Thực hiện thay đổi (Lưu ý sử dụng --allow-root khi chạy với quyền root)**

wp search-replace 'https://linhlt.id.vn' 'https://wp.phucan.vietnix.tech' --allow-root

Bước 3: Cấu hình lại Đường dẫn tĩnh (Permalinks)

Sau khi đổi tên miền, cần làm mới tệp cấu hình đường dẫn để tránh lỗi 404 trên các trang con.
```
wp rewrite flush --allow-root
Khởi động lại các dịch vụ để đảm bảo cấu hình mới được áp dụng hoàn toàn và xóa các tệp tin hỗ trợ tạm thời.

sudo systemctl restart nginx
sudo systemctl restart apache2
```

<img width="1875" height="930" alt="image" src="https://github.com/user-attachments/assets/2b774623-2414-43cc-83fe-a665c6365d59" />

**để kiểm tra xem nginx có thật sự đứng trước apeche không ta thực hiện kiểm tra sau**

```
curl -I http://wp.phucan.vietnix.tech
curl -I https://wp.phucan.vietnix.tech
curl -I http://laravel.phucan.vietnix.tech
curl -I http://laravel.phucan.vietnix.tech
curl -I http://wp.phucan.vietnix.tech/wp-content/themes/twentytwentyone/style.css 
```

<img width="1081" height="736" alt="image" src="https://github.com/user-attachments/assets/19af79ac-a835-45ae-8ecd-4b620e0cbbce" />

_Đây là bằng chứng cho thấy mô hình Hybrid (Nginx + Apache) hoạt động hoàn hảo. Nginx trực tiếp đảm nhận việc trả về các tệp tin tĩnh (CSS, JS, Hình ảnh) nhờ vào Header tùy chỉnh X-Static-Handler. Việc này giúp giảm tải hoàn toàn cho Back-end Apache, giúp hệ thống chịu tải tốt hơn_



