title: "Active Directory Cleanup with Powershell: Finding inactive users"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - Active Directory
  - Microsoft
  - Automation
---

Keeping Active Directory clean can be a very time consuming tasks. This can be especially true in a company like mine where we add and remove 100’s of users a month from the environment. It also does not help that some of our clients have accounts with us for accessing tools and resources, and other shenanigans. With this in mind I set about to write a script to run as a scheduled task every Monday morning, that searches though our AD environment, minus a handful of specific OU’s such as Service Accounts, Leave of Absence users and the like, and finds any account that has not been signed into in over 90 days.

I also needed to get any account that has not been signed into at all. Thankfully when searching by the LastLogonDate property an account that has not been signed into will also show up. Sadly I had to do this because it isn’t uncommon for Vender/Client accounts to be requested and then never used. They all typically are set to expire but why worry about it or have them there if I don’t have to.

So now that I know roughly what I need I’ll use a simple, but long depending on how many OU’s that need to be excluded command to get all of the users in a variable to work with.

```powershell
$Users = Get-ADUser -Properties LastLogonDate, created -Filter * | where {$_.distinguishedname -notmatch "OU=External Accounts,DC=domain,DC=net" -and $_.distinguishedname -notmatch "CN=Microsoft Exchange System Objects,DC=domain,DC=net" -and $_.distinguishedname -notmatch "OU=Exchange Accounts,OU=Accounts,OU=Corporate,DC=domain,DC=net" -and $_.distinguishedname -notmatch "CN=Builtin,DC=domain,DC=net" }
```

Next I’ll search through those users for any users that have been inactive greater than 90 days.

```powershell
$InactiveUsers = $Users | Where { $_.LastLogonDate -le $(Get-Date).AddDays(-90) -and $_.Created -le $(Get-Date).AddDays(-14)}
```

Then we are going to export that to a CSV file to ready it to be emailed into our ticketing system.

```powershell
$InactiveUsers | select name, LastLogonDate, created, distinguishedname | Export-Csv C:\Scripts\Script_Results\90users.csv -NoTypeInformation
```

Tacking that into an email for tracking in our ticketing system.

```powershell
Send-MailMessage -From "InactiveUsersScript <InactiveUserScript@domain.com>" -To "domainitsupport@domain.com" -Cc "serverteam@domain.com", "infosec@domain.com", "ITSCSCTech@domain.com", "ITSCSCIDs@domain.com" -Subject "Users inactive- 90 days" -Body "Please see attatched CSV file" -Attachments "C:\Scripts\Script_Results\90users.csv" -SmtpServer "smtp.domain.net"
```

Finally taking all of these accounts disabling them and moving them into a OU for termed employees. Accounts sit in this OU for 30 days after their last modified date and then another script cleans that up.

```powershell
foreach ($user in $InactiveUsers)
{
	Disable-ADAccount -Identity $user -Confirm:$false
	Get-ADUser $user | Move-ADObject -TargetPath "OU=Term30,OU=Termed Accounts,DC=domain,DC=net"
	
}
```

Check out the full code on GitHub here.