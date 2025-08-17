---
title: Home Lab - Initial Setup
date: 2025-08-10 20:15:23 +0100
categories: [Cybersecurity, Homelab]
tags: [cybersecurity, homelab, project]     # TAG names should always be lowercase
description: First steps to take set up a Windows Homelab
comments: false
---
# Initial Homelab Setup using Sysmon
Sysmon (System Monitor) is a Window userland service and a kernel-land device driver that monitors, records, and displays Windows event logs.
Specific Sysmon configurations here will be done through a separate XML file, which specifies the specs that Syslog will adhere to.

## Requirements
Initial Setup Requirements for a Homelab is as follows:
1. Host OS (Windows 11 in this case)
2. Type 2 Hypervisor (VirtualBox in this case)
3. Guest OS (Windows 11)

## VM Installation and Initial Configs
1. Initialise a VM with a Windows 11 ISO
2. Allow Guest Addition Extentions for convenience
3. Configure display resolutions and network settings if necessary
4. Prepare an administrator account

![VBCapture](/assets/img/posts/homelab/sysmon_install/vbsetup.png)

## Sysmon download and configuration
~~~shell
#Downloading the sysmon binary file
iwr https://download.sysinternals.com/files/Sysmon.zip -outfile sysmon.zip

#Unzipping the binary
expand-archive sysmon.zip

#Download the configuration file
iwr https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -outfile sysmonconfig-export.xml

#Install sysmon with the config file
./sysmon.exe -accepteula -i sysmonconfig-export.xml
~~~

## Installation Check
~~~shell
Get-Service sysmon
~~~
This should flag **Status : Running** if the setup process was successful.

Another check is to update the configuration file.
~~~shell
./sysmon.exe -c sysmonconfig-export.xml
~~~
On successful update, it should look like this:
![updatecheck](/assets/img/posts/homelab/sysmon_install/sysmoncheck.png)

Sysmon should now be running as a background process, logging every event in the OS.
To view the logs, follow the steps:
1. Execute the command to run Event Viewer
~~~shell
eventvwr.exe
~~~
2. Navigate to \Applications and Services Logs\Micosoft\Windows\Sysmon\Operational

You should see the following screen:
![eventviewer](/assets/img/posts/homelab/sysmon_install/eventviewer.png)

## Test Event Generation
These are some further test cases I ran to ensure a successful installation:
![Test Cases](/assets/img/posts/homelab/sysmon_install/testcases.png)

### Simple binary execution: Notepad (Event ID 1)
![notepadtest](/assets/img/posts/homelab/sysmon_install/notepadtest.png)

### Network Connection: Google.com (Event ID 3)
![connectionTest1](/assets/img/posts/homelab/sysmon_install/netconnectiontest.png)

This test case logs not only the connection case with Event ID 3, but also a series of DNS queries

![connectionTest2](/assets/img/posts/homelab/sysmon_install/netconnectiontest2.png)

### Binary Generation: Copying a file (Event ID 11)
~~~shell
copy C:\Windows\System32\notepad.exe C:\Users\Public\notepad_test.exe
~~~

This logs the Event ID of 11, visible in Event Viewer through Sysmon
![filecopytest](/assets/img/posts/homelab/sysmon_install/filecopytest.png)


# Further Steps
Event logs parsed and picked up by Sysmon can be channeled to a SIEM tool for potential malicious commands.