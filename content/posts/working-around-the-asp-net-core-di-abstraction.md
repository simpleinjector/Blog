---
title:  "Working around the ASP.NET Core DI abstraction"
date:   2016-07-06
author: Steven and Peter
draft:  false
aliases:
    - /2016/07
---

For the last couple of years, Microsoft has been building the latest version of the .NET platform, branded .NET Core. One of the core components of this new framework is a DI library. Unfortunately, Microsoft made the mistake of defining a public abstraction for its DI library. In our [previous blog post](/blog/2016/06/whats-wrong-with-the-asp-net-core-di-abstraction/) we described why the existence of this abstraction leads to all sorts of problems.

The goal of this blog post is to explain how you can effectively limit exposure to this abstraction and instead apply proven practices that promote structure, design and maintainability within your application. The summary of this blog post is the following:

**TLDR;**

> Refrain from using a self-developed or third-party provided adapter for the .NET Core DI abstraction. Isolate the registration of application components from the framework and third-party components. Pursue a SOLID way of working and allow your application registrations to be verified and diagnosed by Simple Injector, without concern for incompatibilities with the framework and third-party components.

Microsoft wants its users to start off using the default container and then replace, if you want, with a third-party DI library. This advice of having one container instance that builds up both framework components, third-party components and application components stems from the idea that it is useful for framework components to be injected into application components. Having a single container makes it easy for the container build up object graphs that are a mixture of application and framework components.

Although developers might find this appealing, it’s important to realize that this a violation of the [Dependency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle) (DIP), which states that:

> abstracts are owned by the upper/policy layers. 

