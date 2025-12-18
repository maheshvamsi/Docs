Wazuh migration scenario 👍  
You can move to a **new server**, reinstall everything, **and your agents will reconnect automatically** **without re-adding them**.

The key is **preserving the agent identity and keys**.

---

# ✅ WHAT MAKES AGENTS “REMEMBER” THE MANAGER

Each agent is identified by:

- **Agent ID**
- **Agent key**
- **Agent name**
- **Manager IP / hostname**


These are stored on the **manager**, not on the agent alone.

If you restore these on the new server → agents reconnect automatically.

---

# 🧠 MIGRATION STRATEGY (ZERO AGENT RE-ENROLLMENT)

## 🔑 Files you MUST back up (VERY IMPORTANT)

On your **current Wazuh manager**:

`/var/ossec/etc/client.keys /var/ossec/queue/db/ /var/ossec/var/db/`

Optional but recommended:

`/var/ossec/etc/ossec.conf /var/ossec/etc/shared/`

📌 `client.keys` is the **most critical file**.

---

## 🛠 STEP-BY-STEP MIGRATION (OLD → NEW)

### 1️⃣ Stop Wazuh on old server

`systemctl stop wazuh-manager`

---

### 2️⃣ Backup agent data

`tar -czvf wazuh-agent-backup.tar.gz \ /var/ossec/etc/client.keys \ /var/ossec/queue/db \ /var/ossec/var/db \ /var/ossec/etc/shared`

Copy this archive safely.

---

### 3️⃣ Deploy new server

- Order OVH / AWS / Azure / GCP India
- Install **same Wazuh version** (IMPORTANT)

`curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh bash wazuh-install.sh -a`

---

### 4️⃣ Stop Wazuh on new server

`systemctl stop wazuh-manager`

---

### 5️⃣ Restore agent data

`tar -xzvf wazuh-agent-backup.tar.gz -C / chown -R wazuh:wazuh /var/ossec`

---

### 6️⃣ Update manager IP / hostname

Edit:

`/var/ossec/etc/ossec.conf`

Update:

`<global>`
  `<server_ip>NEWIP</server_ip>`
`</global>`


---

### 7️⃣ Start Wazuh

`systemctl start wazuh-manager`

---

### 8️⃣ Update DNS (IMPORTANT)

If your agents connect using:

- **Hostname** (recommended) → just update DNS A record
- **Hardcoded IP** → you must update agent config (scriptable)

📌 Best practice:

`wazuh.yourdomain.com → NEWIP`

---

# 🚨 WHAT IF AGENTS USE OLD IP?

If agents are hardcoded to Europe IP:

You have **2 options**:

1. Bulk update agent config via:
    
    - GPO
    - Ansible
    - SSH

2. Temporary NAT / IP forwarding (not recommended long-term)

---

# 🔐 SSL CERTIFICATES (Dashboard)

If you’re using Wazuh Dashboard:

- Reinstalling is fine
- Agents do NOT care about dashboard certs

---

# ✅ WHAT YOU WILL SEE AFTER MIGRATION

- Agents reconnect automatically
- Same agent IDs
- Same groups
- Same alerts
- No “duplicate agents”

---

## ❌ WHAT NOT TO DO

❌ Re-run `agent-auth` on agents  
❌ Delete `client.keys`  
❌ Change Wazuh version mid-migration

---

# 🧪 QUICK HEALTH CHECK

After migration:

`/var/ossec/bin/agent_control -l`

Agents should appear as:

`Active/Connected`

---

## 💡 PRO TIP (Very Important)

**Always use hostname instead of IP** for manager in agent config.  
This makes future migrations painless.