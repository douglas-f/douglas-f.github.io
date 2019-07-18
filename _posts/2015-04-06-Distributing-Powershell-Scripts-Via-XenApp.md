---
title: "Distributing Powershell Scripts via XenApp"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - Citrix
  - O365
  - Office 365
  - XenApp
---

We have been busy standing up System Center for the past few weeks, which is akin to adding another full-time job on top of my already full-time job. 

What I've noticed, as we've mostly ignored the ticket queue this week, is the massive amount of email-related requests we get now. A lot of this is due to the migration to hybrid O365 and our helpdesk and ID team not having the same permissions they used to. 

As System Center is not ready to handle a lot of these scriptable tasks yet, and now have even less time to devote to them. I considered ways I could leverage the scripts that I already use,  without giving the help desk and ID team more access than necessary.

I started with a few things. I knew that RBAC in Exchange would be my best bet to give only the permissions that I wanted. However, with the Hybrid O365 deployment, those rights would need to be set up on the local CAS server and in the cloud. Mostly on CAS, I only require users to have the ability to enable a user account to have a remote mailbox, so that was simple enough. I created a service account that had the rights I needed for the local CAS server.

Now that that part was set up, I started looking into RBAC control in the cloud. Here I wanted the help desk/ID team to use their accounts for the tasks they performed, so there was some level of accountability on who did what. Rhoderick Milne, a Microsoft Senior Premier Field Engineer, has a very helpful post here on exploring the options for RBAC settings in the cloud. Using this, I set up roles that allowed these users to create and modify mailboxes distros and contacts.

Next, I needed to modify the assortment of scripts that I already use for managing mail, and determine which scripts would be the most useful for cutting down on my ticket count. I settled on these five scripts initially.

– Create Contact Account

– Create Distro Lists

– Add access to a mailbox

– Delete a mailbox

– Setup a new mailbox

– Convert a user mailbox to a Shared Mailbox

These requests make up about 80% of the mail requests we get each week, mostly for new hires and terminations.

Next, I went to work on making my scripts a bit more user-friendly for non-sysadmins to use. Adding some color to the console and setting up credentials for the CAS account to be stored in the scripts. Now with O365, there are a few requirements I needed on the machines running these scripts, Powershell v4, the sign-in assistant and azure PowerShell management. Here came the issue. As it was simple enough to deploy all of this to the dozen machines the users have, but when working remote, they all use a provisioned Win7 machine on VDI. The issue is that there is one generic image that all IT staff use for VDI and their apps are broken out in XenApp.

So I worked with our Desktop Engineering group that handles our VDI infrastructure to set up the XenApp server to have all of those prereqs installed to run the scripts. Getting that setup was straight forward and simple but I now had a new problem. If users run the scripts hosted on the XenApp server after the script executes they would have full PowerShell permissions to that server. Not a very good idea from a security standpoint.

Here is where PowerGUI came in. PowerGUI is a free Powershell IDE that allows you to package scripts up in an executable. More importantly, it gives you the ability to hide the PowerShell console and or exit the console when the script completes when the EXE is compiled. With this, I do not need to worry about the users seeing the credentials of the CAS service account. And I do not need to worry about them wreaking havoc on the XenApp server as anything they would exit the script closes the XenApp.  They have 45-day trial versions available if you want to give them a try.

With scripts created and deployed to the XenApp server and access given to the users and a little bit of documentation, I've cut down my daily ticket count and have something in place to help until we get System Center fully stood up to take over these processes. I will post the scripts I created for these tasks soon as well.
