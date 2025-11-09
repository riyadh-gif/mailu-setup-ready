# Mailu Reverse Proxy Setup

## Konfigurasi yang Sudah Dilakukan

### 1. Mailu Configuration
Mailu sudah dikonfigurasi untuk bind ke `127.0.0.1` saja, sehingga tidak konflik dengan Nginx eksternal.

**File**: `/srv/mailcow/mailu/.env`
```
BIND_ADDRESS4=127.0.0.1
TLS_FLAVOR=cert  # Nginx akan handle SSL
```

### 2. Nginx Reverse Proxy
**File**: `/etc/nginx/sites-available/mail.madaka.my.id`

Nginx listen pada **IP eksternal** (145.79.11.140) untuk:
- Port 80 → redirect ke HTTPS
- Port 443 → proxy ke Mailu (127.0.0.1:80)

### 3. Port Mapping Lengkap

#### Web Ports (Sudah dikonfigurasi):
- **80** → Nginx eksternal → Mailu internal (80)
- **443** → Nginx eksternal (SSL termination) → Mailu internal (80)

#### SMTP/IMAP Ports (Direct from Mailu Docker):
Ports ini sudah **bind langsung ke 127.0.0.1** oleh Mailu:
- **25** (SMTP) → 127.0.0.1:25
- **465** (SMTPS) → 127.0.0.1:465
- **587** (Submission) → 127.0.0.1:587
- **143** (IMAP) → 127.0.0.1:143
- **993** (IMAPS) → 127.0.0.1:993
- **110** (POP3) → 127.0.0.1:110
- **995** (POP3S) → 127.0.0.1:995

## Firewall/Port Forwarding Required

Jika server berada di belakang firewall/router, pastikan ports berikut **diforward ke 145.79.11.140**:

### Mail Ports (CRITICAL):
```
TCP 25   → SMTP (incoming mail)
TCP 465  → SMTPS (encrypted submission)
TCP 587  → Submission (secure mail sending)
TCP 143  → IMAP
TCP 993  → IMAPS (secure IMAP)
TCP 110  → POP3
TCP 995  → POP3S (secure POP3)
```

### Web Ports:
```
TCP 80   → HTTP (will redirect to HTTPS)
TCP 443  → HTTPS (webmail & admin)
```

## DNS Records Yang Perlu Dikonfigurasi

### Basic DNS:
```
Type: A
Name: mail.madaka.my.id
Value: 145.79.11.140
TTL: 3600
```

### MX Record:
```
Type: MX
Name: madaka.my.id (atau @)
Value: mail.madaka.my.id
Priority: 10
TTL: 3600
```

### SPF Record:
```
Type: TXT
Name: madaka.my.id (atau @)
Value: v=spf1 mx ~all
TTL: 3600
```

### DMARC Record:
```
Type: TXT
Name: _dmarc.madaka.my.id
Value: v=DMARC1; p=quarantine; rua=mailto:admin@madaka.my.id
TTL: 3600
```

### DKIM Record:
Dapatkan dari web admin → Mail domains → madaka.my.id → Details
```
Type: TXT
Name: dkim._domainkey.madaka.my.id
Value: v=DKIM1; k=rsa; p=<public-key-dari-admin>
TTL: 3600
```

## Testing

### Test Web Access:
```bash
curl -I https://mail.madaka.my.id/admin
curl -I https://mail.madaka.my.id/webmail
```

### Test SMTP:
```bash
telnet mail.madaka.my.id 25
```

### Test IMAP:
```bash
openssl s_client -connect mail.madaka.my.id:993 -crlf
```

## Troubleshooting

### Jika webmail tidak bisa diakses:
```bash
# Cek Nginx status
systemctl status nginx
nginx -t

# Cek Mailu status
cd /srv/mailcow/mailu
docker compose ps

# Lihat logs
docker compose logs -f front
docker compose logs -f admin
```

### Jika email tidak bisa kirim/terima:
1. Pastikan port 25, 587, 465 terbuka di firewall
2. Cek DNS records (MX, SPF, DKIM)
3. Test dengan: https://mxtoolbox.com/

### Restart Services:
```bash
# Restart Nginx
systemctl restart nginx

# Restart Mailu
cd /srv/mailcow/mailu
docker compose restart
```

## Architecture Diagram

```
Internet (145.79.11.140)
         |
         ├─ Port 80 ──────┐
         ├─ Port 443 ─────┤
         |                 └─→ Nginx (External) → 127.0.0.1:80 → Mailu Front
         |
         ├─ Port 25 ──────┐
         ├─ Port 465 ─────┤
         ├─ Port 587 ─────┼─→ Direct to 127.0.0.1 → Mailu SMTP/IMAP
         ├─ Port 143 ─────┤
         ├─ Port 993 ─────┤
         ├─ Port 110 ─────┤
         └─ Port 995 ─────┘
```

## Additional Notes

- Mailu certificate sudah di-generate otomatis menggunakan Let's Encrypt
- Certificate path: `/srv/mailcow/mailu/data/certs/letsencrypt/live/mailu/`
- Nginx menggunakan certificate yang sama dari Mailu
- TLS_FLAVOR=cert karena Nginx handle SSL termination untuk web
- SMTP ports tetap menggunakan TLS langsung dari Mailu (STARTTLS)