In other words, in order to conform to the DIP, your application code should not depend on framework abstractions. Typically, code that depends on framework abstractions should exist wholly in the [Composition Root](https://freecontent.manning.com/dependency-injection-in-net-2nd-edition-understanding-the-composition-root/). The DIP and [ISP](https://en.wikipedia.org/wiki/Interface_segregation_principle "Interface Segregation Principle") promote the use of abstractions tailored to your application’s needs and the creation of adapter implementations. Instead of having a framework or external library dictate the size and shape of abstractions, the application under development should define what’s best for its particular needs. Not only does this result in clean and testable code, it makes the code more flexible and reusable.

The [SOLID](https://en.wikipedia.org/wiki/SOLID) principles are of great guidance here, and since the DIP states that our application (upper) layer should only depend on its own abstractions, building up mixed object graphs is an anti-pattern. Having one container build up mixed object graphs leads developers to violate the SOLID principles and will undoubtedly cause pain in the long run.

Instead of aiming for one DI library that builds everything up (one container to rule them all), we should keep these two worlds separate: framework components should be built up by the framework’s container, application components should be built up using our container of choice. To integrate or bridge the two worlds we define small focused adapters on each side. Use the framework’s provided extension points to intercept the creation of root types and forward the creation of those types to your container. On the other side of the container divide we define implementations for application-tailored abstractions, which call-back into framework and third-party library code. A well designed framework will have all the necessary abstractions in place for us to intercept. ASP.NET Core MVC already contains all the required hooks. third-party tool developers should follow the same practice.

Some developers feel uncomfortable with the notion of two containers in single application. But if you view the built-in framework container as a configuration system for the framework, having an independent container for your own application components is a non-issue. Every framework has its own configuration system. ASP.NET Web Forms has its own configuration system (mainly XML based) and MVC & Web API have their own code-first configuration systems. In a sense, nothing much has changed; ASP.NET Core still has a configuration system, be it one that includes an internal container-like structure. Apparently this container gives them a lot of flexibility, which is great. But we never had the need to completely swap out a framework’s configuration system before, so why should we need to for ASP.NET Core?

So how does this work? If we don’t want to swap out the built-in configuration system for .NET Core, what should we do? As said before, good practice is to use the framework’s supplied extension points and override as necessary to redirect/intercept the creation of certain types.

The main interception point for ASP.NET Core MVC is the `IControllerActivator` abstraction. This abstraction allows intercepting the creation of MVC controller types. An implementation for Simple Injector is trivial:

{{< highlight csharp >}}
public sealed class SimpleInjectorControllerActivator : IControllerActivator
{
    private readonly Container container;
    public SimpleInjectorControllerActivator(Container c) => container = c;

    public object Create(ControllerContext c) =>
       container.GetInstance(c.ActionDescriptor.ControllerTypeInfo.AsType());

    public void Release(ControllerContext c, object controller) { }
}
{{< / highlight >}}

To replace the built-in controller activator, you configure the Core container:

{{< highlight csharp >}}
services.AddSingleton<IControllerActivator>(
    new SimpleInjectorControllerActivator(container));
{{< / highlight >}}

Although trivial to implement, we do provide an out-of-the-box implementation for you in our [ASP.NET Core MVC integration package](https://www.nuget.org/packages/SimpleInjector.Integration.AspNetCore.Mvc/) to make your life easier. As a matter of fact, over time we will supply you with with all the convenient methods that allow you to make bootstrapping as seamless as possible. We might not provide you with integration packages for all existing frameworks, but plugging in Simple Injector will always be trivial when the designers provided you with the correct interception points.

What this means is that all framework components and third-party components can keep being composed by the built-in DI container and your application will register and resolve your components through Simple Injector.

Many developers incorrectly assume that having one container for the framework’s internal configuration and another for the application components will mean re-registering hundreds of framework and third-party library components in the application container, but this is simply not necessary. First of all, as we already established, those registrations shouldn’t be in the application container because no application component should directly depend on those abstractions. Secondly, your application will only need to interact with a handful of those services at most, so you’ll handle the abstractions you are actually interested in.

## Examples

Let’s say you have a component that needs access to the `HttpContext` instance, because you want to extract the name of the user from the current request being executed. Since the `HttpContext` can be acquired using the `Microsoft.AspNetCore.Http.IHttpContextAccessor` abstraction, your component requires this abstraction as a constructor argument and your code might look something like this:

{{< highlight csharp >}}
public sealed class CustomerRepository : ICustomerRepository
{
    private readonly IUnitOfWork uow;
    private readonly IHttpContextAccessor accessor;

    public CustomerRepository(IUnitOfWork uow, IHttpContextAccessor accessor)
    {
        this.uow = uow;
        this.accessor = accessor;
    }

    public void Save(Customer entity)
    {
        entity.CreatedBy = this.accessor.HttpContext.User.Identity.Name;
        this.uow.Save(entity);
    }
}
{{< / highlight >}}

There are, however, several problems with this approach:

* The component now takes a dependency on an ASP.NET Core MVC abstraction, which makes it impossible to reuse this component outside the context of ASP.NET Core MVC.
* The component has explicit knowledge about how to get the user name for the application.
* The code that gets the user name will likely be duplicated throughout the application.
* The component becomes much harder to test, because of the [train wreck](https://c2.com/cgi/wiki?TrainWreck) in the `Save` method.

One of the main problems is that the `IHttpContextAccessor` abstraction isn’t designed for the specific needs of this component. The needs of this component are not to access the current `HttpContext`, its need is to get the name of the user on whose behalf the code is running. We should create a specific abstraction for that specific need:

{{< highlight csharp >}}
public interface IUserContext
{
    string Name { get; }
}
{{< / highlight >}}

With this abstraction, we can simplify our component to the following:

{{< highlight csharp >}}
public sealed class CustomerRepository : ICustomerRepository
{
    private readonly IUnitOfWork uow;
    private readonly IUserContext userContext;

    public CustomerRepository(IUnitOfWork uow, IUserContext userContext)
    {
        this.uow = uow;
        this.userContext = userContext;
    }

    public void Save(Customer entity)
    {
        entity.CreatedBy = userContext.Name;
        uow.Save(entity);
    }
}
{{< / highlight >}}

What we have achieved here is that we:

* Decoupled our component from the framework code; it can be reused outside of ASP.NET.
* Prevented this component to have explicit knowledge about how to retrieve the current user’s name.
* Prevented this code from being duplicated throughout the application.
* Reduced test complexity.
* Made the code simpler.

Since we have decoupled our component from the framework code, we can now reuse the component. For instance, it’s quite common to want to run part of our code base in a background Windows Service where there is obviously no HttpContext. To make this work we will create an adapter implementation for IUserContext that is specific to the type of application we are building. For our ASP.NET application, we will need an adapter implementation that contains the original code that retrieves the user’s name. For a Windows Service, we might return the name of the system user.

Here’s our adapter implementation for ASP.NET:

{{< highlight csharp >}}
public sealed class AspNetUserContext : IUserContext
{   
    private readonly IHttpContextAccessor accessor;
    public AspNetUserContext(IHttpContextAccessor a) => accessor = a;
    public string Name => accessor.HttpContext.Context.User.Identity.Name;
}
{{< / highlight >}}

As you can see, this adapter implementation is straightforward, all it does is getting the `HttpContext` for the current request and the user name is determined from the context, as we saw before.

This component can be registered in our application container as follows:

{{< highlight csharp >}}
var accessor =
    app.ApplicationServices.GetRequiredService<IHttpContextAccessor>();
container.RegisterSingleton<IUserContext>(new AspNetUserContext(accessor));
{{< / highlight >}}

The app variable here is ASP.NET Core’s `IApplicationBuilder`abstraction that gets injected into the `Startup.Configure` method.

What we see here is that our `AspNetUserContext` adapter depends directly on the `IHttpContextAccessor` abstraction. We can do this because `IHttpContextAccessor` is one of the framework’s abstractions that we know for sure is registered as a singleton. For most framework and third-party services however, we will have no idea what lifestyle it is registered with, and therefore, resolving them directly using the `ApplicationServices` property of [IApplicationBuilder](https://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Builder/IApplicationBuilder/) is a pretty bad idea.

Due to [another design flaw](https://simpleinjector.org/decisions#dont-allow-resolving-outside-an-active-scope), ASP.NET Core allows resolving scoped instances through the `ApplicationServices` property, but returns those components as singletons! In other words, if we were to request any framework and third-party services through `ApplicationServices`, the chances are that we would get a stale instance that would break our application at runtime—and ASP.NET Core will not inform us of our mistake. Instead of throwing an exception the ASP.NET Core will fail silently and leave our application in a potentially invalid state, maybe causing an `ObjectDisposedException` or worse. This is actually yet another incompatibility with Simple Injector; Simple Injector blocks these types of invalid resolves by throwing an exception.

UPDATE: ASP.NET Core 2.0 mitigates the resolution of scoped instances from the root container, when running in development mode. However, it will still not detect the resolution of any disposable transients from the root container. This will lead to memory leaks.

Instead of using the `ApplicationServices` property, it would be better to resolve services using the `HttpContext.RequestServices` property. The following adapter shows an example when dealing with framework dependencies with a lifestyle that is either not singleton or unknown:

{{< highlight csharp >}}
public sealed class AspNetAuthorizerAdapter : IAuthorizer
{
    private readonly Func<IAuthorizationService> provider;

    public AspNetAuthorizerAdapter(Func<IAuthorizationService> provider)
    {
        this.provider = provider;
    }

    // Implementation here
}
{{< / highlight >}}

Here we have an adapter for a hypothetical `IAuthorizer` abstraction. Instead of depending on ASP.NET’s `IAuthorizationService` directly, this adapter depends on `Func<IAuthorizationService>`, which allows the correctly scoped service to be resolved at runtime. This adapter can be registered as follows:

{{< highlight csharp >}}
container.RegisterSingleton(
    new AspNetAuthorizerAdapter(
        GetAspNetServiceProvider<IAuthorizationService>(app)));
{{< / highlight >}}

The `AspNetAuthorizationAdapter` is created and registered as singleton. The registration makes use of the convenient `GetAspNetServicesProvider<T>` helper method that allows creating the provider delegate:

{{< highlight csharp >}}
private static Func<T> GetAspNetServiceProvider<T>(IApplicationBuilder app)
{
    var accessor =
        app.ApplicationServices.GetRequiredService<IHttpContextAccessor>();
    return () =>
    {
        var context = accessor.HttpContext
            ?? new InvalidOperationException("No HttpContext");
        return context.RequestServices.GetRequiredService<T>();
    };
}
{{< / highlight >}}

When supplied with an `IApplicationBuilder` instance, the `GetAspNetServiceProvider` method will create a `Func<T>` that allows resolving the given service type `T ` from the `RequestServices` collection according to its proper scope.

**NOTE:** With the introduction of ASP.NET Core integration package for Simple Injector v4, we added `GetRequestService<T>()` and `GetRequiredRequestService<T>()` extension methods on `IApplicationBuilder` that allow retrieving request services just like the previous `GetAspNetServiceProvider<T>()` method does. With the introduction of v4.1, we added the notion of [auto crosswiring](https://simpleinjector.org/aspnetcore#cross-wiring-asp-net-and-third-party-services), which simplifies this process even more.

## Using logging in your application

Besides a DI library, .NET Core ships with a logging library out-of-the-box. Any application developer can use the built-in logger abstraction directly in their applications. But should they? If you look at the [`ILogger` abstraction](https://github.com/aspnet/Logging/blob/1.0.0/src/Microsoft.Extensions.Logging.Abstractions/ILogger.cs) supplied to us by Microsoft, it’s hard to deny that the abstraction is very generic in nature and might very well not suit your application’s specific needs. The previous arguments still hold: application code should be in control over the abstraction.

The SOLID principles guide us defining an application-specific abstraction for logging. The exact shape of this abstraction will obviously differ from application to application (but look at [this example](https://stackoverflow.com/questions/5646820/logger-wrapper-best-practice) for inspiration). Again, a simple adapter implementation can do the transition from application code to framework code:

{{< highlight csharp >}}
// your application's logging abstraction
public interface ILog { void Log(LogEntry e); }

public sealed class DotNetCoreLoggerAdapter : ILog
{
    private readonly Microsoft.Extensions.Logging.ILogger logger;
    public DotNetCoreLoggerAdapter(ILogger logger) => this.logger = logger;

    public void Log(LogEntry e) => 
        logger.Log(ToLevel(e.Severity), 0, e.Message, e.Exception,
            (s, _) => s);

    private static LogLevel ToLevel(LoggingEventType s) =>
        s == LoggingEventType.Warning ? LogLevel.Warning :
        s == LoggingEventType.Error ? LogLevel.Error :
        LogLevel.Critical;
}
{{< / highlight >}}

This `DotNetCoreLoggerAdapter` can be registered as singleton as follows:

{{< highlight csharp >}}
container.RegisterSingleton<ILog>(
    new DotNetCoreLoggerAdapter(loggerFactory.CreateLogger("Application")));
{{< / highlight >}}

But there are [other options](https://stackoverflow.com/a/41244169/264697) when it comes to integrating logging with Simple Injector.

## Conclusion

By creating application specific abstractions, we prevent our code from taking unnecessary dependencies on external code, making them more flexible, testable and maintainable. We can define simple adapter implementations for the abstractions we need to use, while hiding the details of connecting to external code. This allows our application to use our container of choice (and supports a [container-less](https://blog.ploeh.dk/2014/06/10/pure-di/) approach). This approach is part of a set of principles and practices that is been taught by experts like Robert C. Martin and others for decades already. Don’t ignore these practices, embrace them and be both productive and successful.