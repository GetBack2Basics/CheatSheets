# Mail Server Setup Guide (Postfix + Dovecot + Roundcube)

## Summary

Set up a complete self-hosted mail stack on Linux using Postfix (SMTP), Dovecot (IMAP/auth), and Roundcube (webmail), with proper DNS records (SPF, DKIM, DMARC, PTR) to avoid spam folders. This guide covers both server-side configuration and the DNS changes you must make at your registrar/hosting provider.

## Tested On

Ubuntu 24.04 LTS, x86.64, MariaDB 10.11, PHP 8.3, Postfix 3.8, Dovecot 2.3, Roundcube 1.6.9

## Requirements

- Linux server with sudo access
- A domain with DNS control (Cloudflare, BIND, etc.)
- Your hosting provider's control panel access (for PTR/reverse DNS)
- Ports 25, 587, 143, 110, 80, 443 open in firewall

---

## Part 1: Server-Side Setup

### Step 1: Install All Packages

```bash
sudo apt update
sudo apt install -y postfix postfix-mysql dovecot-core dovecot-imapd \
  dovecot-lmtpd dovecot-mysql dovecot-pop3d nginx php8.3-fpm \
  php8.3-mysql php8.3-intl php8.3-mbstring php8.3-imap php8.3-gd \
  php8.3-xml php8.3-curl php8.3-zip php8.3-ldap mariadb-server \
  opendkim opendkim-tools certbot python3-certbot-nginx
```

During Postfix installation, select "Internet Site" and enter your domain.

### Step 2: Configure MariaDB

```bash
sudo mysql_secure_installation
```

Create the Roundcube database and user:

```bash
sudo mysql -u root << 'EOF'
CREATE DATABASE roundcube CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'roundcube'@'localhost' IDENTIFIED BY 'CHOOSE_A_PASSWORD';
CREATE USER 'roundcube'@'127.0.0.1' IDENTIFIED BY 'CHOOSE_A_PASSWORD';
GRANT ALL PRIVILEGES ON roundcube.* TO 'roundcube'@'localhost';
GRANT ALL PRIVILEGES ON roundcube.* TO 'roundcube'@'127.0.0.1';
FLUSH PRIVILEGES;
EOF
```

Create the virtual mailbox tables:

```bash
sudo mysql -u root roundcube << 'EOF'
CREATE TABLE virtual_users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  domain VARCHAR(255) NOT NULL,
  active TINYINT(1) NOT NULL DEFAULT 1,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE virtual_aliases (
  id INT AUTO_INCREMENT PRIMARY KEY,
  address VARCHAR(255) NOT NULL,
  goto TEXT NOT NULL,
  domain VARCHAR(255) NOT NULL,
  active TINYINT(1) NOT NULL DEFAULT 1,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
EOF
```

Add a mail user:

```bash
# Generate SHA512-CRYPT hash
HASH=$(python3 -c "import crypt; print(crypt.crypt('USER_PASSWORD', crypt.mksalt(crypt.METHOD_SHA512)))")

sudo mysql -u root roundcube -e "INSERT INTO virtual_users (email, password, domain, active) VALUES ('user@example.com', '$HASH', 'example.com', 1);"
```

### Step 3: Configure Postfix

#### /etc/postfix/main.cf

Key settings:

```ini
myhostname = mail.example.com
mydomain = example.com
inet_interfaces = all
inet_protocols = all

# Virtual mailbox setup
virtual_mailbox_domains = example.com
virtual_mailbox_base = /var/mail/vhosts
virtual_mailbox_maps = mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
virtual_alias_maps = mysql:/etc/postfix/mysql-virtual-alias-maps.cf
virtual_minimum_uid = 100
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000

# SASL authentication via Dovecot
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth

# Restrictions
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
smtpd_recipient_restrictions = permit_mynetworks, reject_unauth_destination

# TLS
smtpd_tls_security_level = may
smtp_tls_security_level = may

# OpenDKIM milter
milter_protocol = 6
milter_default_action = accept
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```

#### /etc/postfix/mysql-virtual-mailbox-maps.cf

```ini
user = roundcube
password = CHOOSE_A_PASSWORD
hosts = 127.0.0.1
dbname = roundcube
query = SELECT CONCAT(SUBSTRING_INDEX(email,'@',-1),'/',SUBSTRING_INDEX(email,'@',1),'/') FROM virtual_users WHERE email='%s' AND active=1
```

#### /etc/postfix/mysql-virtual-alias-maps.cf

```ini
user = roundcube
password = CHOOSE_A_PASSWORD
hosts = 127.0.0.1
dbname = roundcube
query = SELECT goto FROM virtual_aliases WHERE address='%s' AND active=1
```

#### /etc/postfix/master.cf

