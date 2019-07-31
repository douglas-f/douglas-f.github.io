---
title: "Azure Active Directory Password Hash Sync"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Azure
  - O365
  - Office 365
  - Azure AD
  - Microsoft
---

I frequently have conversations with customers around what their authentication options are with Azure Active Directory. More specifically, the conversation leads to reasons why they should use Password Hash Synchronization over other options.

<!--more-->

Generally speaking, there are three options for authentication in the Azure AD space from Active Directory synced accounts.
    - Password Hash Synchronization (PHS)
    - Pass-thru authentication (PTA)
    - Federation (Typically ADFS, other IDP's are possible)

In short, PHS is authentication occurring in Azure AD-based off a hash of the hashed password value stored in Active Directory on-prem, synced by Azure AD Connect. With PTA authentication occurs through Azure AD communicating with an agent on-prem interacting with Active Directory. While using federation authentication requests are sent to the IDP for authentication off Active Directory.

The number one concern I receive from customers around password hash synchronization generally is "I don't want my passwords stored in the cloud for 'security.'" Although, I've yet to have a discussion with someone that can articulate what their security concerns are and why they are more concerned about those risks than currently deployed solutions.

I believe that part of this comes from a misconception on how PHS works. Commonly the people I talk to think that AD Connect is copying the hash value already stored in Active Directory and using that for Azure AD.  PHS does not pull the password hash that is in Active Directory and upload it to Azure AD and make that the Azure AD password.

What occurs can be a blog post in of itself but does get into the cryptographic weeds a bit. Without digging into all that, what happens is: AD Connect takes the hash stored in Active Directory and places it through an [MD4+salt+PBKDF2+HMAC+SHA256 cryptographic process](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-password-hash-synchronization#detailed-description-of-how-password-hash-synchronization-works). At the end of this process is a new hash value stored in Azure AD for authentication purposes. The salt value is unique to the user and is not generic across the environment, this will become important later.

Overall this process accomplishes a few things that are important for the security of Azure accounts.
    - It would not be possible for an attacker to obtain a copy of an Azure AD hash somehow and replay that hash to any on-prem resources as the user
    - Any two users that may have the same password have different hashes. As the salt is done on a per-user basis two users with "password1" as a password would have different hashes
    - Prevents rainbow tables from being effective solutions in brute-forcing attempts on the account as well.
    - The hashed value in Azure AD is cryptographically much stronger than the mechanism that is used currently in Active Directory for generating hashes

Password Hash Sync is the preferred method for authentication users with Azure AD from Active Directory sourced identities, followed by PTA and federation.

The number one reason that companies start leveraging PHS is removing the dependency on on-prem infrastructure for authentication. With PTA and federation if any outages prevent Azure AD from communication to Active Directory users will not be able to authenticate. Note: It is possible to [enable PHS as a backup authentication with ADFS](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/tutorial-phs-backup). I have at least one conversation a month with customers to assist them with migrating off of ADFS to Password Hash Sync. For the organizations that are only using ADFS for O365 authentication is a huge win. Less infrastructure on-premises that they need to manage and maintain, higher availability for authentication, and it is just one less thing they need to worry about. Even for the organizations using ADFS for other claims, they are moving those over to Azure AD as well for SAML authentication and are having a lot of success with it.

Azure AD  is a global resource that operates out of all 54 Azure regions worldwide with a 99.9% [SLA guarantee](https://azure.microsoft.com/en-us/support/legal/sla/active-directory/v1_0/). Basic and Premium only, sorry no SLA guarantees for Azure AD free users. All writes and reads to Azure AD are distributed across multiple Azure regions. All writes to Azure AD are also copied over to the secondary region. The write attempt does not return successfully until the [secondary region acknowledges it has the data](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-architecture), ensuring data integrity. Additionally, Azure performs all writes on the primary region, all read on the secondary region. This provides a balance of resources and spreads the load out.

You can leverage PHS for [seamless SSO](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-sso), automatically signing users into services when they are on corporate devices. (PTA also supports Seamless SSO, ADFS does not) This leads to limiting how often users need to enter passwords while still providing a secure method of authentication. You can even start leveraging password ban lists to prevent users from [setting common to guess passwords](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-password-ban-bad#custom-banned-password-list).

However, all is not perfect and rosy in the PHS world. Like any technology, there are downsides to be aware of that should be considered before deployment into your environment.
First, on-prem lockout policies and restricted hour login settings do not apply to Azure AD. Password changes sync to Azure AD every two minutes from AD connect.

Additionally, passwords set in Azure AD are set never to expire, allowing users to sign in to Azure AD with passwords that have expired in Active Directory. This means if a user's password expires in Active Directory. And if they have no need to sign into Active Directory again, the Azure AD password will continue working. The Azure AD password would only get updated the next time the password is changed on-premises and then synced back.

Also, if you leverage the 'accountExpires' attribute in Active Directory, this value does not get [synced to Azure AD](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/how-to-connect-password-hash-synchronization#account-expiration). The recommendation for this is a PowerShell workflow or some other automated process to take action in Azure AD based on Active Directory. What I generally see is a PowerShell script that runs hourly against Active Directory querying for expired accounts and then using Set-AzureADUser to match on the on-premises value. Disabled accounts on the other hand, are synced and updated with Azure AD with AD Connect updates.

I hope you found this useful for managing identities in your environment and understanding one of the options available to you.
