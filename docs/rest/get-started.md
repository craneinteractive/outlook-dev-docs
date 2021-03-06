---
title: Get Started with the Outlook REST APIs
description: Learn how to use Microsoft Graph via REST requests and responses to access the Outlook API.
author: jasonjoh

ms.topic: article
ms.technology: ms-graph
ms.devlang: rest-api
ms.date: 04/26/2017
ms.author: jasonjoh
---
# Overview of using the Outlook REST APIs

> [!TIP]
> Try out sample REST calls in the [Graph Explorer](https://developer.microsoft.com/graph/graph-explorer). You can use your own account, or one of our test accounts. Once you're done exploring the API, come back here and select your favorite platform on the left. We'll guide you through the steps to write a simple application to retrieve messages from your inbox.
>
> If your preferred platform isn't listed yet, continue reading on this page. We'll go through the same set of steps using raw HTTP requests.

The purpose of this guide is to walk through the process of calling the [Outlook Mail API](/graph/api/resources/message?view=graph-rest-1.0) to retrieve messages in Office 365 and Outlook.com. Unlike the platform-specific getting started guides, this guide focuses on the OAuth and REST requests and responses. It will cover the sequence of requests and responses that an app uses to authenticate and retrieve messages.

This tutorial will use [Microsoft Graph](/graph/overview) to call the Mail API. Microsoft recommends using Microsoft Graph to access Outlook mail, calendar, and contacts. You should use the Outlook APIs directly (via `https://outlook.office.com/api`) only if you require a feature that is not available on the Graph endpoints.

With the information in this guide, you can implement this in any language or platform capable of sending HTTP requests.

## Use OAuth2 to authenticate

In order to call the Mail API, the app requires an access token from Azure Active Directory. In order to do that, the app implements one of the supported OAuth flows in the [Azure v2.0 endpoint](/azure/active-directory/develop/active-directory-appmodel-v2-overview). However, before this will work, the app must be registered in the Application Registration Portal.

### Registering an app

> [!NOTE]
> This example scenario will use the [authorization code flow](/azure/active-directory/develop/active-directory-v2-protocols-oauth-code). The steps for the [implicit flow](/azure/active-directory/develop/active-directory-v2-protocols-implicit) or [client credentials flow](/azure/active-directory/develop/active-directory-v2-protocols-oauth-client-creds) will be slightly different.

You can use the [Application Registration Portal](https://apps.dev.microsoft.com/) to quickly register an app that uses any of the Outlook APIs.

[!include[App Registration Intro](~/includes/rest/app-registration-intro.md)]

Once you register the app, you will have a client ID and secret. These are used in the authorization code flow.

### Getting an authorization code

The first step in the authorization code flow is to get an authorization code. That code is returned to the app by Azure when the user logs on and consents to the level of access the app requires.

First the app constructs a logon URL for the user. This URL must be opened in a browser so that the user can login and provide consent.

The base URL for logon looks like:

```http
https://login.microsoftonline.com/common/oauth2/v2.0/authorize
```

The app appends query parameters to this base URL to let Azure know what app is requesting the logon, and what permissions it is requesting.

- `client_id`: the client ID generated by registering the app. This lets Azure know which app is requesting the logon.
- `redirect_uri`: the location that Azure will redirect to once the user has granted consent to the app. This value must correspond to the value of **Redirect URI** used when registering the app.
- `response_type`: the type of response the app is expecting. For the Authorization Grant Flow, this should always be `code`.
- `scope`: a space-delimited list of access scopes that your app requires. For a full list of Outlook scopes in Microsoft Graph, see [Microsoft Graph permission scopes](/graph/permissions-reference).

For example, an application that requires read access to mail would put all of those together to generate a request URL like the following:

#### Authorization code request

```http
GET https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=<CLIENT ID>&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F&response_type=code&scope=openid+Mail.Read
```

The user will be presented with a sign in screen that displays the name of the app. Once they sign in, if it is their first time using the app, the user will be presented with a list of the app permissions the app requires and asked to allow or deny. Assuming they allow the required access, the browser will be redirected to the redirect URI specified in the initial request.

#### Redirect request after successful sign in

```http
GET  HTTP/1.1 302 Found
Location: http://localhost/myapp/?code= AwABAAAA...cZZ6IgAA&session_state=7B29111D-C220-4263-99AB-6F6E135D75EF&state=D79E5777-702E-4260-9A62-37F75FF22CCE
```

The value of the `code` query parameter in the URL is the authorization code. The next step is to exchange that code for an access token.

### Getting an access token

To get an access token, the app posts form-encoded parameters to the token request URL, which is always:

```http
https://login.microsoftonline.com/common/oauth2/v2.0/token
```

The app includes the following parameters.

- `client_id`: the client ID generated by registering the app.
- `client_secret`: the client secret generated by registering the app.
- `code`: the authorization code obtained in the prior step.
- `redirect_uri`: this value must be the same as the value used in the authorization code request.
- `grant_type`: the type of grant the app is using. For the Authorization Grant Flow, this should always be `authorization_code`.

These parameters are encoded to the `application/x-www-form-urlencoded` content type and sent to the token request URL.

#### Access token request

```http
POST https://login.microsoftonline.com/common/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=AwABAAAA...cZZ6IgAA&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F&client_id=<CLIENT ID>&client_secret=<CLIENT SECRET>
```

Azure responds with a JSON payload which includes the access token.

#### Access token response

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{
  "token_type":"Bearer",
  "expires_in":"3600",
  "access_token":"eyJ0eXAi...b66LoPVA",
  "scope":"Mail.Read",
}
```

The access token is found in the `access_token` field of the JSON payload. The app uses this value to set the `Authorization` header when making REST calls to the API.

## Calling the Mail API

Once the app has an access token, it's ready to call the Mail API. The [Mail API Reference](/graph/api/resources/message?view=graph-rest-1.0) has all of the details. Since the app is retrieving messages, it will use an HTTP GET request to the `https://graph.microsoft.com/v1.0/me/mailfolders/inbox/messages` URL. This will retrieve messages from the inbox.

### Refining the request

Apps can control the behavior of GET requests by using [OData query parameters](/graph/query-parameters). It is recommended that apps use these parameters to limit the number of results that are returned and to limit the fields that are returned for each item. Let's look at an example.

Consider an app that displays messages in a table. The table only displays the subject, sender, and the date and time the message was received. The table displays a maximum of 25 rows, and should be sorted so that the most recently received message is at the top.

To achieve this, the app uses the following query parameters:

- The `$select` parameter is used to specify only the `subject`, `from`, and `receivedDateTime` fields.
- The `$top` parameter is used to specify a maximum of 25 items.
- The `$orderby` parameter is used to sort the results by the `receivedDateTime` field.

This results in the following request.

#### Mail API request for messages in the inbox

```http
GET https://graph.microsoft.com/v1.0/me/mailfolders/inbox/messages?$select=subject,from,receivedDateTime&$top=25&$orderby=receivedDateTime%20DESC

Accept: application/json
Authorization: Bearer eyJ0eXAi...b66LoPVA
X-AnchorMailbox: jason@contoso.onmicrosoft.com
```

#### Mail API Response

```http
HTTP/1.1 200 OK
Content-Type: application/json;odata.metadata=minimal;odata.streaming=true;IEEE754Compatible=false;charset=utf-8

{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users(...)/mailfolders('inbox')messages(subject,from,receivedDateTime)",
  "value": [
    {
      "@odata.etag": "W/\"CQAAABYAAAAoPBSqxXQOT6tuE0pxCMrtAABufX4i\"",
      "id": "AAMkADRmMDExYzhjLWYyNGMtNDZmMC1iZDU4LTRkMjk4YTdjMjU5OABGAAAAAABp4MZ-5xP3TJnNAPmjsRslBwAoPBSqxXQOT6tuE0pxCMrtAAAAAAEMAAAoPBSqxXQOT6tuE0pxCMrtAABufW1UAAA=",
      "subject": "Ruby on Rails tutorial",
      "from": {
        "emailAddress": {
          "address": "jason@contoso.onmicrosoft.com",
          "name": "Jason Johnston"
        }
      },
      "receivedDateTime": "2015-01-29T20:44:53Z"
    },
    {
      "@odata.etag": "W/\"CQAAABYAAAAoPBSqxXQOT6tuE0pxCMrtAABSzmz4\"",
      "id": "AAMkADRmMDExYzhjLWYyNGMtNDZmMC1iZDU4LTRkMjk4YTdjMjU5OABGAAAAAABp4MZ-5xP3TJnNAPmjsRslBwAoPBSqxXQOT6tuE0pxCMrtAAAAAAEMAAAoPBSqxXQOT6tuE0pxCMrtAABMirSeAAA=",
      "subject": "Trip Information",
      "from": {
        "emailAddress": {
          "address": "jason@contoso.onmicrosoft.com",
          "name": "Jason Johnston"
        }
      },
      "receivedDateTime": "2014-12-09T21:55:41Z"
    },
    {
      "@odata.etag": "W/\"CQAAABYAAAAoPBSqxXQOT6tuE0pxCMrtAABzxiLG\"",
      "id": "AAMkADRmMDExYzhjLWYyNGMtNDZmMC1iZDU4LTRkMjk4YTdjMjU5OABGAAAAAABp4MZ-5xP3TJnNAPmjsRslBwAoPBSqxXQOT6tuE0pxCMrtAAAAAAEMAAAoPBSqxXQOT6tuE0pxCMrtAABAblZoAAA=",
      "subject": "Multiple attachments",
      "from": {
        "emailAddress": {
          "address": "jason@contoso.onmicrosoft.com",
          "name": "Jason Johnston"
        }
      },
      "receivedDateTime": "2014-11-19T20:35:59Z"
    },
    {
      "@odata.etag": "W/\"CQAAABYAAAAoPBSqxXQOT6tuE0pxCMrtAAA9yBBa\"",
      "id": "AAMkADRmMDExYzhjLWYyNGMtNDZmMC1iZDU4LTRkMjk4YTdjMjU5OABGAAAAAABp4MZ-5xP3TJnNAPmjsRslBwAoPBSqxXQOT6tuE0pxCMrtAAAAAAEMAAAoPBSqxXQOT6tuE0pxCMrtAAA9x_8YAAA=",
      "subject": "Attachments",
      "from": {
        "emailAddress": {
          "address": "jason@contoso.onmicrosoft.com",
          "name": "Jason Johnston"
        }
      },
      "receivedDateTime": "2014-11-18T20:38:43Z"
    }
  ]
}
```

Now that you've seen how to make calls to the Mail API, you can use the API reference to construct any other kinds of calls your app needs to make. However, bear in mind that your app needs to have the appropriate permissions configured on the app registration for the calls it makes.