Add the submission service (port 587):

```ini
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=may
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_sasl_type=dovecot
  -o smtpd_sasl_path=private/auth
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

### Step 4: Configure Dovecot

#### /etc/dovecot/dovecot.conf

```ini
protocols = imap pop3 lmtp
listen = *, ::
disable_plaintext_auth = no
auth_mechanisms = plain login
mail_location = maildir:/var/mail/vhosts/%d/%n
mail_privileged_group = mail
```

#### /etc/dovecot/conf.d/10-auth.conf

```ini
disable_plaintext_auth = no
auth_mechanisms = plain login
!include auth-sql.conf.ext
```

#### /etc/dovecot/conf.d/auth-sql.conf.ext

```ini
passdb {
  driver = sql
  args = /etc/dovecot/sql/dovecot-sql.conf.ext
}
userdb {
  driver = sql
  args = /etc/dovecot/sql/dovecot-sql.conf.ext
}
```

#### /etc/dovecot/sql/dovecot-sql.conf.ext

```ini
driver = mysql
connect = host=127.0.0.1 dbname=roundcube user=roundcube password=CHOOSE_A_PASSWORD
default_pass_scheme = SHA512-CRYPT
password_query = SELECT email as user, password FROM virtual_users WHERE email='%u' AND active=1
user_query = SELECT 5000 AS uid, 5000 AS gid, '/var/mail/vhosts/%d/%n' AS home FROM virtual_users WHERE email='%u' AND active=1
iterate_query = SELECT email AS username FROM virtual_users WHERE active=1
```

#### /etc/dovecot/conf.d/10-master.conf

```ini
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
  unix_listener auth-userdb {
    mode = 0600
    user = dovecot
  }
  unix_listener auth-master {
    mode = 0600
    user = dovecot
  }
}

service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

#### /etc/dovecot/conf.d/10-ssl.conf

```ini
ssl = yes
ssl_cert = </etc/ssl/certs/ssl-cert-snakeoil.pem
ssl_key = </etc/ssl/private/ssl-cert-snakeoil.key
ssl_min_protocol = TLSv1.2
```

### Step 5: Create Maildir Structure

```bash
sudo mkdir -p /var/mail/vhosts/example.com
sudo groupadd -g 5000 vmail 2>/dev/null
sudo useradd -g vmail -u 5000 vmail -d /var/mail/vhosts -s /usr/sbin/nologin 2>/dev/null
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 770 /var/mail/vhosts
```

### Step 6: Install Roundcube

```bash
cd /tmp
wget https://github.com/roundcube/roundcubemail/releases/download/1.6.9/roundcubemail-1.6.9-complete.tar.gz
tar xzf roundcubemail-1.6.9-complete.tar.gz
sudo mv roundcubemail-1.6.9 /var/www/mail
sudo chown -R www-data:www-data /var/www/mail
```

Import the initial database schema:

```bash
sudo mysql -u root roundcube < /var/www/mail/SQL/mysql.initial.sql
```

#### /var/www/mail/config/config.inc.php

```php
<?php
$config = [];

$config["db_dsnw"] = "mysql://roundcube:CHOOSE_A_PASSWORD@127.0.0.1/roundcube";

$config["imap_host"] = "localhost:143";
$config["smtp_host"] = "localhost:587";
$config["smtp_user"] = "%u";
$config["smtp_pass"] = "%p";

$config["support_url"] = "";
$config["product_name"] = "Roundcube Webmail";
$config["des_key"] = "rcmail-!24ByteDESkey*Str";

$config["plugins"] = ["archive", "zipdownload"];
$config["skin"] = "elastic";
```

IMPORTANT: The DB password must match the one you set in MariaDB step 2. The default config shows "***" — replace with the real password.

### Step 7: Configure Nginx

#### /etc/nginx/sites-enabled/mail

```nginx
server {
  listen 80;
  listen [::]:80;
  server_name mail.example.com _;
  root /var/www/mail;
  index index.php index.html;
  client_max_body_size 50M;

  location / {
    try_files $uri $uri/ /index.php?$args;
  }

  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_split_path_info ^(.+\.php)(/.*)$;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    fastcgi_pass 127.0.0.1:9000;
  }

  location ~* \.(?:css|js|map|svg|gif|png|jpg|jpeg|ico|webp|ttf|woff|woff2)$ {
    expires 1y;
    access_log off;
    add_header Cache-Control "public, immutable";
  }

  error_page 404 /index.php;
}
```

Adjust fastcgi_pass to match your PHP-FPM port (check /etc/php/8.3/fpm/pool.d/www.conf).

### Step 8: Configure OpenDKIM

Generate DKIM keys:

