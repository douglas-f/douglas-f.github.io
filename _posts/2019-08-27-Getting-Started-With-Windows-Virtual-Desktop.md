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

![alt text][permissions]

[permissions]: https://poshops.io/assets/images/azurevdi/wvdPermissions.png "WVD Server Permissions"

After consenting to the Server App go back and repeat the same process for the client application as well.
Please wait at least 30 seconds between granting consent so AAD can process the changes on your tenant.

After consenting to those applications to be a part of your AAD tenant you need to grant a user 'TenantCreator' access to the new Windows Virtual Desktop enterprise application. The name 'TenantCreator' here applies only to the Virtual Desktop Tenant, not your AAD tenant.

In the Azure Portal navigate to enterprise applications by going to 'Azure Active Directory > Enterprise Applications > Search for Windows Virtual Desktop'

You will see the new applications you consented to, 'Windows Virtual Desktop' and 'Windows Virtual Desktop Client'.

![alt text][enterpriseApp]

[enterpriseApp]: https://poshops.io/assets/images/azurevdi/enterpriseAppSearch.png "Enterprise Application Search"

Add an administrator user 'TenantCreator' permissions, as of now in the preview this is the default and only permissions option that can be delegated.

![alt text][tenantcreator]

[tenantcreator]: https://poshops.io/assets/images/azurevdi/wvdTenantCreator.png "Tenant Creator Permissions"

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

![alt text][pstenantcreation]

[pstenantcreation]: https://poshops.io/assets/images/azurevdi/powershellCreateRdsTenant.png "Powershell - Create RDS Tenant"

Next we need to connect to Azure AD

```powershell
$aadcontext = Connect-AzureAD
```

![alt text][connectaad]

[connectaad]: https://poshops.io/assets/images/azurevdi/connectAAD.png "Powershell - Connect Azure AD"

Best practice is to create an AAD Service Principal for the WVD services to run under. You *could* create the deployment under a named user, but it is not recommended.

```powershell
$svcPrincipal = New-AzureADApplication -AvailableToOtherTenants $true -DisplayName "Windows Virtual Desktop Svc Principal"
$svcPrincipalCreds = New-AzureADApplicationPasswordCredential -ObjectId $svcPrincipal.ObjectId
```

![alt text][createsp]

[createsp]: https://poshops.io/assets/images/azurevdi/powershellcreatesp.png "Powershell - Create AAD Service Principal"

After creation of the Service Principal we are going to assign it RDS owner rights

```powershell
New-RdsRoleAssignment -RoleDefinitionName "RDS Owner" -ApplicationId $svcPrincipal.AppId -TenantName $tenantName
```

![alt text][rdspermissions]

[rdspermissions]: https://poshops.io/assets/images/azurevdi/addRdsPermissions.png "Powershell - Add RDS Permissions"

Next up is creating a new credential object and updating that role. 

```powershell
$creds = New-Object System.Management.Automation.PSCredential($svcPrincipal.AppId, (ConvertTo-SecureString $svcPrincipalCreds.Value -AsPlainText -Force))

Add-RdsAccount -DeploymentUrl "https://rdbroker.wvd.microsoft.com" -Credential $creds -ServicePrincipal -AadTenantId $aadContext.TenantId.Guid
```

![alt text][addpermissions]

[addpermissions]: https://poshops.io/assets/images/azurevdi/powershellUpdatePermissions.png "Powershell - Update RDS Role"

We now need to make note of the Application ID and the Service Principal key, keep track of this because after you close this PowerShell session you will have to reset the Service Principal key to get access again. We will use this information in the portal for the WVD deployment.

```powershell
$svcPrincipal.appid 
$svcPrincipalCreds.value
```

![alt text][secrets]

[secrets]: https://poshops.io/assets/images/azurevdi/appsecrets.png "Powershell - App Secrets"

Okay we're almost there! Time to drop into the Azure Portal and create our WVD deployment!
