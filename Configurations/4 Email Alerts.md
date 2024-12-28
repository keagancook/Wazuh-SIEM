# EMAIL ALERTS

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
