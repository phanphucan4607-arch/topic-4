## **Chuẩn bị Source Code**
```
scp /home/an/Downloads/source_wp.zip root@221.132.21.141:/var/www/
scp /home/an/Downloads/laravel_source.zip root@221.132.21.141:/var/www/
```
**Tại vps**
```
cd /var/www/
unzip source_wp.zip && mv source_wp wordpress
unzip laravel_source.zip && mv laravel_source laravel
chown -R www-data:www-data /var/www/
rm *.zip
```
Đổi cổng: sudo nano /etc/apache2/ports.conf
Sửa Listen 80 thành Listen 8080.

<img width="942" height="420" alt="image" src="https://github.com/user-attachments/assets/ee6a6b17-21aa-4b28-9da3-141d049c9d2e" />

