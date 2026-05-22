# Hướng dẫn triển khai Mail Server Open Source bằng mailcow trên Ubuntu 24.04

>
> Mục tiêu: triển khai một hệ thống email server miễn phí, open-source, có webmail, hỗ trợ HTTPS bằng Let's Encrypt, có đầy đủ SPF/DKIM/DMARC để cải thiện khả năng gửi mail không bị vào spam.

---

## 1. Thông tin mô hình triển khai

### 1.1. Thông tin server

| Thành phần | Giá trị mẫu |
|---|---|
| OS | Ubuntu Server 24.04 LTS |
| CPU | 2 vCPU |
| RAM | 4 GB |
| Disk | 100 GB SSD |
| Public IP | `103.205.98.xxx` |
| Hostname/FQDN | `mail.luantc.com` |
| Mail domain | `luantc.com` |
| Webmail URL | `https://mail.luantc.com/` |
| Admin UI URL | `https://mail.luantc.com/admin` |

> Nếu triển khai bằng IP khác, thay toàn bộ `103.205.98.xxx` trong tài liệu này bằng IP thực tế của server.

### 1.2. Giải pháp được chọn

Giải pháp sử dụng là **mailcow: dockerized**.

mailcow là bộ mail server open-source dạng all-in-one, triển khai bằng Docker Compose, bao gồm các thành phần chính:

- Postfix: xử lý SMTP, gửi/nhận mail.
- Dovecot: xử lý IMAP/POP3 và mailbox access.
- SOGo: webmail/groupware cho người dùng cuối.
- Rspamd: anti-spam, DKIM signing, mail filtering.
- ClamAV: antivirus scanning.
- Nginx: reverse proxy/web frontend.
- ACME container: tự động xin và gia hạn chứng chỉ Let's Encrypt.
- MariaDB, Redis, Memcached: backend database/cache.
- Netfilter/Watchdog: hỗ trợ bảo vệ và giám sát container.

Lý do chọn mailcow:

- Phù hợp với mô hình nhỏ 5–10 users.
- Có giao diện quản trị web trực quan.
- Có sẵn webmail SOGo.
- Hỗ trợ Let's Encrypt tự động.
- Có DKIM/SPF/DMARC workflow rõ ràng.
- Quản lý domain, mailbox, alias, quota, quarantine thuận tiện.
- Không cần tự ráp thủ công Postfix + Dovecot + Roundcube + SpamAssassin.

---

## 2. DNS cần chuẩn bị trước khi cài đặt

Domain chính trong tài liệu này là:

```text
luantc.com
```

Mail server hostname là:

```text
mail.luantc.com
```

### 2.1. Record bắt buộc

Trên DNS provider, ví dụ GoDaddy, cần tạo các record sau.

#### A record

```text
Type: A
Name: mail
Value: 103.205.98.xxx
TTL: Default hoặc 600
```

Kết quả mong muốn:

```text
mail.luantc.com -> 103.205.98.xxx
```

#### MX record

```text
Type: MX
Name: @
Value: mail.luantc.com
Priority: 0 hoặc 10
TTL: Default hoặc 600
```

Kết quả mong muốn:

```text
luantc.com MX -> mail.luantc.com
```

#### PTR record

PTR record phải được cấu hình bởi đơn vị cấp IP hoặc nhà cung cấp server/VPS.

```text
103.205.98.xxx -> mail.luantc.com
```

PTR rất quan trọng đối với email server. Nếu không có PTR đúng, mail gửi đến Gmail, Microsoft, Yahoo và nhiều hệ thống khác có khả năng cao bị từ chối hoặc vào spam.

---

## 3. Kiểm tra DNS trước khi cài mailcow

Trên server Ubuntu, cài công cụ cần thiết nếu chưa có:

```bash
sudo apt update
sudo apt install -y dnsutils netcat-openbsd curl
```

Kiểm tra A record:

```bash
dig @8.8.8.8 +short mail.luantc.com A
dig @1.1.1.1 +short mail.luantc.com A
```

Kết quả đúng:

