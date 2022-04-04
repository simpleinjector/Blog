---
title:	"Simple Injector v3 is here!"
date:	2015-08-18
author: Steven
draft:	false
aliases:
    - /2015/08
---

After months of preparation and development we have finally released [Simple Injector v3.0](https://www.nuget.org/packages/SimpleInjector/). In version 3 we are breaking from the past: we have removed legacy methods, simplified parts of the API and added some compelling new features.

_**We expect that almost every developer will have to make changes to their composition root when upgrading to v3.** We did our best to make the upgrade process easy but please be prepared to make changes to your code._

The driver for making these breaking changes is that parts of the API have evolved over time and in doing so have grown confusing (e.g. `RegisterOpenGeneric` and `RegisterManyForOpenGeneric`). In the pursuit of keeping Simple Injector simple we felt obligated to improve the consistency of the API. These decisions have not been taken lightly because we hate breaking your code. Our driving force is, however, a simpler and more compelling library for everyone.

Our goal was to let the API guide you as much as possible through the breaking changes and how to fix them. In most cases removed parts of the API still exist, but are marked with `[Obsolete(error: true)]` attribute with expressive messages that explain what to do instead. This will cause your compiler to show a compilation error with (hopefully) a clear message describing the action to take. This should make it easier for you to migrate from v2.x to v3.0.

***Before you upgrade to v3.0, please make sure you upgrade to the latest 2.8 version of Simple Injector first.*** Some beta testers reported that there were some changes between minor versions of the 2.x branch that broke code and/or unit tests. Upgrading in two steps should make the process much easier.

Besides the clean-up of the API, Simple Injector is now much stricter when it comes to diagnosing your configuration. When calling `Verify()`, Simple Injector 3 will automatically run its diagnostics and it will throw an exception when any diagnostic error occurs. Even without calling `Verify()`, Simple Injector will always check for pesky [Lifestyle Mismatches](https://simpleinjector.org/dialm) when resolving instances from the container and will throw an exception when such a mismatch is detected. These exceptions are intended to provide the quickest feedback on configuration mistakes that we believe you should resolve. From experience we know that this can save you from wasting many hours debugging problems later.

## Breaking Changes

The most prominent breaking changes are the changes to the public API and these can prevent your code from compiling.

Here is a cheat sheet containing a mapping for the most prominent API changes. On the left side are the old v2 API calls, on the right side the related method to use in v3. *Note that in most cases the compiler errors will guide you through the process.*

| v2 API                     | v3 API                                 |
| -------------------------- | -------------------------------------- |
| RegisterSingle             | RegisterSingleton                      |
| RegisterAll                | RegisterCollection                     |
| RegisterSingleDecorator    | RegisterDecorator(Lifestyle.Singleton) |
| RegisterAllOpenGeneric     | RegisterCollection                     |
| RegisterManyForOpenGeneric | Register / RegisterCollection          |
| RegisterOpenGeneric        | Register                               |
| RegisterSingleOpenGeneric  | Register(Lifestyle.Singleton)          |

For a complete list of all the breaking changes, please see the [release notes](https://github.com/simpleinjector/SimpleInjector/releases/tag/v3.0).

## New Features

As well as the breaking changes there are many other big and small improvements to the library. The most prominent of these are:

* the addition of a `Lifestyle.Scoped` property;
* support for conditional and contextual registrations using the new `RegisterConditional` methods.
* `Register` overloads now accept open generic types.
* `RegisterCollection(Type, IEnumerable<Registration>)` now accepts open generic types.
* `Container.Register(Type, IEnumerable<Assembly>)` and `Container.RegisterCollection(Type, IEnumerable<Assembly>)` overloads have been added to simplify batch registration.
* the `Container` class now implements `IDisposable` to allows disposing singletons.

The new `Lifestyle.Scoped` is a small feature that can make your [Composition Root](https://freecontent.manning.com/dependency-injection-in-net-2nd-edition-understanding-the-composition-root/) much cleaner. Most applications use a combination of three lifestyles: `Transient`, `Singleton`, and some scoped lifestyle that is particularly suited for that application type. For example an ASP.NET MVC application will typically use the `WebRequestLifestyle`; a Web API application will use the `WebApiRequestLifestyle`. Instead of using the appropriate `RegisterXXX` extension method of the appropriate integration package, you can now do the following:

{{< highlight csharp >}}
var container = new Container();
// Just define the scoped lifestyle once.
container.Options.DefaultScopedLifesyle = new WebRequestLifestyle();

container.Register<IUserContext, AspNetUserContext>(Lifestyle.Scoped);
container.Register<IUnitOfWork, DbContextUnitOfWork>(Lifestyle.Scoped);
{{< / highlight >}}

Not only does this make your code much cleaner, it also makes it easier to pass the container on to some methods that add some layer-specific configuration to the container. For instance.

{{< highlight csharp >}}
public static void BootstrapBusinessLayer(Container container) {
    // Registrations specific to the business layer here:
    container.Register<IUnitOfWork, DbContextUnitOfWork>(Lifestyle.Scoped);
}
{{< / highlight >}}

The `Lifestyle.Scoped` property makes it easy for the business layer bootstrapper to add registrations using the application’s scoped lifestyle without having to know which lifestyle it actually is. This simplifies reuse of this bootstrapper across the applications in your solution.

Another great improvement is the addition of the `RegisterConditional` methods. These method overloads allow conditional registration of types and registration of contextual types. Take the following conditional registrations:

{{< highlight csharp >}}
container.RegisterConditional<ILogger, NullLogger>(
    c => c.Consumer.ImplementationType == typeof(HomeController));
container.RegisterConditional<ILogger, SqlLogger>(c => !c.Handled);
{{< / highlight >}}

This particular combination of registrations ensures that a `NullLogger` is injected into `HomeController` and all other components get a `SqlLogger`.

For advanced scenarios one of the `RegisterConditional` overloads accepts an implementation type factory delegate. Take a look at the following example:

{{< highlight csharp >}}
container.RegisterConditional(typeof(ILogger),
    c => typeof(Logger<>).MakeGenericType(c.Consumer.ImplementationType),
    Lifestyle.Singleton,
    c => true);
{{< / highlight >}}

This example registers a non-generic `ILogger` abstraction that will be injected as a closed generic version of `Logger<T>`, where `T` is determined based on its context (in this case the consuming component). In other words when your `HomeController` depends on an `ILogger` it will actually get a `Logger<HomeController>`.

For a complete list of all the breaking changes, new features and bug fixes, please view the release notes.

Happy injecting.