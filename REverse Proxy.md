Mô hình Hybrid (Nginx + Apache): Tận dụng sức mạnh của cả hai

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
# --- KHỐI 1: XỬ LÝ HTTP (CỔNG 80) ➡️ APACHE 8080 ---
server {
    listen 80;
    server_name wp.phucan.vietnix.tech;
    
    root /var/www/wp.phucan.vietnix.tech;
    index index.php index.html index.htm;

    # Kiểm tra file tĩnh trước, nếu không thấy thì đẩy sang Apache
    location / {
        try_files $uri $uri/ @apache_http;
    }

    # TỐI ƯU: Nginx tự xử lý file tĩnh, giảm tải cho Apache
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg|eot)$ {
        try_files $uri @apache_http; 
        expires 30d;
        access_log off;
    }

    # Proxy sang Apache HTTP
    location @apache_http {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# --- KHỐI 2: XỬ LÝ HTTPS (CỔNG 443) ➡️ APACHE 8443 ---
server {
    listen 443 ssl;
    server_name wp.phucan.vietnix.tech;
    
    root /var/www/wp.phucan.vietnix.tech;

    ssl_certificate /etc/letsencrypt/live/wp.phucan.vietnix.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wp.phucan.vietnix.tech/privkey.pem;

    location / {
        try_files $uri $uri/ @apache_https;
    }

    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg|eot)$ {
        try_files $uri @apache_https;
        expires 30d;
        access_log off;
    }

    # Proxy sang Apache HTTPS
    location @apache_https {
        proxy_pass https://127.0.0.1:8443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_ssl_verify off; 
    }
}
```

**Cấu hình Virtual Host cho Laravel (Coza Store)**
sudo nano /etc/nginx/sites-available/laravel.phucan.vietnix.tech
```
# --- KHỐI 1: XỬ LÝ HTTP (CỔNG 80) ➡️ APACHE 8080 ---
server {
    listen 80;
    server_name laravel.phucan.vietnix.tech;
    
    # Laravel bắt buộc root phải trỏ vào /public
    root /var/www/laravel.phucan.vietnix.tech/public;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ @apache_http;
    }

    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg|eot)$ {
        try_files $uri @apache_http;
        expires 30d;
        access_log off;
    }

    location @apache_http {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# --- KHỐI 2: XỬ LÝ HTTPS (CỔNG 443) ➡️ APACHE 8443 ---
server {
    listen 443 ssl;
    server_name laravel.phucan.vietnix.tech;
    
    root /var/www/laravel.phucan.vietnix.tech/public;

    ssl_certificate /etc/letsencrypt/live/laravel.phucan.vietnix.tech/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/laravel.phucan.vietnix.tech/privkey.pem;

    location / {
        try_files $uri $uri/ @apache_https;
    }

    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg|eot)$ {
        try_files $uri @apache_https;
        expires 30d;
        access_log off;
    }

    location @apache_https {
        proxy_pass https://127.0.0.1:8443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_ssl_verify off;
    }
}

```
Kích hoạt và Kiểm tra

Sau khi lưu file, cậu cần tạo liên kết (symlink) và khởi động lại Nginx:

kíchkích hoạt cấu hình

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

<img width="1842" height="998" alt="image" src="https://github.com/user-attachments/assets/9bc9b12f-04eb-4745-80a0-7f24380f0ed4" />

