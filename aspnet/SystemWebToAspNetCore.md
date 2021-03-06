# Migration from ASP.NET on System.Web to ASP.NET Core
- [Project Structure](#)
- [Application Initialization](#)
- [Request processing](#)
    - [Default Response Headers](#)
    - [Reading Request bodies](#)
    - [Writing Response bodies](#)
    - [Error handling behavior](#)
    - [Header and Querystring manipulation](#)
    - [Request limits](#)
- [Modules](#)
    - [Authentication](#)
    - [Routing](#)
    - [Session](#)
    - [OutputCaching](#)
    - [StaticFiles](#)
- [Configuration](#)

## System.Web
**System.Web.dll** is the assembly where ASP.NET on .NET Framework resides. It's core assembly upon which lots of frameworks build on top of (Webforms, MVC, SignalR, etc).

## Project structure

Here's an empty ASP.NET project targeting .NET Framework 4.7.2:

![image](https://user-images.githubusercontent.com/95136/88470444-37dd3a80-ceb1-11ea-8ba7-6ff688f4886a.png)

### web.config

```xml
<?xml version="1.0" encoding="utf-8"?>

<!--
  For more information on how to configure your ASP.NET application, please visit
  https://go.microsoft.com/fwlink/?LinkId=169433
  -->
<configuration>
  <system.web>
    <compilation debug="true" targetFramework="4.7.2"/>
    <httpRuntime targetFramework="4.7.2"/>
  </system.web>
  <system.codedom>
    <compilers>
      <compiler language="c#;cs;csharp" extension=".cs"
        type="Microsoft.CodeDom.Providers.DotNetCompilerPlatform.CSharpCodeProvider, Microsoft.CodeDom.Providers.DotNetCompilerPlatform, Version=2.0.1.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35"
        warningLevel="4" compilerOptions="/langversion:default /nowarn:1659;1699;1701"/>
      <compiler language="vb;vbs;visualbasic;vbscript" extension=".vb"
        type="Microsoft.CodeDom.Providers.DotNetCompilerPlatform.VBCodeProvider, Microsoft.CodeDom.Providers.DotNetCompilerPlatform, Version=2.0.1.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35"
        warningLevel="4" compilerOptions="/langversion:default /nowarn:41008 /define:_MYTYPE=\&quot;Web\&quot; /optionInfer+"/>
    </compilers>
  </system.codedom>

</configuration>
```

Every System.Web based application is enabled for runtime page compilation by default (for webforms and razor files as an example). This is a subsystem in ASP.NET called the BuildManager and it's a very pluggable and flexiable system. The default templates configure the compilation for debug (so debugging information is emitting during compile time) and specify the target framework of the compiled pages. 

The `httpRuntime` section describes the target framework for the ASP.NET application itself. Since the .NET Framework is updated in place (replaces the older version), we use this flag to control changes in runtime behavior (referred to as quirking) when changes to the defaults are made. For example, if we wanted to change how many headers were added by default, we might decide to toggle this behavior based on the target framework version. This won't affect older applications but it does affect new ones.

This project has no code files but has lots of functionality built into the system wired up via **web.config** and the `C:\Windows\Microsoft.NET\Framework\v4.0.30319\Config\web.config` (also known as root web.config). ASP.NET on .NET Framework runs on IIS and lots of the functionality is built into Windows and .NET Framework. The root web.config wires up default modules and handlers:

### Built-in Modules

<details>

```xml
<httpModules>
    <add name="OutputCache" type="System.Web.Caching.OutputCacheModule"/>
    <add name="Session" type="System.Web.SessionState.SessionStateModule"/>
    <add name="WindowsAuthentication" type="System.Web.Security.WindowsAuthenticationModule"/>
    <add name="FormsAuthentication" type="System.Web.Security.FormsAuthenticationModule"/>
    <add name="PassportAuthentication" type="System.Web.Security.PassportAuthenticationModule"/>
    <add name="RoleManager" type="System.Web.Security.RoleManagerModule"/>
    <add name="UrlAuthorization" type="System.Web.Security.UrlAuthorizationModule"/>
    <add name="FileAuthorization" type="System.Web.Security.FileAuthorizationModule"/>
    <add name="AnonymousIdentification" type="System.Web.Security.AnonymousIdentificationModule"/>
    <add name="Profile" type="System.Web.Profile.ProfileModule"/>
    <add name="ErrorHandlerModule" type="System.Web.Mobile.ErrorHandlerModule, System.Web.Mobile, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a"/>
    <add name="ServiceModel" type="System.ServiceModel.Activation.HttpModule, System.ServiceModel.Activation, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35"/>
    <add name="UrlRoutingModule-4.0" type="System.Web.Routing.UrlRoutingModule"/>
    <add name="ScriptModule-4.0" type="System.Web.Handlers.ScriptModule, System.Web.Extensions, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35"/>
</httpModules>
```
</details>

IIS modules are part of the [integrated pipeline](https://docs.microsoft.com/en-us/iis/application-frameworks/building-and-running-aspnet-applications/how-to-take-advantage-of-the-iis-integrated-pipeline) since IIS 7.0.

### Built-in Handlers

<details>

```xml
<httpHandlers>
    <add path="eurl.axd" verb="*" type="System.Web.HttpNotFoundHandler" validate="True"/>
    <add path="trace.axd" verb="*" type="System.Web.Handlers.TraceHandler" validate="True"/>
    <add path="WebResource.axd" verb="GET" type="System.Web.Handlers.AssemblyResourceLoader" validate="True"/>
    <add verb="*" path="*_AppService.axd" type="System.Web.Script.Services.ScriptHandlerFactory, System.Web.Extensions, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" validate="False"/>
    <add verb="GET,HEAD" path="ScriptResource.axd" type="System.Web.Handlers.ScriptResourceHandler, System.Web.Extensions, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" validate="False"/>
    <add path="*.axd" verb="*" type="System.Web.HttpNotFoundHandler" validate="True"/>
    <add path="*.aspx" verb="*" type="System.Web.UI.PageHandlerFactory" validate="True"/>
    <add path="*.ashx" verb="*" type="System.Web.UI.SimpleHandlerFactory" validate="True"/>
    <add path="*.asmx" verb="*" type="System.Web.Script.Services.ScriptHandlerFactory, System.Web.Extensions, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" validate="False"/>
    <add path="*.rem" verb="*" type="System.Runtime.Remoting.Channels.Http.HttpRemotingHandlerFactory, System.Runtime.Remoting, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" validate="False"/>
    <add path="*.soap" verb="*" type="System.Runtime.Remoting.Channels.Http.HttpRemotingHandlerFactory, System.Runtime.Remoting, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" validate="False"/>
    <add path="*.asax" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.ascx" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.master" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.skin" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.browser" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.sitemap" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.dll.config" verb="GET,HEAD" type="System.Web.StaticFileHandler" validate="True"/>
    <add path="*.exe.config" verb="GET,HEAD" type="System.Web.StaticFileHandler" validate="True"/>
    <add path="*.config" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.cs" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.csproj" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.vb" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.vbproj" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.webinfo" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.licx" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.resx" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.resources" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.mdb" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.vjsproj" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.java" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.jsl" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.ldb" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.ad" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.dd" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.ldd" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.sd" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.cd" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.adprototype" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.lddprototype" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.sdm" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.sdmDocument" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.mdf" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.ldf" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.exclude" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.refresh" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.svc" verb="*" type="System.ServiceModel.Activation.HttpHandler, System.ServiceModel.Activation, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" validate="False"/>
    <add path="*.rules" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.xoml" verb="*" type="System.ServiceModel.Activation.HttpHandler, System.ServiceModel.Activation, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" validate="False"/>
    <add path="*.xamlx" verb="*" type="System.Xaml.Hosting.XamlHttpHandlerFactory, System.Xaml.Hosting, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" validate="False"/>
    <add path="*.aspq" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.cshtm" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.cshtml" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.vbhtm" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*.vbhtml" verb="*" type="System.Web.HttpForbiddenHandler" validate="True"/>
    <add path="*" verb="GET,HEAD,POST" type="System.Web.DefaultHttpHandler" validate="True"/>
    <add path="*" verb="*" type="System.Web.HttpMethodNotAllowedHandler" validate="True"/>
</httpHandlers>
```

</details>

This is why the web.config in your application is so small. By default System.Web based projects inherit the configuration hierarchy from both your project local configuration, other the system wide configuration files that include a massive set of behavior. This is what makes it possible to install ASP.NET, point IIS at a folder with a single aspx file and have it just work without further configuration.

### ASP.NET Core

![image](https://user-images.githubusercontent.com/95136/88473099-f9567880-cece-11ea-8cd2-70d6d6a287aa.png)

In ASP.NET Core, there are 3 files by default. `appsettings.json`, `Program.cs` and `Startup.cs`. One of the main differences between System.Web based applications and ASP.NET Core applications is that most of the things that were configuration based were moved to code. In essence, your `Startup.cs` is where most of the application and framework components are wired up.

**Program.cs**

```C#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace WebApplication1
{
    public class Program
    {
        public static void Main(string[] args)
        {
            CreateHostBuilder(args).Build().Run();
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                });
    }
}
```

Program.cs has a set of defaults that include:
- Default logging providers 
    - Console logging
    - EventSource logging
    - EventLog errors (on Windows)
- Default configuration providers
    - Envrionment variables
    - Json configuration (for appsetttings.json)
- Default Servers
    - IIS support
    - Kestrel support
- Default middleware
    - Hosting filtering

These defaults can be changed via code and configuration. You can also create a host without these defaults and build your own defaults. This is one of the key tenants of ASP.NET Core. The system is flexible enough and built like buffet so you can build your own system without having to replace everything. 

## Application Initialization

The entrypoint to any System.Web based application is Global.asax. Global.asax is an IIS module that runs as part of the integrated pipeline but has a special `Application_Start` event that serves as a place to put your application bootstrapping logic. It's called once per application and runs the very first time a request is made to the application running in IIS.

### Hello World

**Global.asax**

```C#
using System;

namespace WebApplication1
{
    public class Global : System.Web.HttpApplication
    {
        protected void Application_Start(object sender, EventArgs e)
        {
            // Write initialization logic here
        }
        
        protected void Application_BeginRequest(object sender, EventArgs e)
        {
            Context.Response.Write("Hello World");
            Context.ApplicationInstance.CompleteRequest();
        }
    }
}
```

The above logic is a minimal hello world application using **System.Web**. The **Global.asax** file hooks the BeginRequest event (which is detected by naming convention `Application_{eventname}` with the event method signature) and writes the string "Hello World" to the response (learn more about these events in the (Request Processing)[#] section of this document). Context in this case is an `HttpContext`, one of the central types when interacting with the ASP.NET request pipeline (more on this later). The second line of code calls `CompleteRequest` on the `ApplicationInstance`, this is important because it short circuits the IIS pipeline from executing and will make sure no other modules run and change the outcome of this application. This is one of the fundamental behaviors of the System.Web and IIS programming model. There's a pipeline of modules that run in order and a series of events for each of those modules.

### ASP.NET Core

In ASP.NET Core applications, the entry point has a Main. This is important because one of the guiding principles of ASP.NET Core is to act like a normal console application with a different entrypoint. In System.Web based applications, the application bootstrapping is handled by the framework and you get to write code in a different, framework specific entrypoint. This makes your application less portable and tied to this hosting model. ASP.NET Core moves your application entry point to be a normal .NET application entry point with a Main.

**Startup.cs** is where you wire up your application dependencies and configure the request processing pipeline. Here's a simple hello world application.


```C#
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace WebApplication1
{
    public class Startup
    {
        public void Configure(IApplicationBuilder app)
        {
            app.Run(async context =>
            {
                await context.Response.WriteAsync("Hello World!");
            });
        }
    }
}
```

## Request processing

`IHttpModule` and `IHttpHandler` are fundamental primitives for hooking into the request processing pipeline in System.Web based applications. Modules run in order and have a fixed set of stages that are applicable to various parts of the request processing pipeline described [here](https://docs.microsoft.com/en-us/previous-versions/aspnet/bb470252(v=vs.100)). Handlers can be chosen on a per request basic by a module. They are usally chosen based on extension or route.

**Hello World Module**

**MyModule.cs**

```C#
using System.Web;

namespace WebApplication1
{
    public class MyModule : IHttpModule
    {
        public void Init(HttpApplication app)
        {
            app.BeginRequest += (sender, e) =>
            {
                app.Context.Response.Write("Hello World");
                app.CompleteRequest();
            };
        }
        
        public void Dispose()
        {
        }
    }
}
```

Register the module by type in web.config under the `system.webServer\modules` section.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.web>
    <compilation debug="true" targetFramework="4.7.2"/>
    <httpRuntime targetFramework="4.7.2"/>
  </system.web>
  <system.webServer>
    <modules>
      <add name="mymodule" type="WebApplication1.MyModule, WebApplication1" />
    </modules>
  </system.webServer>
</configuration>
```

Modules are constructed by the runtime and can't be passed any state in the constructor. It should also be noted that modules themselves aren't asynchronous but each event handler for the various pipeline stages can be async (albeit with a very old style async API). `IHttpHandler` can be asynchronous and modules can be used to pick the approrate handler:

```C#
using System.Threading.Tasks;
using System.Web;

namespace WebApplication1
{
    public class MyModule : IHttpModule
    {
        public void Init(HttpApplication app)
        {
            app.BeginRequest += (sender, e) =>
            {
                app.Context.RemapHandler(new MyHandler());
            };
        }

        public void Dispose()
        {
        }
    }

    public class MyHandler : HttpTaskAsyncHandler
    {
        public override async Task ProcessRequestAsync(HttpContext context)
        {
            await Task.Delay(100);

            context.Response.Write("Hello World");
        }
    }
}
```

**ASP.NET Core**

The [RequestDelegate](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.requestdelegate?view=aspnetcore-3.1) is the core primitive for request processing in ASP.NET Core. The signaure is an `HttpContext` in with a `Task` returned (if you squint, you'll see it looks alot like `HttpTaskAsyncHandler.ProcessRequestAsync`). ASP.NET Core [Middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-3.1) is the primitive for chaining request processing components together. 

### Behaviors and Built-ins

Before we dive into each of the various components of System.Web we should discuss some of the default behaviors in the system:
- Default Response Headers
- Request Buffering behavior
- Response Buffering behavior
- Error handling behavior
- Header manipulation
- Request limits
  - Timeouts
  - Size limits

### Default Response Headers

**System.Web**

System.Web based application assume the browser is the client. Here's the default set of response headers:

Here's are the response headers:

| **Key**  | **Value**   |
|---|---|
| Cache-Control  | private  |
| Content-Type  | text/html;charset=utf-8 |
| Content-Encoding  | gzip  |
| Vary  | Accept-Encoding  |
| Server  | Microsoft-IIS/10.0  |
| X-AspNet-Version  | 4.0.30319 |
| X-SourceFiles  | =?UTF-8?B?QzpcVXNlcnNcZGF2aWZvd2xcc291... |
| X-Powered-By  |  ASP.NET |
| Date  | Mon, 27 Jul 2020 01:45:55 GMT |
| Content-Length  | 129 |

- The default Content-Type is text/html
- There are default Cache-Control headers
- There's an X-SourceFiles header generated for localhost requests only

**ASP.NET Core**

Writing the same "Hello World" response in ASP.NET Core has the following content:

**Kestrel**

| **Key**  | **Value**   |
|---|---|
| Date  | Mon, 27 Jul 2020 02:30:50 GMT  |
| Transfer-Encoding  | chunked |
| Server  | Kestrel  |

- There's no assumption on Content-Type
- There are no Cache-Control headers
- The response is chunked by default (more on the response buffering behaviro in a section below)

### Reading Request bodies
There are several ways in System.Web based applications to read the raw body:
- [`HttpRequest.InputStream`](https://docs.microsoft.com/en-us/dotnet/api/system.web.httprequest.inputstream?view=netframework-4.8)
- [`HttpRequest.GetBufferlessInputStream()`](https://docs.microsoft.com/en-us/dotnet/api/system.web.httprequest.getbufferlessinputstream?view=netframework-4.8)
- [`HttpRequest.GetBufferedInputStream()`](https://docs.microsoft.com/en-us/dotnet/api/system.web.httprequest.getbufferedinputstream?view=netframework-4.8)

There are various ways to read the input stream, each with their documented caveats.

#### Reading Forms

#### File Uploads

### Writing Response bodies
Responses are buffered in System.Web based applications by default. That has a few implications:
- Responses have a Content-Length header
- Headers can be manipulated until an explicit Flush/FlushAsync is called

**System.Web**

```C#
public class MyHandler : HttpTaskAsyncHandler
{
    public override Task ProcessRequestAsync(HttpContext context)
    {
        context.Response.Headers["x"] = "1";
        context.Response.Write("Hello World");
        context.Response.Headers["x"] = "2";
        return Task.CompletedTask;
    }
}
```

Here's are the response headers:

| **Key**  | **Value**   |
|---|---|
| Cache-Control  | private  |
| Content-Type  | text/html;charset=utf-8 |
| Content-Encoding  | gzip  |
| Vary  | Accept-Encoding  |
| Server  | Microsoft-IIS/10.0  |
| x  | 2  |
| X-AspNet-Version  | 4.0.30319 |
| X-SourceFiles  | =?UTF-8?B?QzpcVXNlcnNcZGF2aWZvd2xcc291... |
| X-Powered-By  |  ASP.NET |
| Date  | Mon, 27 Jul 2020 01:45:55 GMT |
| Content-Length  | 129 |

Notice the x header is set to 2 not one. This means headers are allowed to be modified until the response is flushed.

```C#
public class MyHandler : HttpTaskAsyncHandler
{
    public override Task ProcessRequestAsync(HttpContext context)
    {
        context.Response.Headers["x"] = "1";
        context.Response.Write("Hello World");
        context.Response.Flush();
        context.Response.Headers["x"] = "2";
        return Task.CompletedTask;
    }
}
```

These are the response headers in this case.

| **Key**  | **Value**   |
|---|---|
| Cache-Control  | private  |
| Transfer-Encoding  | chunked |
| Content-Type  | text/html;charset=utf-8 |
| Content-Encoding  | gzip  |
| Vary  | Accept-Encoding  |
| Server  | Microsoft-IIS/10.0  |
| x  | 1  |
| X-AspNet-Version  | 4.0.30319 |
| X-SourceFiles  | =?UTF-8?B?QzpcVXNlcnNcZGF2aWZvd2xcc291... |
| X-Powered-By  |  ASP.NET |
| Date  | Mon, 27 Jul 2020 01:45:55 GMT |


The above example chunkes the response body (see the transfer encoding header in the response) instead of setting a content length. The example also throws a server side exception because headers are mutated after the response goes out:

![image](https://user-images.githubusercontent.com/95136/88474495-c5ce1b00-cedb-11ea-9688-f861b5afe68a.png)

**NOTE: This is a first chance exception so it may not show up in the debugger unless you change your settings.**

Buffering can also be disabled in System.Web based applications by setting HttpResonse.BufferOutput to false.

```C#
public class MyHandler : HttpTaskAsyncHandler
{
    public override Task ProcessRequestAsync(HttpContext context)
    {
        context.Response.BufferOutput = true;
        context.Response.Headers["x"] = "1";
        context.Response.Write("Hello World");
        return Task.CompletedTask;
    }
}
```

| **Key**  | **Value**   |
|---|---|
| Cache-Control  | private  |
| Transfer-Encoding  | chunked |
| Content-Type  | text/html;charset=utf-8 |
| Content-Encoding  | gzip  |
| Vary  | Accept-Encoding  |
| Server  | Microsoft-IIS/10.0  |
| x  | 1  |
| X-AspNet-Version  | 4.0.30319 |
| X-SourceFiles  | =?UTF-8?B?QzpcVXNlcnNcZGF2aWZvd2xcc291... |
| X-Powered-By  |  ASP.NET |
| Date  | Mon, 27 Jul 2020 01:45:55 GMT |

The response is the same as an explicit call to Flush.

**ASP.NET Core**

The equivalent code in ASP.NET Core looks like this:

```C#
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Hosting;

namespace WebApplication1
{
    public class Startup
    {
        public void Configure(IApplicationBuilder app)
        {
            app.Run(async context =>
            {
                context.Response.Headers["x"] = "1";
                await context.Response.WriteAsync("Hello World");
                context.Response.Headers["x"] = "2";
            });
        }
    }
}
```

Responses are **NOT** buffered by default, this means that writing to the response without explicitly setting a content length can result in an exception:

![image](https://user-images.githubusercontent.com/95136/88474565-9ff54600-cedc-11ea-8639-c7601971549b.png)

In order to generate `Content-Length` based responses, the `ContentLength` property needs to be set explicitly before the response is written:

```C#
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Hosting;

namespace WebApplication1
{
    public class Startup
    {
        public void Configure(IApplicationBuilder app)
        {
            app.Run(async context =>
            {
                context.Response.Headers["x"] = "1";
                context.Response.ContentLength = 11;
                await context.Response.WriteAsync("Hello World");
            });
        }
    }
}
```

### Header and QueryString manipulation

In System.Web based applications, headers and query strings are represented as [NameValueCollection](https://docs.microsoft.com/en-us/dotnet/api/system.collections.specialized.namevaluecollection?view=netcore-3.1). This is a keyed collection where the value may be a `string` or `string[]`. In ASP.NET Core the query string is represented as an [IQueryCollection](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.iquerycollection?view=aspnetcore-3.1) and headers are represented as [IHeaderDictionary](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.iheaderdictionary?view=aspnetcore-3.1). These 2 collections are also keyed collections where the values is a [StringValues](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.primitives.stringvalues?view=dotnet-plat-ext-3.1) struct. [StringValues](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.primitives.stringvalues?view=dotnet-plat-ext-3.1) is a stuct that represents a `string` or `string[]` and has implicit converstions as well as other useful operations for working with them.

#### Reading

#### Writing

**Operation**| **System.Web**  | **ASP.NET Core** 
|---|---|---|
|Add | `context.Response.AddHeader("X-MyHeader", "1");` OR `context.Response.Headers.Add("X-MyHeader", "1");` | `context.Response.Headers.Add("X-MyHeader", "1");` |
|Add Multiple | same as Add | `context.Response.Headers.Append("X-MyHeader", "1"); context.Response.Headers.Append("X-MyHeader", "2");` OR `context.Response.Headers.Add("X-MyHeader", new[] { "1", "2" });` |
|Add comma-separated | Not built-in | `context.Response.Headers.AppendCommaSeparatedValues("X-MyHeader", "1", "2");` |
|Replace | `context.Response.Headers["X-MyHeader"] = "1";` OR  `context.Response.Headers.Set("X-MyHeader", "1");` | `context.Response.Headers["X-MyHeader"] = "1";` |
|Delete | `context.Response.Headers.Remove("X-MyHeader");`| `context.Response.Headers.Remove("X-MyHeader");` |


### Request limits

```C#
internal const int DefaultMaxRequestLength = 4096 * 1024;  // 4MB
internal const int DefaultRequestLengthDiskThreshold = 80 * 1024; // 80KB
```

## Routing

TBD
