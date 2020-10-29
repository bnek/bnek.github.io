---
layout: post
title:  "Say Goodbye to the OAuth implicit flow with MSAL 2"
date:   2020-09-17 16:16:06 +1000
categories: code
tags: azure msal 2.x authentication authorisation code flow pkce
---
TL;DR If you want to skip the blurbs, just go straight to [steps to reproduce](#steps-to-reproduce) or checkout the [code on GitHub](https://github.com/bnek/examples/tree/master/web/msal-2-auth-code-flow-spa-aspnet-core-31).

## Authentication in SPAs using OAuth 2 authorisation code flow with MSAL.js 2.x and consuming a .NET Core 3.1 API

This post provides the steps to secure your ASP.NET core API using OAuth2, authenticating a Single Page App against Azure AD using Microsoft Authentication Library for JavaScript (MSAL.js) 2.x and the authorisation code flow with PKCE and calling the API with an access token.

## MSAL 1.x, Implicit Flow and Security Considerations

Authentication in single page apps using v1.x involves the use of the [implicit flow](https://tools.ietf.org/html/rfc6749#section-1.3.2). The OAuth security best practices have been updated and advise [against using the implicit flow](https://tools.ietf.org/html/draft-ietf-oauth-security-topics-15#section-2.1.2) due to a few vulnerabilities most of which result from the fact that the access token is part of the return URL (and therefore could be intercepted before arriving in the app it is intended for).

Instead, the authorisation code flow is recommended.


## MSAL 2, Authorisation Code Flow and Azure AD
Using MSAL.js 2.x, SPAs can now authenticate against Azure AD using the authorisation code flow (Azure AD B2C was not supported at the time this blog was written). Let's get started...

## Steps to reproduce
* Prerequisites
  * An app registration in AzureAD, set up for authentication with the platform 'Single-page Application' [as described here](https://docs.microsoft.com/en-gb/azure/active-directory/develop/scenario-spa-app-registration).
* What I used:
  * Windows 10 
  * Visual Studio 2019
  * dotnet sdk v3.1.401
* **[Commit 1](https://github.com/bnek/examples/commit/c5d411fecad47802208289a215cc9700d1fb99e1)**: Creating an ASP.NET Core 3.1 web app with React
  * File->New project
  * ASP.NET Core Web Application (name it i.e. "Msal2") -> Create
  * Choose .NET Core 3.1, React.js (the auth code won't be React-specific)
  * Choose 'No Authentication'
  * Create (the app will be scaffolded)
* **[Commit 2](https://github.com/bnek/examples/commit/7eab6bcc62f5da318b213b096a0e23a50d5f8857)**: Let's modify `Home.js` to fetch and display today's weather forecast. Authetication is not yet enabled in the API so we can get away with an unauthenticated call - [here is the link to the complete Home.js file](https://github.com/bnek/examples/blob/7492989c0241789285e9d041ab3e37c0a8e8116f/web/msal-2-auth-code-flow-spa-aspnet-core-31/Msal2/ClientApp/src/components/Home.js) for this step. The App will now display tomorrow's weather.
  ```jsx
  //...
    fetchWeather() {
      fetch("/weatherforecast").then(response => {
        response.json().then(result => {
          this.setState(result);
        })
      });
    }
  //...
  ```
* **[Commit 3](https://github.com/bnek/examples/commit/b77263c3885302ae7b56f814943b73e42e3e6dbd)**: Secure the API by adding middleware to handle OpenID Connect Bearer token.
  * Install nuget package `Microsoft.AspNetCore.Authentication.JwtBearer`
  * Add the following lines to the beginning of the `Startup.ConfigureServices` method:

    ```c#
    services.AddAuthentication(options =>
    {
      options.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(jwtOptions =>
    {
      var instance = Configuration["AzureAd:Instance"];
      var domain = Configuration["AzureAd:Domain"];

      jwtOptions.Authority = $"{instance}/{domain}/v2.0/";
      jwtOptions.Audience = Configuration["AzureAd:ClientId"];
    });
    ```
  * Add the following lines to the `Startup.Configure` method to configure the pipeline to use the auth middleware:
  ```c#
    app.UseAuthentication();
    app.UseAuthorization();
  ```
  * Open `appsettings.json` and replace the values for `TenantId`, `ClientId` and `Domain` with valid values from your Azure AD app registration.
  * Finally, we add the `[Authorize]` attribute to the weather forecast endpoint.
  * Now the `fetch('/weatherforecast')` call will result in a 401 (Unauthorized) response due to a missing valid JWT.
* **[Commit 4](https://github.com/bnek/examples/commit/c8d55d47f76ef0a942b910e1ccadd01180f4da08)**: Finally, we add the package `@azure/msal-browser` to our client code and acquire an access token from Azure AD that we then use to call the API.
  * Install msal-browser by typing `npm install --save @azure/msal-browser`
  * The call to `myMSALObj.loginRedirect(loginRequest)` will request an authorisation code (`response_type: code`) which then will be used to request an access token.
  * We then use that access token to call the API:
  ```js
  // ...
    auth.getToken().then(accessToken => {
      fetch("/weatherforecast", { headers: { Authorization: `Bearer ${accessToken}` } }).then(response => {
        response.json().then(result => {
          this.setState(result);
        })
      });
    });
  // ...
  ```
 
## Conclusion
MSAL.js 2.x supports the authorisation code flow with PKCE as opposed to MSAL.js 1.x (and ADAL.js), which uses the implicit flow. With the updates OAuth security best practices advising [against using the implicit flow](https://tools.ietf.org/html/draft-ietf-oauth-security-topics-15#section-2.1.2) it's a good idea to choose MSAL.js 2.x over version 1.x for new projects and update existing projects to use the most recent version.