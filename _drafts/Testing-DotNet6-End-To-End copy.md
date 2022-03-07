---
layout: post
tags: [.net, e2e, tests, xUnit, .net 6, .net identity]
title: .NET 6 Identity and E2E Functional Tets. Am I doing it right? 
author: emir_osmanoski
comments: true
image: /images/2022-02-28-Identity-And-E2E-Testing/01_ListLogo.png
cover-img: /images/2022-02-28-Identity-And-E2E-Testing/00_Dotnet6Logo.png
share-img: /images/2022-02-28-Identity-And-E2E-Testing/01_ListLogo.png
meta-description: .NET 6 Identity and E2E Testing.
---

# Intro

Recently I've been re-exploring some specific aspects of .NET 6 through a small
side-project as a way to get back into the latest and greatest of the .NET
world.

The side-project is a simple API Application based on the Ardalis Clean
Architecture template. It's meant to be a simple Marketplace where Users - named
Merchants can post/sell and buy or show interest for specific products.
Currently the "products" are used games which is why some of the code we will
look at references a Game entity.

More importantly the template used comes with a couple of interesting concepts:

   - API Single Endpoint Handlers
      - Each API Endpoint for a given resource (POST, GET, PUT, DELETE) has it's
        own handler that only contains code related to handling that request.
   - Onion/Clean Architecture  ( Kernel < Core < Infrastructure < API )
      - Defines a code organization approach where layer dependencies always go
        towards the Core/Kernel which aims to be Technology/Framework/Library
        agnostic.
   - Repository Pattern with Specifications
      - A generic repository that handles "Specifications", classes that express
        queries.
   - Mediator, through `[MediatR](https://github.com/jbogard/MediatR)` for
     Business Events

It has some DDD flavour in how things are organized and it was very easy to kick
things off by adding Domain entities related to the specific needs of the
application I had in mind.

What id did not have, was any Authentication and Authorization support.

One way to go about introducing that was to start off with the documentation in
the official .NET Guides and read through the explanations and samples.

Another recommendation was to reference the setup in another Microsoft Template
& Reference Project, namely `E-Shop on Web`, which is also somewhat based on the
same template: `Ardalis Clean Architecture`. 

In the next section we'll combine both to get an initial better grasp of what we
need for the side-project setup.

# .NET 6 ASP Identity Review

> From this point on we will reference `E-Shop on Web` as ESW. The following
> sections will contain code from both ESW and the side-project.

The starting point for this article, beyond ESW, is going to be introductory
documents from the official guides:

