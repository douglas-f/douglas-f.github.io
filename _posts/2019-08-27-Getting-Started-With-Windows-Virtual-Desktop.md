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

---

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

Okay we're almost there! Time to drop into the Azure Portal and create our WVD deployment! This is where all of the infrastructure we need to leverage WVD gets created.

In the Portal click 'create a resource and search for 'Windows Virtual Desktop - Provision a Host Pool' > Click create.

![alt text][hostPool]

[hostPool]: https://poshops.io/assets/images/azurevdi/hostpool.png "Azure Portal - Create WVD Host Pool"

The deployment wizard will start, the first step is providing basic information about the HostPool that will host your VDI instances/applications.

Few things of note here. If you are going to host both VDI and applications you should create a separate host pool for VDI and another for applications. As of now you cannot have a user in an Application Group for VDI *and* in an Application Group for a hosted application within the same pool. Which means, if you want Jane User to be able to access a Windows 10 VDI instance and a published LoB application at the same time then the app has to be in a different pool than the VDI hosts. If you have no need for users to have access to both then feel free to create just one Hostpool.

In addition this is the step where you determine what type of Hostpool this will be. The options are 'pooled' or 'personal.' A personal pool is a dedicated, non-persistent VDI environment with a one-to-one mapping between user and VDI instance. The system will automatically assign the user to an instance when signing in. However, if you create a personal hostpool with only one VDI instance in it, and assign two users to it. Only the user that signs into the pool first will be able to get a VM. The second user will receive a no resources error, even if the first user is *not* signed into the VM at the time.

I highly recommend thinking through a solid naming convention that will survive changing needs of the organization and what you may be using VDI for. I'm going to create resources under this format
*env-purpose-pooltype-region-iteration*

For example, a development pool, used for hosting personal VDI will be "dev-vdi-personalpool-ussc-01", conversely a shared application hostpool would be "dev-app-sharedpool-ussc-01" I'm going to use the same convention with my resource groups, just adding an 'rg' in the name as there is a one-to-one relationship with pools and resource groups.

Speaking of user assignments, right now you can only grant access via a user's UPN, not through an AAD group. Don't worry about getting all of the user assignments added right now, you can add and remove user later via PowerShell or the management application.

Finally, the resource group must be empty and not contain any resources, it's easier to then make a new resource group through this deployment wizard.

![alt text][deploystep1]

[deploystep1]: https://poshops.io/assets/images/azurevdi/deploystep1.png "Azure Portal - Deployment Basics"

In step two we're going to size our VDI deployment. There are 4 options for how the wizard is going to determine the compute requirements.

* Light - 6 users per vCPU
* Medium - 4 users per vCPU
* Heavy - 2 users per vCPU
* Custom - you just provide a number of VM's

What the deployment wizard is creating is a Virtual Machine Availability set. Once created, VM Availability Sets *cannot* have VM's added or removed. You will be able to scale the size of these VM's up and down as your needs grow, and shut them down to save on host but to remove one you will need to remove them all.

By default the wizard selects D8s v3 VM's for deployment. Right now that is ~$330/month per VM, plus networking/storage costs. You can select different types of VM's, such as compute or memory optimized as your needs and budget require.

With the size of the VM you've choosen, the usage profile and number of users inputted the wizard will math out how many VM's will be required to support that workload. There isn't any additional buffer included other than what may be extra just to how the users to vCPU calculations compute out.

For example, if you do a light profile for 500 users on D8s v3 VM's the wizard will want to deploy 11 VM's, totalling 88 vCPU's. 500 users, divided by 6 users/vCPU is 84 vCPU's rounding up, giving 4 extra cores. In the event that a VM goes down with all users online, there would be a resource outage.

The last important piece on this step is the VM name prefix, this is in place to prevent name collision in AD/WVD. Anything you provide here will have a '-#' appended to it. For example 'wvdvm' becomes 'wvdvm-0' and 'wvdvm-1'.

![alt text][deploystep2]

[deploystep2]: https://poshops.io/assets/images/azurevdi/deploystep2.png "Azure Portal - Config Virtual Machines"

Step 3 are the VM configuration settings. This is where you can choose what OS you want to deploy, and how/where you want it deployed. The quickest way to get started is deploying either the Windows 10 or Server 2016 images from the Gallery. Additonally, you can deploy an image from a VHD you have on blob storage or from a Managed Azure Instance. This is how you'd deploy custom applications if you're going that route.

Next, select the disk type, provide a UPN to perform the domain-join, provide a specific OU for the computers to join if you'd like and select the Azure vNet and subnet for the VM's to reside on.

The 'gotchas' in this section are ensuring you provide a domain-join account as a UPN, *not* as a 'Contoso\Admin' format as you may be used to. In addition to that the wizard by default wants to create a new vNet. *Do not do this.* The VM's *have* to have the ability to talk to Active Directory services. If you create a new vNet then these machines will not be able to talk to your domain services and the deployment will fail. They can be on different subnets but if you deploy to a new vNet the deployment fails. By default there is not vNet to vNet traffic allowed.

![alt text][deploystep3]

[deploystep3]: https://poshops.io/assets/images/azurevdi/deploystep3.png "Azure Portal - VM Settings"

Step Four is authentication to Windows Virtual Desktop. Leave the tenant group name default, provide the tenant name you created earlier in powershell and change RDS owner from UPN to Service Principal. This is where you provide the App ID, password and AAD tenant ID info we saved earlier. 

![alt text][deploystep4]

[deploystep4]: https://poshops.io/assets/images/azurevdi/deploystep4.png "Azure Portal - VM Auth"

Step Six is just a summary of the info you've provided so far, this is your chance to review all the settings you've provided. The first time you run this it may take a few minutes for the summary page to complete loading so you can proceed. This is so any needed resource providers that are needed but you don't have in place can be registered.

Once validation has passed and you're happy with the settings click ok.



