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

I’ve been playing with some AD stuff. Partially because I’ve been avoiding tracking down Dev people and trying to get them to move their shit off a 2003 server that needs to die. Which is more frustrating than it sounds because every time I ask questions about what apps use what shares I either do not get answers at all or am told bullshit about how they do not know. Which really makes me want to just migrate shares and tell them to GFY and update their shit because then they will for sure know what writes where. But sadly my manager keeps telling me no.

Anyway on to the powershell goodness.

The first thing I’ve got that is slightly interesting is Getting all users in the domain and outputting their name, the last time their password was reset and true/false if the account is set to never expire.

First thing first, we need to get all the users in the domain

```powershell
Get-AdUser -Filter *
```

Now this gives you a fair bit of data by default

Most of which is useless for this purpose as the only thing in here that I’m looking for is “Name”. To get nice and clean data that I want I’m going to use the properties flag to get “passwordlastset” and “passwordneverexpires”

```powershell
Get-ADUser -Filter * -Properties passwordlastset, passwordneverexpires
```

Now we have the data we want, but it’s listed with other values we don’t want and it’s still not very readable as you need to skim through want you don’t want. So I’m going to pipe that data to select the name, passwordlastset, passwordneverexpires data and pipe it again to sort it by the passwordneverexpires field

```powershell
| Select Name, passwordlastset, PasswordneverExpires | Sort-Object passwordnever expires
```

Tossing that all together and we get this wonderful one liner.

```powershell
get-aduser -filter * -properties passwordlastset, passwordneverexpires | select Name, passwordlastset, Passwordneverexpires | Sort-Object passwordneverexpires
```

Which when ran gives out this nice little table formatted by the users name, the last time their password was changed and true/false if the account is enabled.

You could then take this pipe it again, to a txt, csv or one of my favorites for quickly viewing data other than in the console, pipe it to “Out-GridView” Enjoy and remember, as a sysadmin pipes are your best friend.