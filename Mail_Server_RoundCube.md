  GNU nano 7.2                                    mailserver-setup.md                                             
s.quit()
"

# Test IMAP
python3 -c "
import imaplib
m = imaplib.IMAP4('127.0.0.1', 143)
m.login('user@example.com', 'PASSWORD')
m.select('INBOX')
print('IMAP: OK')
m.logout()
"

# Test webmail
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:80/
```

## Troubleshooting

### Roundcube shows "Internal Error"
Most likely a DB password mismatch. Check:
1. Password in /var/www/mail/config/config.inc.php matches the MySQL user
2. Test: mysql -u roundcube -pPASSWORD -h 127.0.0.1 roundcube -e "SELECT 1;"
3. Check /var/www/mail/logs/errors.log

### Mail not being delivered
1. Check mail queue: postqueue -p
2. Check logs: sudo tail -f /var/log/mail.log
3. Verify maildir exists: ls -la /var/mail/vhosts/example.com/

### SASL auth fails
1. Verify Dovecot is running: sudo systemctl status dovecot
2. Check auth socket exists: ls -la /var/spool/postfix/private/auth
3. Test Dovecot auth: doveadm auth test user@example.com PASSWORD

### Permission denied on maildir
```bash
sudo chown -R vmail:vmail /var/mail/vhosts
sudo chmod -R 770 /var/mail/vhosts
```

## Adding New Mail Users

```bash
# Generate password hash
HASH=$(python3 -c "import crypt; print(crypt.crypt('PASSWORD', crypt.mksalt(crypt.METHOD_SHA512)))")

# Add to database

^G Help         ^O Write Out    ^W Where Is     ^K Cut          ^T Execute      ^C Location     M-U Undo
^X Exit         ^R Read File    ^\ Replace      ^U Paste        ^J Justify      ^/ Go To Line   M-E Redo
