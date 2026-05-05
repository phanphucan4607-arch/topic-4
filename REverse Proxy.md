 **Mô hình Hybrid (Nginx + Apache): Tận dụng sức mạnh của cả hai**
#### Giai đoạn 1 cài đặt môi trường và đổi cổng Apache 
-- Cập nhật hệ thống và cài đặt LEMP/LAMP kết hợp

```
apt update && apt upgrade -y
# Cài Nginx, Apache và MySQL
apt install nginx apache2 mysql-server php8.1-fpm php8.1 libapache2-mod-php8.1 php8.1-mysql php8.1-curl php8.1-gd php8.1-
apt install nginx apache2 mysql-server php8.3-fpm php8.3 libapache2-mod-php8.3 php8.3-mysql php8.3-curl php8.3-gd
mbstring php8.1-xml php8.1-zip -y
```
- đổi cổng: Mặc định cả 2 điều muốn giành cổng 80. chúng ta sẽ bắt Apache lùi về phí sau, chạy cổng 8080(http) 8443(hhtps)
```
nano /etc/apache2/ports.cconf
```
```
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
# Đẩy file SQL của WordPress
scp sqlwp_db_vps.zip root@221.132.21.144:/root/

# Đẩy file SQL của Laravel
scp sqllaravel_db_vps.zip root@221.132.21.144:/root/

```
- trên vps
```
cd /root

# Giải nén 2 file (Nếu VPS chưa có unzip thì chạy: apt install unzip -y)
unzip sqlwp_db_vps.zip
unzip sqllaravel_db_vps.zip

# Kiểm tra xem tên file .sql sau khi giải nén là gì
ls -lh *.sql
```

```
A. Đẩy toàn bộ chứng chỉ SSL:
scp * root@221.132.21.144:/etc/nginx/ssl/
```

```
B. Đẩy Source Code (giả sử bạn đã nén thành file .zip):
scp laravel_source_vps.zip root@221.132.21.144:/var/www/laravel.phucan.vietnix.tech/
scp wp_source_vps.zip root@221.132.21.144:/var/www/wp.phucan.vietnix.tech/
```
3. Giải nén và Cấp quyền (Cửa sổ SSH trên VPS)
```
A. Giải nén Laravel:

cd /var/www/laravel.phucan.vietnix.tech
unzip laravel_source_vps.zip

B. Giải nén WordPress:

cd /var/www/wp.phucan.vietnix.tech
unzip wp_source_vps.zip
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

Laravel có đặc thù là thư mục chạy phải trỏ vào /public.
Tạo file cấu hình:
```
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

2. Cấu hình VirtualHost cho WordPress
WordPress thì chạy ngay tại thư mục gốc của nó.
Tạo file cấu hình:
```
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

3. kích hoạt cấu hình module Rewrite
Apache cần module rewrite để chạy các đường dẫn đẹp của worpress và laravel
```
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

1. Cấu hình Default Vhost (Chặn truy cập qua IP)
Tạo file:
```
 nano /etc/nginx/sites-available/default
```
Dán nội dung (chỉ để hiện trang trắng hoặc thông báo lỗi):
```
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
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name laravel.phucan.vietnix.tech;

    # ĐÚNG TÊN FILE SSL CỦA BẠN
    ssl_certificate     /etc/nginx/ssl/ssl.laravel.phucan.vietnix.tech.pem;
    ssl_certificate_key /etc/nginx/ssl/ssl.laravel.phucan.vietnix.tech.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8080; 
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
    server_name laravel.phucan.vietnix.tech;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name laravel.phucan.vietnix.tech;

    # ĐÚNG TÊN FILE SSL CỦA BẠN
    ssl_certificate     /etc/nginx/ssl/ssl.laravel.phucan.vietnix.tech.pem;
    ssl_certificate_key /etc/nginx/ssl/ssl.laravel.phucan.vietnix.tech.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:8080; 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
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

3. Cho đưa data lên sql WordPress:
```
# Giả sử file giải nén ra là sqlwp_db_vps.sql
mysql -u wp_admin -pAdmin@123 wp_admin < /root/sqlwp_db_vps.sql
```
4. Cho đưa data lên sql Laravel:
```
# Giả sử file giải nén ra là sqllaravel_db_vps.sql
mysql -u lara_admin -pAdmin@123 lara_admin < /root/sqllaravel_db_vps.sql
```
5. Kết nối Code Laravel
mở file .env
```
nano /var/www/laravel.phucan.vietnix.tech/.env
```
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
# lênh dọn dẹp
cd /var/www/laravel.phucan.vietnix.tech
php artisan config:clear
php artisan cache:clear
```
6. Kết nối với Wordpress
```
nano /var/www/wp.phucan.vietnix.tech/wp-config.php
```
```
# sửa lại các chỗ sau
define( 'DB_NAME', 'wp_admin' );
define( 'DB_USER', 'wp_admin' );
define( 'DB_PASSWORD', 'Admin@123' );
define( 'DB_HOST', 'localhost' );
````

##### CỰC KỲ QUAN TRỌNG (Để WordPress hiểu đang chạy qua Proxy HTTPS):
Dán đoạn này lên ngay trên dòng /* That's all, stop editing! Happy publishing. */:
```
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}
define('WP_HOME', 'https://wp.phucan.vietnix.tech');
define('WP_SITEURL', 'https://wp.phucan.vietnix.tech');
```
7.  Cấp quyền sở hữu

```
chown -R www-data:www-data /var/www/laravel.phucan.vietnix.tech
chown -R www-data:www-data /var/www/wp.phucan.vietnix.tech
chmod -R 775 /var/www/laravel.phucan.vietnix.tech/storage
chmod -R 775 /var/www/laravel.phucan.vietnix.tech/bootstrap/cache
```

