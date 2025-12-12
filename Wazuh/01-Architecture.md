# 01 – Architecture Overview

## 1. Purpose of this document

This file explains the overall architecture of your Wazuh deployment so that:

- Anyone can understand **what runs where** before touching configs.
- The rest of the runbooks (install, agents, RBAC, email) make sense in context.
- You can scale this setup later (more agents, multi‑tenant clients, clusters) without redesigning from scratch.

This document covers:

- Components and data flow
- Network layout and ports
- Single‑server lab vs production‑style cluster
- How multi‑tenancy (clients) and email alerting fit into the picture

---

## 2. High‑level components

At a high level, Wazuh consists of four main building blocks:

1. **Wazuh agents**  
   - Software running on endpoints (Linux, Windows, macOS, servers, VMs).  
   - Collect logs, OS events, file integrity data, inventory, and security configuration info.  
   - Send this data securely to the Wazuh manager.

2. **Wazuh manager (server)**  
   - Core analysis engine.  
   - Receives data from agents and agentless sources (e.g., syslog devices).  
   - Applies decoders and rules, correlates events, generates alerts.  
   - Manages agent enrollment, keys, groups, and centralized configuration.

3. **Wazuh indexer**  
   - Stores and indexes alerts and events (OpenSearch/Elasticsearch under the hood).  
   - Provides fast search and analytics over large volumes of security data.

4. **Wazuh dashboard**  
   - Web UI used by you and clients/SOC analysts.  
   - Visualizes alerts, agent status, vulnerabilities, SCA results, etc.  
   - Handles multi‑tenancy (tenants), RBAC, and custom dashboards.

Your deployment also includes:

5. **SMTP relay (Postfix)**  
   - Runs on the same VPS as the manager.  
   - Relays Wazuh alert emails to Gmail via `smtp.gmail.com:587`.

6. **External consumers (you / clients)**  
   - Access Wazuh dashboard over HTTPS (port 443).  
   - Receive email alerts.  
   - Optionally, per‑client users restricted to only their agent group(s) using groups, labels, RBAC, and tenants.

---

## 3. Current single‑server architecture (your lab / MVP)

### 3.1 Logical diagram

```
[Ubuntu VM agent]  
[Windows 10 agent] >----> [OVH VPS: Wazuh all-in-one] ---> [You / Clients]  
[Other future agents] (manager + indexer + dashboard)  
|  
+-- Postfix SMTP relay --> Gmail

```
### 3.2 What runs on the OVH VPS

The OVH VPS hosts:

- **wazuh-manager** service
- **wazuh-indexer** service
- **wazuh-dashboard** service
- **Postfix** (SMTP relay to Gmail)
- Any reverse proxy (if you put Nginx/Apache in front of the dashboard on 443)

This was installed via the Wazuh automated script:

```
curl -sO [https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```


The script sets up:

- Single Wazuh manager
- Single indexer node
- Single dashboard instance
- Self‑signed TLS for the dashboard (you access it on 443)
- Internal credentials printed at the end of the install

### 3.3 What runs on your agents

Examples:

- **Ubuntu VM (VirtualBox)**  
  - Runs `wazuh-agent` service.  
  - Configured with your VPS public IP as `<server><address>...</address></server>` in `ossec.conf`.  
  - Uses the modern apt repository with `signed-by` keyring.  
  - Syscollector, SCA, and vulnerability detection enabled via `ossec.conf`.

- **Windows 10 VM**  
  - `WazuhSvc` Windows service from MSI installer.  
  - Agent points to your VPS public IP.  
  - Uses `agent-auth.exe` for enrollment and collects Windows Event Logs (via `eventchannel`).

---

## 4. Network and ports

### 4.1 External / public‑facing

On the OVH VPS you expose:

- **22/tcp** – SSH (admin access).
- **443/tcp** – Wazuh dashboard HTTPS (either native or via reverse proxy).
- **1514/tcp + 1514/udp** – Agent data to manager (default).
- **1515/tcp** – Agent‑manager control connection (registration/commands).
- **55000/tcp** – `wazuh-authd` enrollment port (for agent‑auth).
- **587/tcp (outbound)** – Postfix to Gmail (`smtp.gmail.com:587`).

