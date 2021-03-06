---
title: "Start/Stop Maintenance Scripts for Exchange 2016"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Powershell
  - O365
  - Office 365
  - Exchange
  - Windows
  - Microsoft
  - Automation
---

Coming from Exchange 2010 one of the things that I love/hate about Exchange 2016 is when a new CU is installed it is essentially a full install of Exchange. In some ways, this is nice because it gets rid of a lot of the weird bugs that would occur post update after CUs that had weird triggers, and you get a nice fresh new install of Exchange, for the most part, every quarter. However, on the downside of this, the CU update process now takes much longer to complete in 2016 because, well you’re getting a nice fresh install. Also deploying that ~5GB ISO file around is a tad bit annoying at times.

In my experience after the initial install, and the 3 or 4 CU’s I’ve done since, it takes approximately one hour per server for an install/CU upgrade from the moment that the setup program is executing its installation steps. Probably an hour and a half from setup launch/prereq check/install.

In addition, you need to place the Exchange host in maintenance before starting the CU upgrade which can take some more time, especially if you have to look up commands again because I don’t know about you, but I don’t do Exchange CU updates frequently enough to remember the exact syntax for all the Powershell cmdlets.

So that I could cut down on time spent interacting with the host during CU upgrade changes I made myself two small helper functions for maintenance tasks. First places the Exchange host in maintenance, drains active database copies and pauses cluster membership. Then the second reverses all of that. This probably saves me 15-20 minutes per host now, and that adds up when I have 4 Exchange hosts to patch. Also, with a bit more error handling and comment-based help, I eventually can turn these helper functions over to my GNOC and have them patch.

First off, I run an elevated Exchange PowerShell module session on the server I’m starting the maintenance from. Import this function to my profile, execute it providing the name of the Exchange server I am currently working on, and another Exchange server for roles to move over to.

```powershell
<# 
.NOTES
===========================================================================
Created with: SAPIEN Technologies, Inc., PowerShell Studio 2018 v5.5.151
Created on: 5/31/2018 2:10 PM
Created by: Douglas Francis
Organization: PoshOps.IO
Filename: Start-ExchangeMaintenance.ps1
===========================================================================
.DESCRIPTION
Places an exchange 2016 server in maintenance in prep for patching.
#>

function Start-ExchangeMaintenance
{
    param
   (
    [parameter(Mandatory = $true)]
    [validateNotNullOrEmpty()]
    [String]$maintServer,
    [Parameter(Mandatory = $true)]
    [ValidateNotNullOrEmpty()]
    [String]$failoverServerFQDN
    )

    BEGIN
    {

    }

    PROCESS
    {
        Write-Verbose "Draining Mail Queues"
        Set-ServerComponentState $maintServer -Component HubTransport -State Draining -Requester Maintenance

        Write-Verbose "Restarting Transport Services"
        Restart-Service MSExchangeTransport
        Restart-Service MSExchangeFrontEndTransport

        Write-Verbose "Redicting pending mail to Failover Server"
        Redirect-Message -Server $maintServer -Target $failoverServerFQDN

        Write-Verbose "Suspending DAG activity"
        Suspend-ClusterNode $maintServer

       Write-Verbose "Moving any Active Database Ownership to other servers"
       Set-MailboxServer $maintServer -DatabaseCopyActivationDisabledAndMoveNow $True

       Write-Verbose "Blocking mmaintenance server from hosting active database copies"
       Set-MailboxServer $maintServer -DatabaseCopyAutoActivationPolicy Blocked

       Write-Verbose "Placing Server in maintenance"
       Set-ServerComponentState $maintServer -Component ServerWideOffline -State Inactive -Requester Maintenance
    }

    END
    {

    } 
}
```

I frequently like to use the “Write-Verbose” cmdlet for small things like this instead of comments Verbose provides info during execution and reading in the script provides as much info as a comment would in simple things like this.

Everything that happens here is getting mail queues off the server, stopping membership in the DAG, moving over database ownership and placing exchange in maintenance.

Once maintenance on the server completes, I import and run this script that reverses the actions of the first one allowing the exchange server to function in the DAG again fully.

```powershell
<#    
    .NOTES
    ===========================================================================
     Created with:     SAPIEN Technologies, Inc., PowerShell Studio 2018 v5.5.151
     Created on:       5/31/2018 3:45 PM
     Created by:       Douglas Francis
     Organization:     Afni    
     Filename:         Stop-ExchangeMaintenance.ps1
    ===========================================================================
    .DESCRIPTION
        Stops exchange 2016 server from being in maintenance and function in the DAG/mail flow again.
#>

function Stop-ExchangeMaintenance
{
    param
    (
        [parameter(Mandatory = $true)]
        [validateNotNullOrEmpty()]
        [String]$maintServer
    )
    
    BEGIN 
    {
    }
    PROCESS
    {
        Write-Verbose "Taking Server out of Maintenance"
        Set-ServerComponentState $maintServer -Component ServerWideOffline -State Active -Requester Maintenance
        
        Write-Verbose "Restart DAG activity"
        Resume-ClusterNode $maintServer
        
        Write-Verbose "Allow Database Activation"
        Set-MailboxServer $maintServer -DatabaseCopyActivationDisabledAndMoveNow $False
        
        Write-Verbose "Set database back to the original setting"
        Set-MailboxServer $maintServer -DatabaseCopyAutoActivationPolicy Unrestricted
        
        Write-Verbose "Reactivate hub transport"
        Set-ServerComponentstate $maintServer -Component HubTransport -State Active -Requester Maintenance
        
        Write-Verbose "Restart Transport Services"
        Restart-service msexchangetransport
        Restart-service msexchangefrontendtransport
    }
    END
    {
    }
}
```

Its quick, simple helper scripts like this that makes things go a bit quicker and while it still takes an hour to do a CU upgrade it cuts some of the pre/post prep work out of the equation and could also be modified for SCCM to use in patching maintenance as well to help automate that process.
