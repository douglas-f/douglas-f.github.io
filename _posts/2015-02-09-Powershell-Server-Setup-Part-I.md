---
title: "Powershell Server Setup: Part I"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - Windows Server
  - Microsoft
  - Automation
---

Working on my environment and needing to build out new servers fairly quickly, and not having SCCM or another provisioning tool running at the moment I setout to leverage some powershell scripts that would take me from base install off ISO to a functioning box with a lot less effort on my part.

This is broken up into three parts, based off the three scripts I created for this process, this was developed on 2012R2 using PowerShell v3.

<!--more-->

The first script in this setup is laying the groundwork for the rest of the deployment.

- IP info is inputted and set
- Hostname is set
- Domain join occurs
- Server reboot to apply those changes.

This will be a partially interactive script, using 'Read-Host' to get info when required.

```powershell
# Prompting for the IP info
$IPAddress = Read-Host "Enter IP Address"
$Subnet = Read-Host "Enter Subnet Address"
$Gateway = Read-Host "Enter Gateway Address"
$DNS1 = Read-Host "Enter First DNS Address"
$DNS2 = Read-Host "Enter Second DNS Address"
$DNS = ($DNS1, $DNS2) # this is set like this bc the SetDNSServerSearchOrder only takes one option
```

Next step is some WMI calls to update the IP address and DNS info on the server, followed by a small wait to ensure that the network link comes up before proceeding.

```powershell
# Sets the var to the wmi object so commands can be ran against it w/o typing the whole thing each time
$wmi = get-WmiObject win32_networkadapterconfiguration -filter "ipenabled = 'true'"
# Uses the var set above with it's system calls to updated IP/Subnet/Gateway/DNS
$wmi.EnableStatic($IPAddress, $Subnet)
$wmi.SetGateways($Gateway, 1)
$wmi.SetDNSServerSearchOrder($DNS)

# Waiting for network link to be established.
Start-Sleep -s 10
```

At this point we have a server that has network connectivity and… that’s about it. However, it is a good start and you need to walk before you can run. Next up is hostname configuration and domain join.

The only thing you’ll need to change in this block is putting your appropriate domain in. And of course providing credentials of a user on the domain that has the authority to add/remove computers and change their names.

```powershell
# Adding computer to the domain & getting domain join creds
$domain = "contoso.com"
$user = Get-Credential
$username = "$domain\$user"

Add-Computer -DomainName $domain -Credential $user

# Updates computer name
$hostname = Read-Host "Enter Hostname"
Rename-Computer -NewName $hostname -DomainCredential $user
```

The next step of this process is rebooting the server for domain/name changes to go into effect. 

```powershell
# Restart notification & reboot
Write-Output "Setup complete: Server will now restart"
Start-Sleep -s 5

Restart-Computer
```

We've laid a solid groundwork here for getting this server operational Part II will continue with activation, updates, and prep for software install.

```powershell
#######################################################################################
# Douglas Francis
# New Server Setup Part 1
# Version 0.7
# Created: 9/30/2014
# Modified 01/27/2015
# New to this version-
# Modified script to change hostname after domain join as otherwise it's been failing.
#
# This will do some basic setup tasks for new servers
#
#
# Future to do
# - disable IPv6
# - store DNS entries for Different data centers and prompt user which one to use
#
#######################################################################################
# First going to config the IP info of the NIC

#Prompting for the IP info
$IPAddress = Read-Host "Enter IP Address"
$Subnet = Read-Host "Enter Subnet Address"
$Gateway = Read-Host "Enter Gateway Address"
$DNS1 = Read-Host "Enter First DNS Address"
$DNS2 = Read-Host "Enter Second DNS Address"
$DNS = ($DNS1, $DNS2) # this is set like this bc the SetDNSServerSearchOrder only takes one option

# Sets the var to the wmi object so commands can be ran against it w/o typing the whole thing each time
$wmi = get-WmiObject win32_networkadapterconfiguration -filter "ipenabled = 'true'"

# Uses the var set above with it's system calls to updated IP/Subnet/Gateway/DNS
$wmi.EnableStatic($IPAddress, $Subnet)
$wmi.SetGateways($Gateway, 1)
$wmi.SetDNSServerSearchOrder($DNS)

# Waiting for network link to be established.
Start-Sleep -s 10

# Adding computer to the domain
# Setting up creds
$domain = "contoso.com"
$user = Get-Credential
$username = "$domain\$user"

# this runs the actual adding of the server to the domain.
Add-Computer -DomainName $domain -Credential $user

# Updates computername
$hostname = Read-Host "Enter Hostname"
Rename-Computer -NewName $hostname -DomainCredential $user

# Restart notification
Write-Host "Setup complete Server will now restart"
Start-Sleep -s 5

# restarting sever with new host/domain info
Restart-Computer
```
