---
title: "Automated User Account Removal and O365 Cleanup"
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

In my previous post here, I talked about finding, disabling and placing inactive users into a separate OU. In this post I will use powershell to remove users from that OU along with any child objects they have, such as mobile devices, along with removing their license for O365.

This script runs as a scheduled task on my Powershell automation server every Monday. It searches though all users in the Term30 OU which is where our ID Team places terminated users and it also where my inactive user script places users. Any user that has not been modified in over 30 days gets picked up by this script. I went with modified date instead of last login date because I would rather be on the side of caution. This typically means user accounts sit here for 30-60 days before they are removed. The script also creates a CSV report and creates a ticket for removal of their home directory if they have one. Sadly there are years of bad habits and people putting in mappings to other shares in users AD profiles so I cannot have the script nuke them automatically.

I start off by using the transcription feature of powershell to log what the script does with a file name based on date/time the script ran. This way I can go back and view logs either for error reporting or knowing when an account is deleted in case someone tosses a fit they no longer have access to a terminated employees mailbox, which has happened. I highly recommend any automated scripts that you create have some kind of logging. They do not take up much space and are better to have than not.

```powershell
# Powershell transcription setup
# Create a filename based on a time stamp.
$Filename = ((Get-date).Month).ToString() + "-" +`
((Get-date).Day).ToString() + "-" +`
((Get-date).Year).ToString() + "-" +`
((Get-date).Hour).ToString() + "-" +`
((Get-date).Minute).ToString() + "-" +`
((Get-date).Second).ToString() + "-PurgeTerm30" + ".txt"
# Set the storage path.
$Path = "C:\Scripts\PS Transcripts"
# Turn on PowerShell transcripting. 
Start-Transcript -Path "$Path\$Filename"
```

UPDATE: as pointed out in the comments there is a much better way to create a file name for the transcript.

```powershell
$Filename = (Get-date).ToString(“MM-dd-yyyy-HH-mm-ss”) + “-PurgeTerm30.txt”
```

Now I connect to O365 with a service account created specifically for this purpose. Remember to set the account password to not expire or otherwise you will have to update the script. Also side note, this runs as a packaged .exe created with Powershell Studio so I’m not overly concerned with having credentials stored in here.

```powershell
# setting up modules and O365 connections

Function New-Connection
{
	#Credential setup for O365
	$O365User = "deleting@domain4.onmicrosoft.com"
	$O365Pass = "p@ssw0rd" | ConvertTo-SecureString -AsPlainText -Force
	$O365Creds = New-Object System.Management.Automation.PSCredential ($O365User, $O365Pass)
	
	# O365 Function
	Function Connect-O365
	{
		$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri https://ps.outlook.com/powershell/ -Credential $O365Creds -Authentication Basic -AllowRedirection
		Import-PSSession $Session -AllowClobber
		Import-Module Msonline
		Connect-MSOLService -credential $O365Creds
	}
	

	
	# Connecting and improrting modules
	Connect-O365
	Import-Module ActiveDirectory
	Import-Module MSOnline
}

New-Connection
```

Next, I search though the Term30 OU for users accounts that are disabled, in case someone accidentally put an active account in there, have not been modified for 30 days or greater and the homedirectory value. Then export that data to a CSV file

```powershell
$OldUsers = Get-ADUser -Searchbase "OU=Term30, OU=Termed Accounts, DC=domain, DC=com" -Filter * -Properties samaccountname, Modified, homedirectory | Where { $_.Modified -le $(Get-Date).AddDays(-30) -and $_.enabled -eq $false } # | select samaccountname, Modified