- [ASP.NET Core 6.0 Security and
  Identity](https://docs.microsoft.com/en-us/aspnet/core/security/?view=aspnetcore-6.0)
  - Starting Page for all Security and Identity concerns including
    Authentication.
- [Authentication
  Overview](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/?view=aspnetcore-6.0)
  - Overview of the Authentication specific features.

The basic concepts here are that **Authentication** is handled by an out of the
box implementation of the `IAuthenticationService` interface. This is invoked
through middleware we can register in our Startup.cs/Program.cs files.

This service internally uses one of multiple `AuthenticationSchemes`. An
authentication scheme is a combination of a class that implements
`IAuthenticationHandler` and a combination of options
(`AuthenticationSchemeOptions`) associated with that handler.

The usage of the `IAuthenticateService` and any `Schemes` can be configured in
the Startup.cs/Program.cs files through out of the box extension methods or
through more direct setup. 

For example, the following piece of code does exactly that by setting up the
JSONWebToken Scheme:

``` csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options => Configuration.Bind("JwtSettings", options));
```

It is possible to have multiple schemes configured.  If we do configure multiple
schemes we can use `Authorization Policies` or `Attributes` to specify what
scheme to specifically use authenticate the user. 

In the previous example we also setup the `JwtBearer` Scheme as the default by
passing the `JwtBearerDefaults.AuthenticationScheme` in the `AddAuthentication`
method.

> Note that the JWT Handler, plainly, checks that the requests contain an
> encoded token that contains the information for the user trying to perform the
> request. What we've registered here does not deal with creating any Tokens or
> verifying usernames or passwords.

So far we've discussed setting up the services needed for Authentication. To
wrap this initial intro section we can also mention that we need to register the
Middleware that will use the IAuthenticationService and whatever Scheme we have
configured. We can do this by setting up the proper middleware:

```
app.UseAuthentication();
```

These are the basics of how Authentication works. Middleware that users
Services(IAuthenticationService) and Scheme/Handlers (which are on their own
Injectable services) to verify that the requests that we are processing trying
to access a specific endpoint contain the needed Headers or Cookies or whatever
data to be allowed access. 

This allows us to add the `[Authorize]` attribute on Controller endpoints or
full Controllers or globally to forbid the access to the resources without an
Authenticated User. Setting this up properly

So, now the question becomes how do we track Users and Roles?
This is where **ASP.NET Core Identity** comes into play.

## ASP.NET Core Identity

ASP.NET Core Identity is best described in the documentation:

- [Introduction to Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-6.0&tabs=visual-studio)
- Is an **API that supports user interface (UI) login functionality**.
- Manages users, passwords, profile data, roles, claims, tokens, email
  confirmation, and more.
- Identity is typically configured using a SQL Server database to store user
  names, passwords, and profile data. Alternatively, another persistent store
  can be used, for example, Azure Table Storage.

While exploring all of this I ran into different approaches to Identity. One of
the things that kept popping up were The Microsoft Identity Platform and
Identity Server 4.

All of these frameworks can be used to authenticate users, but we will focus on
ASP.NET Core Identity for now.

As a short summary of each of these and their differences we can reference the
following page in the documentation:

- [Introduction to
  Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-6.0&tabs=visual-studio)

The mentioned document page present **ASP.NET Core Identity ** as a set of
features that are described best with the following key points:

- Is an API that supports user interface (UI) login functionality.
- Manages users, passwords, profile data, roles, claims, tokens, email
  confirmation, and more.
- Identity is typically configured using a SQL Server database to store user
  names, passwords, and profile data. Alternatively, another persistent store
  can be used, for example, Azure Table Storage.

We then have** Microsoft Identity Platform** described as:

- An evolution of the Azure Active Directory (Azure AD) developer platform.
- Unrelated to ASP.NET Core Identity.

As ASP.NET Core Identity is described as something to be used where we have a UI
that can be used for logging, the docs recommend we use the following to secure
Web APIs and SPAs:

- Azure Active Directory
- Azure Active Directory B2C (Azure AD B2C)
- IdentityServer4 (became Duende Identity Server)

Finally from the above we come to IdentityServer4 (Duende Identity Server)
described as an OpenID Connect and OAuith2.0 Framework with the following
features:

- Authentication as a Service (AaaS)
- Single sign-on/off (SSO) over multiple application types
- Access control for APIs
- Federation Gateway

**So what is the deal with all of this?**

## Reference project setup

ESW Authentication is implemented through several aspects in the project:

- Program.cs File contains the needed Identity Setup that stores User
  Information
  - This includes the IdentityDbContext setup.
  - Noting here that the ESW App uses a separate database to Store User Identity
    information, which is how I also approached it in my application, with a
    slight modification to the claims stored on the token to reference business
    entities.
  - There is a way to use a single Context for both the Identity and the
    Application tables, but for now we will use this "default" approach.
- Program.cs File Contains the needed Service Registration and Middleware setup
  for JWT Authentication
- The ITokenClaimsService and IdentityTokenClaimService implementation used to
  create the JWT Token from an Identity User.
- The Authenticate Endpoint that takes in the Username and Password for a User
  and uses the Identity classes SignInManager and UserManager as well as the
  ITokenClaimsService to create the token.

These are some of the key important concepts around .NET 6 Identity so let us go
into some of these in a bit more detail.

### Identity Setup and Db Context

The first thing we are going to look at is how we will store the User and Role
information in our application. When Users are registered and created we will
store their information in the database, which will also include any role
information.

The code that configures this can be found in the Program.cs file in the ESW:

``` csharp
builder.Services.AddDbContext<AppIdentityDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("IdentityConnection")));

builder.Services.AddIdentity<ApplicationUser, IdentityRole>()
    .AddEntityFrameworkStores<AppIdentityDbContext>()
    .AddDefaultTokenProviders();
```
