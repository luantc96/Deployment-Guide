## HƯỚNG DẪN CÀI ĐẶT NEXTCLOUD TRÊN UBUNTU 20.04

> Cài đặt Ubuntu Server 20.04 và update hệ thống.

``` shell
apt update -y
apt upgrade -y
```

### BƯỚC 1: Cài Đặt LEMP Stack.

> 'Cài đặt NGINX.'

``` shell
apt install nginx -y
```

> Kích hoạt và khởi động NGINX.

``` shell
systemctl enable nginx
systemctl start nginx
systemctl status nginx
```

> Cài đặt MariaDB Database Server.

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

> Cài đặt PHP.


### BƯỚC 3: Cấu Hình SSL Certificate.

> Tạo directory lưu trữ certificate.

``` shell
mkdir -p /.minio/certs
```

> Tạo file Certificate và Private key. Sau đó copy nội dung Certificate vào 2 file này.

``` shell
touch /.minio/certs/public.crt
touch /.minio/certs/private.key
```
> Cấp quyền cho user **minio-user** đọc được certificate.

``` shell
chown minio-user:minio-user /.minio/certs/public.crt
chown minio-user:minio-user /.minio/certs/private.key
```
> Restart dịch vụ MinIO

``` shell
systemctl restart minio.service
```
> **Như vậy là hoàn thành cài đặt MinIO, truy cập giao diện quản trị bằng địa chỉ https://minio-url:9000**