```bash
sudo mkdir -p /etc/opendkim/keys/example.com
sudo opendkim-genkey -s mail -d example.com -D /etc/opendkim/keys/example.com -b 2048
sudo chown -R opendkim:opendkim /etc/opendkim/keys/example.com
sudo chmod 600 /etc/opendkim/keys/example.com/mail.private
```

View the DNS record you need to add:

```bash
sudo cat /etc/opendkim/keys/example.com/mail.txt
```

Create OpenDKIM config files:

```bash
# Write via temp files (avoids sudo tee issues)
cat > /tmp/trusted.hosts << 'EOF'
127.0.0.1
localhost
mail.example.com
example.com
YOUR_SERVER_IP
EOF

cat > /tmp/signing.table << 'EOF'
*@example.com    mail._domainkey.example.com
EOF

cat > /tmp/key.table << 'EOF'
mail._domainkey.example.com    example.com:mail:/etc/opendkim/keys/example.com/mail.private
EOF

sudo cp /tmp/trusted.hosts /etc/opendkim/trusted.hosts
sudo cp /tmp/signing.table /etc/opendkim/signing.table
sudo cp /tmp/key.table /etc/opendkim/key.table
```

#### /etc/opendkim.conf

```ini
Syslog                  yes
LogWhy                  yes
Canonicalization        relaxed/simple
Mode                    sv
SubDomains              no
OversignHeaders         From

KeyTable                refile:/etc/opendkim/key.table
SigningTable             refile:/etc/opendkim/signing.table
ExternalIgnoreList      /etc/opendkim/trusted.hosts
InternalHosts           /etc/opendkim/trusted.hosts
Socket                  inet:8891@localhost
PidFile                 /run/opendkim/opendkim.pid
UserID                  opendkim
```

Create PID directory and restart:

```bash
sudo mkdir -p /run/opendkim
sudo chown opendkim:opendkim /run/opendkim
sudo systemctl restart opendkim postfix
```

### Step 9: Start and Enable All Services

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now postfix dovecot nginx php8.3-fpm mariadb opendkim
sudo systemctl restart postfix dovecot nginx php8.3-fpm opendkim
```

### Step 10: SSL with Let's Encrypt

```bash
sudo certbot --nginx -d mail.example.com
```

After getting the cert, update Dovecot SSL paths in /etc/dovecot/conf.d/10-ssl.conf:

```ini
ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.example.com/privkey.pem
```

Then:
```bash
sudo systemctl restart dovecot
```

---

## Part 2: DNS Records to Add (USER ACTION REQUIRED)

These changes must be made at your DNS provider (e.g. Cloudflare) and hosting provider control panel. The server cannot do these for you.

### 2a. DNS Records at Your Registrar/Cloudflare

Add these records to your domain's DNS zone:

| Type | Name | Value |
|------|------|-------|
| A | mail | YOUR_SERVER_IP |
| MX | @ | 10 mail.example.com |
| TXT | @ | v=spf1 mx ip4:YOUR_SERVER_IP ~all |
| TXT | mail._domainkey | (value from `sudo cat /etc/opendkim/keys/example.com/mail.txt`) |
| TXT | _dmarc | v=DMARC1; p=none; rua=mailto:postmaster@example.com |

Replace:
- `YOUR_SERVER_IP` with your server's actual public IPv4 (e.g. 185.208.207.241)
- `mail._domainkey` value with the output from the mail.txt file above

### 2b. PTR Record at Your Hosting Provider

This is NOT in Cloudflare or your DNS zone. It is set in your hosting provider's server control panel.

1. Log in to your hosting provider's control panel (e.g. Contabo, Hetzner, DigitalOcean)
2. Go to your server's settings
3. Find "Reverse DNS" or "PTR Record" (often under Network, IP Addresses, or DNS settings)
4. Set the PTR for YOUR_SERVER_IP to `mail.example.com`
5. Save

This is critical. Without a correct PTR, Gmail will almost certainly mark your mail as spam.

To verify after making the change:
```bash
dig -x YOUR_SERVER_IP +short
```
Should return: `mail.example.com.`

---

## Part 3: Verification

### Check Services

```bash
sudo systemctl is-active postfix dovecot nginx php8.3-fpm mariadb opendkim
```

All should show "active".

### Test SMTP (port 25)

```bash
echo "EHLO test" | nc -w 3 127.0.0.1 25
```
Expected: `220 mail.example.com ESMTP Postfix`

### Test SMTP AUTH (port 587)

```bash
python3 -c "
import smtplib
s = smtplib.SMTP('127.0.0.1', 587)
s.ehlo()
s.login('user@example.com', 'PASSWORD')
print('SMTP AUTH: OK')
s.quit()
"
```

### Test IMAP

```bash
python3 -c "
import imaplib
m = imaplib.IMAP4('127.0.0.1', 143)
m.login('user@example.com', 'PASSWORD')
m.select('INBOX')
print('IMAP: OK')
m.logout()
"
```

### Test Webmail

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:80/
```
Expected: 200 (or 301/302 if redirecting to HTTPS)

