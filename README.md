# **WAZUH SIEM CONSOLE**

### **OBJECTIVE**
The goal of this repository is to document the installation and management of a Wazuh instance. This Wazuh server will act as a SIEM console for a home network. This system will aggregate logs from multiple sources, and use Suricata to monitor network traffic and generate alerts.

The purpose of this system is, primarily, to increase the security posture and visibility of a public facing web server running a Nextcloud fileserver instance. In addition, this project was created as an educational project and proof of concept.

----
#### Key Features
* Log aggregation with Elasticsearch
* Custom dashboards
* Suricata integration for IDS
* Virustotal integration for active file scanning
* Custom scripts for active response and firewall automation
* Routine vulnerability assessment
* Automated Security Configuration Assessments (SCAs)
* Ability to generate reports and filter alerts for relevant information
* Slack integration for instant mobile alerts
#### **Technical Stack**
* Server: Ubuntu Linux
- SIEM Tools: ELK Stack, Splunk, Wazuh
- IDS/IPS: Suricata, Snort
- Log Management: Elasticsearch, Kibana
- Programming/Scripting: Python, Bash
#### **Results/Impact**
* Prevented over 200 SSH bruteforce attempts per week with automated firewall blocking
* Automated 100% of log aggregation tasks
* Enhanced network reliability by blocking malicious traffic, improving browsing speeds for 5 users
* Gained practical expertise in deploying and configuring Suricata IDS for real-time threat detection.
* Collected and analyzed over 2000 log entries with the ELK Stack, gaining insights into network behavior





