# Mailu Mail Server Setup

Self-hosted mail server menggunakan Mailu dengan Nginx reverse proxy.

## 📋 Spesifikasi

- **Mail Server**: Mailu (Docker-based)
- **Domain**: mail.madaka.my.id
- **Web Server**: Nginx (reverse proxy)
- **SSL**: Let's Encrypt (auto-generated)
- **Services**: SMTP, IMAP, Webmail (Roundcube), Admin Panel

## 🔧 Persyaratan

- Ubuntu/Debian Server
- Docker & Docker Compose
- Nginx
- Domain dengan DNS yang sudah dikonfigurasi
- Port yang terbuka: 25, 80, 443, 465, 587, 143, 993

## 📦 Instalasi

### 1. Install Dependencies

```bash
# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
apt install docker-compose-plugin -y

# Install Nginx
apt install nginx -y
```

### 2. Clone Repository

```bash
cd /srv
git clone <your-repo-url> mailcow
cd mailcow/mailu
```

### 3. Konfigurasi Environment

Edit file `.env` dan sesuaikan:

```bash
# Domain configuration
DOMAIN=your-domain.com
HOSTNAMES=mail.your-domain.com

# Bind to localhost (Nginx will reverse proxy)
BIND_ADDRESS4=127.0.0.1

# Generate secret key
SECRET_KEY=$(openssl rand -hex 16)

# TLS flavor (notls because Nginx handles SSL)
TLS_FLAVOR=notls

# Path configuration
ROOT=/srv/mailcow/mailu/data
```

### 4. Setup Nginx Reverse Proxy

Copy konfigurasi Nginx:

```bash
cp nginx-config/mail.conf /etc/nginx/sites-available/mail.your-domain.com
ln -s /etc/nginx/sites-available/mail.your-domain.com /etc/nginx/sites-enabled/

# Edit file dan ganti domain
nano /etc/nginx/sites-available/mail.your-domain.com

# Test dan reload nginx
nginx -t
systemctl reload nginx
```

### 5. Start Mailu

```bash
cd /srv/mailcow/mailu
docker compose up -d
```

### 6. Tunggu hingga semua containers healthy

```bash
docker compose ps
```

## 🎯 Membuat Admin User

```bash
cd /srv/mailcow/mailu
docker compose exec admin flask mailu admin admin your-domain.com 'YourPassword123'
```

Login credentials:
- Email: `admin@your-domain.com`
- Password: `YourPassword123`

## 🌐 DNS Configuration

### A Record
```
Type: A
Name: mail.your-domain.com
Value: YOUR_SERVER_IP
TTL: 3600
```

### MX Record
```
Type: MX
Name: your-domain.com (atau @)
Value: mail.your-domain.com
Priority: 10
TTL: 3600
```

### SPF Record
```
Type: TXT
Name: your-domain.com (atau @)
Value: v=spf1 mx ~all
TTL: 3600
```

### DMARC Record
```
Type: TXT
Name: _dmarc.your-domain.com
Value: v=DMARC1; p=quarantine; rua=mailto:admin@your-domain.com
TTL: 3600
```

### DKIM Record

1. Login ke admin panel: https://mail.your-domain.com/admin
2. Navigasi ke **Mail domains** → **your-domain.com** → **Details**
3. Klik **Regenerate keys** untuk generate DKIM
4. Copy public key dan tambahkan ke DNS:

```
Type: TXT
Name: dkim._domainkey.your-domain.com
Value: v=DKIM1; k=rsa; p=<public-key-dari-admin>
TTL: 3600
```

## 🔐 Access Points

- **Admin Panel**: https://mail.your-domain.com/admin
- **Webmail**: https://mail.your-domain.com/webmail

## 📧 Mail Ports

| Port | Protocol | Deskripsi |
|------|----------|-----------|
| 25   | SMTP     | Incoming mail (port wajib) |
| 465  | SMTPS    | Secure SMTP submission |
| 587  | SMTP     | Mail submission (recommended) |
| 143  | IMAP     | Mail retrieval |
| 993  | IMAPS    | Secure IMAP |
| 110  | POP3     | Legacy mail retrieval |
| 995  | POP3S    | Secure POP3 |

## 🛠️ Management Commands

### Membuat User Baru
```bash
docker compose exec admin flask mailu user username your-domain.com 'password'
```

### Membuat User dengan Quota
```bash
docker compose exec admin flask mailu user username your-domain.com 'password' --quota 1000000000
```

### List Semua Users
```bash
docker compose exec admin flask mailu user-list
```

### Membuat Alias
```bash
docker compose exec admin flask mailu alias alias@your-domain.com destination@your-domain.com
```

### Restart Services
```bash
docker compose restart
```

### View Logs
```bash
# Semua services
docker compose logs -f

# Service tertentu
docker compose logs -f front
docker compose logs -f admin
docker compose logs -f smtp
```

### Stop Services
```bash
docker compose down
```

### Update Mailu
```bash
docker compose pull
docker compose up -d
```

## 🏗️ Architecture

