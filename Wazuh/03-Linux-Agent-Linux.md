# 03 – Linux Agent (Ubuntu VM)

## Goal

Install and enroll an **Ubuntu VM (VirtualBox)** as a Wazuh agent to the public‑facing VPS manager so that logs, inventory, SCA, and vulnerabilities from this VM appear in the Wazuh dashboard.

---

## Step 1 – Prepare the Ubuntu VM

1. Create an Ubuntu VM in VirtualBox:
   - OS: Ubuntu 22.04/24.04.
   - Network mode: **Bridged Adapter** (recommended so VM can directly reach `VPS_PUBLIC_IP`).
2. Boot and update:

```
sudo apt update && sudo apt upgrade -y
```

> Note: Replace `VPS_PUBLIC_IP` below with your real Wazuh server IP.

---

## Step 2 – Add Wazuh APT Repository (keyring method)

Create keyring and add repo:

```
sudo mkdir -p /usr/share/keyrings

curl -s [https://packages.wazuh.com/key/GPG-KEY-WAZUH](https://packages.wazuh.com/key/GPG-KEY-WAZUH)  
| gpg --dearmor  
| sudo tee /usr/share/keyrings/wazuh-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/wazuh-keyring.gpg] [https://packages.wazuh.com/4.x/apt/](https://packages.wazuh.com/4.x/apt/) stable main"  
| sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
```

---

## Step 3 – Install Wazuh Agent

```
sudo apt install wazuh-agent -y
```

Enable and start service (we’ll restart again after config, but this confirms install):

```
sudo systemctl enable wazuh-agent  
sudo systemctl start wazuh-agent  
sudo systemctl status wazuh-agent
```

---

## Step 4 – Point Agent to the Manager

Edit `/var/ossec/etc/ossec.conf`:

```
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<server>` section (or add if missing) and set your VPS IP:

```
<server> <address>VPS_PUBLIC_IP</address> </server> ```
```

Save and exit.
## Step 5 – Register the Agent with the Manager

## 5.1 Create agent entry and get key (on VPS)

On the Wazuh manager (VPS):

text

`sudo /var/ossec/bin/manage_agents`

In the menu:

1. Choose option to **Add an agent**.
    
2. Give it a name, e.g. `ubuntu-vm`.
    
3. When asked for IP, just press **Enter** (or type `0.0.0.0`) so any source IP is allowed.
    
4. After adding, choose option to **Extract the key** for that agent ID.
    
5. Copy the long key string and keep it handy.
    

Exit `manage_agents`.

## 5.2 Enroll from the Ubuntu VM

On the Ubuntu VM:

text

`sudo /var/ossec/bin/agent-auth -m VPS_PUBLIC_IP -A ubuntu-vm -k PASTED_KEY_HERE`

- `-m`: manager IP (your VPS public IP).
    
- `-A`: agent name (must match the name you created).
    
- `-k`: the copied key.
    

---

## Step 6 – Start/Restart the Agent

On the Ubuntu VM:

```
sudo systemctl restart wazuh-agent sudo systemctl enable wazuh-agent sudo systemctl status wazuh-agent
```

---

## Step 7 – Verify from the Manager and Dashboard

## 7.1 CLI verification (VPS)

text

`sudo /var/ossec/bin/agent_control -l` 

You should see a line similar to:

text

`ID: 001, Name: ubuntu-vm, IP: any, Status: Active`

## 7.2 Dashboard verification

1. Log in to the Wazuh dashboard.
    
2. Go to **Agents**.
    
3. Confirm `ubuntu-vm` appears as **Active**.
    
4. Go to **Security events** or **Discover**, filter by `agent.name:"ubuntu-vm"` and verify events are coming in.
    

---

## Optional – Enable Inventory (syscollector), SCA, and Vulnerability Detection

On the Ubuntu VM, in `/var/ossec/etc/ossec.conf`, ensure these blocks exist:
```
<wodle name="syscollector">
  <disabled>no</disabled>
  <interval>1h</interval>
  <scan_on_start>yes</scan_on_start>
  <hardware>yes</hardware>
  <os>yes</os>
  <network>yes</network>
  <packages>yes</packages>
  <ports all="no">yes</ports>
  <processes>yes</processes>
</wodle>

<sca>
  <enabled>yes</enabled>
  <scan_on_start>yes</scan_on_start>
  <interval>12h</interval>
</sca>
```


Then:

`sudo systemctl restart wazuh-agent`

This allows:

- **Inventory** and **Vulnerability detection** to work (syscollector).
    
- **Configuration assessment (SCA)** scans to run on this VM.
    

---

## Checklist

- [ ]  Ubuntu VM uses **Bridged** networking and can ping `VPS_PUBLIC_IP`.
    
- [ ]  Wazuh APT repo added using keyring (`/usr/share/keyrings/wazuh-keyring.gpg`).
    
- [ ]  `wazuh-agent` installed and service is enabled.
    
- [ ]  `<server><address>VPS_PUBLIC_IP</address></server>` set in `ossec.conf`.
    
- [ ]  Agent registered with key via `agent-auth`.
    
- [ ]  `agent_control -l` on VPS shows `ubuntu-vm` as Active.
    
- [ ]  Dashboard shows `ubuntu-vm` agent with events flowing.