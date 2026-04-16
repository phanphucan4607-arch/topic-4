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

Kiểm tra:

<img width="942" height="420" alt="image" src="https://github.com/user-attachments/assets/ee6a6b17-21aa-4b28-9da3-141d049c9d2e" />


<img width="930" height="984" alt="image" src="https://github.com/user-attachments/assets/184f3191-44fa-452a-aa87-135244bb7a96" />

<img width="930" height="984" alt="image" src="https://github.com/user-attachments/assets/13cd3056-7c72-4b71-9c50-d2dd203accb0" />


<img width="908" height="841" alt="image" src="https://github.com/user-attachments/assets/0ba03b8b-b14f-4d07-947a-693ea3a9fb02" />

