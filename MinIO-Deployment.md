## HƯỚNG DẪN CÀI ĐẶT MINIO SERVER TRÊN UBUNTU 20.04
> Khuyến khích nên dùng 2 disk để triển khai. 1 disk chạy OS, 1 disk dùng làm datastore cho MinIO. Hướng dẫn này dùng 2 disk.

> Cài đặt Ubuntu Server 20.04 và update hệ thống

``` shell
apt update -y
apt upgrade -y
```
### BƯỚC 1: Init MinIO Datastore

> Tạo phân vùng cho MinIO datastore (/dev/sdb) và chuyển sang dạng LVM

``` shell
fdisk /dev/sdb
```
<img width="666" alt="image" src="https://github.com/luantc96/Deployment-Guide/assets/108060416/3fa653b9-bda4-425f-b547-affbe5ebe8d7">

> Tạo Physical Volume

``` shell
pvcreate /dev/sdb1
```

> Tạo Volume Group

``` shell
vgcreate vg-minio /dev/sdb1
```

> Tạo Logical Volume nhận toàn bộ dung lượng của Physical Volume

``` shell
lvcreate -n lv-minio -l 100%FREE vg-minio
```

> Định dạng Filesystem là kiểu xfs

``` shell
mkfs -t xfs /dev/vg-minio/lv-minio
```

> Tạo Directory chứa dữ liệu MinIO và mount vào LV ở trên

``` shell
mkdir -p /minio-data
echo "/dev/vg-minio/lv-minio /minio-data xfs defaults 0 0" >> /etc/fstab
mount -a
```

> Kiểm tra việc init disk đã thành công

``` shell
df -hT
```
<img width="605" alt="image" src="https://github.com/luantc96/Deployment-Guide/assets/108060416/9557d49a-6b7b-480c-b582-723bdc47cb49">

### BƯỚC 2: Cài Đặt MinIO

> Tải gói cài đặt từ trang chủ và cài đặt

``` shell
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20240628090649.0.0_amd64.deb -O minio.deb
dpkg -i minio.deb
```
