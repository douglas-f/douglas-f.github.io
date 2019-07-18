---
title: "Inactive Computer Account Cleanup"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - O365
  - Office 365
  - Active Directory
  - Windows
  - Microsoft
  - Automation
---

Previously, I talked about cleaning up inactive user accounts. This is how I am automating the same process for computer accounts. Note: in my environment, some *nix based computers did not seem to consistently update the value we are looking at, which may cause some issues. Thankfully the only *nix based computers in my environment are servers, and I am excluding this from my automation. 

My scenario is having a multitude of locations across a dozen locations where the local techs have access to add and remove computers from the domain. These computers don’t always get removed from the environment, and I needed a way to do some cleanup. I created this script to run as a scheduled task on my PoSH automation server every week.

What I’m doing is searching for all computer objects in Active Directory, excluding the OU’s that my servers reside in. Mostly because if the issue I noted with *nix-based computers and avoiding unnecessary outages is a good thing.

Once the computers that have been identified as inactive; in my case 60 days since checking in with the domain, they are disabled and moved to a term computers OU.

I am getting all computer objects, excluding those in my Server OU’s and the termed computer OU, assigning those to the $computers variable.

```powershell
Import-Module ActiveDirectory
$computers = Get-ADComputer -Filter * -Properties lastlogondate | where { $_.distinguishedName -notmatch "OU=Server_New,OU=Corporate,DC=domain,DC=net" -and $_.distinguishedName -notmatch "OU=Servers,OU=Corporate,DC=domain,DC=net" -and $_.distinguishedName -notmatch "OU=TermComputerAccounts,OU=Termed Accounts,DC=domain,DC=net" }
```

Now that I have all the computer objects, I filter those out by the ones that have not contacted the domain in over 60 days, while also excluding newly created objects made within the previous week.

```powershell
$InactiveComputers = $computers | Where { $_.LastLogonDate -le $(Get-Date).AddDays(-60) -and $_.Created -le $(Get-Date).AddDays(-7) }
```

Next, I export those results to a CSV that I keep. If I ever need to know when a specific computer was removed along with what OU it was in I have the log.

```powershell
$Path = "C:\Scripts\Script_Results\TermComputers"
$Filename = (Get-date).ToString(“MM-dd-yyyy-HH-mm-ss”) + “ - 60DayInActiveComputers.csv”
$InactiveComputers | select name, lastlogondate, distinguishedname | Export-Csv $Path\$Filename -NoTypeInformation
```

Now I will loop through all of those results, disable the AD account, and move it to the termed computers OU.

```powershell
foreach ($computer in $InactiveComputers)
{
	Get-ADComputer $computer | Disable-ADAccount -Confirm:$false 
	Get-ADComputer $computer | Move-ADObject -TargetPath "OU=TermDesktops,OU=TermComputerAccounts,OU=Termed Accounts,DC=domain,DC=net"
}
```

The rest of the script is for formatting the results into HTML and sending out an email notification.

```powershell
# Setting up email and HTML for sending the report on what was disabled. 
$html = "<style>" $html = $html + "BODY{background-color:white;}" $html = $html + "TABLE{border-width: 1px;border-style: solid;border-color: black;border-collapse: collapse;}" $html = $html + "TH{border-width: 1px;padding: 10px;border-style: solid;border-color: black;background-color:Crimson}" $html = $html + "TD{border-width: 1px;padding: 10px;border-style: solid;border-color: black;background-color:Yellow}" $html = $html + "</style>" $InactiveComputers = $InactiveComputers | select name, lastlogondate, distinguishedname | ConvertTo-Html -Head $html
 # emailing report
 $smtpServer = "smtp.domian.net" $smtpFrom = "PoSH_Reporting@domain.com" $message = New-Object System.Net.Mail.MailMessage $message.From = "PoSH_Reporting@domain.com" $message.to.Add("infosec@domain.com") $Message.To.Add("serveradmin@domain.com") $Message.to.add("DTS_Engineering@domain.com") $message.Attachments.Add("$Path\$Filename") $message.IsBodyHtml = $true $message.Subject = "60 Day InActive Computers Report" $message.Body = "Computer Objects that have not signed into the domain in over 60 days. <br/> These objects have been disabled and moved to the TermDesktops OU" + $InactiveComputers $smtp = New-Object Net.Mail.SmtpClient($smtpServer) $smtp.Send($message)
```

