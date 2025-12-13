## **CONCEPTUAL OVERVIEW: Why we're doing this**

Imagine your Wazuh server is a hotel. Different clients (Zebra, Acme, Ford) are different companies staying there. Without isolation:

- Zebra's security team sees **everyone's** data (Acme's logs, Ford's alerts).
    
- No privacy, huge security risk.
    

With our setup:

- **Agent groups** = assign agents to "Zebra floor" vs "Acme floor".
    
- **Roles with resource filters** = Zebra's user can only access "Zebra floor" agents.
    
- **Dashboard tenants** (optional) = separate visual workspaces per client.
    

Each client logs in and sees **only their own data**.

---

## **STEP 1: Create the "Zebra" agent group**

## Why?

Groups organize agents logically. All Zebra's agents go into the `zebra` group so we can apply permissions to the entire group at once (instead of per-agent).

## Via GUI:

1. Open **Wazuh Dashboard** → log in as **admin**.
    
2. Click **☰** (hamburger menu, top-left) → **Agents management** → **Endpoint Groups**.
    
3. Click **+ Create group** (top-right).
    
4. Enter name: `zebra`
    
5. Click **Create**.
    

You now have an empty `zebra` group. ✓

## Via Command (on VPS):

bash

`sudo /var/ossec/bin/agent_groups -a -g zebra -q`

---

## **STEP 2: Add a label to the Zebra group (for dashboard filtering)**

## Why?

When Wazuh generates alerts, it needs to "tag" them with the group so the dashboard can filter. This label makes the alert searchable by `agent.labels.group: zebra`.

## Via GUI:

1. **Agents management** → **Endpoint Groups** → click on **zebra** group.
    
2. Click **Files** tab.
    
3. Click **Edit group configuration**.
    
4. Paste this inside the `<agent_config>` block:
    

xml

`<labels>`
  `<label key="group">zebra</label>`
`</labels>`


5. Click **Save**.
    

The group config now tags all alerts from Zebra's agents with `group: zebra`. ✓

## Via Command:

bash

`sudo nano /var/ossec/etc/shared/zebra/agent.conf`

Add the label block above, save.

---

## **STEP 3: Create a Wazuh API Policy**

## Why?

A **policy** is a set of permissions. It says "this user can read agents in the `zebra` group, create/modify rules, restart agents, etc."

## Via GUI:

1. Click **☰** → **Indexer management** → **Security** → **Policies**.
    
2. Click **Create policy**.
    
3. Fill in:
    
    - **Name**: `zebra-admin-policy`
        
    - **Policy document**: replace with this:
        

`{`
  `"actions": [`
    `"agent:read",`
    `"agent:modify_group_assignment",`
    `"agent:restart",`
    `"rule:read",`
    `"rule:update",`
    `"decoder:read",`
    `"decoder:update",`
    `"mitre:read"`
  `],`
  `"resources": [`
    `"agent:group:zebra"`
  `],`
  `"effect": "allow"`
`}`


4. Click **Create policy**.
    

This policy says: "Allow full read/write on agents belonging to the `zebra` group only." ✓

---

## **STEP 4: Create a Wazuh API Role**

## Why?

A **role** is a container that holds policies. One role can have multiple policies. Later, we assign this role to the Zebra user.

## Via GUI:

1. Go to **Indexer management** → **Security** → **Roles**.
    
2. Click **Create role**.
    
3. Fill in:
    
    - **Name**: `zebra-admin-role`
        
    - Under **Policies**, click **Add policy** and select `zebra-admin-policy` (created above).
        
4. Click **Create role**.
    

Now the role `zebra-admin-role` contains the `zebra-admin-policy` permissions. ✓

---

## **STEP 5: Create an internal user "zebra"**

## Why?

This is the username + password that the Zebra team will use to log into the Wazuh dashboard and API.

## Via GUI:

1. Click **☰** → **Indexer management** → **Security** → **Internal users**.
    
2. Click **Create internal user**.
    
3. Fill in:
    
    - **Username**: `zebra`
        
    - **Password**: `SecurePassword123!` (give this to your client)
        
    - **Confirm password**: `SecurePassword123!`
        
4. Click **Create**.
    

The user `zebra` now exists but has no roles assigned yet. ✓

---

## **STEP 6: Map the user to the role**

## Why?

This connects the user `zebra` to the role `zebra-admin-role`, so when Zebra logs in, they get the permissions from that role.

## Via GUI:

1. Go to **Indexer management** → **Security** → **Roles**.
    
2. Click on the **zebra-admin-role** role.
    
3. Click the **Mapped users** tab.
    
4. Click **Manage mapping**.
    
5. Under **Users**, click to add and select `zebra`.
    
6. Click **Update**.
    

Now the user `zebra` has the `zebra-admin-role` role, which includes permissions to `agent:group:zebra` only. ✓

---

## **STEP 7: Test (optional but recommended)**

1. **Log out** as admin.
    
2. **Log in** as `zebra` / `SecurePassword123!`.
    
3. Go to **Agents** → you should see **only Zebra's agents** (whichever ones are assigned to the `zebra` group).
    
4. Go to **Security events** → filter shows **only events from Zebra's agents**.
    
5. Try to view an agent from a different group (e.g., if Acme has an agent) → **access denied** or not visible.
    

Perfect! Zebra is now isolated. ✓

## **SINGLE COMMAND FOR YOUR CLIENT (Zebra) TO SELF-REGISTER THEIR AGENT**

Send this to your client Zebra. They run it **once** on their machine (Linux/Mac/Windows WSL):

`sudo /var/ossec/bin/agent-auth -m YOUR_VPS_PUBLIC_IP -A zebra-client-01 -g zebra -i auto`

## Adding a new User will b as simple as it will b once this is done

Repeat steps 1–8 with:

- Group: `acme` (or `ford`)
    
- Policy: `acme-admin-policy`
    
- Role: `acme-admin-role`
    
- User: `acme` (or `ford`)
    
- Tenant: `acme-tenant`
    

And give them a similar command:

`sudo /var/ossec/bin/agent-auth -m YOUR_VPS_PUBLIC_IP -A acme-client-01 -g acme -i auto`
