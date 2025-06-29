# SOC Analyst Home Lab Documentation

## Author
Keegen Love

Date Started: June 27th, 2025 (06.27.2025) @ 12:57PM EST
Last Updated: June 29th, 2025 (06.29.2025) @ 4:03PM EST

# Description:
This document details the build, configuration, and operational steps taken to deploy a Security Operations Center (SOC) home lab environment. The lab consists of a Windows-based Splunk SIEM server and an Ubuntu virtual machine acting as a log source. Included are configuration procedures, troubleshooting notes, and lessons learned to serve as a skills portfolio and replicable guide for SOC analyst job preparation.

# Table of Contents

	1. [Lab Host Setup](#lab-host-setup)
	2. [Linux Virtual Machine (Ubuntu) Setup](#linux-virtual-machine-ubuntu-setup)
	3. [Splunk Setup on Host (Windows)](#splunk-setup-on-host-windows)
	4. [Splunk Setup on VM (Ubuntu)](#splunk-setup-on-vm-ubuntu)
	5. [Troubleshooting Log Forwarding Issues](#troubleshooting-log-forwarding-issues)
	6. [Other Key Troubleshooting Steps](#other-key-troubleshooting-steps)

## Lab Host Setup

	- Install VirtualBox on your Windows host machine
	- Install the VirtualBox Extension Pack (for added functionality like USB passthrough)
	- Plan to run Splunk on the Windows host

## Linux Virtual Machine (Ubuntu) Setup

    Download Ubuntu 24.04.2 LTS from ubuntu.com

    Create a new Virtual Machine in VirtualBox:

        Name: Ubuntu24

        Type: Linux

        Version: Ubuntu (64-bit)

        RAM: minimum 4GB

        Storage: minimum 25GB dynamically allocated

    Mount the Ubuntu ISO under Controller: IDE in VirtualBox storage settings

    Boot the VM and select Install Ubuntu (not Try Ubuntu)

    Complete the installation and set a username/password

    After installation, remove the ISO from the VM’s Controller: IDE

        You can confirm this by locating the Controller: IDE section. It should not have any disk selected.

    Reboot the VM and confirm it boots to the Ubuntu login screen

    Open a terminal (Ctrl+Alt+T) and run:

		`sudo apt update && sudo apt upgrade -y`

    Optionally, install OpenSSH Server for later log shipping or remote access by running the following commands in the terminal:
	
		`sudo apt install openssh-server -y`
    	`sudo systemctl enable ssh`
    	`sudo systemctl start ssh`

## Splunk Setup on Host (Windows)

    Download Splunk Enterprise (Windows .msi) from splunk.com (free 60-day trial)

    Run the installer and accept the license terms

    Use the default installation path or choose a custom location if desired

    Set up the Splunk admin username and password (record this securely)

    Complete installation

    Launch Splunk from:

	http://localhost:8000

    and log in with the credentials you created
	
	From here, you should be able to see an empty dashboard
	
	Click Settings -> Data Inputs

	Custom TCP Setup
	
		Choose TCP (common for syslog forwarding) from the list

		Click New Local TCP in the top right (green box)

		(Pick a port — recommended 1514 — to avoid the Windows permission issues of port 514, which is the syslog TCP default)

		Click Next at the top

		Under Source Type, from the drop down, select "syslog". Leave the remaining sections as their default (Search & Reporting (search) for App Context, DNS for Host, and Default for Index)

		Click Review and then Submit at the top
	
	
	Default UDP Setup
	
		Choose UDP (default for syslog forwarding) from the list
		
		Click New Local UDP in the top right (green box)
		
		Enter 514 into the port section (this is the default port for syslog reports, and it is important to use this port, or you will need to allow this port through your firewall through a special rule)
		
		Default syslog port 514 is widely supported and usually allowed through Windows firewalls, reducing friction compared to nonstandard ports.
		
		Click Next at the top
		
		Under Source Type, from the drop down, select "syslog". Leave the remaining sections as their default
		
		Click Review and then Submit at the top
	
    

## Splunk Setup on VM (Ubuntu)

    Inside your Ubuntu VM terminal, run: `sudo apt install rsyslog -y` (Installs rsyslog in case it isn't already present)

    Then, edit the config file (opening it with the following command: `sudo nano /etc/rsyslog.conf`)

    At the VERY END of the file, emplace the following line:

	For Custom TCP Setup:
	
		*.* @@<YOUR_WINDOWS_IP>:1514
		
		(Replace 1514 with the port you chose, when relevant)
	
	
	For Default UDP Setup:
	
		*.* @<YOUR_WINDOWS_IP>:514
	
	
	Replace <YOUR_WINDOWS_IP> (including the <>) with the Splunk host's IPv4 address (found via ipconfig for Windows systems)

	You can see an example of where to place this configuration line in [screenshots/ubuntu_syslog_conf_location.png](./screenshots/ubuntu_syslog_conf_location.png).

	Edit the /etc/rsyslog.conf file to enable omfwd (the forwarding module) by adding:
    
		module(load="omfwd")
		
	
	You can see an example of where to place this configuration line in [screenshots/ubuntu_omfwd_enabled.png](./screenshots/ubuntu_omfwd_enabled.png).
	
    Save & Exit (`ctrl+O` & `enter` to save (keep the exact same file naming format), `ctrl+X` to close the file)

    Restart rsyslog to apply those changes via the following command in the terminal:

		`sudo systemctl restart rsyslog`
		
	You can test that rsyslog is working by sending a manual log:
	
		`logger "Test message to Splunk"`


# Troubleshooting Log Forwarding Issues

	Initially configured Splunk to listen on TCP port 1514 to avoid Windows firewall issues with privileged ports.

	Ubuntu VM was set to forward logs to the Windows host on port 1514 via TCP.

	No logs appeared in Splunk despite confirmed network connectivity and active rsyslog forwarding.

	Verified that Splunk’s TCP input on port 1514 was enabled and listening.

	Confirmed Ubuntu VM could reach Windows host on port 1514 using `nc -vz <Windows_IP> 1514`.

	Discovered Windows Defender Firewall was blocking inbound TCP traffic on port 1514.

	Decided to switch to the default syslog port UDP 514 to avoid firewall configuration.

	In Splunk, created a new UDP input on port 514, source type syslog.

	On Ubuntu VM, updated /etc/rsyslog.conf to forward logs via UDP on port 514:

	*.* @<Windows_IP>:514
	
	On Ubuntu VM, updated /etc/rsyslog.conf to enable omfwd under module section with the following line:
	
	module(load="omfwd")

	Restarted rsyslog service on Ubuntu.

	Verified logs successfully appeared in Splunk using search:

	source="udp:514" sourcetype="syslog"
	
	Confirmed Splunk indexed the logs from the VM, verifying end-to-end connectivity.

	Lesson learned: Windows Firewall blocks custom TCP ports by default; using standard UDP syslog port 514 can simplify home lab setups.


# Other Key Troubleshooting Steps

    Confirmed Splunk’s management port (8000) was accessible by launching Splunk in the browser after install

    Verified VirtualBox VM networking (NAT/Bridged) so the VM and host could communicate

    Removed the Ubuntu ISO from VirtualBox after installation to ensure proper boot

    Used `ping <host>` to verify network reachability between VM and host

    Used logger commands to manually test syslog messages from Ubuntu

    Checked for rsyslog configuration errors with:

		`sudo systemctl status rsyslog`

	Verified Splunk’s port listener using netstat on the Windows host:

		`netstat -an | find "514"`
	
	Reviewed Splunk Data Inputs page to confirm the correct port and sourcetype

	Avoided default port 514 TCP initially due to Windows permission restrictions, but circled back once firewall issues were understood