# Migrating from ASP.NET MVC 5.x to ASP.NET Core MVC

This article will walk you through several of the major changes from ASP.NET MVC to ASP.NET Core MVC. Because applications vary greatly in scope, feature usage, and dependencies, this article is not a step-by-step migration guide, but rather a source of information that will help you migrate your application.

Definitions:

* ASP.NET MVC, also known as System.Web.Mvc and Microsoft.AspNet.Mvc, is a web framework that runs on .NET Framework. It shipped as a set of NuGet packages.
* ASP.NET Core MVC is a web framework that runs on .NET Core. (Older versions also ran on .NET Framework.) It first shipped as NuGet packages, and later as part of the ASP.NET Core runtime.

Both ASP.NET MVC and ASP.NET Core MVC make heavy use of their underlying platforms - ASP.NET (System.Web) and ASP.NET Core, respectively. For that reason, this guide will focus on features of MVC itself and platform features that are used closely with MVC. Other topics that are more general in the underlying platform are covered by other articles in this set [TODO: Link].

## Project configuration

ASP.NET MVC projects depend on the `Microsoft.AspNet.Mvc` NuGet package as well as several other supporting packages. ASP.NET Core MVC is contained almost entirely in the ASP.NET Core runtime and does not use NuGet packages (since version X?).

These are the common MVC-related NuGet packages used in ASP.NET MVC projects and their replacements in ASP.NET Core MVC:

* `Microsoft.AspNet.Mvc` - N/A. Replaced by the ASP.NET Core runtime.
* `Microsoft.AspNet.Razor` - N/A. Replaced by the ASP.NET Core runtime.
* `Microsoft.AspNet.Web.Optimization` - No exact replacement. CSS and JS bundling and optimization is typically done using 3rd party systems.
* `Microsoft.AspNet.WebPages` - N/A. Replaced by the ASP.NET Core runtime.
* `Microsoft.CodeDom.Providers.DotNetCompilerPlatform` - N/A. Replaced by the ASP.NET Core runtime.
* `Microsoft.jQuery.Unobtrusive.Validation` - No exact substitute. TODO: What's the recommendation?
* `Microsoft.Web.Infrastructure` - N/A. Replaced by the ASP.NET Core runtime.
* `Newtonsoft.Json` - While most JSON functionality is now built-in, to use Json.NET in ASP.NET Core MVC you can add a reference to the `Microsoft.AspNetCore.Mvc.NewtonsoftJson` NuGet package.

## MVC app setup

The configuration of an ASP.NET MVC app is typically defined in these places:

* `web.config`?
* `global.asax`?
* Various startup code files, typically in the `~/App_Start` folder of the application

## Defining routes

* Conventional routes
* Attribute routes

## Filters

* Filters
* Global filters

## Using DI

Using DI in:

* Models
* Views
* Controllers
* 3rd party DI

## Model binding and validation

* Model binding
* Validation
* Client side validation

## Controllers

And action results

## Views

* Views
* Razor
* HTML helpers
* No more ASPX/WebFormsViewEngine

## Child actions

## Areas

## MVC request lifecycle

* https://docs.microsoft.com/en-us/aspnet/mvc/overview/getting-started/lifecycle-of-an-aspnet-mvc-5-application/_static/lifecycle-of-an-aspnet-mvc-5-application1.pdf

## Auth

## Async actions

Everything allows async, no support for legacy async

## Testing

## Data access
