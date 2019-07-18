---
title: "Disabling License Features in O365"
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

With the changes Microsoft has deployed to O365, adding new Skype plan options to K1 accounts and updating the plan on other license plans, I needed to remove this license for ~5000 users.

Some of my K1 users have licenses solely for Sharepoint Online while some have it for both Exchange and Sharepoint.

What I’ve done is get all the MSOL users in my domain storing that in  $allusers

Then I loop through getting all of my K1 accounts, do a Get-Mailbox to determine if they have a mailbox so that I don’t remove the license options for Exchange for people that already have it.

Depending on that erroring out or not, I create a new MSOL license then assign the right permissions.

If you only need to remove the Skype plan, this would be the new license option.

```powershell
$licensePlan = New-MsolLicenseOptions -AccountSkuId COMPANY:DESKLESSPACK -DisabledPlans "MCOIMP"
```

This isn't the fastest way to run this but I have a server dedicated to running powershell scripts so I just kick it off and wait.

```powershell
<#	
	.NOTES
	===========================================================================
	 Created with: 	SAPIEN Technologies, Inc., PowerShell Studio 2016 v5.3.130
	 Created on:   	1/10/2017 1:08 PM
	 Created by:   	Douglas Francis
	 Organization: 	PoshOps.io
	 Filename:     	Remove-SkypeK1License
	===========================================================================
	.DESCRIPTION
		A description of the file.
#>

# Getting all MSOL users
$allusers = Get-MsolUser -all

# getting all the k1 users from that
$k1users = $allusers | Where-Object { $_.islicensed -eq 'true' -and $_.licenses.accountskuid -contains 'COMPANY:desklesspack' }

#counter for displaying progress
$totalcount = $k1users.count
$count = 1

#looping through all users
foreach ($user in $k1users)
{
	Write-Host -ForegroundColor Red "Getting user $count out of $totalcount"
	Get-Mailbox $user.userprincipalname 
	if ($? -eq $true) # if has mailbox, setting disabled options leaving exchange
	{
		$user.userprincipalname | Out-File C:\Scripts\Script_Results\k1mail.txt -Append
		$licensePlan = New-MsolLicenseOptions -AccountSkuId COMPANY:DESKLESSPACK -DisabledPlans "SWAY", "INTUNE_O365", "YAMMER_ENTERPRISE", "MCOIMP"
		Set-MsolUserLicense -UserPrincipalName $Upn -RemoveLicenses $lic.licenses.accountskuid -AddLicenses 'COMPANY:DESKLESSPACK' -LicenseOptions $licensePlan
	}
	
	else
	{
		# setting disabled options for users that only get sharepoint
		$user.userprincipalname | Out-File C:\Scripts\Script_Results\k1nomail.txt -Append
		$licensePlan = New-MsolLicenseOptions -AccountSkuId COMPANY:DESKLESSPACK -DisabledPlans "SWAY", "INTUNE_O365", "YAMMER_ENTERPRISE", "EXCHANGE_S_DESKLESS", "MCOIMP"
		Set-MsolUserLicense -UserPrincipalName $Upn -RemoveLicenses $lic.licenses.accountskuid -AddLicenses 'COMPANY:DESKLESSPACK' -LicenseOptions $licensePlan
	}
	
	$count++
	
}
```