```text
103.205.98.xxx
```

Kiểm tra MX record:

```bash
dig @8.8.8.8 +short luantc.com MX
```

Kết quả đúng:

```text
0 mail.luantc.com.
```

hoặc:

```text
10 mail.luantc.com.
```

Kiểm tra PTR record:

```bash
dig @8.8.8.8 +short -x 103.205.98.xxx
```

Kết quả đúng:

```text
mail.luantc.com.
```

Kiểm tra outbound SMTP port 25:

```bash
nc -vz gmail-smtp-in.l.google.com 25
```

Kết quả tốt:

```text
Connection to gmail-smtp-in.l.google.com 25 port [tcp/smtp] succeeded!
```

Nếu outbound TCP/25 bị chặn, mail server vẫn có thể nhận mail, nhưng gửi mail trực tiếp ra ngoài sẽ gặp vấn đề. Khi đó cần yêu cầu nhà cung cấp mở port 25 hoặc dùng SMTP relay như Amazon SES, Mailgun, SendGrid.

---

## 4. Chuẩn bị hostname và hệ điều hành

### 4.1. Cấu hình hostname

Set hostname:

```bash
sudo hostnamectl set-hostname mail.luantc.com
```

Kiểm tra:

```bash
hostname -f
```

Kết quả mong muốn:

```text
mail.luantc.com
```

### 4.2. Cấu hình `/etc/hosts`

Mở file:

```bash
sudo nano /etc/hosts
```

Nội dung khuyến nghị:

```text
127.0.0.1 localhost
127.0.1.1 mail.luantc.com mail

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```

Kiểm tra lại:

```bash
hostname -f
```

### 4.3. Cấu hình timezone và NTP

```bash
sudo timedatectl set-timezone Asia/Ho_Chi_Minh
sudo timedatectl set-ntp true
timedatectl
```

---

## 5. Kiểm tra port conflict trước khi cài

Trước khi chạy mailcow, server không nên có Nginx/Apache/Postfix/Dovecot khác đang chiếm port mail/web.

Kiểm tra:

```bash
sudo ss -tlpn | grep -E ':(25|80|110|143|443|465|587|993|995|4190)\b'
```

Nếu server sạch, lệnh này có thể không trả về gì.

Các port quan trọng mailcow sẽ sử dụng:

| Port | Giao thức | Mục đích |
|---|---|---|
| 25/tcp | SMTP | Gửi/nhận mail server-to-server |
| 80/tcp | HTTP | Let's Encrypt HTTP-01 challenge, redirect web |
| 443/tcp | HTTPS | Admin UI, SOGo webmail |
| 465/tcp | SMTPS | SMTP over SSL/TLS |
| 587/tcp | Submission | SMTP STARTTLS cho mail client |
| 993/tcp | IMAPS | IMAP over SSL/TLS |
| 995/tcp | POP3S | POP3 over SSL/TLS, optional |
| 4190/tcp | ManageSieve | Quản lý filter rule |

---

## 6. Cài Docker và Docker Compose plugin

Cài Docker bản chính thức:

```bash
curl -sSL https://get.docker.com/ | CHANNEL=stable sh
sudo systemctl enable --now docker
```

Cài Docker Compose plugin nếu chưa có:

```bash
sudo apt update
sudo apt install -y docker-compose-plugin
```

Kiểm tra version:

```bash
docker --version
docker compose version
```

Ví dụ kết quả tốt:

```text
Docker version 29.4.1
Docker Compose version v5.1.3
```

---

## 7. Cài đặt mailcow

### 7.1. Cài package phụ trợ

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git openssl curl gawk coreutils grep jq ca-certificates
```

### 7.2. Clone source mailcow

```bash
sudo -i
umask 0022

