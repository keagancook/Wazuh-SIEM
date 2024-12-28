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

###### **2.0 Expand File System**
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

----
