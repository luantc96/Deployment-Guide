## HƯỚNG DẪN CÀI ĐẶT GRAYLOG 4.3 TRÊN UBUNTU 20.04

> Cài đặt Ubuntu Server 20.04 và update hệ thống

``` shell
apt-get update && apt-get upgrade
```

### BƯỚC 1: Cài Đặt Java OpenJDK

> Quá trình cài đặt Graylog chúng ta cần tiến hành kiểm tra xem Java OpenJDK đã được cài đặt chưa vì đây là là thứ cần thiết để Graylog có thể hoạt động.
Để kiểm tra xem Java đã được cài đặt hay chưa, hãy chạy lệnh:

``` shell
java -version
```
> Nếu chưa cài đặt javajdk chúng ta tiến hành cài đặt javajdk và các gói phụ thuộc bằng lệnh sau:

``` shell
apt -y install bash-completion apt-transport-https uuid-runtime pwgen openjdk-11-jre-headless
```
> Kiểm tra phiên bản javajdk đã cài đặt:

``` shell
java -version
```
### BƯỚC 2: Cài đặt Elasticsearch
> Nhập Elasticsearch PGP key bằng cách thực thi lệnh sau:
``` shell
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
```
>Tiến thành thêm Elasticsearch từ repository.

``` shell
echo "deb https://artifacts.elastic.co/packages/oss-6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
```
> Cập nhật và tiến hành cài đặt Elasticsearch:

``` shell
apt update -y
apt -y install elasticsearch-oss
```

> Tiến hành chỉnh sửa file cấu hình của Elasticsearch /etc/elasticsearch/elasticsearch.yml thêm 2 dòng sau vào cuối file:

_cluster.name: graylog_

_action.auto_create_index: false_

> Thực hiện khởi động lại dịch vụ daemon và Elasticsearch:

``` shell
systemctl daemon-reload && systemctl restart elasticsearch && systemctl enable elasticsearch
```

### BƯỚC 3: Cài đặt MongoDB

> Thực thi lệnh sau để tiến hành cài đặt MongoDB:

``` shell
apt install mongodb-server -y
```
> Tiến hành khởi động dịch vụ MongoDB và cho MongoDB khởi động cùng với hệ thống:

``` shell
systemctl start mongodb && systemctl enable mongodb
```
### BƯỚC 4: Cài đặt Graylog

> Thực hiện thêm Graylog repository:

``` shell
wget https://packages.graylog2.org/repo/packages/graylog-4.3-repository_latest.deb
```
> Tiến hành cài đặt Graylog server package:

``` shell
dpkg -i graylog-4.3-repository_latest.deb
```
> Tiến hành cài đặt Graylog:

``` shell
apt update -y
apt install graylog-server -y
```
> Sau khi cài đặt hoàn tất, chúng ta cần tạo một khóa bí mật để bảo mật mật khẩu người dùng. Chạy lệnh sau để tạo khóa bí mật:

``` shell
pwgen -N 1 -s 96
```

> Chọn một mật khẩu cho tài khoản admin và mã hóa bằng hàm băm 64 ký tự.

``` shell
echo -n Welcome..... | sha256sum
```
> Sau đó chúng ta tiến hành cấu hình Graylog như sau:
Sử dụng trình soạn thảo để tiến hành chỉnh sửa file /etc/graylog/server/server.conf. Cụ thể:
>
> Chỉnh sửa password_secret với chuỗi 96 ký tự ngẫu nhiên mà chúng ta đã tạo trước đó.
> 
> Chỉnh sửa root_password_sha2 với hàm băm 64 ký tự của mật khẩu admin.
> 
> Chỉnh sửa http_bind_address thành 0.0.0.0:9000
