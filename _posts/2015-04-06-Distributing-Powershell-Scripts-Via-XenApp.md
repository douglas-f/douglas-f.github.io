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

We have been busy standing up System Center for the past few weeks which is akin to adding another full time job on top of my already full time job. This has made me realize a few things thanks to the fact that I have gone days without looking at new tickets assigned to us. One thing I have realized is the huge amount of email related requests we get now that with running on a hybrid O365 environment our help desk and ID team do not have permissions to do. Also I have realized System Center, in all of its powerful glory is not going to be the quick silver bullet to managing the infrastructure I was lead to believe, but that is another task entirely.

As System Center is not ready to handle a lot of these scriptable tasks yet and now have even less time to devote to them I considered ways I could leverage the scripts that I already use to create/modify mail related things without giving the help desk and ID team more access than I really want them to have.

I started out with a few things. I knew that RBAC in Exchange would be my best bet to give only the permissions that I wanted. However, with the Hybrid O365 deployment those rights would need to be setup on the local CAS server and in the cloud. Mostly on CAS I only need users to have the ability to enable a user account to have a remote mailbox so that was simple enough. As I did not want to give the users access to the server itself and have multiple accounts, one for on premise one for cloud I created a service account that had the rights I needed for the local CAS server.

Now that that part was setup I started looking into RBAC control in the cloud. Here I wanted the help desk/ID team to use their own accounts for the tasks they performed so there was some level of accountability on who did what. Rhoderick Milne, a Microsoft Senior Premier Field Engineer, has a very helpful post here on exploring the options for RBAC settings in the cloud. Using this I setup roles that allowed these users to create and modify mailboxes distros and contacts.

Next I needed to modify the assortment of scripts that I use all of the time to manage our mail environment and determine which scripts would be the most useful for them and cut down on my ticket count. I settled on these five scripts initially

– Create Contact Account

– Create Distro Lists

– Add Access to a mailbox

– Delete a mailbox

– Setup a new mailbox.

– Convert a User mailbox to a Shared Mailbox

These requests make up about 80% of the mail requests we get each week, mostly for new hires and terminations.

Next I went to work on making my scripts a bit more user friendly for non sysadmins to use. Adding some color to the console and setting up credentials for the CAS account to be stored in the scripts. Now with O365 there are a few requirements I needed on the machines running these scripts, Powershell v4, the sign in assistant and azure powershell management. Here came the issue as it was simple enough to deploy all of this to the dozen machines the users have but when working remote they all use a provisioned Win7 machine on VDI. The issue is that there is one generic image that all IT staff use for VDI and their apps are broken out in XenApps.

So I worked with our Desktop Engineering group that handles our VDI infrastructure to setup the XenApp server to have all of those prereqs installed to run the scripts on. Getting that setup was straight forward and simple but I now had a new problem. If users run the scripts hosted on the XenApp server after the script executes they would have full powershell permissions to that server. Not a very good idea from a security standpoint.

This is where PowerGUI came in. PowerGUI is a free Powershell IDE that allows you to package scripts up in an executable. More importantly it gives you the ability to hide the powershell console and or exit the console when the script completes when the EXE is compiled. With this I do not need to worry about the users seeing the credentials of the CAS service account and I do not need to worry about them wrecking havoc on the XenApp server as anything they would to to exit the script closes the XenApp. Also SAPIEN Technologies also have a few programs that also allow you to create scripts as EXEs and provide a lot more functionality you can check them out here. They have 45 day trial versions available if you want to give them a try.

With scripts created and deployed to the XenApp server and access given to the users and a little bit of documentation I’ve cut down my daily ticket count and have something in place to help until we get System Center fully stood up to take over these processes. I will post the scripts I created for these tasks soon as well.