### Test DKIM Signing

```bash
# Send a test email, then check the delivered message for DKIM header
echo "Test body" | mail -s "DKIM test" -a "From: user@example.com" user@example.com
sleep 2
sudo grep "DKIM-Signature" /var/mail/vhosts/example.com/user/new/*
```
Expected: A DKIM-Signature header with `s=mail` and `d=example.com`

### Test DNS Records

```bash
dig +short TXT example.com                  # SPF
dig +short TXT mail._domainkey.example.com  # DKIM
dig +short TXT _dmarc.example.com           # DMARC
dig -x YOUR_SERVER_IP +short                # PTR
```

### Full End-to-End Test

Send a test email to an external address (e.g. your Gmail) and check:
1. It arrives (not in spam)
2. Gmail shows "Signed by" (DKIM pass)
3. Gmail shows SPF=pass, DKIM=pass, DMARC=pass (view original email headers)

---

## Part 4: Troubleshooting

### Roundcube shows "Internal Error"

Most likely a DB password mismatch. The default config has `***` as the password.

1. Check the actual password in `/var/www/mail/config/config.inc.php`
2. Test DB login: `mysql -u roundcube -pPASSWORD -h 127.0.0.1 roundcube -e "SELECT 1;"`
3. Check logs: `sudo tail -20 /var/www/mail/logs/errors.log`
4. If you changed the password via `ALTER USER`, restart PHP-FPM: `sudo systemctl restart php8.3-fpm`

### Mail not being delivered

1. Check queue: `postqueue -p`
2. Check logs: `sudo tail -f /var/log/mail.log`
3. Verify maildir exists: `ls -la /var/mail/vhosts/example.com/`
4. Check if OpenDKIM is rejecting: `sudo grep milter /var/log/mail.log | tail -10`

### OpenDKIM fails to start

1. Check key permissions: `sudo chmod 600 /etc/opendkim/keys/example.com/mail.private`
2. Check PID directory: `sudo chown opendkim:opendkim /run/opendkim`
3. Test key loading: `cat email.txt | opendkim -s mail -d example.com -b sv`
4. Check logs: `sudo journalctl -u opendkim -n 20`

### SASL auth fails

1. Verify Dovecot is running: `sudo systemctl status dovecot`
2. Check auth socket: `ls -la /var/spool/postfix/private/auth`
3. Test Dovecot auth directly: `doveadm auth test user@example.com PASSWORD`
4. Check Dovecot logs: `sudo journalctl -u dovecot -n 20`

### Permission denied on maildir

```bash
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 770 /var/mail/vhosts
```

### Gmail marks email as spam

Checklist:
1. PTR record matches your server hostname (`dig -x YOUR_IP +short` should return `mail.example.com`)
2. SPF record exists and includes your server IP
3. DKIM is signing (check email headers for DKIM-Signature)
4. DMARC record exists
5. Your server IP is not blacklisted: check at https://mxtoolbox.com/blacklists.aspx
6. You are not sending bulk mail from a new server (build reputation slowly)

---

## Part 5: Adding New Mail Users

```bash
# Generate password hash
HASH=$(python3 -c "import crypt; print(crypt.crypt('PASSWORD', crypt.mksalt(crypt.METHOD_SHA512)))")

# Add to database
sudo mysql -u root roundcube -e "INSERT INTO virtual_users (email, password, domain, active) VALUES ('newuser@example.com', '$HASH', 'example.com', 1);"

# Create maildir
sudo mkdir -p /var/mail/vhosts/example.com/newuser
sudo chown vmail:vmail /var/mail/vhosts/example.com/newuser
```

## Adding Email Aliases

```bash
sudo mysql -u root roundcube -e "INSERT INTO virtual_aliases (address, goto, domain, active) VALUES ('alias@example.com', 'user@example.com', 'example.com', 1);"
```

---

## Notes and Cautions

- Replace all placeholder values (example.com, user@example.com, PASSWORD, YOUR_SERVER_IP) before use.
- Exposing SMTP/IMAP publicly requires strict TLS, DNS (MX/SPF/DKIM/DMARC), and firewall hardening.
- Keep credentials out of shell history where possible.
- The `~all` in SPF is softfail (recommended initially). Change to `-all` after confirming everything works.
- DMARC `p=none` monitors only. Move to `p=quarantine` or `p=reject` after confirming delivery works.
- PTR record changes can take a few minutes to 24 hours to propagate.
- New server IPs have no reputation. Send small amounts of mail initially to build trust.
