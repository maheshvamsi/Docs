# 04 – Windows Agent (Windows 10 VM)

## Goal

Install and enroll a **Windows 10 VM** as a Wazuh agent so its logs, inventory, SCA and vulnerability data appear in your Wazuh dashboard. The agent must talk to your public Wazuh manager on the OVH VPS.

---

## Step 1 – Prepare the Windows VM

1. Create a Windows 10 VM (VirtualBox / other).
2. Network mode: **Bridged Adapter** (recommended so the VM can reach `VPS_PUBLIC_IP` directly).
3. Ensure it has outbound access to:
   - `VPS_PUBLIC_IP:1514` (TCP/UDP)
   - `VPS_PUBLIC_IP:1515` (TCP)
   - `VPS_PUBLIC_IP:55000` (TCP)
   - `VPS_PUBLIC_IP:443` (for dashboard access from the VM if needed)

---

## Step 2 – Download Wazuh Windows Agent

In the Windows VM (browser):

1. Go to:

[https://packages.wazuh.com/4.x/windows/](https://packages.wazuh.com/4.x/windows/)

2. Download the MSI matching your server version, for example:

wazuh-agent-4.9.2-1.msi

(Adjust version as needed.)

Save it, e.g. `C:\Users\<You>\Downloads\wazuh-agent-4.9.2-1.msi`.

---

## Step 3 – Install Agent (Easy GUI Method)

1. Right‑click the MSI → **Run as administrator**.
2. In the setup wizard:
- **Manager address**: `VPS_PUBLIC_IP`
- **Agent name**: e.g. `win10-vm`
- Group: leave as `default` for now (you can move to `vm` group later).
3. Complete the installation.
4. Let the installer start the service if there’s an option for that.

> Note: The Windows service name is **`WazuhSvc`** (not `wazuh`).

---

## Step 4 – (Optional) Register via agent-auth

If you want to explicitly register using a pre‑created key:

### 4.1 On the manager (VPS)

sudo /var/ossec/bin/manage_agents

In the menu:

- Add new agent:
  - Name: `win10-vm`
  - IP: (press Enter / `0.0.0.0` for any)
- Extract the key for this agent and copy it.

### 4.2 On Windows

Open **PowerShell as Administrator**:

```
cd "C:\Program Files (x86)\ossec-agent"  
.\agent-auth.exe -m VPS_PUBLIC_IP -A win10-vm -k PASTED_KEY_HERE
```

If you don’t use `-k`, `agent-auth` will talk to `wazuh-authd` directly and register dynamically; both are valid.

---

## Step 5 – Start and Verify the Windows Agent Service

In an **elevated PowerShell**:

```
Get-Service WazuhSvc  
Start-Service WazuhSvc
```

Check status:

```
Get-Service WazuhSvc | Select-Object Name,Status
```

You should see `Status : Running`.

---

## Step 6 – Verify from the Manager and Dashboard

### 6.1 On the manager (VPS)

```
sudo /var/ossec/bin/agent_control -l
```

Look for something like:

```
ID: 002, Name: win10-vm, IP: any, Status: Active
```

### 6.2 In the dashboard

1. Log in to Wazuh dashboard.
2. Go to **Agents**.
3. Confirm `win10-vm` is listed and **Active**.
4. Go to **Security events** and filter by:

```
agent.name:"win10-vm"
```

You should see events appearing as the machine is used.

---

## Step 7 – Default Windows Log Collection

By default, Windows agent collects core event channels via `ossec.conf`:

- Application
- Security
- System

You can confirm or modify in:

```
C:\Program Files (x86)\ossec-agent\ossec.conf
```

## Step 8 – Enable Inventory, SCA & Vulnerability Detection (Optional but Recommended)

In `C:\Program Files (x86)\ossec-agent\ossec.conf` ensure:
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
Then restart the service:
```
Restart-Service WazuhSvc
```
This allows:

- **Inventory** and **Vulnerability detection** to work on Windows.
    
- **Configuration assessment (SCA)** scans using policies like `cis_win10_enterprise.yml` (enabled on the manager).
## Checklist

- [ ]  Windows 10 VM uses **Bridged** networking and can ping `VPS_PUBLIC_IP`.
    
- [ ]  Wazuh agent MSI installed, service name `WazuhSvc`.
    
- [ ]  Manager address set to `VPS_PUBLIC_IP` during install (or via `ossec.conf`).
    
- [ ]  `agent-auth.exe` run if using manual key enrollment.
    
- [ ]  `WazuhSvc` service is **Running**.
    
- [ ]  `agent_control -l` on VPS shows `win10-vm` as Active.
    
- [ ]  Dashboard shows `win10-vm` with events under **Security events**.



