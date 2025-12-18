Architecture: How the integration works
`Agent → Wazuh Manager → VirusTotal API → Enriched Alert → Dashboard`
- Agent detects file event
- Manager extracts file hash
- Manager queries VirusTotal API
- VT response is attached to alert
- Alert severity & rule ID updated
## Prerequisites

### ✔️ VirusTotal API Key

- Create account: [https://www.virustotal.com](https://www.virustotal.com)
- Get **API key**
- Free key is fine (rate-limited)
- Premium = faster + more queries

### ✔️ Wazuh version

- Works on **Wazuh 4.x (you are on 4.14.1)**
## Prerequisites

### ✔️ VirusTotal API Key

- Create account: [https://www.virustotal.com](https://www.virustotal.com)
- Get **API key**
- Free key is fine (rate-limited)
- Premium = faster + more queries

### ✔️ Wazuh version

- Works on **Wazuh 4.x (you are on 4.14.1)**
	`sudo nano /var/ossec/etc/ossec.conf`
Add **inside `<ossec_config>`**:
`<integration>`
  `<name>virustotal</name>`
  `<api_key>YOUR_VIRUSTOTAL_API_KEY</api_key>`
  `<group>syscheck</group>`
  `<alert_format>json</alert_format>`
`</integration>`
💡 `group=syscheck` means:

- File integrity events trigger VT checks    

---

### 🔹 Step 2: Enable Syscheck (if not already)

Still in `ossec.conf`:

`<syscheck>`
  `<disabled>no</disabled>`
  `<frequency>43200</frequency>`
  `<scan_on_start>yes</scan_on_start>`

  `<directories check_all="yes">/bin,/sbin,/usr/bin,/usr/sbin</directories>`
`</syscheck>`


This allows Wazuh to:

- Monitor binaries
- Extract hashes
- Send to VirusTotal

---

### 🔹 Step 3: Restart Wazuh Manager

`sudo systemctl restart wazuh-manager`

Check status:

`sudo systemctl status wazuh-manager`

---

## 6️⃣ How to verify it’s working

### 🔍 Logs

`sudo tail -f /var/ossec/logs/ossec.log`

You should see:

`INFO: VirusTotal integration enabled INFO: Querying VirusTotal for hash...`

