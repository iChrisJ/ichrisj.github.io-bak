---
layout:     post
title:      Azure AD Authentication with CRM
date:       2019-05-20
author:     alphac
header-img: 
catalog: true
tags: Dynamics CRM
---

Lots of scenarios we want to be authenticated in Dynamics CRM without using CRM SDK, especially in programming applications to fetch data from CRM. In that case, we need to set up Azure AD authentication with Dynamics CRM.

## Set up Azure AD authentication

### App Registration

Go to https://portal.azure.com and log on as Global Administrator, navigate to **Azure Active Directory** -> **App registrations** -> click **New registration** to register an application.

- *Name*: this is the display name of the appliction, any name is OK.
- *Supported account types*: choose who can use the application, I choose the first one - "Accounts in this organizational directory only (<- account name ->)"
- *Redirect URL (optional)*: choose "Web" and set "https://localhost:1234". We can update it later if needed.

On the following app overview page, copy and store the *Application (client) ID*, we will use it later.
![clientid](https://alphacland.blob.core.windows.net/landimages/clientid.jpg)

### Create Client Secret

On the Application page, move to **Certificate & secrets** tab, click **New client secret** to create a client secret and copy/store the value, we will use it later as well.

### Add CRM API Permission

Navigate to **API permissions**, click **Add a permission** button and choose **Dynamics CRM** tile, toggle the "user_impersonation", click "Add permission", then click **Grant admin consent for xxxx**
![apipermission](https://alphacland.blob.core.windows.net/landimages/apipermission.jpg)


### Create an Application User

Go to your CRM orgainzation, navigate to Settings -> Security -> Users, switch the View to ***Application Users***, click *New* ribbon button, switch the form to ***Application User***, fill out the "User Name", "Full Name" and "Primary Email" with anything you want. Fill out the "Application ID" with *Application (client) ID* we copied previously and save the record, then the "Application ID URI" and "Azure AD Object ID" will be populated.

## Use it in a Console Application 

Let's use the basic Web Request to acquire the token and fetch data from CRM.

{% highlight C# %}
using System;
using System.IO;
using System.Net;
using System.ServiceModel;
using Microsoft.Xrm.Sdk;
using Newtonsoft.Json;

namespace DemoConsole
{
	class Program
	{
		public static void Main(string[] args)
		{
			try
			{
				//set these values for your D365 instance, user credentials and Azure AD clientid/token endpoint
				string crmorg = "https://<--org name-->.crm5.dynamics.com";
				string clientid = "<-- Application (client) ID -->";
				string username = "<-- username -->";
				string userpassword = "<-- password -->";
				string tokenendpoint = "https://login.microsoftonline.com/<-- domain name or tanent id -->/oauth2/token";
				string clientsecret = "<-- client secret -->";

				//relative path to web api endpoint
				string crmwebapi = "/api/data/v9.1";

				//web api query to execute - in this case all accounts that start with "F"
				string crmwebapipath = "/contacts";

				//instantiate a new entity collection to hold the records we'll return later
				EntityCollection results = new EntityCollection();

				//build the authorization request for Azure AD
				var reqstring = "client_id=" + clientid;
				reqstring += "&resource=" + Uri.EscapeUriString(crmorg);
				reqstring += "&username=" + Uri.EscapeUriString(username);
				reqstring += "&password=" + Uri.EscapeUriString(userpassword);
				reqstring += "&client_secret=" + clientsecret;
				reqstring += "&grant_type=password";
				//make the Azure AD authentication request
				HttpWebRequest req = (HttpWebRequest)WebRequest.Create(tokenendpoint);
				req.ContentType = "application/x-www-form-urlencoded";
				req.Method = "POST";
				req.Headers.Add("Cache-Control", "no-cache");

				using (var streamWriter = new StreamWriter(req.GetRequestStream()))
				{
					streamWriter.Write(reqstring);
					streamWriter.Flush();
					streamWriter.Close();
				}

				HttpWebResponse resp = (HttpWebResponse)req.GetResponse();
				StreamReader tokenreader = new StreamReader(resp.GetResponseStream());
				string responseBody = tokenreader.ReadToEnd();
				tokenreader.Close();

				//deserialize the Azure AD token response and get the access token to supply with the web api query
				var tokenresponse = JsonConvert.DeserializeObject<Newtonsoft.Json.Linq.JObject>(responseBody);
				var token = tokenresponse["access_token"];

				//make the web api query
				WebRequest crmreq = WebRequest.Create(crmorg + crmwebapi + crmwebapipath);
				crmreq.Headers = new WebHeaderCollection();

				//use the access token from earlier as the authorization header bearer value
				crmreq.Headers.Add("Authorization", "Bearer " + token);
				crmreq.Headers.Add("OData-MaxVersion", "4.0");
				crmreq.Headers.Add("OData-Version", "4.0");
				crmreq.Headers.Add("Prefer", "odata.maxpagesize=500");
				crmreq.Headers.Add("Prefer", "odata.include-annotations=OData.Community.Display.V1.FormattedValue");
				crmreq.ContentType = "application/json; charset=utf-8";
				crmreq.Method = "GET";

				HttpWebResponse crmresp = (HttpWebResponse)crmreq.GetResponse();
				StreamReader crmreader = new StreamReader(crmresp.GetResponseStream());
				string crmresponseBody = crmreader.ReadToEnd();
				crmreader.Close();

				//deserialize the response
				var crmresponseobj = JsonConvert.DeserializeObject<Newtonsoft.Json.Linq.JObject>(crmresponseBody);
                // We can use crmresponeObj to get the data we want.
			}
			catch (FaultException<OrganizationServiceFault> ex)
			{
				string message = ex.Message;
				throw;
			}
		}
	}
}
{% endhighlight %}

## Use PostMan to Fetch CRM Data

Now let's try to use PostMan to get data from CRM organization.

1. Open a new tab in PostMan and set value of URL eg: GET https://<-orgname->.api.crm5.dynamics.com/api/data/v9.1/contacts 
2. Select the *Authorization* tab, set the Type to *OAuth 2.0* and click on "Get New Access Token" button 
![postmanauth](https://alphacland.blob.core.windows.net/landimages/postmanauth.jpg)
3. Set The following values in the Get New Access Token dialog
 - Token Name: <Any Name is OK>
 - Grant Type: Authorization Code
 - Callback URL: http://localhost:1234 (This is the *Redirect URL (optional)* we set in creating app registration step. )
 - Auth URL: https://login.windows.net/common/oauth2/authorize?resource=<URL Encoded Url to Web API Endpoint> eg: https://login.windows.net/common/oauth2/authorize?resource=https%3A%2F%2Fcontoso.crm5.dynamics.com%2F
 - Access Token URL: https://login.microsoftonline.com/common/oauth2/token
 - Client ID: *Application (client) ID* we copy and stored previously.
 - Client Secret: client secret we created and stored previously. 
![new access token](https://alphacland.blob.core.windows.net/landimages/newtoken.jpg)

 - Click "Request Token" to get the token. Once get the token, scroll down to click "Use Token"
4. Click "Send" to send the request to restrieve data from CRM.

## Reference

https://danishnaglekar.wordpress.com/2019/03/10/azure-ad-authentication-with-dynamics-crm/
https://alexanderdevelopment.net/post/2018/05/28/using-dynamics-365-virtual-entities-to-show-data-from-an-external-organization/

* content
{:toc}
