---
title: "Powershell Server Setup: Part III"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - Windows Server
  - Microsoft
  - Automation
---

This is the final part of my server setup script. If you missed the previous posts, check out Part I and Part II. I suppose I should point out when I use all of these I’ve got an ISO with the scripts that I store on our datastores in ESX/Hyper-V and mount them, so I don’t need a working network connection to run this all.

Now the final part of my setup script doesn’t do a whole lot, mostly getting packages for AV/Logging/Backup and storing it in the folder I created earlier along with install the SNMP role for our monitoring software.  Getting the packages is rather simple.e

<!--more-->

```powershell
# Getting  AV

Write-Host "Getting AV Agent"
Get-ChildItem –path "\\FILELOCATIONHERE" | copy-item -Destination C:\PowerShell_Install_Temp

# Getting Logging
Write-Host "Getting Logging Agent"
Get-ChildItem –path "\\FILELOCATIONHERE" | copy-item -Destination C:\PowerShell_Install_Temp

# Getting Backup Client
Write-Host "Getting Backup Client"
Get-ChildItem –path "\\FILELOCATIONHERE" | copy-item -Destination C:\PowerShell_Install_Temp

&nbsp;
```

And then installing SNMP is straightforward as well.

```powershell
# Installing SNMP Feature
# Setting up creds
$domain = "EXAMPLE.NET"
$user = Get-Credential
$username = "$domain\$user"

Write-Host "Installing SNMP Feature"
Import-Module ServerManager
Get-WindowsFeature -Name SNMP* | Add-WindowsFeature -IncludeManagementTools -Credential $user | Out-Null
```

And that's it. Now just run all the installers from the share instead of navigating to different shares. Also, it helps that you delete them as they're installed, so you know what you've done as inevitably something will pull you away from this.

```powershell
# Douglas Francis
# New Server Setup Part 3
# Version 0.7
# Created: 9/30/2014
# Modified 01/26/2015
# New to this version-
# Breaking the script up in pieces by function
#
# New to this version.
# - installing snmp service
#
# This will do some basic setup tasks for new servers
#
# Future to do
# - when the script is done creat a file that provides a list of everything done
#
##############################################################################
&nbsp;

# Getting  AV
Write-Host "Getting AV Agent"
Get-ChildItem –path "\\FILELOCATIONHERE" | copy-item -Destination C:\PowerShell_Install_Temp

# Getting Logging
Write-Host "Getting Logging Agent"
Get-ChildItem –path "\\FILELOCATIONHERE" | copy-item -Destination C:\PowerShell_Install_Temp

# Getting Backup Client
Write-Host "Getting Backup Client"
Get-ChildItem –path "\\FILELOCATIONHERE" | copy-item -Destination C:\PowerShell_Install_Temp

# Installing SNMP Feature
# Setting up creds
$domain = "EXAMPLE.NET"
$user = Get-Credential
$username = "$domain\$user"

Write-Host "Installing SNMP Feature"
Import-Module ServerManager
Get-WindowsFeature -Name SNMP* | Add-WindowsFeature -IncludeManagementTools -Credential $user | Out-Null
```

I hope my little set of scripts will make your server builds quicker and less painful. As I add and edit them, I’ll put updates out.
