## HƯỚNG DẪN CÀI ĐẶT NEXTCLOUD TRÊN UBUNTU 20.04

> Cài đặt Ubuntu Server 20.04 và update hệ thống.

``` shell
apt update -y
apt upgrade -y
```

### BƯỚC 1: Cài Đặt LEMP Stack.

> __Cài đặt NGINX.__

``` shell
apt install nginx -y
```

> Kích hoạt và khởi động NGINX.

``` shell
systemctl enable nginx
systemctl start nginx
systemctl status nginx
```

> __Cài đặt MariaDB Database Server.__

``` shell
apt install mariadb-server mariadb-client -y
```

> Khởi động và kích hoạt MariaDB.

``` shell
systemctl enable mariadb
systemctl start mariadb
systemctl status mariadb
```

> Cấu hình bảo mật MariaDB.

``` shell
mysql_secure_installation
```

![image](https://github.com/user-attachments/assets/69f2965c-65d9-474b-ad72-567a88eb627e)

> __Cài đặt PHP.__

``` shell
apt install imagemagick php-imagick php7.4-imagick php7.4-common php7.4-mysql php7.4-fpm php7.4-gd php7.4-json php7.4-curl  php7.4-zip php7.4-xml php7.4-mbstring php7.4-bz2 php7.4-intl php7.4-bcmath php7.4-gmp php7.4-zip
apt install -y libmagickcore-6.q16-6-extra
```

> Kích hoạt và khởi động PHP cùng hệ thống.

``` shell
systemctl enable php7.4-fpm
systemctl start php7.4-fpm
systemctl status php7.4-fpm
```


### BƯỚC 2: Cài đặt NextCloud.

> Tạo directory lưu trữ certificate.

``` shell
mkdir /ssl_cert
```

> Tạo file Certificate và Private key. Sau đó copy nội dung Certificate vào 2 file này.

``` shell
touch /ssl_cert/fullchain.pem
touch /ssl_cert/privkey.pem
```
> Tải source NextCloud từ trang chủ về.

``` shell
wget https://download.nextcloud.com/server/releases/nextcloud-23.0.3.zip
```
> Giải nén file NextCloud vừa tải về.

``` shell
unzip nextcloud-*.zip -d /usr/share/nginx/
chown -R www-data:www-data /usr/share/nginx/nextcloud/
```

> Tạo Database cho NextCloud.

> Tiếp theo, đăng nhập vào mysql bằng lệnh *mysql* và thực hiện tạo database_name và database_user cho Nextcloud.
