## HƯỚNG DẪN CẤU HÌNH SFTP SERVER TRÊN UBUNTU 24.04
> Khuyến khích nên dùng 2 disk để triển khai. 1 disk chạy OS, 1 disk dùng làm data storage. Hướng dẫn này dùng 2 disk.
>
> SFTP server sử dụng chính Open-ssh có sẵn trên Ubuntu.
>

> Cài đặt Ubuntu Server 24.04 và update hệ thống.

``` shell
echo "<IP_server>  <public_domain>" >> /etc/hosts
```

``` shell
systemctl enable --now systemd-resolved
systemctl restart systemd-resolved
ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
echo "127.0.1.1 $hostname" >> /etc/hosts
```

``` shell
apt update -y && apt upgrade -y
```

### BƯỚC 1: Tạo SFTP user và SFTP Group.

> Lưu ý: Tạo user không cho phép login SSH. Đặt mật khẩu cho user

``` shell
groupadd sftpusers
useradd -m -g sftpusers -s /usr/sbin/nologin <tên_user>
passwd <tên_user>
```

> Tạo directory lưu trữ dữ liệu cho User và phân quyền.

``` shell
mkdir -p /sftp/<tên_user>/upload
chown root:root /sftp/<tên_user>
chmod 755 /sftp/<tên_user>
chown <tên_user>:sftpusers /sftp/<tên_user>/upload/
```

>Cấu hình cho phép SSH bằng password: vi /etc/ssh/sshd_config.d/60-cloudimg-settings.conf

``` shell
PasswordAuthentication yes
```

> Cấuh hình file ssh config. /etc/ssh/sshd_config
> Tìm dòng: Subsystem sftp /usr/lib/openssh/sftp-server sửa thành:

``` shell
Subsystem       sftp    internal-sftp
```
> Thêm vào cuối file /etc/ssh/sshd_config
``` shell
Match Group sftpusers
    ChrootDirectory /sftp/%u
    ForceCommand internal-sftp -d /upload
    X11Forwarding no
    AllowTcpForwarding no
```

### BƯỚC 2: Tạo phân vũng lưu trữ và mount volume.

> Init Disk: Tạo phân vùng sdb1 và định dạng type "Linux file System"

``` shell
fdisk /dev/sdb
```

> Format phân vùng đã tạo.

``` shell
mkfs.ext4 /dev/sdb1
```

> Cấu hình file /etc/fstab.

``` shell
/dev/sdb1	/sftp	ext4	defaults	0	0
```
> Reload cấu hình storage mapping.

``` shell
mount -a
systemctl daemon-reload
```

### BƯỚC 3: Test cấu hình.

> Dùng Winscp test kết nối.