cd /opt
git clone https://github.com/mailcow/mailcow-dockerized
cd /opt/mailcow-dockerized
```

### 7.3. Generate cấu hình mailcow

Chạy script:

```bash
./generate_config.sh
```

Khi script hỏi hostname:

```text
Mail server hostname (FQDN): mail.luantc.com
```

Khi hỏi timezone:

```text
Timezone [Etc/UTC]: Asia/Ho_Chi_Minh
```

Khi hỏi branch:

```text
Which branch of mailcow do you want to use?
[1] master branch stable updates, default, recommended
[2] nightly branch unstable/testing
[3] legacy branch deprecated/security updates only
```

Chọn:

```text
1
```

hoặc bấm Enter nếu mặc định là `[1]`.

Nếu server không có IPv6 public, script có thể báo:

```text
Only link-local IPv6 present and no external connectivity - disabling IPv6 support.
ENABLE_IPV6=false
```

Điều này bình thường. Nếu không có IPv6 public hoạt động đầy đủ, không nên tạo AAAA record cho `mail.luantc.com`.

Script cũng có thể báo:

```text
Generating snake-oil certificate...
Copying snake-oil certificate...
```

Đây là certificate tạm thời để hệ thống khởi động. Sau khi mailcow chạy, ACME container sẽ tự xin certificate Let's Encrypt thật.

### 7.4. Kiểm tra file cấu hình

```bash
grep -E 'MAILCOW_HOSTNAME|HTTP_PORT|HTTPS_PORT|SKIP_LETS_ENCRYPT|SKIP_SOGO|SKIP_CLAMD' /opt/mailcow-dockerized/mailcow.conf
```

Kết quả mong muốn:

```text
MAILCOW_HOSTNAME=mail.luantc.com
HTTP_PORT=80
HTTPS_PORT=443
SKIP_LETS_ENCRYPT=n
SKIP_CLAMD=n
SKIP_SOGO=n
```

Ý nghĩa:

- `MAILCOW_HOSTNAME=mail.luantc.com`: hostname chính của mail server.
- `HTTP_PORT=80`: dùng cho HTTP và Let's Encrypt challenge.
- `HTTPS_PORT=443`: dùng cho giao diện web HTTPS.
- `SKIP_LETS_ENCRYPT=n`: bật Let's Encrypt tự động.
- `SKIP_CLAMD=n`: bật ClamAV.
- `SKIP_SOGO=n`: bật SOGo webmail.

---

## 8. Pull image và khởi động mailcow

Đứng trong thư mục mailcow:

```bash
cd /opt/mailcow-dockerized
```

Pull image:

```bash
docker compose pull
```

Start mailcow:

```bash
docker compose up -d
```

Kiểm tra container:

```bash
docker compose ps
```

Xem log ACME/Let's Encrypt:

```bash
docker compose logs --tail=150 acme-mailcow
```

Kết quả tốt sẽ có các dòng tương tự:

```text
Found A record for mail.luantc.com: 103.205.98.xxx
Confirmed A record 103.205.98.xxx
Verifying mail.luantc.com...
mail.luantc.com verified!
Certificate signed!
Certificate successfully obtained
```

Nếu thấy các dòng trên, nghĩa là Let's Encrypt đã cấp chứng chỉ thành công.

Kiểm tra SSL:

```bash
echo | openssl s_client -connect mail.luantc.com:443 -servername mail.luantc.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

Kết quả mong muốn issuer là Let's Encrypt.

---

## 9. Kiểm tra firewall UFW

Kiểm tra trạng thái UFW:

```bash
sudo ufw status
```

Nếu UFW đang inactive, có thể bỏ qua.

