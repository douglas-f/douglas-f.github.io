---
title: "Powershell Server Setup: Part II"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - Windows Server
  - Microsoft
  - Automation
---

In Part I of this series, we took a new 2012 R2 server from fresh ISO to domain join, today we will build upon that by performing these tasks.

- Windows Datacenter License activation
- Install Michal Gajda's PowerShell module for Windows Update.
- Create a folder for storing our software packages
- Enable RDP
- Update Firewall rules
- Grab and install the latest updates.

We will be doing a lot more work in Part II than we did in Part I so let's dig into it.

Using a datacenter/volume licensing server license, we'll apply that first with a WMI call.

```powershell
# Activating windows
Write-Host "Activating Windows"
$computer = gc env:computername
$key = "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"
$service = get-wmiObject -query "select * from SoftwareLicensingService" -computername $computer
$service.InstallProductKey($key)
$service.RefreshLicenseStatus()
```
All good there, next making a folder to store the installation files for software that needs to be on every server.

```powershell
# Making folder and moving to it.
New-Item -ItemType directory -Path C:\PowerShell_Install_Temp
cd C:\PowerShell_Install_Temp
```

Next step, we're going to get the ZIP file of Michal Gajda's Windows update module from the TechNet gallery. Optionally, you could have this on a file share somewhere and grab it from there.

```powershell
# Pulling the zip from the net
Write-Host "Getting the PS Windows Update Module"
Invoke-WebRequest "http://gallery.technet.microsoft.com/scriptcenter/2d191bcd-3308-4edd-9de2-88dff796b0bc/file/41459/43/PSWindowsUpdate.zip" -OutFile C:\PowerShell_Install_Temp\PSWindowsUpdate.zip

# Extracting and moving the .zip contents to the PS module directory.
$shell = new-object -com shell.application
$zip = $shell.NameSpace(“C:\PowerShell_Install_Temp\PSWindowsUpdate.zip”)

foreach($item in $zip.items())
{
$shell.Namespace(“C:\Windows\System32\WindowsPowerShell\v1.0\Modules”).copyhere($item)
}
```

If you have standard firewall rules, you should apply them here. As a *terrible* example this is disabling all of the firewall rules. Please don't do this and create a real profile and apply them.

Following that quick RDP config and check for windows updates using the module we grabbed earlier.

```powershell
# Turning windows firewall off
Write-Output "Turning Windows Firewall off"
Set-NetFirewallProfile -profile domain -enabled false
Set-NetFirewallProfile -profile public -enabled false
Set-NetFirewallProfile -profile private -enabled false

# Allowing RDP connections to machines *NOTE* this only sets the setting to be true it does not grant anyone permissions to do so.
Write-Output "Allowing RDP connections"
(Get-WmiObject Win32_TerminalServiceSetting -Namespace root\cimv2\TerminalServices).SetAllowTsConnections(1,1) | Out-Null
(Get-WmiObject -Class "Win32_TSGeneralSetting" -Namespace root\cimv2\TerminalServices -Filter "TerminalName='RDP-tcp'").SetUserAuthenticationRequired(0) | Out-Null

# Installing windows updates
Write-Output "Getting updates"
Import-Module PSWindowsUpdate
Get-WUInstall
```

The windows update module will run, asking if you want to grab all updates or just one, then it will ask if you want to reboot after updates complete.

Here is the whole thing.

```powershell
#######################################################################################
# Douglas Francis
# New Server Setup Part 2
# Version 0.6
# Created: 9/30/2014
# Modified 01/27/2015
# New to this version-
# Breaking the script up in pieces by function
#
# This will do some basic setup tasks for new servers
#
#
#######################################################################################

#activating windows
Write-Host "Activating Windows"
$computer = gc env:computername
$key = "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"
$service = get-wmiObject -query "select * from SoftwareLicensingService" -computername $computer
$service.InstallProductKey($key)
$service.RefreshLicenseStatus()

# This section gets a powershell module for windows update and stores it in a temp folder for installation tasks.
# Next it will extract the file and place it in the modules folder where it can be used.

# Making folder and moving to it.
New-Item -ItemType directory -Path C:\PowerShell_Install_Temp
cd C:\PowerShell_Install_Temp

# Pulling the zip from the net
Write-Host "Getting the PS Windows Update Module"
Invoke-WebRequest "http://gallery.technet.microsoft.com/scriptcenter/2d191bcd-3308-4edd-9de2-88dff796b0bc/file/41459/43/PSWindowsUpdate.zip" -OutFile C:\PowerShell_Install_Temp\PSWindowsUpdate.zip

#Extracting and moving the .zip contents to the PS module directory.
$shell = new-object -com shell.application
$zip = $shell.NameSpace(“C:\PowerShell_Install_Temp\PSWindowsUpdate.zip”)

foreach($item in $zip.items())
{
$shell.Namespace(“C:\Windows\System32\WindowsPowerShell\v1.0\Modules”).copyhere($item)
}

# Turning windows firewall off
Write-Host "Turning Windows Firewall off"
Set-NetFirewallProfile -profile domain -enabled false
Set-NetFirewallProfile -profile public -enabled false
Set-NetFirewallProfile -profile private -enabled false

# Allowing RDP connections to machines *NOTE* this only sets the setting to be true it does not grant anyone permissions to do so.
Write-Host "Allowing RDP connections"
(Get-WmiObject Win32_TerminalServiceSetting -Namespace root\cimv2\TerminalServices).SetAllowTsConnections(1,1) | Out-Null
(Get-WmiObject -Class "Win32_TSGeneralSetting" -Namespace root\cimv2\TerminalServices -Filter "TerminalName='RDP-tcp'").SetUserAuthenticationRequired(0) | Out-Null

# Installing windows updates
Write-Host "Getting updates"
Import-Module PSWindowsUpdate
Get-WUInstall
```
