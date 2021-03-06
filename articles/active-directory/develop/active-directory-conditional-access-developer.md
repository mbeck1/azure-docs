---
title: Developer Guidance for Azure Active Directory Conditional Access
description: Developer guidance and scenarios for Azure AD conditional access
services: active-directory
keywords: 
author: danieldobalian
manager: mtillman
editor: PatAltimore
ms.author: dadobali
ms.date: 07/19/2017
ms.service: active-directory
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: identity
---

# Developer Guidance for Azure Active Directory Conditional Access

Azure Active Directory (AD) offers several ways to secure your app and protect a service.  One of these unique features is conditional access.  Conditional access enables developers and enterprise customers to protect services in a multitude of ways including:

* Multi-factor authentication
* Allowing only Intune enrolled devices to access specific services
* Restricting user locations and IP ranges

For more information on the full capabilities of conditional access, see [Conditional access in Azure Active Directory](../active-directory-conditional-access-azure-portal.md). 

In this article, we focus on what conditional access means to developers building apps for Azure AD.  It assumes knowledge of [single](active-directory-integrating-applications.md) and [multi-tenant](active-directory-devhowto-multi-tenant-overview.md) apps and [common authentication patterns](active-directory-authentication-scenarios.md).

We'll dive into the impact of accessing resources you don't have control over that may have conditional access policies applied.  Moreover, we explore the implications of conditional access in the on-behalf-of flow, web apps, accessing Microsoft Graph, and calling APIs.

## How does conditional access impact an app?

### App types impacted

In most common cases, conditional access does not change an app's behavior or requires any changes from the developer.  Only in certain cases when an app indirectly or silently requests a token for a service, an app requires code changes to handle conditional access "challenges".  It may be as simple as performing an interactive sign-in request. 

Specifically, the following scenarios require code to handle conditional access "challenges": 

* Apps accessing Microsoft Graph
* Apps performing the on-behalf-of flow
* Apps accessing multiple services/resources
* Single page apps using ADAL.js
* Web Apps calling a resource

Conditional access policies can be applied to the app, but also can be applied to a web API your app accesses. To learn more about how to configure a conditional access policy, please see [Getting started with Azure Active Directory Conditional Access](../active-directory-conditional-access-azure-portal-get-started.md).

Depending on the scenario, an enterprise customer can apply and remove conditional access policies at any time.  In order for your app to continue functioning when a new policy is applied, you need to implement the "challenge" handling. The following examples illustrate challenge handling. 

### Conditional access examples

Some scenarios require code changes to handle conditional access whereas others work as is.  Here are a few scenarios using conditional access to do multi-factor authentication that gives some insight into the difference.

* You are building a single-tenant iOS app and apply a conditional access policy.  The app signs in a user and doesn't request access to an API.  When the user signs in, the policy is automatically invoked and the user needs to perform multi-factor authentication (MFA). 
* You are building a multi-tenant web app that uses Microsoft Graph to access Exchange, among other services.  An enterprise customer who adopts this app sets a policy on Exchange.  When the web app requests a token for MS Graph, the app will not be challenged to comply with the policy.  The end-user is signed in with valid tokens. When the app attempts to use this token against the Microsoft Graph to access Exchange data, a claims "challenge" is returned to the web app through the ```WWW-Authenticate``` header.  The app can then use the ```claims``` in a new request, and the end user will be prompted to comply with the conditions. 
* You are building a native app that uses a middle tier service to access a downstream API.  An enterprise customer at the company using this app applies a policy to the downstream API.  When an end-user signs in, the native app requests access to the middle tier and sends the token.  The middle tier performs on-behalf-of flow to request access to the downstream API.  At this point, a claims "challenge" is presented to the middle tier. The middle tier sends the challenge back to the native app, which needs to comply with the conditional access policy.

### Complying with a conditional access policy

