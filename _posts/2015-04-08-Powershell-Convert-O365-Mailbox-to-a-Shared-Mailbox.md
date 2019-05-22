---
title: "Powershell Convert O365 Mailbox to a Shared Mailbox"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - Hybrid Exchange
  - Microsoft
  - Automation
  - Office 365
  - O365
  - Mailboxes
---

UPDATE: 09/24/2015- While converting mailboxes to shared is still a useful feature it turns out managing NLE users this way in a hybrid O365 environment was more trouble than it was worth. With the users AD object still existing we would frequently get loop failures as mail would bounce between O365 and our local exchange server. In this post, I write about how I eventually settled on automating the removal of O365 mailboxes for NLE users.

With moving to a Hybrid Office 365 mail enviroment we need to be extremely careful on our license usage. At any given time we only have a few dozen licenses available so when we employees leave the company frequently there is a need for their manager or former team members to have access to the employees mailbox to handle any mail they receive for a period of time. The issue is depending on the level of the employee this could be for a few days to a few months where that mailbox is using a unneeded license. To resolve this we convert these mailboxes to shared, remove the license and then are able to keep the mailbox allow people to have access to it but now have a license we can give to a current employee. Here is the powershell script I came up with to make this a quicker and easier process. And more importantly to give to our Helpdesk and ID Team so I have less ticket requests. And I’m a HUGE fan of having less ticket requests so I can actually get more important things done.


Starting out just a simple function to connect to O365.

```powershell
# O365 Function
Function Connect-O365
{
Write-Host -ForegroundColor Green "Enter Credentials for O365"
$Creds = Get-Credential
$Session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri https://ps.outlook.com/powershell/ -Credential $Creds -Authentication Basic -AllowRedirection
Import-PSSession $Session -AllowClobber
Import-Module Msonline
Connect-MSOLService -credential $creds
}
# Connecting to O365
Connect-O365
```

Now, with a company of our size it is not uncommon to have multiple requests come in on our daily termination report that have mailboxes that need to be changed. As I didn’t want to have to rerun this script and enter credentials each time I used a DO loop to run the command multiple times without running the script each time.

```powershell
# Getting Name of the mailbox to be converted
Do {
Write-Host -ForegroundColor Green "Enter the email address of the mailbox to be converted, or 'q' to quit: " -NoNewline
$Mailbox = Read-Host</code>

If ($Mailbox -NotLike "q")
{
Get-Mailbox $Mailbox | Set-Mailbox -Type Shared

Get-Mailbox $Mailbox | Set-MsolUserLicense -RemoveLicenses "TENANTID:EXCHANGEARCHIVE_ADDON"
Get-Mailbox $Mailbox | Set-MsolUserLicense -RemoveLicenses "TENANTID:STANDARDPACK"

Get-MsolUser -UserPrincipalName $Mailbox | Ft UserPrincipalName,DisplayName,Licenses
}

}
Until ($Mailbox -Like "q")
```

As I’m giving these to helpdesk/ID team staff that have varying levels of technical knowledge I wanted to make things slightly more user friendly. And nice green colors make everyone feel better right? Right. The issue I had was that the Read-Host cmdlet does not allow you to change the color of the prompt. To get around this I use Write-Host with the Foreground color and end it with the -NoNewLine flag. So when I follow that up with the Read-Host everything is on one nice line.

As for the loop a simple Do until $Mailbox -Like “q” runs the script section over and over again until the user hits the ‘q’ key and it finally exits. And finally I grab what license are assigned to that user and display it. I do this for a couple of reasons. First I’ve had other scripts that work great until something changes in the cloud and all of the sudden it doesn’t work so it is nice to have verification. Secondly with multiple licenses that not all users are assigned the script will give those. I didn’t want to code it in and have the script error out which would lead to a confused helpdesk/ID team person and eventually leading to them just ignoring errors. So now if a license shows up after this is ran the users can shoot me an email and I can remove it. Until I trust them enough to do things in the web portal even with their limited access.

Finally if you do not know what licenses your company has a simple “Get-MsolAccountSku” command will give you a list of licenses that you have along with the number you have and the number used.

One quick word there are a few things you need for this to work. The Microsoft Online Service Sign-in Assistant and the Azure Active Directory Module for Windows Powershell to be able to connect along with powershell v4 with latest updates of .Net. If you’d like to see how I ended up giving this to the helpdesk/ID team staff check out my XenApp post here.