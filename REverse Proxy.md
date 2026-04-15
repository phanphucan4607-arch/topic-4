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

