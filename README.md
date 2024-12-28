# **WAZUH SIEM CONSOLE**

### **OBJECTIVE**
The goal of this repository is to document the installation and management of a Wazuh instance. This Wazuh server will act as a SIEM console for a home network. This system will aggregate logs from multiple sources, and use Suricata to monitor network traffic and generate alerts.

The purpose of this system is for education and proof of concept, along with increasing home network security posture.

----
### **HARDWARE**
Mac Mini (mid 2011)
Intel(R) Core(TM) i5-2415M CPU @ 2.30GHz
RAM: 16gb
500GB Hard Drive
### **OPERATING SYSTEM**
Ubuntu Server 24.04.1 LTS

----
### **INSTALLATION PROCEDURES**
##### 1.0 Ubuntu Installation
###### **1.1 Creating Installation Media**
First, download a copy of Ubuntu Server 24.04.1 LTS from the official Ubuntu website. To verify the iso integrity, use a program like 7zip and generate a SHA256 checksum. Cross reference this with the value on the Ubuntu website and make sure they match. In this case and version, the hash value was:
	`e240e4b801f7bb68c20d1356b60968ad0c33a41d00d828e74ceb3364a0317be9`

Next, write the ISO to a USB drive using a program like Balena Etcher (Windows) or dd (MacOS).

Insert the installation media and select it from your boot manager to begin installation.

###### **1.2 Install and Update**
Ensure the system has an ethernet connection and begin the installation. Follow the on screen instructions and use the default settings. Make sure to check the box to install SSH to be able to access the server remotely later. 

After first boot, connect over SSH and run a full upgrade:
	`sudo apt update && sudo apt full-upgrade`

Next, verify tmux is installed to use persistent SSH sessions in the event of disconnection
	`sudo apt install tmux`
	`tmux`

##### **2.0 Expand File System**
###### **Explanation**
The Ubuntu default installation uses a Logical Volume Manager (LVM) to group separate block devices (partitions) together into Volume Groups (VGs), and then chop those VGs up into logical block devices, or Logical Volumes (LVs). LVs are the abstracted block devices upon which your usable file system resides.

With the default installation settings, Ubuntu won't take advantage of the entire disk. Bellow are the steps to correct this and maximize disk utilization.
###### **2.1 Expand the LVG**
Run the command `sudo vgdisplay` to check free space

Next, run `sudo lvdisplay` to check the LV Path and size.

Run `sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv` to extend the LV to the maximum usable size. 

Run `lvdisplay` one more time to verify the change

Now we need to extend the filesystem to take advantage of the extended volume. Run `df -h` to verify your root file system, then run `sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv` to expand the filesystem

Run `df -h` one more time to verify the filesystem was expanded.

##### 3.0 Installing Wazuh

Within a tmux session, run the Wuzuh installation script using the following command:
	curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
Installation succeeded and the web interface was confirmed to be working with https

****note, make sure to verify the URL in the above command is up to date**


### **CONIGURATIONS**
#### AGENT CONFIGURATION
Setting up an agent is very easy with Wazuh

First, open `Endpoints` on the Wazuh Dashboard. Then, select `Deploy New Agent`

Select the correct information for the particular agent you are attempting to integrate, enter the Wazuh Server IP address, and run the commands it outputs on your agent device. DONE!

*****note*
On this particular system, 3 agents were configured. The first is running on a Windows 11 personal desktop computer, next an Ubuntu File Server running Nextcloud, and finally a Raspberry Pi board running Pi-Hole DNS.

#### SURICATA AGENT CONFIGURATION
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

#### PORT MONITORING WITH SURICATA
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

#### HOW TO START SURICATA IN NFQ MODE AT BOOT TIME

Unfortunately these variables in `/etc/default` have been made obsolete in 2016. AF_PACKET is now the preselected (and hardcoded) mode in the Suricata package’s systemd unit file. The `init.d` scripts are no longer used in recent Debian versions, AFAIK. Systemd is the way to go.

It’s not a good idea to directly edit the package systemd unit file (`/lib/systemd/system/suricata.service`) though, since it will be overwritten with each package update.  
If you want, you can override the command line in the packaged systemd unit file using systemd’s drop-in feature. Just create a file `/etc/systemd/system/suricata.service.d/override.conf` with the following content:

`[Service] ExecStart= ExecStart=/usr/bin/suricata -D -q 0 -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid`

This overrides the default AF_PACKET start by first clearing the previous `ExecStart` directive and then sets a new one, which you can now tailor as you wish.  
Then run

`sudo systemctl daemon-reload`

and then

`sudo systemctl restart suricata`

which will then use the overridden command line (note the `Drop-In` field):

