# Mail Server Setup (Postfix + Dovecot + Roundcube)

## Summary

Set up a basic self-hosted mail stack on Linux using Postfix (SMTP), Dovecot (IMAP/auth), and Roundcube (webmail), then verify end-to-end delivery and authentication.

## Discussion

This guide focuses on a practical local/server deployment path and fast troubleshooting for common failures (database auth mismatch, SASL issues, permissions, and delivery problems).

## Requirements

- Linux server with sudo access
- Postfix, Dovecot, and Roundcube installed
- MySQL/MariaDB database available for Roundcube and virtual mailbox/auth data
- A mail domain configured (example uses `example.com`)
- Firewall and DNS configured for your deployment target

## Workflow

### Step 1: Verify core services are running

```bash
sudo systemctl status postfix
sudo systemctl status dovecot
sudo systemctl status apache2 || sudo systemctl status nginx
```

### Step 2: Verify SMTP authentication path

Confirm Dovecot auth socket exists for Postfix SASL:

```bash
ls -la /var/spool/postfix/private/auth
```

### Step 3: Validate Roundcube database connectivity

Confirm the credentials in Roundcube config match database credentials:

- Roundcube config path (common): `/var/www/mail/config/config.inc.php`

Test DB login:

```bash
mysql -u roundcube -pPASSWORD -h 127.0.0.1 roundcube -e "SELECT 1;"
```

### Step 4: Test IMAP login

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

### Step 5: Test webmail endpoint

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:80/
```

Expected: HTTP 200/301/302 depending on your web server and redirect setup.

### Step 6: Verify mail queue and logs

```bash
postqueue -p
sudo tail -f /var/log/mail.log
```

### Step 7: Verify mailbox storage permissions

```bash
ls -la /var/mail/vhosts/example.com/
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 770 /var/mail/vhosts
```

### Step 8: Add a new mail user (virtual mailbox workflow)

Generate a password hash:

```bash
HASH=$(python3 -c "import crypt; print(crypt.crypt('PASSWORD', crypt.mksalt(crypt.METHOD_SHA512)))")
echo "$HASH"
```

Then insert the user into your mailbox/auth tables per your schema.

## Verification

- IMAP login succeeds and prints `IMAP: OK`.
- Webmail endpoint responds with valid HTTP status.
- Test messages appear in queue/log flow and are delivered.
- SASL auth succeeds for SMTP submissions.
- Maildir ownership and permissions allow mail read/write operations.

## Troubleshooting

### Roundcube shows "Internal Error"

Most likely a DB credential mismatch.

1. Confirm `/var/www/mail/config/config.inc.php` DB credentials.
2. Re-test DB login with the same user/pass/host.
3. Check Roundcube logs (common path): `/var/www/mail/logs/errors.log`.

### Mail not being delivered

1. Inspect queue: `postqueue -p`
2. Follow logs: `sudo tail -f /var/log/mail.log`
3. Confirm recipient maildir exists under `/var/mail/vhosts/<domain>/`.

### SASL auth fails

1. Confirm Dovecot is running.
2. Confirm `/var/spool/postfix/private/auth` exists and permissions are correct.
3. Test credentials directly:

```bash
doveadm auth test user@example.com PASSWORD
```

### Permission denied on maildir

Reapply ownership and mode:

```bash
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 770 /var/mail/vhosts
```

## Notes and Cautions

- Replace placeholder values (`example.com`, `user@example.com`, `PASSWORD`) before use.
- Exposing SMTP/IMAP publicly requires strict TLS, DNS (MX/SPF/DKIM/DMARC), and firewall hardening.
- Keep credentials out of shell history where possible.
