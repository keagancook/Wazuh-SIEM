# PORT MONITORING WITH SURICATA
##### **Overview**
This guide explains how to configure port mirroring through the Wazuh Server, and how to configure Suricata to capture the mirrored traffic. This will allow us to capture network packets for all devices on our switch. For this configuration, we will need 2 NIC adapters on our Wazuh Server, one for the management connection, and one for the mirrored connection.

To set this up, we will use the following outline: First, we will have to configure a SPAN port on our switch to mirror the traffic of all other ports, then we will configure the appropriate NIC to accept this traffic, finally we will set up Suricata to interpret this data and send alerts to Wazuh.

##### **1.0 Configuring the SPAN Port on Network Switch**
###### 1.1 Connecting the Switch
For this project, we are using a TL-SG108E network switch. This switch as 8 ports.

First, we need to make the proper connections on the switch ports. Port 1 was used for our LAN connection, Ports 2-6 were connected to our network devices, port 7 was connected to the motherboard ethernet connection on our Wazuh server to be used for the management connection, and port 8 was connected to our USB ethernet NIC to be used as our SPAN port

###### 1.2 Configuring the Switch
Next, we will log into the management console of our switch with the assigned IP address. This can be found by checking the DHCP settings on the router.

Under Monitoring > Port Mirror, we selected port 8 to be our monitoring port. Next, we selected ports 2-6 and enabled both ingress and egress mirroring. Port 7 was not enabled because this is our management port and it would be redundant to monitor this port.

##### **2.0 Configuring Monitoring Interface**
###### 2.1 Netplan configuration
First, run `ip link show` to identify our monitoring interface. In this case, it was `enx00249b8a3345`.

Next, create a new network configuration file for our interface:
	`sudo nano /etc/netplan/01-netcfg.yaml`

Enter the following configuration (make sure to replace `enx00249b8a3345` with the correct interface name):

`network:`
`  version: 2`
`  ethernets:`
`    enx00249b8a3345:`
`      dhcp4: no`
`      dhcp6: no`
`      addresses: []`
 `     optional: true`

Apply the proper permissions to the file:
	`sudo chown root:root /etc/netplan/01-netcfg.yaml && sudo chmod 600 /etc/netplan/01-netcfg.yaml`

Now apply the configuration:
	`sudo netplan apply`

Finally, enable promiscuous mode on the interface:
	`sudo ip link set enx00249b8a3345 promisc on`

##### **3.0 Install and Configure Suricata**
###### 3.1 Install Suricata
	sudo apt update
	sudo apt install suricata -y

###### 3.2 Edit the Suricata configuration file:
	sudo nano /etc/suricata/suricata.yaml

Find the interface section and set it to the monitoring NIC `(enx00249b8a3345)`:
	`af-packet:`
  - `interface: enx00249b8a3345`
    `cluster-id: 99`
    `cluster-type: cluster_flow`
    `defrag: yes`

At the very end of the config file under the default-rule-path section, make sure the following is present to accept all rules in the "rules" folder:
	`default-rule-path: /var/lib/suricata/rules`
`rule-files:`
  `- "*.rules"`

###### 3.3 Download and Configure Suricata Rules

Download default Suricata rules:
	`sudo suricata-update`
Verify the rule file location after the update:
	`ls /var/lib/suricata/rules/suricata.rules`
	If file exits, you're good to go

Download and extract the Emerging Threats Suricata ruleset:
	`cd /tmp/ && curl -LO https://rules.emergingthreats.net/open/suricata/emerging.rules.tar.gz`
	`sudo tar -xvzf emerging.rules.tar.gz && sudo mv rules/*.rules /etc/suricata/rules/`
	`sudo chmod 640 /etc/suricata/rules/*.rules`

Test Suricata :
	`sudo suricata -T -c /etc/suricata/suricata.yaml`

If no errors are detected, Suricata is successfully configured and you can proceed.

###### 3.4 Reload and Start Suricata
	`sudo systemctl daemon-reload`
	`sudo systemctl enable suricata`
	`sudo systemctl start suricata`

##### **4.0 Integrate Suricata with Wazuh**
###### 4.1 Configure Wazuh Server to monitor Suricata logs
To integrate Suricata with Wazuh, configure Wazuh to process Suricata logs. On the Wazuh server, add a `<localfile>` entry in the configuration file:

	sudo nano /var/ossec/etc/ossec.conf

Add the following lines at the end of the file, just before `</ossec_config>` to configure Wazuh to read the Suricata log files:

`<localfile>`   
	`<log_format>syslong</log_format>`   
	`<location>/var/log/suricata/fast.log</location>`
`</localfile>`

Restart the Wazuh manager:
	`sudo systemctl restart wazuh-manager`

###### 4.2 Verify Traffic Monitoring
Use the Suricata logs to confirm that traffic is being captured and analyzed:
	`tail -f /var/log/suricata/fast.log`
Check that Wazuh is receiving the logs:
	`sudo cat /var/ossec/logs/alerts/alerts.json | jq`

You should now see the Suricata events on your Wazuh dashboard.
