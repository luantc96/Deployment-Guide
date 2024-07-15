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

``` shell
create database nextcloud_db;
create user nextcloud@localhost identified by 'password';
grant all privileges on nextcloud_db.* to nextcloud@localhost identified by 'password';
flush privileges;
exit;
```

> Tạo file cấu hình NGINX NextCloud.

``` shell
vi /etc/nginx/conf.d/nextcloud.conf
```

> Sau đó nhập vào nội dung file cấu hình mẫu bên dưới vào
>
> Lưu ý: Thay server_name drive-demo.tpcloud.com.vn; bằng tên server_name muốn sử dụng. Và thay ssl_certificate, ssl_certificate_key bằng đường dẫn SSL đã tạo ở BƯỚC 2.

``` shell
upstream php-handler {
    #server 127.0.0.1:9000;
    server unix:/var/run/php/php7.4-fpm.sock;
}

server {
    listen 80;
    listen [::]:80;
    server_name drive-demo.tpcloud.com.vn;
    # enforce https
    return 301 https://$server_name:443$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name drive-demo.tpcloud.com.vn;

    ssl_certificate /ssl_cert/fullchain.pem; 
    ssl_certificate_key /ssl_cert/privkey.pem;

    # Add headers to serve security related headers
    # Before enabling Strict-Transport-Security headers please read into this
    # topic first.
    #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
    #
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    add_header Referrer-Policy "no-referrer" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Download-Options "noopen" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Permitted-Cross-Domain-Policies "none" always;
    add_header X-Robots-Tag "none" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security 'max-age=31536000; includeSubDomains; preload';

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Path to the root of your installation
    root /usr/share/nginx/nextcloud/;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # The following 2 rules are only needed for the user_webfinger app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
    #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json last;

    # The following rule is only needed for the Social app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/webfinger /public.php?service=webfinger last;

    location = /.well-known/carddav {
      return 301 $scheme://$host:$server_port/remote.php/dav;
    }
    location = /.well-known/caldav {
      return 301 $scheme://$host:$server_port/remote.php/dav;
    }

    # set max upload size
    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Uncomment if your server is build with the ngx_pagespeed module
    # This module is currently not supported.
    #pagespeed off;

    location / {
        rewrite ^ /index.php;
    }

    location ~ ^\/(?:build|tests|config|lib|3rdparty|templates|data)\/ {
        deny all;
    }
    location ~ ^\/(?:\.|autotest|occ|issue|indie|db_|console) {
        deny all;
    }

    location ~ ^\/(?:index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+)\.php(?:$|\/) {
        fastcgi_split_path_info ^(.+?\.php)(\/.*|)$;
        set $path_info $fastcgi_path_info;
        try_files $fastcgi_script_name =404;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;
        # Avoid sending the security headers twice
        fastcgi_param modHeadersAvailable true;
        # Enable pretty urls
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
    }

    location ~ ^\/(?:updater|oc[ms]-provider)(?:$|\/) {
        try_files $uri/ =404;
        index index.php;
    }

    # Adding the cache control header for js, css and map files
    # Make sure it is BELOW the PHP block
    location ~ \.(?:css|js|woff2?|svg|gif|map)$ {
        try_files $uri /index.php$request_uri;
        add_header Cache-Control "public, max-age=15778463";
        # Add headers to serve security related headers (It is intended to
        # have those duplicated to the ones above)
        # Before enabling Strict-Transport-Security headers please read into
        # this topic first.
        #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
        #
        # WARNING: Only add the preload option once you read about
        # the consequences in https://hstspreload.org/. This option
        # will add the domain to a hardcoded list that is shipped
        # in all major browsers and getting removed from this list
        # could take several months.
        add_header Referrer-Policy "no-referrer" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Download-Options "noopen" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Permitted-Cross-Domain-Policies "none" always;
        add_header X-Robots-Tag "none" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # Optional: Don't log access to assets
        access_log off;
    }

    location ~ \.(?:png|html|ttf|ico|jpg|jpeg|bcmap)$ {
        try_files $uri /index.php$request_uri;
        # Optional: Don't log access to other assets
        access_log off;
    }
}
```
> Sau khi nhập file cấu hình xong, kiểm tra xem có lỗi không bằng lệnh sau:

``` shell
nginx -t
```

> Nếu nhận được thông báo syntax is ok hãy khởi động lại NGINX.
