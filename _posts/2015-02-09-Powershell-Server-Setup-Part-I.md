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

I love powershell. I think it is fantastic, powerful and the fact that common linux commands such as “cat”, “ls” and “cd” work nearly bring warmth to my twisted soul. So it was about the third time in a week that I had a sudden “urgent” server build to do, along with what I was already working on I vowed I would use powershell to make the task of building out new severs less annoying until we get System Center stood up at the office.

This will be a three part post, mostly because I have my setup ran in three scripts. I’ve found this makes life easier in case something breaks and hey you have to reboot for updates about 60 million times anyway.

Just a quick disclaimer, while I use this every week at work I’m not responsible for you fat fingering a command and breaking shit. the -WhatIf flag is your friend. If you do not understand a command us -WhatIf to see what will happen without actually doing it. Secondly, I have only tested this on 2012R2 servers however, I do not believe there is anything in here preventing you from running this on 2k8 servers if you need to.

<!--more-->

Alright so lets get to it. Part I of my setup script does not do a whole lot but does some very important tasks. First IP info is inputted and set on the NIC, the server is joined to the domain, the hostname is set and the server reboots. So far simple and easy lets get into it.

Now we do not have DHCP on any of our server VLANS which means once the OS is installed it is not even worth my time to look at the server until the networking dudes get me IP info. So first things first in my script is to set the IP, gateway, subnet and DNS servers.

To start out I am using “Read-Host” to get a nice prompt for the info.

```powershell
#Prompting for the IP info
$IPAddress = Read-Host "Enter IP Address"
$Subnet = Read-Host "Enter Subnet Address"
$Gateway = Read-Host "Enter Gateway Address"
$DNS1 = Read-Host "Enter First DNS Address"
$DNS2 = Read-Host "Enter Second DNS Address"
$DNS = ($DNS1, $DNS2) # this is set like this bc the SetDNSServerSearchOrder only takes one option
```

Now that our script knows what the IP info is and has it assigned to variables we can use we can use this obscure bit of WMI calls I found long ago trying to figure out how to do this.

```powershell
# Sets the var to the wmi object so commands can be ran against it w/o typing the whole thing each time
$wmi = get-WmiObject win32_networkadapterconfiguration -filter "ipenabled = 'true'"
# Uses the var set above with it's system calls to updated IP/Subnet/Gateway/DNS
$wmi.EnableStatic($IPAddress, $Subnet)
$wmi.SetGateways($Gateway, 1)
$wmi.SetDNSServerSearchOrder($DNS)
```

Finally I like to just have the script wait a second for the network link to establish before we proceed. Mostly because sometimes I forget to change the VLAN in the VM config

```powershell
 Waiting for network link to be established.
Start-Sleep -s 10
```

So at this point we have a server that has network connectivity and… that’s about it. But it is a good start. Next I wan to add the server to the domain and give it a hostname. But funny not about this before we get to it. My first version of this script changed the hostname then added to the domain. But about two weeks ago that just stopped working so switched it around to domain join then hostname change which seems to be working fine.

The only thing you’ll need to change in this block is putting your appropriate domain in. And of course providing credentials of a user on the domain that has the authority to add/remove computers and change their names.

```powershell
# Adding computer to the domain
# Setting up creds
$domain = "afni.net"
$user = Get-Credential
$username = "$domain\$user"

# this runs the actual adding of the server to the domain.
Add-Computer -DomainName $domain -Credential $user

# Updates computername
$hostname = Read-Host "Enter Hostname"
Rename-Computer -NewName $hostname -DomainCredential $user
```

Now of course you will get a warning in your powershell window telling you that changes will not work until the server is restarted. So I’ll go ahead and write-host that the server is going to be restarted, wait a few seconds to give me time to cancel the restart if I want to and then have the server reboot. Once it’s done We will have a nice happy domain joined server and then we can start doing the fun stuff.

```powershell
# Restart notification
Write-Host "Setup Compelete Server will now restart"
Start-Sleep -s 5

# restarting sever with new host/domain info
Restart-Computer
```

So far so good nothing to crazy or insane in here. The WMI settings that I use to set the IP info is the most advanced part of the script. Though to be honest when making this I did have a hell of a time getting the two DNS fields to work. But here is my full part one script in it’s entirety.

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
# - store DNS entries for Different datacenters and prompt user which one to use
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
$domain = "example.net"
$user = Get-Credential
$username = "$domain\$user"

# this runs the actual adding of the server to the domain.
Add-Computer -DomainName $domain -Credential $user

# Updates computername
$hostname = Read-Host "Enter Hostname"
Rename-Computer -NewName $hostname -DomainCredential $user

# Restart notification
Write-Host "Setup Compelete Server will now restart"
Start-Sleep -s 5

# restarting sever with new host/domain info
Restart-Computer
```
