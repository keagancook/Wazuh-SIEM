# SURICATA AGENT CONFIGURATION
##### **Overview**

This guide explains how to set up **Suricata** as an IDS on  **Ubuntu 24.04.1 Server** agent. Suricata will be used to inspect the traffic and generate alerts for suspicious activity, and we'll integrate it with **Wazuh** for centralized security monitoring.

##### **1. Installing Suricata**

###### **1.1 Install Suricata on Ubuntu Agent**

First add the apt repository for suricata:
	`sudo add-apt-repository ppa:oisf/suricata-stable

Next, install Suricata with the following commands:

	sudo apt update && sudo apt install suricata -y

###### **1.2 Configure Suricata**

To configure Suricata with the network, edit the configuration file:

	sudo nano /etc/suricata/suricata.yaml

First, change the value `HOME_NET:` to the IP address of the Wazuh server, in this case it was `[192.168.0.128]`

Next, find the `Linux high speed capture support` configuration section and change the interface to `enp3s0f0` (network interface, verify by running `ifconfig`):

	`af-packet:   - interface: enp3s0f0`

Finally, set the directory of your rule files 
	`rule-files:`
		 `-"*.rules"` 
This will include all rule files in the rule path

Exit out of this file and save.

###### **1.3 Test Suricata**

Run Suricata in test mode to ensure it’s configured properly and can start capturing traffic:

	sudo suricata -T -c /etc/suricata/suricata.yaml

This will run Suricata in a test mode, checking the configuration for errors. If everything looks good, proceed to the next step.


##### **2. Configuring Suricata Rules**

###### **2.1 Download and Update Suricata Rules**

Suricata uses a set of rules to detect threats in the network traffic. These rules need to be downloaded and updated. To do this:

	sudo suricata-update

This will download the latest rule sets for Suricata.

###### **2.2 Download Custom Rules**

With Suricata, many different rule files can be use. In this case, we are using the Emerging Threats Ruleset

Download and extract the Emerging Threats Suricata ruleset:
	`cd /tmp/ && curl -LO https://rules.emergingthreats.net/open/suricata-7.0.3/emerging.rules.tar.gz'
	`sudo tar -xvzf emerging.rules.tar.gz && sudo mv rules/*.rules /etc/suricata/rules/`
	`sudo chmod 640 /etc/suricata/rules/*.rules`

Test Suricata again:
	`sudo suricata -T -c /etc/suricata/suricata.yaml`

If no errors are detected, Suricata is successfully configured and you can proceed.

Reload and start Suricata:
	`sudo systemctl daemon-reload`
	`sudo systemctl enable suricata`
	`sudo systemctl start suricata`

**ERRORS**
In this installation, errors were detected with some of the rule files. These rule files were removed and the errors were mitigated. These rules were removed:
	`dnp3-events.rules`
	`https2-events.rules`
	`modbus-events.rules`

---

##### **3. Configuring Wazuh to Integrate with Suricata**

###### **3.1 Configure Wazuh Agent to Send Suricata Agent Logs to Manager**

To integrate Suricata with Wazuh, configure Wazuh to process Suricata logs. On the Wazuh agent, add a `<localfile>` entry in the configuration file:

	sudo nano /var/ossec/etc/ossec.conf

Add the following lines at the end of the file, just before `</ossec_config>` to configure Wazuh to read the Suricata log files:

`<localfile>`   
	`<log_format>syslong</log_format>`   
	`<location>/var/log/suricata/fast.log</location>`
`</localfile>`


###### **3.2 Restart Wazuh Agent**

After configuring Wazuh, restart the Wazuh manager to apply the changes:

	sudo systemctl restart wazuh-agent

###### **3.3 Verify Suricata Alerts in Wazuh**

Now, you should be able to view Suricata alerts within Wazuh. The alerts will be available in the Wazuh interface under **Security Events**.