Nếu UFW đang active, mở các port cần thiết:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 25/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 465/tcp
sudo ufw allow 587/tcp
sudo ufw allow 993/tcp
sudo ufw allow 995/tcp
sudo ufw allow 4190/tcp
sudo ufw reload
```

---

## 10. Truy cập giao diện quản trị mailcow

Mở trình duyệt:

```text
https://mail.luantc.com/admin
```

Tài khoản mặc định:

```text
Username: admin
Password: moohoo
```

Sau khi đăng nhập lần đầu, cần đổi password admin ngay.

Giao diện webmail cho user cuối:

```text
https://mail.luantc.com
```

---

## 11. Tạo mail domain `luantc.com`

Vào menu:

```text
E-Mail -> Configuration -> Domains -> Add domain
```

Điền thông tin:

```text
Tên miền: luantc.com
Mô tả: Mail domain for Kbuor Tech
Template: Default
Thẻ: để trống
```

Cấu hình quota đề xuất:

```text
Số lượng bí danh tối đa có thể tạo: 400
Số lượng hộp thư tối đa có thể tạo: 10
Hạn ngạch hộp thư mặc định: 5120 MiB
Hạn ngạch tối đa mỗi hộp thư: 10240 MiB
Tổng hạn ngạch tên miền: 102400 MiB
```

Ý nghĩa:

- Mỗi mailbox mặc định 5 GiB.
- Một mailbox tối đa 10 GiB.
- Toàn domain tối đa 100 GiB.
- Phù hợp với server 200 GB SSD, vẫn chừa dung lượng cho log, database, Docker volume, index, quarantine và backup tạm.

Các lựa chọn khác:

```text
Danh sách địa chỉ toàn cục GAL: tick
Đang hoạt động: tick
Giới hạn tốc độ: Disabled
Bộ chọn DKIM: dkim
Độ dài khóa DKIM: 2048
```

Phần chuyển tiếp:

```text
Chuyển tiếp tên miền này: không tick
Chuyển tiếp tất cả người nhận: không tick
Chỉ chuyển tiếp hộp thư không tồn tại: không tick
```

Vì `luantc.com` là mail domain chính, không phải relay/forwarding domain.

Cuối cùng bấm:

```text
Thêm tên miền và khởi động lại SOGo
```

---

## 12. Tạo mailbox

Vào:

```text
E-Mail -> Configuration -> Mailboxes -> Add mailbox
```

### 12.1. Mailbox quản trị đầu tiên

Tạo mailbox:

```text
admin@luantc.com
```

Thông tin đề xuất:

```text
Tên người dùng: admin
Tên đầy đủ: Kbuor Tech Admin
Tên miền: luantc.com
Template: Default
Mật khẩu: đặt password mạnh
Hạn ngạch: 5120 MiB
Tagged mail: In subfolder
Quarantine notifications: Daily
Quarantine notification category: All categories
Encryption policy: không bật Enforce TLS incoming/outgoing
Allowed protocols: IMAP, SMTP, Sieve
Rate limit: Disabled
Trạng thái: Đang hoạt động
Direct forwarding to SOGo: tick
```

> Nếu vẫn muốn hỗ trợ POP3 cho user cũ, có thể bật thêm POP3. Tuy nhiên với hệ thống hiện đại, nên ưu tiên IMAP thay vì POP3.

### 12.2. Các mailbox hệ thống nên có

Tạo thêm:

```text
postmaster@luantc.com
abuse@luantc.com
dmarc@luantc.com
```

Gợi ý quota:

```text
postmaster@luantc.com: 1024 MiB
abuse@luantc.com: 1024 MiB
dmarc@luantc.com: 1024 MiB
```

Ý nghĩa:

- `postmaster@luantc.com`: mailbox chuẩn cho vận hành mail domain.
- `abuse@luantc.com`: mailbox chuẩn để nhận abuse report.
- `dmarc@luantc.com`: mailbox nhận DMARC aggregate report.

Với `dmarc@luantc.com`, có thể để quarantine notification là `Never` để tránh nhận thông báo dư thừa.

---

## 13. Cấu hình SPF, DKIM, DMARC

Đây là phần rất quan trọng để cải thiện deliverability.

### 13.1. SPF record

Trên GoDaddy, tạo TXT record:

```text
Type: TXT
Name / Host: @
Value: v=spf1 mx a ip4:103.205.98.xxx -all
TTL: Default hoặc 600
```

Lưu ý:

- Một domain chỉ nên có một SPF record bắt đầu bằng `v=spf1`.
- Nếu đã có SPF cũ, không tạo thêm record thứ hai. Hãy sửa record cũ để gộp lại.

Kiểm tra:

```bash
dig @8.8.8.8 TXT luantc.com
```

Kết quả mong muốn có:

```text
v=spf1 mx a ip4:103.205.98.xxx -all
```

### 13.2. DKIM record

Trong mailcow, vào:

```text
E-Mail -> Configuration -> Domains
```

Ở dòng `luantc.com`, bấm nút:

```text
DNS
```

Tìm dòng DKIM, thường có dạng:

```text
dkim._domainkey.luantc.com
```

Value có dạng:

```text
v=DKIM1;k=rsa;t=s;s=email;p=MIIBIjANBgkqh...
```

Trên GoDaddy, tạo TXT record:

```text
Type: TXT
Name / Host: dkim._domainkey
Value: copy toàn bộ chuỗi bắt đầu bằng v=DKIM1;k=rsa;t=s;s=email;p=...
TTL: Default hoặc 600
```

Lưu ý quan trọng:

- Trong ô `Name / Host` của GoDaddy, thường chỉ nhập `dkim._domainkey`.
- Không nhập full `dkim._domainkey.luantc.com`, vì GoDaddy thường tự nối thêm domain phía sau.
- Nếu nhập sai, record có thể thành `dkim._domainkey.luantc.com.luantc.com`.

Kiểm tra:

```bash
dig @8.8.8.8 TXT dkim._domainkey.luantc.com
dig @1.1.1.1 TXT dkim._domainkey.luantc.com
```

Kết quả mong muốn có:

```text
v=DKIM1;k=rsa;t=s;s=email;p=...
```

### 13.3. DMARC record

Giai đoạn đầu nên để DMARC ở chế độ monitor:

```text
Type: TXT
Name / Host: _dmarc
Value: v=DMARC1; p=none; rua=mailto:dmarc@luantc.com; adkim=s; aspf=s
TTL: Default hoặc 600
```

Kiểm tra:

```bash
dig @8.8.8.8 TXT _dmarc.luantc.com
```

Kết quả mong muốn:

```text
v=DMARC1; p=none; rua=mailto:dmarc@luantc.com; adkim=s; aspf=s
```

Sau khi hệ thống gửi/nhận ổn định, có thể nâng policy theo lộ trình:

```text
Giai đoạn 1: p=none
Giai đoạn 2: p=quarantine
Giai đoạn 3: p=reject
```

Không nên để `p=reject` ngay từ đầu khi chưa xác nhận SPF/DKIM/DMARC pass ổn định.

---

## 14. Autodiscover và Autoconfig

Để mail client dễ tự động nhận cấu hình, thêm các record sau.

### 14.1. Autodiscover

```text
Type: CNAME
Name / Host: autodiscover
Value: mail.luantc.com
TTL: Default hoặc 600
```

### 14.2. Autoconfig

```text
Type: CNAME
Name / Host: autoconfig
Value: mail.luantc.com
TTL: Default hoặc 600
```

Kiểm tra:

```bash
dig @8.8.8.8 CNAME autodiscover.luantc.com
dig @8.8.8.8 CNAME autoconfig.luantc.com
```

Kết quả mong muốn:

```text
mail.luantc.com.
```

---

## 15. Kiểm tra toàn bộ DNS sau khi cấu hình

Chạy trên server:

```bash
dig @8.8.8.8 +short mail.luantc.com A
dig @8.8.8.8 +short luantc.com MX
dig @8.8.8.8 +short -x 103.205.98.xxx
dig @8.8.8.8 TXT luantc.com
dig @8.8.8.8 TXT _dmarc.luantc.com
dig @8.8.8.8 TXT dkim._domainkey.luantc.com
dig @8.8.8.8 CNAME autodiscover.luantc.com
dig @8.8.8.8 CNAME autoconfig.luantc.com
```

Checklist kết quả:

```text
mail.luantc.com A                  -> 103.205.98.xxx
luantc.com MX                      -> mail.luantc.com
103.205.98.xxx PTR                 -> mail.luantc.com
luantc.com TXT                     -> v=spf1 ...
_dmarc.luantc.com TXT              -> v=DMARC1 ...
dkim._domainkey.luantc.com TXT     -> v=DKIM1 ...
autodiscover.luantc.com CNAME      -> mail.luantc.com
autoconfig.luantc.com CNAME        -> mail.luantc.com
```

---

## 16. Test đăng nhập webmail

Truy cập:

```text
https://mail.luantc.com/SOGo
```

Đăng nhập bằng mailbox đã tạo:

```text
admin@luantc.com
```

Nếu đăng nhập thành công, kiểm tra các mục:

- Inbox
- Sent
- Drafts
- Junk
- Calendar/Contacts nếu dùng SOGo groupware

---

## 17. Test gửi nhận mail

### 17.1. Test nhận mail

Từ Gmail hoặc một mail bên ngoài gửi vào:

```text
admin@luantc.com
```

Sau đó kiểm tra trong SOGo.

Nếu chưa nhận được, xem log:

```bash
cd /opt/mailcow-dockerized
docker compose logs -f postfix-mailcow
```

### 17.2. Test gửi mail ra ngoài

Từ SOGo, dùng `admin@luantc.com` gửi mail đến Gmail.

Trong Gmail, mở email nhận được, chọn:

```text
Show original
```

Cần thấy:

```text
SPF: PASS
DKIM: PASS
DMARC: PASS
```

Nếu cả 3 đều PASS, nền tảng xác thực email đã đúng.

---

## 18. Cấu hình mail client thủ công

Nếu người dùng không dùng webmail mà dùng Outlook, Thunderbird, Apple Mail hoặc mobile mail client, có thể cấu hình thủ công như sau.

### 18.1. Incoming IMAP

```text
Protocol: IMAP
Server: mail.luantc.com
Port: 993
Security: SSL/TLS
Username: địa chỉ email đầy đủ, ví dụ admin@luantc.com
Password: mật khẩu mailbox
```

### 18.2. Outgoing SMTP

```text
Protocol: SMTP Submission
Server: mail.luantc.com
Port: 587
Security: STARTTLS
Authentication: Required
Username: địa chỉ email đầy đủ, ví dụ admin@luantc.com
Password: mật khẩu mailbox
```

Hoặc dùng SMTPS:

```text
Protocol: SMTPS
Server: mail.luantc.com
Port: 465
Security: SSL/TLS
Authentication: Required
```

Khuyến nghị dùng port `587` với STARTTLS cho mail client.

---

## 19. Các lệnh vận hành cơ bản

### 19.1. Xem trạng thái container

```bash
cd /opt/mailcow-dockerized
docker compose ps
```

### 19.2. Xem log tổng

```bash
docker compose logs --tail=100
```

### 19.3. Xem log ACME/Let's Encrypt

```bash
docker compose logs --tail=150 acme-mailcow
```

### 19.4. Xem log Postfix

```bash
docker compose logs -f postfix-mailcow
```

Dùng khi debug gửi/nhận SMTP.

### 19.5. Xem log Dovecot

```bash
docker compose logs -f dovecot-mailcow
```

Dùng khi debug IMAP/POP3/login mailbox.

### 19.6. Xem log Rspamd

```bash
docker compose logs -f rspamd-mailcow
```

Dùng khi debug spam filtering, DKIM signing, quarantine.

### 19.7. Restart toàn bộ mailcow

```bash
cd /opt/mailcow-dockerized
docker compose restart
```

### 19.8. Stop mailcow

```bash
cd /opt/mailcow-dockerized
docker compose down
```

### 19.9. Start mailcow

```bash
cd /opt/mailcow-dockerized
docker compose up -d
```

---

## 20. Update mailcow

Update bằng script chính thức trong thư mục mailcow:

```bash
cd /opt/mailcow-dockerized
./update.sh
```

Trước khi update production, nên:

- Đọc release note nếu có.
- Backup trước khi update.
- Update vào khung giờ ít người dùng.
- Kiểm tra lại `docker compose ps` sau update.
- Test gửi/nhận mail sau update.

---

## 21. Backup cơ bản

mailcow có script backup riêng trong thư mục helper.

Ví dụ chạy backup:

```bash
cd /opt/mailcow-dockerized
./helper-scripts/backup_and_restore.sh backup all
```

Khi chạy, script sẽ hỏi đường dẫn lưu backup.

Khuyến nghị:

- Không lưu backup lâu dài trên cùng disk với mail server.
- Đồng bộ backup sang storage ngoài hoặc object storage.
- Kiểm tra restore định kỳ.
- Backup trước mỗi lần update lớn.

Các thành phần cần backup:

- Mailbox data.
- Database.
- Redis/Rspamd data nếu cần.
- `mailcow.conf`.
- Custom config nếu có.
- DKIM keys.

---

## 22. Một số best practices chống vào spam

Không có mail server self-host nào cam kết 100% không vào spam. Tuy nhiên, hệ thống nên đạt các tiêu chí sau:

```text
A record đúng
PTR record đúng
MX record đúng
SPF pass
DKIM pass
DMARC pass
TLS hoạt động
IP không nằm trong blacklist lớn
Không có open relay
Không gửi bulk mail ngay từ đầu
Không gửi nội dung giống spam
Không dùng link rút gọn trong email
Không gửi attachment đáng ngờ
Warm-up IP/domain từ từ
Theo dõi Google Postmaster Tools nếu gửi nhiều đến Gmail
```

### 22.1. Warm-up domain/IP

Với server mới, không nên gửi lượng lớn mail ngay trong ngày đầu.

Gợi ý:

```text
Ngày 1–3: gửi test nội bộ và một số email thật số lượng nhỏ
Ngày 4–7: tăng dần nếu mail không vào spam
Sau 1–2 tuần: mới dùng ổn định cho vận hành hằng ngày
```

### 22.2. Nếu vẫn vào spam dù SPF/DKIM/DMARC PASS

Các nguyên nhân thường gặp:

- IP mới, chưa có reputation.
- IP từng bị lạm dụng trước đây.
- ASN/datacenter có reputation kém.
- Nội dung mail có nhiều dấu hiệu marketing/spam.
- Domain mới, chưa có lịch sử gửi mail.
- Người nhận không tương tác hoặc đánh dấu spam.

Phương án xử lý:

- Kiểm tra blacklist.
- Gửi mail tự nhiên, tăng dần volume.
- Tránh nội dung quá giống quảng cáo.
- Dùng chữ ký email rõ ràng.
- Đăng ký Google Postmaster Tools.
- Nếu cần deliverability cao hơn, dùng outbound SMTP relay như Amazon SES/Mailgun/SendGrid.

---

## 23. Troubleshooting thường gặp

### 23.1. Let's Encrypt không cấp được certificate

Kiểm tra:

```bash
dig @8.8.8.8 +short mail.luantc.com A
sudo ss -tlpn | grep -E ':(80|443)\b'
cd /opt/mailcow-dockerized
docker compose logs --tail=150 acme-mailcow
```

Nguyên nhân thường gặp:

- A record chưa trỏ đúng IP.
- Port 80 bị firewall chặn.
- Có Nginx/Apache khác đang chiếm port 80/443.
- DNS chưa propagate.

### 23.2. Không nhận được mail từ bên ngoài

Kiểm tra:

```bash
dig @8.8.8.8 +short luantc.com MX
sudo ss -tlpn | grep ':25'
cd /opt/mailcow-dockerized
docker compose logs -f postfix-mailcow
```

Nguyên nhân thường gặp:

- MX sai.
- Port 25 inbound bị chặn.
- Firewall/security group chưa mở port 25.
- Domain chưa được add trong mailcow.
- Mailbox chưa tồn tại.

### 23.3. Gửi mail ra ngoài không được

Kiểm tra outbound port 25:

```bash
nc -vz gmail-smtp-in.l.google.com 25
```

Xem log:

```bash
cd /opt/mailcow-dockerized
docker compose logs -f postfix-mailcow
```

Nguyên nhân thường gặp:

- Outbound TCP/25 bị provider chặn.
- DNS resolver lỗi.
- IP bị blacklist.
- SPF/DKIM/DMARC chưa đúng.

### 23.4. DKIM không PASS

Kiểm tra DNS:

```bash
dig @8.8.8.8 TXT dkim._domainkey.luantc.com
```

Các lỗi hay gặp:

- Nhập sai Host ở GoDaddy.
- Nhập full `dkim._domainkey.luantc.com` khiến record bị thành `dkim._domainkey.luantc.com.luantc.com`.
- Copy thiếu chuỗi `p=`.
- DNS chưa propagate.
- Tạo DKIM selector khác nhưng DNS lại dùng selector `dkim`.

### 23.5. SPF bị PermError

Kiểm tra:

```bash
dig @8.8.8.8 TXT luantc.com
```

Nếu thấy nhiều record cùng bắt đầu bằng `v=spf1`, cần gộp lại còn một record duy nhất.

Ví dụ đúng:

```text
v=spf1 mx a ip4:103.205.98.xxx -all
```

### 23.6. DMARC không PASS

Kiểm tra:

```bash
dig @8.8.8.8 TXT _dmarc.luantc.com
```

Giai đoạn đầu dùng:

```text
v=DMARC1; p=none; rua=mailto:dmarc@luantc.com; adkim=s; aspf=s
```

Đảm bảo SPF hoặc DKIM pass và aligned với domain `luantc.com`.

---

## 24. Checklist hoàn tất triển khai

### 24.1. Server

```text
Hostname là mail.luantc.com
/etc/hosts cấu hình đúng
Timezone là Asia/Ho_Chi_Minh
Docker chạy ổn
mailcow container đều Up
Let's Encrypt certificate đã cấp thành công
```

### 24.2. DNS

```text
A mail.luantc.com -> 103.205.98.xxx
MX luantc.com -> mail.luantc.com
PTR 103.205.98.xxx -> mail.luantc.com
TXT SPF tồn tại và chỉ có một record SPF
TXT DKIM tồn tại đúng selector dkim._domainkey
TXT DMARC tồn tại ở _dmarc
CNAME autodiscover -> mail.luantc.com
CNAME autoconfig -> mail.luantc.com
Không tạo AAAA nếu không có IPv6 public chuẩn
```

### 24.3. mailcow

```text
Admin password đã đổi
Domain luantc.com đã add
Mailbox admin@luantc.com đã tạo
Mailbox postmaster@luantc.com đã tạo
Mailbox abuse@luantc.com đã tạo
Mailbox dmarc@luantc.com đã tạo
DKIM key 2048-bit đã tạo
SOGo login thành công
```

### 24.4. Deliverability

```text
Gmail nhận được mail từ admin@luantc.com
Gmail Show original: SPF PASS
Gmail Show original: DKIM PASS
Gmail Show original: DMARC PASS
Mail không bị reject
Mail không nằm trong spam ở các test đầu tiên hoặc nếu có thì cần warm-up/reputation tuning
```

---

## 25. Kết luận

Với cấu hình Ubuntu 24.04, 4 CPU, 8 GB RAM, 200 GB SSD và Public IP trực tiếp, mailcow là lựa chọn phù hợp để triển khai email server open-source cho domain `luantc.com` với quy mô 5–50 users.

Kiến trúc này đáp ứng các yêu cầu chính:

- Gửi/nhận email bằng domain riêng.
- Có webmail cho người dùng cuối qua SOGo.
- Có giao diện quản trị mailbox/domain.
- Có HTTPS bằng Let's Encrypt.
- Có SPF/DKIM/DMARC để cải thiện deliverability.
- Có anti-spam, antivirus, quarantine.
- Dễ vận hành bằng Docker Compose.

Điểm cần tiếp tục theo dõi sau khi golive:

- Reputation của IP `103.205.98.xxx`.
- Tỷ lệ mail vào spam khi gửi đến Gmail/Microsoft/Yahoo.
- Log Postfix/Rspamd.
- Dung lượng mailbox.
- Backup định kỳ.
- Update mailcow định kỳ.
- Chính sách DMARC nâng dần từ `p=none` lên `p=quarantine`, sau đó mới cân nhắc `p=reject`.
