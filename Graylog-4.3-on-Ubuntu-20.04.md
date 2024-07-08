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