```
┌─────────────────────────────────────────┐
│         Internet (Your IP)              │
└───────────────┬─────────────────────────┘
                │
                ├─── Port 80/443 (HTTP/HTTPS)
                │    │
                │    └──> Nginx (SSL Termination)
                │         │
                │         └──> Mailu Front (127.0.0.1:80)
                │              │
                │              ├──> Admin (Python/Flask)
                │              └──> Webmail (Roundcube)
                │
                ├─── Port 25/587/465 (SMTP)
                │    └──> Mailu SMTP (Postfix)
                │
                └─── Port 143/993 (IMAP)
                     └──> Mailu IMAP (Dovecot)
```

## 📁 Struktur Directory

```
/srv/mailcow/mailu/
├── data/                    # Data storage
│   ├── certs/              # SSL certificates
│   ├── dkim/               # DKIM keys
│   ├── filter/             # Rspamd data
│   ├── mail/               # Mailboxes
│   ├── mailqueue/          # Mail queue
│   └── webmail/            # Webmail data
├── docker-compose.yml      # Docker orchestration
├── .env                    # Environment configuration
├── README.md               # This file
└── REVERSE_PROXY_SETUP.md  # Reverse proxy documentation
```

## 🧪 Testing

### Test Web Access
```bash
curl -I https://mail.your-domain.com/admin
curl -I https://mail.your-domain.com/webmail
```

### Test SMTP Connection
```bash
telnet mail.your-domain.com 25
# atau
openssl s_client -connect mail.your-domain.com:587 -starttls smtp
```

### Test IMAP Connection
```bash
openssl s_client -connect mail.your-domain.com:993 -crlf
```

### Check Mail Server Score
- https://www.mail-tester.com/
- https://mxtoolbox.com/SuperTool.aspx

## 🔍 Troubleshooting

### Container tidak healthy
```bash
docker compose ps
docker compose logs -f [service-name]
```

### Webmail tidak bisa diakses
1. Cek nginx status: `systemctl status nginx`
2. Cek nginx config: `nginx -t`
3. Cek Mailu logs: `docker compose logs -f front`

### Email tidak bisa kirim/terima
1. Pastikan port 25, 587, 465 terbuka di firewall
2. Cek DNS records (MX, SPF, DKIM, DMARC)
3. Test dengan: https://mxtoolbox.com/
4. Cek logs: `docker compose logs -f smtp`

### SSL Certificate Error
```bash
# Check certificate
openssl s_client -connect mail.your-domain.com:443

# Restart Nginx
systemctl restart nginx
```

### DNS Propagation
Check DNS propagation: https://dnschecker.org/

### Port sudah digunakan
```bash
# Cek port yang digunakan
netstat -tlnp | grep -E ":(80|443|25)"

# Pastikan Mailu bind ke 127.0.0.1
grep BIND_ADDRESS .env
```

## 🔒 Security Best Practices

1. **Ganti password admin** setelah first login
2. **Enable 2FA** di admin panel (Settings → Password)
3. **Setup firewall** (UFW atau iptables)
4. **Regular updates**: `docker compose pull && docker compose up -d`
5. **Backup data** directory secara berkala
6. **Monitor logs** untuk suspicious activities
7. **Enable fail2ban** untuk protection brute force

### Setup UFW (Firewall)
```bash
apt install ufw
ufw allow 22/tcp      # SSH
ufw allow 80/tcp      # HTTP
ufw allow 443/tcp     # HTTPS
ufw allow 25/tcp      # SMTP
ufw allow 587/tcp     # Submission
ufw allow 465/tcp     # SMTPS
ufw allow 143/tcp     # IMAP
ufw allow 993/tcp     # IMAPS
ufw enable
```

## 💾 Backup & Restore

### Backup
```bash
# Stop containers
docker compose down

# Backup data directory
tar -czf mailu-backup-$(date +%Y%m%d).tar.gz data/

# Backup .env
cp .env .env.backup

# Start containers
docker compose up -d
```

### Restore
```bash
# Stop containers
docker compose down

# Restore data
tar -xzf mailu-backup-YYYYMMDD.tar.gz

# Start containers
docker compose up -d
```

## 📚 Resources

- **Mailu Documentation**: https://mailu.io/
- **Docker Documentation**: https://docs.docker.com/
- **Nginx Documentation**: https://nginx.org/en/docs/
- **Email Standards**: https://www.rfc-editor.org/

## 🤝 Contributing

1. Fork repository ini
2. Buat branch: `git checkout -b feature/AmazingFeature`
3. Commit changes: `git commit -m 'Add some AmazingFeature'`
4. Push to branch: `git push origin feature/AmazingFeature`
5. Submit Pull Request

## 📝 License

Project ini menggunakan lisensi MIT - lihat file LICENSE untuk detail.

## 👤 Author

**Riyadh**
## ⚠️ Important Notes

- **Port 25** harus terbuka untuk menerima email dari server lain
- **DNS propagation** bisa memakan waktu 24-48 jam
- **DKIM, SPF, DMARC** wajib dikonfigurasi untuk deliverability yang baik
- **Backup** data directory secara berkala
- **Monitor** logs untuk mendeteksi masalah lebih awal
- **Update** Mailu secara berkala untuk security patches

