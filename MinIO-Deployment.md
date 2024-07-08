## HƯỚNG DẪN CÀI ĐẶT MINIO SERVER TRÊN UBUNTU 20.04
> Khuyến khích nên dùng 2 disk để triển khai. 1 disk chạy OS, 1 disk dùng làm datastore cho MinIO. Hướng dẫn này dùng 2 disk.

> Cài đặt Ubuntu Server 20.04 và update hệ thống

``` shell
apt update -y
apt upgrade -y
```
> Tạo phân vùng cho MinIO datastore (/dev/sdb)
