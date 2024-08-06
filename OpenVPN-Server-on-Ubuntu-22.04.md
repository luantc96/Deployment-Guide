## HƯỚNG DẪN CÀI ĐẶT OPENVPN Server TRÊN UBUNTU 22.04

> Cài đặt Ubuntu Server 22.04 và update hệ thống.

``` shell
apt update -y
apt upgrade -y
```

### BƯỚC 1: Tải Và Cài Đặt OPENVPN Từ Script.

``` shell
wget https://git.io/vpn -O openvpn-ubuntu-install.sh
chmod -v +x openvpn-ubuntu-install.sh
bash openvpn-ubuntu-install.sh
```

<img width="538" alt="image" src="https://github.com/user-attachments/assets/80e7b2d8-a103-4166-a253-f99ce948c20c">

> Hiện dòng Finished! như hình là thành công. File cấu hình download tại /root
> 
<img width="1521" alt="image" src="https://github.com/user-attachments/assets/f7e7ccd8-eaf2-4018-b6e4-d57b23054608">

