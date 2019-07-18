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

I've been playing with some AD stuff, partially because I've been avoiding tracking down Dev people about migrating off the last of the 2003 servers that need to die.

Anyway, on to the PowerShell goodness.

The first thing I've got is getting all users in the domain and outputting their name, the last time their password was reset and true/false if the account is set never to expire.

First thing first, we need to get all the users in the domain.

```powershell
Get-AdUser -Filter *
```

Now this gives you a fair bit of data by default.

Most of which is useless for this purpose as the only thing in here that I'm looking for is 'Name'. To get nice and clean data that I want, I'm going to use the properties flag to get 'passwordlastset' and 'passwordneverexpires'.

```powershell
Get-ADUser -Filter * -Properties passwordlastset, passwordneverexpires
```

We now have the data we want, but it’s with other values we don’t want, and it’s still not very readable. So I’m going to pipe that data to select the name, passwordlastset, passwordneverexpires data and pipe it again to sort it by the passwordneverexpires field.

```powershell
| Select Name, passwordlastset, PasswordneverExpires | Sort-Object passwordnever expires
```

Tossing that all together, and we get this wonderful one-liner.

```powershell
get-aduser -filter * -properties passwordlastset, passwordneverexpires | select Name, passwordlastset, Passwordneverexpires | Sort-Object passwordneverexpires
```

Which when run gives out this sweet little table formatted by the users' name, the last time their password was changed and true/false if the account is enabled.

You could then take this pipe it again, to a text, CSV or one of my favorites for quickly viewing data "Out-GridView" Enjoy and remember, as a sysadmin pipes are your best friend.