Security groups / firewall must allow:

- Inbound from the Internet to 443, 1514/1515/55000 (or restricted to your expected agent locations if you harden it later).
- Outbound from VPS to `smtp.gmail.com:587` for email alerts.

### 4.2 VM network modes

- VirtualBox **bridged** networking for your Ubuntu/Windows VMs, so they:
  - Get IPs on your LAN.
  - Can directly reach the VPS public IP on 1514/1515/55000 and 443.

NAT will usually work for outbound agent connections, but bridged is simpler and closer to real‑world networks.

---

## 5. Multi‑tenant / per‑client model

You’re designing this as something an MSSP / freelancer could use for multiple clients.

### 5.1 Agent groups

- You create **agent groups** such as:
  - `vm` – your lab VMs
  - `client` – future customer assets
- Agents are assigned to the appropriate group when registered or afterward.

### 5.2 Labels attached via group config

Each group adds labels to all its agents:

```
<agent_config>  
<labels>
<label key="group">vm</label>  
</labels>
</agent_config>
```

Now every event from those agents carries `agent.labels.group:"vm"` in the index.

You used this to:

- Filter dashboards / Discover.
- Build RBAC rules that restrict users to a single group.

### 5.3 Tenants and RBAC

At the dashboard/indexer level:

- **Tenants** are isolated workspaces (separate dashboards, visualizations, saved searches).
- **Roles** use:
  - **Index permissions + DLS** to restrict which documents a user can see.  
    Example: a role with DLS filter:

    ```
    {
      "bool": {
        "must": {
          "match": { "agent.labels.group": "vm" }
        }
      }
    }
    ```

    ensures the user only sees documents from agents with that label.

  - **Tenant permissions** so a user only uses a specific tenant (e.g. `local`).

At the Wazuh RBAC/API level:

- **Policies** such as:

  - Action: `agent:read`  
  - Resource: `agent:group`  
  - Resource identifier: `vm`  
  - Effect: `allow`

- **Wazuh roles** based on these policies and mapped to users.

The combination gives:

- `vm_user` → sees only group `vm` agents and only events from those agents.
- Future `clientA_user` → sees only group `clientA` etc.

This is the core of your “per‑client SIEM view on a shared backend”.

---

## 6. Email alerting architecture

Email alerting is a side‑channel off the manager:

```
Wazuh manager (wazuh-maild)  
|  
v  
Postfix (local SMTP relay on VPS)  
|  
v  
smtp.gmail.com:587 --> your Gmail inbox
```

- `wazuh-maild` consults `ossec.conf` `<global>` + `<alerts>` tags:
  - `email_notification`
  - `smtp_server` (set to `127.0.0.1`)
  - `email_from` / `email_to`
  - `email_alert_level` (e.g. ≥ 12)

- Postfix is configured with:
  - `relayhost = [smtp.gmail.com]:587`
  - SASL auth using a Gmail App Password.

Anything Wazuh decides should be emailed is handed to Postfix, which relays it to Gmail.

---

## 7. Scaling from this lab to production

Your current setup = **single all‑in‑one node**. For >1000 agents or many clients, the architecture evolves into:

```
Agents (1000+)  
|  
v  
[Load Balancer] (round-robin / health checks)  
|  
v  
Manager cluster (3–5 nodes, master + workers)  
|  
v  
Indexer cluster (3–5 OpenSearch nodes)  
|  
v  
Dashboard cluster (2–3 nodes, behind reverse proxy)  
|  
v  
SOC / clients (per-tenant RBAC)
```

Key differences vs your lab:

- Multiple manager nodes (cluster) for high availability and more agent connections.
- Multiple indexer nodes to handle more data and queries.
- Dashboard cluster so UI remains responsive for many users.
- Load balancer in front of managers and possibly dashboards.
- Same strategy of **groups + labels + tenants + RBAC** for per‑client isolation, just at larger scale.