# Exporting users to CSV for verifcation
$OldUsers | Export-Csv C:\Scripts\Script_Results\Term30.csv -NoTypeInformation
```

Once I have my list of users into a CSV that gets emailed to the helpdesk to create a ticket for removal of the home directories. If you trust all the users have the correct value in their AD profile this could be automated as well, the service account this runs under would just need the appropriate permissions on the file server to remove the users folder.

```powershell
Send-MailMessage -From "Term30Script <Term30Script@domain.com>" -To "helpdesk@domain.com" -Subject "Network Shares from Term30" -Body "Task to Server team. Please see attatched CSV file" -Attachments "C:\Scripts\Script_Results\Term30.csv" -SmtpServer "smtp.domain.net"
```

Now on to the O365 part, fair warning though. Once a license is removed the mailbox is gone forever and cannot be recovered. You may wish to place mailboxes into a litigation hold depending on your companies requirements for retaining mail. Most of our users this is not needed and if it is required we receive the request to apply the litigation hold when the user is terminated/quits so I am not worried about it here.

I simply loop through each of the users in $oldusers removing the license and removing the mailbox from the recycle bin. I have the error action to silently continue because 90% of our employees only get internal email only through iMail from IPSwitch.

```powershell
foreach ($user in $OldUsers)
{
	Remove-MsolUser -UserPrincipalName "$User" -Force -ErrorAction SilentlyContinue
	Remove-MsolUser -UserPrincipalName "$User" -RemoveFromRecycleBin -Force -ErrorAction SilentlyContinue
}
```

Finally, the script removes the AD user object itself along with any child objects it has. This is the biggest difference, apart from the O365 license removal over our old solution as it would break on accounts with child objects.

```powershell
foreach ($user in $OldUsers)
{
	Remove-ADobject (Get-ADUser $user).distinguishedname -Recursive -Confirm:$false
}
```

Here is the complete script, just note you will need the Azure/Microsoft Sign in assistant software installed for the O365 part to work.

```powershell
<#	
	.NOTES
	===========================================================================
	 Created with: 	SAPIEN Technologies, Inc., PowerShell Studio 2015 v4.2.86
	 Created on:   	7/16/2015 10:20 AM
	 Created by:   	DouglasFrancis
	 Organization: 	Snarkysysadmin.com
	 Filename:     	PurgeTerm30_V_0_2
	===========================================================================
	.DESCRIPTION
		Searches through the Term30 OU and find any accounts that have not been modified in the last 30 days.
		It then deletes these accounts along with it's child objects and O365 license.
#>

# Powershell transcription setup
# Create a filename based on a time stamp.
$Filename = ((Get-date).Month).ToString() + "-" +`
((Get-date).Day).ToString() + "-" +`
((Get-date).Year).ToString() + "-" +`
((Get-date).Hour).ToString() + "-" +`
((Get-date).Minute).ToString() + "-" +`
((Get-date).Second).ToString() + "-PurgeTerm30" + ".txt"
# Set the storage path.
$Path = "C:\Scripts\PS Transcripts"
# Turn on PowerShell transcripting. 
Start-Transcript -Path "$Path\$Filename"

# setting up modules and O365 connections

Function New-Connection
{
	#Credential setup for O365
	$O365User = "deleting@domain4.onmicrosoft.com"
	$O365Pass = "p@ssw0rd" | ConvertTo-SecureString -AsPlainText -Force
	$O365Creds = New-Object System.Management.Automation.PSCredential ($O365User, $O365Pass)
	
	# O365 Function
	Function Connect-O365
	{
		$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri https://ps.outlook.com/powershell/ -Credential $O365Creds -Authentication Basic -AllowRedirection
		Import-PSSession $Session -AllowClobber
		Import-Module Msonline
		Connect-MSOLService -credential $O365Creds
	}
	

	
	# Connecting and improrting modules
	Connect-O365
	Import-Module ActiveDirectory
	Import-Module MSOnline
}

New-Connection

# Searching through term 30 OU for accounts that have not been modified in 30 days or greater.
# NOTE: uncomment out the end of the line and the export line to export data to CSV. 
# However, with the select-object cmdlet in place the remove-adobject cmdlet will not work.

$OldUsers = Get-ADUser -Searchbase "OU=Term30, OU=Termed Accounts, DC=domain, DC=net" -Filter * -Properties samaccountname, Modified, homedirectory | Where { $_.Modified -le $(Get-Date).AddDays(-30) -and $_.enabled -eq $false } # | select samaccountname, Modified

# Exporting users to CSV for verifcation
$OldUsers | Export-Csv C:\Scripts\Script_Results\Term30.csv -NoTypeInformation

# Emailing CSV
Send-MailMessage -From "Term30Script <Term30Script@domain.com>" -To "domainitsupport@domain.com" -Subject "Network Shares from Term30" -Body "Task to Server team. Please see attached CSV file" -Attachments "C:\Scripts\Script_Results\Term30.csv" -SmtpServer "smtp.domain.net"

# Removing the user from O365
# NOTE: actions are commented out bc there is no -whatif flag for the Remove-msoluser cmdlet 

foreach ($user in $OldUsers)
{
	Remove-MsolUser -UserPrincipalName "$User" -Force -ErrorAction SilentlyContinue
	Remove-MsolUser -UserPrincipalName "$User" -RemoveFromRecycleBin -Force -ErrorAction SilentlyContinue
}

# Removing the AD User accounts. and it's child objects
# NOTE: remove the -whatif flag for prod use

foreach ($user in $OldUsers)
{
	Remove-ADobject (Get-ADUser $user).distinguishedname -Recursive -Confirm:$false
}
```