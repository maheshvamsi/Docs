Assume:

- You’ll use **Gmail SMTP directly** (simpler than adding Mailjet/SMTP2GO right now).
    
- Sender and recipient are the same Gmail, e.g. `yourname@gmail.com`.
    

---

## 1. Prepare the Gmail account

1. Turn on **2‑Step Verification** for that Gmail.
    
2. In Google Account → **Security → App passwords**:
    
    - App: Mail
        
    - Device: Other → type `wazuh-vps`
        
    - Google gives a 16‑character **app password**. Keep it safe.  
        This is the password you will use for SMTP; your normal Gmail password will not work.​
        

---

## 2. Install a simple SMTP relay (Postfix) on the VPS

On your Wazuh VPS (Ubuntu/Debian):

bash

`sudo apt update sudo apt install postfix libsasl2-modules -y`

- When asked for configuration type, choose **Internet Site**.
    
- “System mail name” can be your VPS hostname (e.g. `vps.example.com`).
    

Now configure Postfix to relay via Gmail:

bash

`sudo postconf -e 'relayhost = [smtp.gmail.com]:587' sudo postconf -e 'smtp_use_tls = yes' sudo postconf -e 'smtp_sasl_auth_enable = yes' sudo postconf -e 'smtp_sasl_security_options = noanonymous' sudo postconf -e 'smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd' sudo postconf -e 'smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt'`

Create the credentials file:

bash

`echo "[smtp.gmail.com]:587 yourname@gmail.com:APP_PASSWORD_HERE" | sudo tee /etc/postfix/sasl_passwd sudo postmap /etc/postfix/sasl_passwd sudo chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db sudo systemctl restart postfix`

Quick test:

bash

`echo "test from wazuh smtp" | mail -s "smtp test" yourname@gmail.com`

If that arrives, SMTP is ready.​

---

## 3. Point Wazuh to use this SMTP

Edit `/var/ossec/etc/ossec.conf` on the manager:

bash

`sudo nano /var/ossec/etc/ossec.conf`

Inside `<ossec_config>...</ossec_config>`, ensure you have:

xml

`<global>   <email_notification>yes</email_notification>  <smtp_server>127.0.0.1</smtp_server>  <email_from>yourname@gmail.com</email_from>  <email_to>yourname@gmail.com</email_to> </global> <alerts>   <!-- send only medium+ severity to avoid spam at first -->  <email_alert_level>12</email_alert_level> </alerts>`

Save and restart Wazuh:

bash

`sudo systemctl restart wazuh-manager`

This tells Wazuh’s `wazuh-maild` daemon to hand alerts to Postfix on localhost, which then relays them to Gmail.

---

## 4. Verify Wazuh alert emails

1. Trigger something noisy (e.g. multiple failed SSH logins to a monitored host).
    
2. Check `/var/ossec/logs/ossec.log` for lines mentioning `wazuh-maild` and no errors.​
    
3. Check Gmail inbox; you should see alert emails from `yourname@gmail.com`.
    

If you hit any specific error messages (Postfix or `ossec.log`), paste them and a focused fix can be given.