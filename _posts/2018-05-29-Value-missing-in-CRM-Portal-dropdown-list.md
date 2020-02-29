---
layout:     post
title:      Value missing in CRM Portal dropdown list
subtitle:   
date:       2018-05-29
author:     alphac
header-img: 
catalog: true
tags: CRM Portal
---

In CRM Portal, we are able to make a Lookup field on an entity form render as a dropdown list, just add an entity form metadata. For example, in an OOB CSS Portal, we add the **Source Campaign (campaignid)** lookup field to the **contact us** page *(an Lead entity form)*. In the **contact us** entity form, add an entity form metadata record for the **Source Campaign (campaignid)** and make sure the *Control Style* is *Render Lookup as Dropdown*.

![](https://alphacland.blob.core.windows.net/landimages/001.png)

Then we can see the lookup field show as a dropdown list.

![](https://alphacland.blob.core.windows.net/landimages/002.png)

However, after recent Portal release (about around v8.4.2.19), the values of the dropdown list don't show, like an empty list. But if we change the dropdown back the lookup, the values show again in the pop up dialog.

![](https://alphacland.blob.core.windows.net/landimages/003.png)

This is because there is a design change (should be a security enhancement), we need to provide additional **Append** permission for the Lookup field related entity.

For above example, as the **Source Campaign (campaignid)** is a Lead entity attribute and contains the relationship between Lead entity and Campaign entity. So please go to CRM > Portals > Entity Permissions, create a new entity permission record, 

Name: *"any name you like"*; Entity Name: *campaign*; Website: *"the portal website"*; Scope: *Global*;

***Privileges*** section, make sure the **Append** is checked.

Scroll down to the *Web Roles* section, you can add the web roles you want this kind of Portal users able to see the values in the dropdown list. Here I choose **Anonymous Users**, as **contact us** page can be accessed anonymously. PS: if we only choose **Anonymous Users** web role, after we sign in the portal, we are unable to see the values in the dropdown.

![](https://alphacland.blob.core.windows.net/landimages/004.png)

Then, save the record, clear portal cache, we can see the values in the dropdown list show again. Cheers!!

* content
{:toc}