For several different app topologies, a conditional access policy is evaluated when the session is established.  As a conditional access policy operates on the granularity of apps and services, the point at which it is invoked depends heavily on the scenario you're trying to accomplish.

When your app attempts to access a service with a conditional access policy, it may encounter a conditional access challenge.  This challenge is encoded in the `claims` parameter that comes in a response from Azure AD or Microsoft Graph.  Here's an example of this challenge parameter: 

```
claims={"access_token":{"polids":{"essential":true,"Values":["<GUID>"]}}}
```

Developers can take this challenge and append it onto a new request to Azure AD.  Passing this state prompts the end-user to perform any action necessary to comply with the conditional access policy. In the following scenarios, specifics of the error and how to extract the parameter are explained. 

## Scenarios

### Prerequisites

Azure AD conditional access is a feature included in [Azure AD Premium](../active-directory-whatis.md#choose-an-edition).  You can learn more about licensing requirements in the [unlicensed usage report](../active-directory-conditional-access-unlicensed-usage-report.md).  Developers can join the [Microsoft Developer Network](https://msdn.microsoft.com/dn308572.aspx), which includes a free subscription to the Enterprise Mobility Suite which includes Azure AD Premium.

### Considerations for specific scenarios

The following information only applies in these conditional access scenarios:

* Apps accessing Microsoft Graph
* Apps performing the on-behalf-of flow
* Apps accessing multiple services/resources
* Single page apps using ADAL.js

In the following sections, we'll get into common scenarios in which are more complex.  The core operating principle is conditional access policies are evaluated at the time the token is requested for the service that has a conditional access policy applied unless it's being accessed through Microsoft Graph.

## Scenario: App accessing Microsoft Graph

In this scenario, we walk through the case when a web app requests access to Microsoft Graph. The conditional access policy in this case could be assigned to SharePoint, Exchange, or some other service that is accessed as a workload through Microsoft Graph.  In this example, let's assume there's a conditional access policy on Sharepoint Online.

![App accessing Microsoft Graph flow diagram](media/active-directory-conditional-access-developer/app-accessing-microsoft-graph-scenario.png)

The app first requests authorization to Microsoft Graph which requires accessing a downstream workload without conditional access.  The request succeeds without invoking any policy and the app receives tokens for Microsoft Graph.  At this point, the app may use the access token in a bearer request for the endpoint requested. Now, the app needs to access a Sharepoint Online endpoint of Microsoft Graph, for example: `https://graph.microsoft.com/v1.0/me/mySite`

The app already has a valid token for Microsoft Graph, so it can perform the new request without being issued a new token. This request fails and a claims challenge is issued from Microsoft Graph in the form of an HTTP 403 Forbidden with a ```WWW-Authenticate``` challenge.
Here's an example of the response: 

```
HTTP 403; Forbidden 
error=insufficient_claims
www-authenticate="Bearer realm="", authorization_uri="https://login.windows.net/common/oauth2/authorize", client_id="<GUID>", error=insufficient_claims, claims={"access_token":{"polids":{"essential":true,"values":["<GUID>"]}}}"
```

The claims challenge is inside the ```WWW-Authenticate``` header, which can be parsed to extract the claims parameter for the next request.  Once it's appended to the new request, Azure AD knows to evaluate the conditional access policy when signing in the user and the app is now in compliance with the conditional access policy.  Repeating the request to the Sharepoint Online endpoint succeeds.

The ```WWW-Authenticate``` header does have a unique structure and is not trivial to parse in order to extract values.  Here's a short method to help.

```csharp
        /// <summary>
        /// This method extracts the claims value from the 403 error response from MS Graph. 
        /// </summary>
        /// <param name="wwwAuthHeader"></param>
        /// <returns>Value of the claims entry. This should be considered an opaque string. 
        /// Returns null if the wwwAuthheader does not contain the claims value. </returns>
        private String extractClaims(String wwwAuthHeader)
        {
            String ClaimsKey = "claims=";
            String ClaimsSubstring = "";
            if (wwwAuthHeader.Contains(ClaimsKey))
            {
                int Index = wwwAuthHeader.IndexOf(ClaimsKey);
                ClaimsSubstring = wwwAuthHeader.Substring(Index, wwwAuthHeader.Length - Index);
                string ClaimsChallenge;
                if (Regex.Match(ClaimsSubstring, @"}$").Success)
                {
                    ClaimsChallenge = ClaimsSubstring.Split('=')[1];
                }
                else
                {
                    ClaimsChallenge = ClaimsSubstring.Substring(0, ClaimsSubstring.IndexOf("},") + 1);
                }
                return ClaimsChallenge;
            }
            return null; 
        }
```

For code samples that demonstrate how to handle the claims challenge, refer to the [On-behalf-of code sample](https://github.com/Azure-Samples/active-directory-dotnet-webapi-onbehalfof-ca) for ADAL .NET.

## Scenario: App performing the on-behalf-of flow

In this scenario, we walk through the case in which a native app calls a web service/API.  In turn, this service does [the "on-behalf-of" flow](active-directory-authentication-scenarios.md#application-types-and-scenarios) to call a downstream service.  In our case, we've applied our conditional access policy to the downstream service (Web API 2) and are using a native app rather than a server/daemon app. 

![App performing the on-behalf-of flow diagram](media/active-directory-conditional-access-developer/app-performing-on-behalf-of-scenario.png)

The initial token request for Web API 1 does not prompt the end-user for multi-factor authentication as Web API 1 may not always hit the downstream API.  Once Web API 1 tries to request a token on-behalf-of the user for Web API 2, the request fails since the user has not signed in with multi-factor authentication.

Azure AD returns an HTTP response with some interesting data: 

> [!NOTE]
> In this instance it's a multi-factor authentication error description, but there's a wide range of `interaction_required` possible pertaining to conditional access.  

```
HTTP 400; Bad Request 
error=interaction_required
error_description=AADSTS50076: Due to a configuration change made by your administrator, or because you moved to a new location, you must use multi-factor authentication to access '<Web API 2 App/Client ID>'.
claims={"access_token":{"polids":{"essential":true,"Values":["<GUID>"]}}}
```

In our Web API 1, we catch the error `error=interaction_required`, and send back the `claims` challenge to the desktop app.  At that point, the desktop app can make a new `acquireToken()` call and append the `claims`challenge as an extra query string parameter.  This new request requires the user to do multi-factor authentication and then send this new token back to Web API 1 and complete the on-behalf-of flow.

To try out this scenario, see our [.NET code sample](https://github.com/Azure-Samples/active-directory-dotnet-webapi-onbehalfof-ca).  It demonstrates how to pass the claims challenge back from Web API 1 to the native app and construct a new request inside the client app. 

## Scenario: App accessing multiple services

In this scenario, we walk through the case in which a web app accesses two services one of which has a conditional access policy assigned.  Depending on your app logic, there may exist a path in which your app does not require access to both web services.  In this scenario, the order in which you request a token plays an important role in the end-user experience.

Let's assume we have web service A and B and web service B has our conditional access policy applied.  While the initial interactive auth request requires consent for both services, the conditional access policy is not required in all cases.  If the app requests a token for web service B, then the policy is invoked and subsequent requests for web service A also succeeds as follows.

![App accessing multiple-services flow diagram](media/active-directory-conditional-access-developer/app-accessing-multiple-services-scenario.png)

Alternatively, if the app initially requests a token for web service A, the end-user does not invoke the conditional access policy.  This allows the app developer to control the end-user experience and not force the conditional access policy to be invoked in all cases. The tricky case is if the app subsequently requests a token for web service B. At this point, the end-user needs to comply with the conditional access policy.  When the app tries to `acquireToken`, it may generate the following error (illustrated in the following diagram): 

```
HTTP 400; Bad Request
error=interaction_required
error_description=AADSTS50076: Due to a configuration change made by your administrator, or because you moved to a new location, you must use multi-factor authentication to access '<Web API App/Client ID>'.
claims={"access_token":{"polids":{"essential":true,"Values":["<GUID>"]}}}
``` 

![App accessing multiple services requesting a new token](media/active-directory-conditional-access-developer/app-accessing-multiple-services-new-token.png)

If the app is using the ADAL library, a failure to acquire the token is always retried interactively.  When this interactive request occurs, the end-user has the opportunity to comply with the conditional access.  This is true unless the request is a `AcquireTokenSilentAsync` or `PromptBehavior.Never` in which case the app needs to perform an interactive ```AcquireToken``` request to give the end use the opportunity to comply with the policy. 

## Scenario: Single Page App (SPA) using ADAL.js

In this scenario, we walk through the case when we have a single page app (SPA), using ADAL.js to call a conditional access protected web API.  This is a simple architecture but has some nuances that need to be taken into account when developing around conditional access.

In ADAL.js, there are a few functions that obtain tokens: `login()`, `acquireToken(...)`, `acquireTokenPopup(…)`, and `acquireTokenRedirect(…)`. 

* `login()` obtains an ID token through an interactive sign-in request but does not obtain access tokens for any service (including a conditional access protected web API).  
* `acquireToken(…)` can then be used to silently obtain an access token meaning it does not show UI in any circumstance.  
* `acquireTokenPopup(…)` and `acquireTokenRedirect(…)` are both used to interactively request a token for a resource meaning they always show sign-in UI.

When an app needs an access token to call a Web API, it attempts an `acquireToken(…)`.  If the token session is expired or we need to comply with a conditional access policy, then the *acquireToken* function fails and the app uses `acquireTokenPopup()` or `acquireTokenRedirect()`.

![Single page app using ADAL flow diagram](media/active-directory-conditional-access-developer/spa-using-adal-scenario.png)

Let's walk through an example with our conditional access scenario.  The end-user just landed on the site and doesn’t have a session.  We perform a `login()` call, get an ID token without multi-factor authentication.  Then the user hits a button that requires the app to request data from a web API.  The app tries to do an `acquireToken()` call but fails since the user has not performed multi-factor authentication yet and needs to comply with the conditional access policy.

Azure AD sends back the following HTTP response: 

```
HTTP 400; Bad Request 
error=interaction_required
error_description=AADSTS50076: Due to a configuration change made by your administrator, or because you moved to a new location, you must use multi-factor authentication to access '<Web API App/Client ID>'.
```

Our app needs to catch the `error=interaction_required`.  The application can then use either `acquireTokenPopup()` or `acquireTokenRedirect()` on the same resource.  The user is forced to do a multi-factor authentication. After the user completes the multi-factor authentication, the app is issued a fresh access token for the requested resource.

To try out this scenario, see our [JS SPA On-behalf-of code sample](https://github.com/Azure-Samples/active-directory-dotnet-webapi-onbehalfof-ca).  This code sample uses the conditional access policy and web API you registered earlier with a JS SPA to demonstrate this scenario. It shows how to properly handle the claims challenge and get an access token that can be used for your Web API. Alternatively, checkout the general [Angular.js code sample](https://github.com/Azure-Samples/active-directory-angularjs-singlepageapp) for guidance on an Angular SPA


## See also

* To learn more about the capabilities, see [Conditional Access in Azure Active Directory](../active-directory-conditional-access-azure-portal.md).
* For more Azure AD code samples, see [Github Repo of Code Samples](https://github.com/azure-samples?utf8=%E2%9C%93&q=active-directory). 
* For more info on the ADAL SDK's and access the reference documentation, see [library guide](active-directory-authentication-libraries.md).
* To learn more about multi-tenant scenarios, see [How to sign in users using the multi-tenant pattern](active-directory-devhowto-multi-tenant-overview.md).
