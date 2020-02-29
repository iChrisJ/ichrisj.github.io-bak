---
layout:     post
title:      Portal External Authentication - ADFS
subtitle:   Seamless sign in Portal with ADFS
date:       2018-06-04
author:     alphac
header-img: 
catalog: true
tags: CRM Portal ADFS
---

Start from CRM Portal v8.0, all the Portal instances are online deployed (except one open source Portal). Portal also provide multiple entries from user to register/sign in it. That means domain users are also to sign in Portal. However, sometimes it's annoying that every time we need to type our credentials to sign in the Portal even we are in the intranet. Whether we can implement seamless sign in Portal under an intranet?

Here is one way: ADFS (WS-Federation)

To implement this, we need to add make some configurations on two parts: ADFS server and the Portal associated CRM org.
> No include Employee Self Service Portal as it is only configured Azure AD to login

## ADFS Server

In order to configure ADFS with Portal online, please make sure the ADFS is accessible from internet. That means we are able to access <https://adfs.contoso.com/FederationMetadata/2007-06/FederationMetadata.xml> and <https://adfs.contoso.com/adfs/ls/idpinitiatedsignon.aspx>.

After that, let's add some settings to AD FS Management tool. Refer to https://docs.microsoft.com/en-us/dynamics365/customer-engagement/portals/configure-ws-federation-settings#create-an-ad-fs-relying-party-trust

### Create an AD FS relying party trust

Using the AD FS Management tool, go to **Trust Relationships** &gt; **Relying Party Trusts**.

1.  Select **Add Relying Party Trust**.
2.  Welcome: Select **Start**.
3.  Select Data Source: Select **Enter data about the relying party manually**, and then select **Next**.
4.  Specify Display Name: Enter a **name**, and then select **Next**.
    Example: https://portal.contoso.com/
5.  Choose Profile: Select **AD FS 2.0 profile**, and then select **Next**.
6.  Configure Certificate: Select **Next**.
7.  Configure URL: Select the **Enable support for the WS-Federation Passive protocol** check box.

Relying party WS-Federation Passive protocol URL: Enter https://portal.contoso.com/signin-federation

- Note: AD FS requires that the portal run on **HTTPS**.

    > The resulting endpoint has the following settings:              
    > Endpoint type: **WS-Federation**                                 
    > -   Binding: **POST**                                            
    > -   Index: n/a (0)                                               
    > -   URL: **https://portal.contoso.com/signin-federation** 

8.  Configure Identities: Specify https://portal.contoso.com/, select **Add**, and then select **Next**.
    If applicable, more identities can be added for each additional relying party portal. Users will be able to authenticate across any or all of the available identities.
9.  Choose Issuance Authorization Rules: Select **Permit all users to access this relying party**, and then select **Next**.
10.  Ready to Add Trust: Select **Next**.
11.  Select **Close**.

Add the **Name ID** claim to the relying party trust:

**TransformWindows account name** to **Name ID** claim (Transform an Incoming Claim):
- Incoming claim type: Windows account name
- Outgoing claim type: Name ID
- Outgoing name ID format: Unspecified
- Pass through all claim values

## CRM Org
After the configurations on AD FS, we can add some Portal related settings on CRM.

### Create AD FS related site settings

Go to CRM > Portals > Site Settings, apply portal site settings referencing the above AD FS Relying Party Trust. Refer to https://docs.microsoft.com/en-us/dynamics365/customer-engagement/portals/configure-ws-federation-settings#create-ad-fs-site-settings

A standard AD FS (STS) configuration only uses the following settings (with example values): 
- Authentication/WsFederation/ADFS/MetadataAddress - https://adfs.contoso.com/FederationMetadata/2007-06/FederationMetadata.xml
- Authentication/WsFederation/ADFS/AuthenticationType - http://adfs.contoso.com/adfs/services/trust
  - Use the value of the **entityID** attribute in the root element of the Federation Metadata (open the **MetadataAddress URL** in a browser that is the value of the above site setting)
- Authentication/WsFederation/ADFS/Wtrealm - https://portal.contoso.com/
- Authentication/WsFederation/ADFS/Wreply - https://portal.contoso.com/signin-federation
  
### Auto navigate to sign in page

In order to automatically navigate to the sign in after typing the Portal URL, we need to create an "Web Page Access Control Rule" record.

| Name      | Value                                                        |
| :-------- | :----------------------------------------------------------- |
| Name      | A title used for reference. eg: "Grant Access to Authenticated Users" |
| Website   | Portal website                                               |
| Web Page  | Home                                                         |
| Right     | Restrict Read                                                |
| Scope     | Exclude direct child web files                               |
| Web Roles | Add "Authenticated Users" web role                           |

### Set ADFS as default log in entry

Add/update this site setting to make Portal set AD FS as default sign in channel. When we navigate to sign in page, it will automatically navigate the AD FS sign page, just like the ADFS button is clicked.

- Authentication/Registration/LoginButtonAuthenticationType - http://adfs.contoso.com/adfs/services/trust

### Addtional

1. We need to add the AD FS url http://adfs.contoso.com as Local Intranet website in IE

    > To locate the **Local intranet** dialog box in **Internet Explorer**, click **Tools**, click **Internet Options**, click **Security**, and then click **Local intranet**.

2. In AD FS Management tool, make sure the "Windows Authentication" is ticked for Intranet.
    > to find "Windows Authentication" in the AD FS Management tool, go to **Authentication Policies** &gt;**Primary Authentication** &gt; **Global Setting** &gt; Edit &gt; Intranet section

 