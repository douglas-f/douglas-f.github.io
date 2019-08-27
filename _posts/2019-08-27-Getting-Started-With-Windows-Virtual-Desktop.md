---
title: "Getting Started with Azure Virtual Desktop"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Azure
  - VDI
  - Microsoft
  - Virtual Desktop
  - PaaS
---

Microsoft has announced the Preview release of Windows Virtual Desktop (WVD) running in Azure giving enterprises the ability to deploy Windows 10 VDI and applications. As this feature is in Preview it is not as polished as other Azure offerings but there is a lot of good features and options available and would be great for a Proof of Concept deployment with IT staff. Or if some of the unpolished management tools don't phase you then this could be used for non-technical users as well. The end user experience is really good, especially when leveraging the Windows Virtual Desktop Remote Desktop Application.

<!--more-->

There are a few dependencies you'll need to meet before getting started with WVD.

First you need access to *Active Directory*, this can either be provided by IaaS VM's running Active Directory Domain Services or through an Azure Active Directory Domain Services deployment. In this environment I will be leveraging AADDS. The VM's we will be deploying later *must* be domain joined, AAD join is not an option.

Second, you need rights to Azure AD to create and register Enterprise Applications and Service Principals along with the ability to deploy Azure resources to a subscription.

Additionally, you need the Azure AD Directory ID (GUID) of your tenant and of the Azure Subscription you will be deploying resources too. Keep that information handy, you'll need it throughout the deployment process.

Finally, you need to install the PowerShell module for WVD

 ```powershell
 Install-Module -Name Microsoft.RDInfra.RDPowerShell
 ```

Step one to deploying WVD is navigating to the [Windows Virtual Desktop Consent Page](https://rdweb.wvd.microsoft.com/). The consent page is where we are going to get two new Enterprise applications in the AAD tenant for managing the WVD servers and for client access.
![alt text][consent]

[consent]: https://poshops.io/assets/images/azurevdi/vdiConsentPage.png "Windows Virtual Desktop Consent"

Input the Azure Active Directory Directory ID (GUID) for the AAD tenant users will be authenticating with. Once the submit button is hit you'll need to authenticate to Azure AD and consent to application permissions. This is read access to Azure AD and read assess to the Graph API.
![WVD Server Permissions][permissions]
[permissions]: https://poshops.io/assets/images/azurevdi/wvdPermissions.png

After consenting to the Server App go back and repeat the same process for the client application as well.
Please wait at least 30 seconds between granting consent so AAD can process the changes on your tenant.

After consenting to those applications to be a part of your AAD tenant you need to grant a user 'TenantCreator' access to the new Windows Virtual Desktop enterprise application. The name 'TenantCreator' here applies only to the Virtual Desktop Tenant, not your AAD tenant.

In the Azure Portal navigate to enterprise applications by going to 'Azure Active Directory > Enterprise Applications > Search for Windows Virtual Desktop'

You will see the new applications you consented to, 'Windows Virtual Desktop' and 'Windows Virtual Desktop Client'.

![Enterprise Application Search][enterpriseApp]
[enterpriseApp]: https://poshops.io/assets/images/azurevdi/enterpriseAppSearch.png

Add an administrator user 'TenantCreator' permissions, as of now in the preview this is the default and only permissions option that can be delegated.

![Tenant Creator Permissions][tenantcreator]
[tenantcreator]: https://poshops.io/assets/images/azurevdi/wvdTenantCreator.png

Now it's time to drop into PowerShell and setup everything we need to go deploy the resources. Note: As of now you cannot use PowerShell Core and need to use Windows Powershell. 

Add the deployment URL for the VDI environment. As of right now in Preview this is the only option. I expect the GA release to allow for custom domains here in the future.

```powershell
Add-RdsAccount -DeploymentUrl "https://rdbroker.wvd.microsoft.com"
```

Next the creation of the Windows Virtual Desktop Tenant, you need the Azure Active Directory Directory ID and the ID of the Azure subscription you'll be adding resources in. I'm also going to create a variable here for the TenantName, it'll be used throughout the setup.

```powershell
$TenantName = 'PoshOps.io'
$AadDirectory = 'XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX'
$AzSubscription = 'XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX'
New-RdsTenant -Name $tenantName -AadTenantId $AadDirectory -AzureSubscriptionId $AzSubscription
```