Here is the entire thing beginning to end. Code on GitHub.

```powershell
<#	
	.NOTES
	===========================================================================
	 Created with: 	SAPIEN Technologies, Inc., PowerShell Studio 2015 v4.2.95
	 Created on:   	10/9/2015 2:05 PM
	 Created by:   	Douglas Francis
	 Organization: 	PoshOps.io
	 Filename:     	60DayInactiveComputers
	 Version: 		1.0
	===========================================================================
	.DESCRIPTION
		Searches AD for computer objects that have not signed into the domain in over 60 days.
		Excluding the Servers & Servers_New OU
		Creates a CSV file of the servers found for history.
		Disables the computers moves them to the Termed Computers OU.
#>
# Getting all computers, excluding Servers OU and Termed Computers OU
Import-Module ActiveDirectory
$computers = Get-ADComputer -Filter * -Properties lastlogondate | where { $_.distinguishedName -notmatch "OU=Server_New,OU=Corporate,DC=contoso,DC=net" -and $_.distinguishedName -notmatch "OU=Servers,OU=Corporate,DC=contoso,DC=net" -and $_.distinguishedName -notmatch "OU=TermComputerAccounts,OU=Termed Accounts,DC=contoso,DC=net" }

# Filtering all computers down to those that have not signed in in 60 days. Excluding computers created in the past 7 days
$InactiveComputers = $computers | Where { $_.LastLogonDate -le $(Get-Date).AddDays(-60) -and $_.Created -le $(Get-Date).AddDays(-7) }

# Exporting Results to CSV
$Path = "C:\Scripts\Script_Results\TermComputers"
$Filename = (Get-date).ToString(“MM-dd-yyyy-HH-mm-ss”) + “ - 60DayInActiveComputers.csv”
$InactiveComputers | select name, lastlogondate, distinguishedname | Export-Csv $Path\$Filename -NoTypeInformation

# Disabling all accounts and moving to term computers OU
foreach ($computer in $InactiveComputers)
{
	Get-ADComputer $computer | Disable-ADAccount -Confirm:$false 
	Get-ADComputer $computer | Move-ADObject -TargetPath "OU=TermDesktops,OU=TermComputerAccounts,OU=Termed Accounts,DC=contoso,DC=net"
}

# Setting up email and HTML for sending the report on what was disabled.
$html = "<style>"
$html = $html + "BODY{background-color:white;}"
$html = $html + "TABLE{border-width: 1px;border-style: solid;border-color: black;border-collapse: collapse;}"
$html = $html + "TH{border-width: 1px;padding: 10px;border-style: solid;border-color: black;background-color:Crimson}"
$html = $html + "TD{border-width: 1px;padding: 10px;border-style: solid;border-color: black;background-color:Yellow}"
$html = $html + "</style>"

$InactiveComputers = $InactiveComputers | select name, lastlogondate, distinguishedname | ConvertTo-Html -Head $html

# emailing report 
$smtpServer = "smtp.contoso.net"
$smtpFrom = "PoSH_Reporting@contoso.com"

$message = New-Object System.Net.Mail.MailMessage
$message.From = "PoSH_Reporting@contoso.com"
$message.to.Add("infosec@contoso.com")
$Message.To.Add("serveradmin@contoso.com")
$Message.to.add("DTS_Engineering@contoso.com")
$message.Attachments.Add("$Path\$Filename")
$message.IsBodyHtml = $true
$message.Subject = "60 Day InActive Computers Report"
$message.Body = "Computer Objects that have not signed into the domain in over 60 days. <br/> These objects have been disabled and moved to the TermDesktops OU" + $InactiveComputers

$smtp = New-Object Net.Mail.SmtpClient($smtpServer)
$smtp.Send($message)
```
