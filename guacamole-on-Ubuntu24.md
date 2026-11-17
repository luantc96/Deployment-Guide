# Runbook: Apache Guacamole Remote Gateway — `remote.luantc.com`

> Triển khai Apache Guacamole 1.6.0 trên Ubuntu 24.04 làm điểm remote (RDP/SSH/VNC) vào LAN nội bộ.
> Yêu cầu: HTTPS (Let's Encrypt, tự gia hạn), 2FA (TOTP), fail2ban chống brute-force.
> Ngày triển khai: 2026-11-16

---

## 1. Thông tin môi trường

| Hạng mục | Giá trị |
|---|---|
| Hostname | `OVPN-Server` (VMware VM, cũng chạy OpenVPN server) |
| OS | Ubuntu 24.04 LTS |
| IP host | `10.0.24.2/24` (`ens192`), gateway `10.0.24.1` |
| LAN nội bộ | `10.0.24.0/24` |
| OpenVPN client pool | `10.8.0.0/24` (`tun0`) |
| Docker address pool | `10.250.0.0/20` (chia /24) |
| FQDN | `remote.luantc.com` → `103.141.177.136` (IP public firewall biên) |
| Port-forward (firewall biên) | TCP 80, 443 → `10.0.24.2` (DNAT, **không SNAT**) |
| Guacamole | 1.6.0 (Docker), PostgreSQL 16 |
| Thư mục stack | `/opt/guacamole` |

### Kiến trúc

```
Internet ──443──> Firewall biên (DNAT) ──> Nginx (host, cert Let's Encrypt)
                                              └─> 127.0.0.1:8080 guacamole (container)
                                                     ├─> guacd (container) ──NAT──> RDP/SSH 10.0.24.x
                                                     └─> PostgreSQL (container: user, connection, TOTP secret)

fail2ban (host) ── đọc log container guacamole qua journald ──> ban IP bằng nftables (port 80/443)
```

### Các lớp bảo vệ

| Lớp | Cơ chế | Ngưỡng |
|---|---|---|
| Transport | HTTPS TLS 1.2/1.3, HSTS | — |
| Xác thực | Password (PostgreSQL) + TOTP bắt buộc | — |
| Brute-force L1 | Extension `guacamole-auth-ban` (tầng ứng dụng) | 5 lần sai → chặn 900s |
| Brute-force L2 | fail2ban (nftables, tầng host) | 5 lần sai / 15 phút → ban 1h, tăng dần nếu tái phạm |

---

## 2. Điều kiện tiên quyết

- DNS A record `remote.luantc.com` trỏ về IP public của firewall biên.
- Firewall biên DNAT **TCP 80 và 443** → `10.0.24.2`. Port 80 phải mở vĩnh viễn (Let's Encrypt HTTP-01 dùng khi cấp và khi gia hạn).
- Firewall biên **không được SNAT** inbound traffic — nếu SNAT, mọi request mang IP firewall và fail2ban sẽ ban chính firewall.
- Port 80/443 trên host chưa bị dịch vụ khác chiếm:
  ```bash
  ss -ltnup | grep -E ':(80|443)\s'     # mong đợi: rỗng
  ```
- Đồng bộ thời gian (TOTP phụ thuộc giờ hệ thống):
  ```bash
  timedatectl | grep -E "synchronized|NTP"   # System clock synchronized: yes
  ```

---

## 3. Cài Docker Engine

### 3.1. Cấu hình address pool **trước** khi cài

> ⚠️ **Bài học:** pool Docker không được trùng với bất kỳ route nào trên host (LAN `10.0.24.0/24`, VPN `10.8.0.0/24`) hoặc subnet nội bộ mà guacd cần truy cập. Nếu trùng, `dockerd` không khởi động được với lỗi:
> `error creating default "bridge" network: all predefined address pools have been fully subnetted`
>
> Pool cũng phải đủ lớn để chia ít nhất 2 subnet (bridge `docker0` + network của compose) — `base /24` với `size 24` chỉ cho 1 subnet → không đủ.

```bash
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<'EOF'
{ "default-address-pools": [ { "base": "10.250.0.0/20", "size": 24 } ] }
EOF
python3 -m json.tool /etc/docker/daemon.json
```

### 3.2. Cài từ repo chính hãng Docker

```bash
apt-get update
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3.3. Kiểm tra

```bash
systemctl is-active docker            # active
ip -4 addr show docker0               # inet 10.250.0.1/24
iptables -S FORWARD | head -1         # -P FORWARD ACCEPT (không đổi so với trước khi cài)
```

> Vì `net.ipv4.ip_forward=1` đã có sẵn (do OpenVPN), Docker không đổi policy FORWARD → traffic OpenVPN client vào LAN không bị ảnh hưởng. Kiểm tra lại bằng một VPN client sau khi cài.

---

## 4. Triển khai stack Guacamole

### 4.1. Sinh schema PostgreSQL

```bash
mkdir -p /opt/guacamole/init && cd /opt/guacamole
docker run --rm guacamole/guacamole:1.6.0 /opt/guacamole/bin/initdb.sh --postgresql > init/initdb.sql
grep -c "CREATE TABLE" init/initdb.sql   # > 0
```

> Dùng file SQL mount vào `/docker-entrypoint-initdb.d` thay vì cách named volume dùng chung (docs method 2) — tránh race condition khi Compose tạo container postgres trước khi volume có schema.

### 4.2. File `.env`

```bash
cat > /opt/guacamole/.env <<EOF
POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=')
EOF
chmod 600 /opt/guacamole/.env
```

### 4.3. `docker-compose.yml`

```bash
cat > /opt/guacamole/docker-compose.yml <<'EOF'
services:
  guacd:
    image: guacamole/guacd:1.6.0
    container_name: guacd
    restart: unless-stopped
    networks: [guacnet]

  postgres:
    image: postgres:16
    container_name: guac-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: guacamole_db
      POSTGRES_USER: guacamole_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - ./init:/docker-entrypoint-initdb.d:ro
      - pgdata:/var/lib/postgresql/data
    networks: [guacnet]

  guacamole:
    image: guacamole/guacamole:1.6.0
    container_name: guacamole
    restart: unless-stopped
    depends_on: [guacd, postgres]
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRESQL_ENABLED: "true"
      POSTGRESQL_HOSTNAME: postgres
      POSTGRESQL_DATABASE: guacamole_db
      POSTGRESQL_USERNAME: guacamole_user
      POSTGRESQL_PASSWORD: ${POSTGRES_PASSWORD}
      TOTP_ENABLED: "true"
      TOTP_ISSUER: "Guacamole Remote Access"
      REMOTE_IP_VALVE_ENABLED: "true"
      BAN_MAX_INVALID_ATTEMPTS: "5"
      BAN_ADDRESS_DURATION: "900"
    ports:
      - "127.0.0.1:8080:8080"
    logging:
      driver: journald
      options:
        tag: guacamole
    networks: [guacnet]

volumes:
  pgdata:

networks:
  guacnet:
EOF
```

Giải thích các biến quan trọng:

| Biến | Mục đích |
|---|---|
| `POSTGRESQL_USERNAME` | Tên chính xác theo docs 1.6.0 (không phải `POSTGRESQL_USER`) |
| `TOTP_ENABLED` | Bật extension TOTP — TOTP cần DB backend để lưu secret |
| `REMOTE_IP_VALVE_ENABLED` | Bật Tomcat RemoteIpValve để nhận IP client thật từ header `X-Forwarded-For` của Nginx. Mặc định tin proxy có IP private (gateway Docker bridge thỏa điều kiện) |
| `BAN_*` | Extension ban bật mặc định trên Docker; nâng thời gian ban từ 300s lên 900s |
| `127.0.0.1:8080` | Chỉ Nginx trên host truy cập được; không bị lộ dù Docker bypass UFW |
| `logging: journald` | fail2ban đọc log ổn định qua `CONTAINER_NAME`, không phụ thuộc container ID |

### 4.4. Khởi động và kiểm tra

```bash
cd /opt/guacamole
docker compose config -q && echo "compose OK"
docker compose up -d
docker compose ps                                                     # 3 container Up
docker logs guacamole 2>&1 | grep -iE "error|exception|started" | tail
docker exec guac-postgres psql -U guacamole_user -d guacamole_db -c '\dt' | head   # bảng guacamole_*
curl -sI http://127.0.0.1:8080/guacamole/ | head -1                   # HTTP/1.1 200
docker logs guacd 2>&1 | tail -3                                      # "Listening on host 0.0.0.0, port 4822"
```

Log khởi động cần thấy 3 extension được load:

```
Extension "Brute-force Authentication Detection/Prevention" (ban) loaded.
Extension "PostgreSQL Authentication" (postgresql) loaded.
Extension "TOTP TFA Authentication Backend" (totp) loaded.
Addresses will be automatically banned for 900 seconds after 5 failed authentication attempts.
```

> **Ghi chú:**
> - guacd hiển thị `health: starting` trong ~5 phút đầu là bình thường — healthcheck của image có `Interval` = 5 phút (`nc -z 127.0.0.1 4822`).
> - Log trong container dùng giờ **UTC** (chậm hơn giờ VN 7 tiếng). journald/fail2ban dùng giờ host nên không bị ảnh hưởng.

---

## 5. Nginx reverse proxy + Let's Encrypt

### 5.1. Cài Nginx và certbot

```bash
apt-get install -y nginx
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
```

### 5.2. Cấp certificate

```bash
certbot certonly --nginx -d remote.luantc.com \
  --deploy-hook "systemctl reload nginx" \
  -m luantc@luantc.com --agree-tos --no-eff-email
```

> `--deploy-hook` được lưu vào file renewal → mỗi lần gia hạn thành công Nginx tự reload để nạp cert mới.

### 5.3. Cấu hình site

```bash
cat > /etc/nginx/sites-available/guacamole <<'EOF'
server {
    listen 80;
    listen [::]:80;
    server_name remote.luantc.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name remote.luantc.com;

    ssl_certificate     /etc/letsencrypt/live/remote.luantc.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/remote.luantc.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    add_header Strict-Transport-Security "max-age=31536000" always;

    location = / { return 302 /guacamole/; }

    location /guacamole/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        client_max_body_size 1g;
        access_log off;
    }
}
EOF

ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

| Directive | Lý do |
|---|---|
| `proxy_buffering off` | Bắt buộc theo docs — buffering làm nghẽn HTTP tunnel của Guacamole |
| `Upgrade` / `Connection` | Cho phép WebSocket |
| `X-Forwarded-For $remote_addr` | **Khác docs** (docs dùng `$proxy_add_x_forwarded_for`). Nginx là edge nên ghi đè header bằng IP thật, ngăn client tự chèn IP giả để né ban hoặc làm fail2ban ban nhầm |
| `client_max_body_size 1g` | Nginx mặc định giới hạn upload 1MB |

### 5.4. Kiểm tra

```bash
curl -sI http://127.0.0.1 -H "Host: remote.luantc.com" | head -1                                  # 301
curl -sI https://remote.luantc.com/guacamole/ --resolve remote.luantc.com:443:127.0.0.1 | head -1   # 200
certbot renew --dry-run                                                                            # succeeded
systemctl list-timers | grep -i certbot                                                            # snap.certbot.renew.timer
grep renew_hook /etc/letsencrypt/renewal/remote.luantc.com.conf                                    # systemctl reload nginx
```

Từ mạng ngoài (vd. 4G): mở `https://remote.luantc.com` → màn hình đăng nhập Guacamole, cert hợp lệ.

---

## 6. Đăng nhập lần đầu & TOTP

> Làm ngay khi port 443 thông ra Internet — `guacadmin/guacadmin` là credential mặc định công khai.

1. Đăng nhập `guacadmin` / `guacadmin` → quét QR, đăng ký TOTP.
2. **Settings → Users → New User**: tạo admin riêng (không dùng tên dễ đoán như `admin`), mật khẩu mạnh, tick **Administer system**.
3. Đăng xuất → đăng nhập bằng admin mới → đăng ký TOTP.
4. **Settings → Users → guacadmin → Delete**.
5. Xác nhận ô **Disable TOTP** của admin mới **không** được tick.

Kiểm tra:

```bash
# Chỉ còn admin mới
docker exec guac-postgres psql -U guacamole_user -d guacamole_db -c "SELECT name FROM guacamole_entity WHERE type='USER';"

# IP ghi nhận phải là IP public thật của client (không phải 10.0.24.1 hay 10.250.x.x)
docker logs guacamole 2>&1 | grep -i "successfully authenticated" | tail -3
```

Mẫu log đúng:

```
User "luantc" (authenticated by "postgresql") successfully authenticated from 116.108.160.221
```

### Quản trị TOTP

| Tình huống | Thao tác |
|---|---|
| User mất điện thoại | Settings → Users → user → tick **Clear TOTP secret** → Save. User đăng ký lại ở lần login sau |
| Cho user quét lại cùng key | Bỏ tick **TOTP key confirmed** → Save |
| Miễn TOTP cho user/group cụ thể (1.6.0+) | Tick **Disable TOTP** trên user hoặc group |

---

## 7. fail2ban

### 7.1. Cài đặt

```bash
apt-get update && apt-get install -y fail2ban python3-systemd
dpkg -l fail2ban | tail -1      # phải là 1.0.2-3ubuntu0.1 trở lên
```

> Bản `1.0.2-3` gốc của Ubuntu 24.04 không khởi động được (`No module named 'asynchat'` do Python 3.12) — đã fix trong `1.0.2-3ubuntu0.1` (noble-updates, LP#2055114). Các dòng `SyntaxWarning` khi cài nằm trong file test của package, bỏ qua được.

### 7.2. Format log xác thực thất bại (Guacamole 1.6.0)

Guacamole 1.6.0 ghi log qua `EventLoggingListener`, **khác** format `AuthenticationService` của các bản cũ mà nhiều hướng dẫn trên Internet đang dùng:

```
WARN  o.a.g.event.EventLoggingListener - Authentication attempt from 116.108.160.221 for user "luantc" failed: Invalid login (rejected by "postgresql")
WARN  o.a.g.event.EventLoggingListener - Authentication attempt from 116.108.160.221 for user "luantc" failed: Provided TOTP code is not valid. (rejected by "totp")
```

Đã xác minh thực tế:
- Sai password, sai username, sai TOTP → đều sinh dòng `failed` ở trên.
- Đăng nhập hợp lệ (bước password → chờ TOTP → nhập đúng) **không** sinh dòng `failed` → không ban nhầm.
- Extension ban sinh thêm dòng `InMemoryAuthenticationFailureTracker - Authentication has failed for address ...` — filter **không** bắt dòng này để tránh đếm đôi.

### 7.3. Filter

```bash
cat > /etc/fail2ban/filter.d/guacamole-docker.conf <<'EOF'
[Definition]
failregex = EventLoggingListener - Authentication attempt from \[?<HOST>(?:,[^\]]*)?\]? for user "<F-USER>[^"]*</F-USER>" failed:
ignoreregex =
EOF
```

### 7.4. Jail

> Đặt tên `guacamole-docker` để không kế thừa jail `guacamole` có sẵn trong `jail.conf`.

```bash
cat > /etc/fail2ban/jail.d/guacamole.local <<'EOF'
[guacamole-docker]
enabled      = true
backend      = systemd
journalmatch = CONTAINER_NAME=guacamole
filter       = guacamole-docker
port         = http,https
maxretry     = 5
findtime     = 15m
bantime      = 1h
bantime.increment = true
ignoreip     = 127.0.0.1/8 ::1 10.0.24.0/24 10.8.0.0/24
EOF
```

> `maxretry` nên ≤ `BAN_MAX_INVALID_ATTEMPTS` — sau khi extension ban chặn, các request tiếp theo có thể không còn sinh log `failed`.

### 7.5. Test regex và kích hoạt

```bash
# Test trên journal (cần có sẵn vài lần đăng nhập sai)
fail2ban-regex -m "CONTAINER_NAME=guacamole" systemd-journal /etc/fail2ban/filter.d/guacamole-docker.conf | grep -A3 "^Failregex"

systemctl enable --now fail2ban
fail2ban-client status
fail2ban-client status guacamole-docker
fail2ban-client get guacamole-docker actions     # nftables (mặc định Ubuntu 24.04)
```

> - Ubuntu 24.04 dùng banaction **nftables** → kiểm tra rule bằng `nft`, không phải `iptables`.
> - Backend systemd lấy timestamp từ journal (giờ host) → không bị lệch do log container dùng UTC.
> - Khi khởi động, fail2ban đọc lại các entry trong khoảng `findtime`.

### 7.6. Test ban thực tế

Từ một IP ngoài (không thuộc `ignoreip`), đăng nhập sai 5 lần trong 15 phút:

```bash
fail2ban-client status guacamole-docker      # Banned IP list có IP test
nft list ruleset | grep -B2 -A10 f2b         # set chứa IP + rule reject port 80,443
tail -n 3 /var/log/fail2ban.log              # NOTICE [guacamole-docker] Ban <IP>
```

Truy cập từ IP bị ban → bị từ chối ở tầng TCP. Gỡ ban:

```bash
fail2ban-client set guacamole-docker unbanip <IP>
```

> Extension ban của Guacamole giữ IP thêm tối đa 900s và tự hết hạn.

---

## 8. Tạo connection RDP

Kiểm tra kết nối từ host trước:

```bash
nc -zv 10.0.24.196 3389
```

**Settings → Connections → New Connection:**

| Trường | Giá trị |
|---|---|
| Protocol | RDP |
| Hostname | `10.0.24.<x>` |
| Port | 3389 |
| Security mode | `Any` |
| Ignore server certificate | ✓ (cert RDP self-signed mặc định của Windows) |

> - Server đích thấy IP nguồn là `10.0.24.2` (guacd đi qua NAT của Docker). Nếu firewall/GPO trên server đích giới hạn nguồn RDP, phải cho phép `10.0.24.2`.
> - Nên gán connection cho group thay vì từng user (least privilege).

---

## 9. Vận hành

### Lệnh thường dùng

```bash
cd /opt/guacamole
docker compose ps
docker logs guacamole --since 1h 2>&1 | grep -E "Authentication attempt|successfully authenticated"
journalctl CONTAINER_NAME=guacamole -f -o cat
fail2ban-client status guacamole-docker
fail2ban-client set guacamole-docker unbanip <IP>
certbot certificates
```

### Backup DB

`pgdata` chứa toàn bộ user, connection, TOTP secret.

```bash
mkdir -p /root/guac-backup
docker exec guac-postgres pg_dump -U guacamole_user guacamole_db | gzip > /root/guac-backup/guac-$(date +%F).sql.gz
```

Cron hằng ngày (lưu ý escape `%` trong crontab):

```
30 2 * * * docker exec guac-postgres pg_dump -U guacamole_user guacamole_db | gzip > /root/guac-backup/guac-$(date +\%F).sql.gz
```

Backup kèm: `/opt/guacamole/{docker-compose.yml,.env}`, `/etc/nginx/sites-available/guacamole`, `/etc/fail2ban/{filter.d/guacamole-docker.conf,jail.d/guacamole.local}`, `/etc/docker/daemon.json`, `/etc/letsencrypt/`.

### Thay đổi cấu hình / nâng cấp

- Biến môi trường Docker chỉ áp dụng khi container được **tạo lại** → `docker compose up -d`. Thao tác này recreate container `guacamole` và **ngắt toàn bộ session đang mở** → làm ngoài giờ.
- Nâng cấp (vd. lên 1.6.1 khi phát hành — đang chờ fix regression GUACAMOLE-2118): đọc release notes (có thể có schema upgrade SQL), backup DB, đổi tag image, `docker compose pull && docker compose up -d`.

### Reboot host

Sau reboot, stack tự lên (`restart: unless-stopped`, nginx/fail2ban/docker đã `enable`). Kiểm tra:

```bash
docker compose -f /opt/guacamole/docker-compose.yml ps
systemctl is-active docker nginx fail2ban
fail2ban-client status guacamole-docker
```

> Reboot ảnh hưởng cả OpenVPN và session Guacamole → chỉ thực hiện trong maintenance window.

---

## 10. Troubleshooting

| Triệu chứng | Nguyên nhân / Xử lý |
|---|---|
| `dockerd` fail: `all predefined address pools have been fully subnetted` | Pool trong `daemon.json` trùng route của host hoặc quá nhỏ. Sửa pool (mục 3.1) → `systemctl reset-failed docker.service docker.socket && systemctl start docker` |
| `docker.service: Start request repeated too quickly` | systemd khóa unit sau nhiều lần fail → `systemctl reset-failed docker.service docker.socket` trước khi start lại |
| `certbot --nginx`: `Could not find a usable 'nginx' binary` | Chưa cài nginx |
| Log Guacamole hiện IP `10.250.x.x` | RemoteIpValve chưa hoạt động — kiểm tra `REMOTE_IP_VALVE_ENABLED` |
| Log Guacamole hiện IP `10.0.24.1` | Firewall biên đang SNAT inbound — tắt SNAT cho rule DNAT 80/443 |
| fail2ban không ban dù có log sai | Kiểm tra `fail2ban-regex` (mục 7.5); kiểm tra IP có nằm trong `ignoreip` |
| `fail2ban-client`: `Failed to access socket path` ngay sau restart | Daemon chưa kịp tạo socket — đợi vài giây rồi chạy lại |
| guacd `health: starting` lâu | Bình thường trong 5 phút đầu (healthcheck interval 5 phút) |

---
