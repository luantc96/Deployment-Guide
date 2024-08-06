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

### BƯỚC 2 (option): Tạo thêm user VPN.

``` shell
bash openvpn-ubuntu-install.sh
```

> Chọn Option 1 và đặt username.

<img width="519" alt="image" src="https://github.com/user-attachments/assets/40ff81ae-693c-47af-bc3f-332889f23d0f">

### BƯỚC 3 (option): Tạo link Website Để Tải File Config.

``` shell
cd /root && python3 -m http.server
```

![image](https://github.com/user-attachments/assets/7004de9b-e64e-442c-ac6c-aceb617cdfa0)

> Truy cập vào *http://IP:8000* để tải file cấu hình về máy.

<img width="877" alt="image" src="https://github.com/user-attachments/assets/1ae4e201-8871-411d-82c1-85c09ee846db">

