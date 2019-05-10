---
title: "Powershell Server Setup: Part I"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - Windows Server
  - Microsoft
  - Automation
---

I love powershell. I think it is fantastic, powerful and the fact that common linux commands such as “cat”, “ls” and “cd” work nearly bring warmth to my twisted soul. So it was about the third time in a week that I had a sudden “urgent” server build to do, along with what I was already working on I vowed I would use powershell to make the task of building out new severs less annoying until we get System Center stood up at the office.

This will be a three part post, mostly because I have my setup ran in three scripts. I’ve found this makes life easier in case something breaks and hey you have to reboot for updates about 60 million times anyway.

Just a quick disclaimer, while I use this every week at work I’m not responsible for you fat fingering a command and breaking shit. the -WhatIf flag is your friend. If you do not understand a command us -WhatIf to see what will happen without actually doing it. Secondly, I have only tested this on 2012R2 servers however, I do not believe there is anything in here preventing you from running this on 2k8 servers if you need to.

<!--more-->