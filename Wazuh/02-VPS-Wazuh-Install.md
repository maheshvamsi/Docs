# 02 – VPS Wazuh Install (OVH)

## Goal

Set up an OVH VPS running a public‑facing Wazuh all‑in‑one server (manager + indexer + dashboard) that agents and users can reach over the internet.

---

## Step 1 – Provision OVH VPS

1. In OVH control panel, create a VPS:
   - OS: **Ubuntu 22.04 LTS** (or newer)
   - Size: **≥ 2 vCPU / 4 GB RAM**, 40–80 GB disk
2. Ensure it has a **public IPv4**.
3. Add your SSH key or note the root password.

Connect:

ssh root@VPS_PUBLIC_IP
# or:

# ssh ubuntu@VPS_PUBLIC_IP

# sudo -i

	Update system: apt update && apt upgrade -y

---

## Step 2 – Install Wazuh All‑in‑One

Download installer:

```
cd /root  
curl -sO [https://packages.wazuh.com/4.14/wazuh-install.sh](https://packages.wazuh.com/4.14/wazuh-install.sh)  
chmod +x wazuh-install.sh

```

Run all‑in‑one install:

sudo ./wazuh-install.sh -a

> Note: At the end, record the **dashboard URL, username, and password** printed by the script.

---

## Step 3 – Verify Services

```
sudo systemctl status wazuh-manager  
sudo systemctl status wazuh-indexer  
sudo systemctl status wazuh-dashboard
```

All three should be `active (running)`.

---

## Step 4 – Open Firewall Ports (ufw example)

```
sudo ufw allow 22/tcp # SSH  
sudo ufw allow 443/tcp # HTTPS dashboard  
sudo ufw allow 1514/tcp # agents → manager  
sudo ufw allow 1514/udp  
sudo ufw allow 1515/tcp # manager control  
sudo ufw allow 55000/tcp # agent enrollment  
sudo ufw enable
```

---

## Step 5 – Access Wazuh Dashboard

In a browser:

- URL: `https://VPS_PUBLIC_IP/`  (or `https://VPS_PUBLIC_IP:443`)
- Login with the credentials provided by the installer.
- Optionally change the admin password immediately.

---

## Checklist

- [ ] SSH access to `root@VPS_PUBLIC_IP` works  
- [ ] `wazuh-manager` is running  
- [ ] `wazuh-indexer` is running  
- [ ] `wazuh-dashboard` is running  
- [ ] Firewall allows 22, 443, 1514, 1515, 55000  
- [ ] Dashboard reachable via HTTPS and admin login works