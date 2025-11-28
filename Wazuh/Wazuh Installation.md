## Install Wazuh (Automated Script Method)

- Download and run the Wazuh installation assistant:
    
    bash
    
    `curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh sudo bash ./wazuh-install.sh -a`
    
- This script sets up the full Wazuh stack (manager, indexer, dashboard, Filebeat).
## Checklist to Verify Wazuh Automated Installation

- **Installation Completed Without Errors**  
    No errors or failures reported in the terminal during the install script (`wazuh-install.sh -a`).
    
- **Service Status Checks**  
    Verify all Wazuh services are active and running:
    
    bash
    
    `sudo systemctl status wazuh-manager sudo systemctl status wazuh-indexer sudo systemctl status wazuh-dashboard sudo systemctl status filebeat`
    
    All should show active (running) without errors.

	**https://<vps.ip>**
	