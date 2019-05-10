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

Last night I was feeling all productive and whatnot writing out part one. Today I’m consuming my prework coffee annoyed and amused that whoever was on call for our database team this past weekend apparently decided it was a good idea to ignore a P1 alert for a SQL service being down. Fuck em, let it cause an outage I am not a huge fan of the DB team manager or most of her staff. And their recent bitching claiming that you cannot virtulize SQL is just annoying me.

<!--more-->

Moving on, in the previous post our script was pretty short but still it was some very much needed details to be done. Now in Part II of my script I use a lot of “Write-Host” commands. Mostly because I just like to know what the script is doing throughout it’s progress and if it takes a shit on me I know where it happened.

I’m going to do a few things in this part, going to activate windows with a key I stored in the script, at least our expensive as hell license makes this easier on me. Next going to grab a fantastic PS module for windows update created my Microsoft MVP Michal Gajda, which you can check out here. Create a folder to store all of the software installers for our AV/Logging/Backup software. Allow RDP connections, turn windows firewall off (I know, I know, I know, Not my choice) and finally use that module to get round one of windows updates.

Okay so first thing’s first I’m going to activate windows, which is fairly simple and straight forward. Just drop your license key in there and you’re good to go. Or you could make a prompt where you could manually put one in but then what’s the point of scripting it. Because lets be honest, putting in license keys blows so if you’re license allows it to be used more than one go for it.

```powershell
#activating windows
Write-Host "Activating Windows"
$computer = gc env:computername
$key = "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"
$service = get-wmiObject -query "select * from SoftwareLicensingService" -computername $computer
$service.InstallProductKey($key)
$service.RefreshLicenseStatus()
```

And that’s it! On to the next phase. I create a folder in C: to store some install files and the PS update module in.

```powershell
# Making folder and moving to it.
New-Item -ItemType directory -Path C:\PowerShell_Install_Temp
cd C:\PowerShell_Install_Temp
```

Simple enough right? Right. Feel free to call the folder whatever you want or use env settings to store it on the users desktop. I just prefer C because simple quick and I’m just going to nuke it after I’m done so it doesn’t matter.

Next step I’m going to get the ZIP file of Michal Gajda’s Windows update module from the net, if you want you could just download it and stick it on a share somewhere. Not sure why I don’t but meh. Next unzip the folder and tell it to wait. As we need to install updates and reboot I want to do that part last

```powershell
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
```

See it’s not so bad. And for you that are learning powershell just think, as long as you focus on doing one thing at a time then it isn’t so bad. I am almost to the end here all that Part II still has to do is turn off the firewall, allow RDP sessions and install Windows updates. Which is done with this wonderful little bit of powershell goodness.

```powershell
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
Now one quick note about the windows update module. once it runs it will sit there and look for updates, once it gets updates it asks do you want to approve this one or all. So of course I hit the key for all. Then it will download install updates just like normal and when it is done the powershell window will give you the windows must reboot do you want to reboot now y/n crap and of course I hit yes, the server reboots and off to part III I go. I will say that having a WSUS server with at least one patching farm that has all patches applied that you add the server into while you’re building it will save you hours. At some point I’ll probably add a bit in here to move the server into that OU but considering I just got that patching farm setup last week baby steps.

And once again here is the whole thing.

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
#
#
#
# This will do some basic setup tasks for new servers
#
#
# Future to do
# - give user option to select which License key they want to use
# - when the script is done create a file that provides a list of everything done
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

If you missed Part I it’s here and the next and final Part III is here.