``$ sudo systemctl status suricata * suricata.service - Suricata IDS/IDP daemon      Loaded: loaded (/lib/systemd/system/suricata.service; enabled; preset: enabled)     Drop-In: /etc/systemd/system/suricata.service.d              `-override.conf      Active: active (running) since Mon 2024-05-13 21:39:15 CEST; 2min 27s ago        Docs: man:suricata(8)              man:suricatasc(8)              https://suricata.io/documentation/    Main PID: 1748 (Suricata-Main)       Tasks: 16 (limit: 1595)         CPU: 38.062s      CGroup: /system.slice/suricata.service              `-1748 /usr/bin/suricata -D -q 0 -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid``

This is expected to be upgrade-safe since it does not touch the original packaged unit file.
#### EMAIL ALERTS

##### **Overview**

This guide explains how to configure Wazuh to send email alerts through Gmail for any event over a specified threat level. Wazuh does not directly support SMTP servers with authentication, like Gmail. To circumvent this, we will use a server relay to send emails. In this case, we are using Postfix

##### **1. Installing and Configuring Postfix**
###### 1.1 Installing Postfix

On the Wazuh Server, run the following command to install Postfix:
	`apt-get install postfix mailutils libsasl2–2 ca-certificates libsasl2-modules`
During installation, select `no configuration`

###### 1.2 Configuring Postfix

Edit the Postfix configuration file:
	`sudo nano /etc/postfix/main.cf`

Add the following configuration:
	`relayhost = [smtp.gmail.com]:587`  
	`smtp_sasl_auth_enable = yes`  
	`smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd`  
	`smtp_sasl_security_options = noanonymous`  
	`smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt`  
	`smtp_use_tls = yes`

Run the following commands as root to configure google credentials. Replace USERNAME with your google username, and PASSWORD with your google app password:
	`echo "[smtp.gmail.com]:587 USERNAME@gmail.com:PASSWORD" > /etc/postfix/sasl_passwd`
	`postmap /etc/postfix/sasl_passwd`
	`chmod 400 /etc/postfix/sasl_passwd`  
	`chown root:root /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db`  
	`chmod 0600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db`
	`systemctl restart postfix`

Finally, test the configuration:
	` echo "Hi! We are testing Postfix!" | mail -s "Test Postfix" destinationmail@testserver1.com`

##### **2. Setting Up Wazuh Manager Configuration File**
###### 2.1 Modify the Configuration File

Open the Wazuh config file:
	`sudo nano /var/ossec/etc/ossec.conf`

Ensure the following is present (replace `sender` and `receiver` with their respective values):
	`<global>`
	`<email_notification>yes</email_notification>`
	`<smtp_server>localhost</smtp_server>`
	`<email_from>sender@gmail.com</email_from>`
	`<email_to>receiver@gmail.com</email_to>`
	`<email_maxperhour>60</email_maxperhour>`
	`</global>`
	`<alerts>`
	`<email_alert_level>12</email_alert_level>`
	`</alerts>`

This will send an email alert for any event over level 12. You can change this value to be more or less sensitive.

Now restart wazuh-manager:
	`sudo systemctl restart wazuh-manager`

##### **3. Testing Alerts**

To test alerts, you can repeatedly attempt to SSH on an agent to trip the alert system:
	`ssh nobody@localhost`


#### ACTIVE RESPONSE
##### **Overview**
Active response allows Wazuh to run commands on an agent in response to certain triggers. These triggers could be a particular rule alert, a rule group, or a rule level. Note these are cascading categories, each one becoming more broad.

In this guide, we will explain how to set up an active response to prevent SSH brute force attack. Wazuh will notice when rule 5712 (brute force detected) is triggered, and our will activate a command to tell the agent to add the IP to their block list.

##### **1.0 Configuring the Active Response**
###### 1.1 Defining the Active Response in Configuration
Now that we have defined the conditions for when an active response will be executed, we will define these conditions in our configuration file. For this active response, we will sue the script `firewall-drop`. This is a default script included with Wazuh and it tells the agent to add an IP to its drop list.

Here is the active response configuration we will use:
  `<active-response>
`    <command>firewall-drop</command> <!--script to drop ip on agent-->`
`    <location>local</location>` <!--adds firewall rule on local agent, can also be set to all to add rule on all agents -->`
`    <level>7</level> <!-- The alert level at which the response is triggered (set this according to your needs) -->`
 `   <rules_id>5712</rules_id> <!-- Specify the rule ID that triggers the response, in this case, rule 5712 is a bruteforce detection -->`
`    <timeout>3600</timeout> <!-- Timeout (in seconds) for blocking IP (e.g., 1 hour) -->`
`    <repeated_offenders>120,240,360</repeated_offenders> <!-- increases time after each repeat offense to 120, 240, then 360 minutes -->`
`</active-response>`

