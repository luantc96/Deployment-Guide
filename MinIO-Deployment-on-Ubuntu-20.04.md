## HƯỚNG DẪN CÀI ĐẶT MINIO SERVER TRÊN UBUNTU 22.04
> Khuyến khích nên dùng 2 disk để triển khai. 1 disk chạy OS, 1 disk dùng làm datastore cho MinIO. Hướng dẫn này dùng 2 disk.
>
> MinIO sử dụng port 9000 cho UI và 9001 cho API. Cần Open 2 port này trên gateway để sử dụng dịch vụ.
>
> Link docs hãng: https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-single-node-single-drive.html

> Cài đặt Ubuntu Server 22.04 và update hệ thống.

``` shell
apt update -y
apt upgrade -y
```
### BƯỚC 1: Init MinIO Datastore.

> Tạo phân vùng cho MinIO datastore (/dev/sdb) và chuyển sang dạng LVM.

``` shell
fdisk /dev/sdb
```
<img width="666" alt="image" src="https://github.com/luantc96/Deployment-Guide/assets/108060416/3fa653b9-bda4-425f-b547-affbe5ebe8d7">

> Tạo Physical Volume.

``` shell
pvcreate /dev/sdb1
```

> Tạo Volume Group.

``` shell
vgcreate vg-minio /dev/sdb1
```

> Tạo Logical Volume nhận toàn bộ dung lượng của Physical Volume.

``` shell
lvcreate -n lv-minio -l 100%FREE vg-minio
```

> Định dạng Filesystem là kiểu xfs.

``` shell
mkfs -t xfs /dev/vg-minio/lv-minio
```

> Tạo Directory chứa dữ liệu MinIO và mount vào LV ở trên.

``` shell
mkdir -p /minio-data
echo "/dev/vg-minio/lv-minio /minio-data xfs defaults 0 0" >> /etc/fstab
mount -a
```

> Kiểm tra việc init disk đã thành công.

``` shell
df -hT
```
<img width="605" alt="image" src="https://github.com/luantc96/Deployment-Guide/assets/108060416/9557d49a-6b7b-480c-b582-723bdc47cb49">

### BƯỚC 2: Cài Đặt MinIO.

> Tải gói cài đặt từ trang chủ và cài đặt.

``` shell
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20241107005220.0.0_amd64.deb -O minio.deb
dpkg -i minio.deb
```
<img width="1665" alt="image" src="https://github.com/luantc96/Deployment-Guide/assets/108060416/d10867af-d5fa-4323-b7a9-559723401baf">

> Tạo User, Group và phân quyền cho MinIO.

``` shell
groupadd -r minio-user
useradd -M -r -g minio-user minio-user
chown minio-user:minio-user /minio-data
```

> Cấu hình MinIO. Tạo file `/etc/default/minio` bằng trình soạn thảo với nội dung sau:
>
> Cần quan tâm đến các thông số:
>
> `MINIO_ROOT_USER=` đặt user name.
>
> `MINIO_ROOT_PASSWORD=` đặt password.
>
> `MINIO_VOLUMES=` chỉ định nơi lưu trữ data.
>
> `MINIO_SERVER_URL=` đại chỉ URL public dự kiến sẽ sử dụng.

``` shell
#MINIO_ROOT_USER and MINIO_ROOT_PASSWORD sets the root account for the MinIO server.
# This user has unrestricted permissions to perform S3 and administrative API operations on any resource in the deployment.
# Omit to use the default values 'minioadmin:minioadmin'.
# MinIO recommends setting non-default values as a best practice, regardless of environment

MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=Welcome...
MINIO_OPTS="--certs-dir /.minio/certs --address :9000 --console-address :9001"

# MINIO_VOLUMES sets the storage volume or path to use for the MinIO server.

MINIO_VOLUMES="/minio-data"
MINIO_SERVER_URL="https://minio.tpcloud.vn:9000"
# MINIO_SERVER_URL sets the hostname of the local machine for use with the MinIO Server
# MinIO assumes your network control plane can correctly resolve this hostname to the local machine
```

> Start động dịch vụ MinIO và bật khởi động cùng OS.

``` shell
systemctl enable minio.service --now
```

> Kiểm tra dịch vụ start thành công.

``` shell
systemctl status minio.service
```
<img width="1343" alt="image" src="https://github.com/luantc96/Deployment-Guide/assets/108060416/236869c9-adcb-48b4-b170-87aecf71d737">

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
