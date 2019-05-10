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