###### 1.2 Proof of Concept
Now that we have defined the active response config, we can attempt to trigger the response. We will do this by attempting to SSH into our agent with invalid username and password multiple times.

First we will start listening to the log file to make sure we see the response in real time.
	`tail -f /var/ossec/logs/alerts/alerts.log`
Next we will spam the SSH command to our agent with invalid credentials
	`ssh baduser@192.168.0.210`

We should see that a drop command was issued, and our attacking machine should stop connecting. We can verify this by trying to ping the agent.
	ping 192.168.0.210
If everything worked correctly, the ping will not go through.

###### 1.3 Generating an Alert When Active Response is Fired
Every agent has a log file where the active response activities are registered:
	`tail -f /var/ossec/logs/active-responses.log`
For our manager to read this file and detect the active responses, we need to edit our configuration file and point it to this log file*. First, open the config file:
	`sudo nano /var/ossec/etc/ossec.conf`
Next, add the log file entry the end of the file near the similar entries:
`<localfile>`
`  <log_format>syslog</log_format>`
`  <location>/var/ossec/logs/active-responses.log</location>`
`</localfile>`

*****note, if this log file doesn't exist already you will have to create it.

###### 1.4 White list
We can also set a list of IP addresses that should never be blocked by the active response. In global section of `ossec.conf` in the Manager, use the field `white_list`. It allows IP address or netblock.

`<ossec_config>`

`<global>`
`<jsonout_output>yes</jsonout_output>`
`<email_notification>no</email_notification>`
`<logall>yes</logall>`
`<white_list>10.0.0.6</white_list>`
`</global>`

#### UFW BLOCKLIST WITH IPSET

Adding a Blocklist to UFW

To add a blocklist to UFW (Uncomplicated Firewall) on Ubuntu, you can follow these steps:

1. **Install ipset**: UFW uses ipset to manage large blocklists. Install ipset using the following command:

```
sudo apt install ipset
```

2. **Create an ipset**: Create a new ipset named `ufw-blocklist` using the following command:

```
sudo ipset create ufw-blocklist hash:ip
```

3. **Load the blocklist**: Load the blocklist into the ipset using a script or a command. For example, you can use a script that reads a file containing IP addresses and adds them to the ipset. Here’s an example:

```
#!/bin/bash

while read -r ip; do
  sudo ipset add ufw-blocklist $ip
done < blocklist.txt
```

Replace `blocklist.txt` with the actual file containing the IP addresses you want to block.

4. **Apply the blocklist to UFW**: Once the ipset is populated, apply the blocklist to UFW using the following command:

```
sudo ufw insert 1 deny from ufw-blocklist to any
```

This will add a deny rule to UFW that blocks traffic from the IP addresses in the `ufw-blocklist` ipset.

5. **Verify the blocklist**: Verify that the blocklist is applied by checking the UFW status:

```
sudo ufw status numbered
```

This should show you the deny rule you just added.

**Tips and Variations**:

- You can specify a specific port or protocol in the deny rule, for example: `sudo ufw insert 1 deny from ufw-blocklist to any port 80 proto tcp`
- You can use a different ipset type, such as `hash:net` for CIDR ranges.
- You can use a script to automate the process of updating the blocklist and applying it to UFW.
- You can use a cron job to schedule the update and application of the blocklist at regular intervals.

**Example Blocklist Script**:

Here’s an example script that loads a blocklist from a file and applies it to UFW:

```
#!/bin/bash

# Load blocklist from file
while read -r ip; do
  sudo ipset add ufw-blocklist $ip
done < blocklist.txt

# Apply blocklist to UFW
sudo ufw insert 1 deny from ufw-blocklist to any
```

Save this script to a file (e.g., `apply_blocklist.sh`), make it executable with `chmod +x apply_blocklist.sh`, and then run it with `./apply_blocklist.sh`. You can also schedule this script to run at regular intervals using cron.

Remember to replace `blocklist.txt` with the actual file containing the IP addresses you want to block.

### **ADDITIONAL SOFTWARE**
##### MOTD

A custom ascii art welcome message was added to /etc/motd :


██╗    ██╗ █████╗ ███████╗██╗   ██╗██╗  ██╗
██║    ██║██╔══██╗╚══███╔╝██║   ██║██║  ██║
██║ █╗ ██║███████║  ███╔╝ ██║   ██║███████║
██║███╗██║██╔══██║ ███╔╝  ██║   ██║██╔══██║
╚███╔███╔╝██║  ██║███████╗╚██████╔╝██║  ██║
 ╚══╝╚══╝ ╚═╝  ╚═╝╚══════╝ ╚═════╝ ╚═╝  ╚═╝

       Security out the wazuh!!
******************************************
In addition, a script was added at /etc/update-motd.d/99-ip-address which displays the public facing IP address at every SSH
login

##### VPN


### **FIXES AND PATCHES**


