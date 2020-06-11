---
title:	"Simple Injector v5"
date:	2020-06-12
author: Steven
draft:	false
---

It has been over three years that we released Simple Injector 4.0. The number of features that required a major version bump were piling up on our backlog, which is why we started working on the next major release a few months ago. And it's finally here. We removed legacy methods, improved performance, fixed bugs, added features, and once more pushed the library towards a [best-practice strategy](https://simpleinjector.org/principles+best-practices).

**We introduced quite some breaking changes, which will likely impact you when migrating from v4.10 to v5.0 without any problems. There are two changes in particular that will likely impact you, which are the new auto-verification feature and how Simple Injector handles unregistered concrete types. Please read on to understand what changed.**

Also review the [release notes](https://github.com/simpleinjector/SimpleInjector/releases/tag/v4.5.0) closely to check whether there's any breaking change that would affect you.

**Before you upgrade to v5.0, please make sure you upgrade to the latest v4.x version of Simple Injector first.**

Below, I discuss the most prominent changes we made in this release.


### Unregistered concrete types are no longer resolved.

Simple Injector has always promoted best practices and tries to prevent you from shooting yourself in the foot. But this is an evolving process. Over the years, for instance, we figured that it was better to require:

- an active scope for resolving scoped instances
- collection to be registered, even when they're empty
- container-uncontrolled collections to be wrapped with Transient decorators solely

All these insights where gained later on, and we decided to introduce a breaking change because we felt the breaking change was worth it.

Resolving unregistered concrete types is a similar case. While the ability to resolve and inject unregistered types can be an appealing concept, developers trip over this behavior regularly. Over the years we introduced new verification features (such as the [Short-Circuited Dependencies diagnostic warning](https://simpleinjector.org/diasc)) to prevent you from running into troubles, but even with all these checks in place, errors are still made easily.

Changing this behavior has been long on my radar, which is why I started discussing this with the other contributors even before the release of v4—more than three years ago. Unfortunately, we had to postpone this feature until v5.

With the introduction of v5, resolution of unregistered concrete types is now disabled by default. We advise to keep the default behavior as-is and register all concrete types directly. In case your current application heavily depends on unregistered concrete types being resolved, you can still flip the switch and use the old behavior. Just set `Container.Options.ResolveUnregisteredConcreteTypes` to `true`.


### Container is now automatically verified when first resolved.

Just like it's generally a good idea to explicitly register all types up front, we came to realize that it is a good idea to trigger verification of the container automatically on the first resolve.

Many developers that start using Simple Injector don't realize its full potential and forget to call `Container.Verify()`. We've seen many developers run into problems that a a call to `Verify()` would have prevented. This is why in v5 we help those new developers out by automatically triggering `Verfiy()` when the first registration is resolved.

This wasn't an easy call, though, because this is a quite severe breaking change. It's severe, because the change of behavior is very subtle.

**When upgrading to Simple Injector v5, check whether your code base deliberately skips verification because of performance concerns.** And when this is the case, you should suppress auto-verification. There are two likely scenarios where you want to suppress verification:

- **When running integration tests where each integration tests creates a new `Container` instance.** In that case verifying the container in each test might cause the integration test suite to slow down considerably, because `Verify()` is a costly operation.
- **When running a large application where start-up time is important.** For big applications, the verification process could take up a considerate amount of time. In that case you would move the `Verify` call to a unit/integration test. That allows the application to start-up quickly, and the performance hit to be spread out over a longer period of time.

Disabling auto-verification can be done by setting `Container.Options.EnableAutoVerification` to `false`.


### No more .NET 4.0

We decided to drop support for .NET 4.0. This means that **the minimum supported versions are .NET 4.5 and .NET Standard 1.0.**

.NET 4.0 was released on 12 April 2010, which is more than a decade ago. It has been superseded with .NET 4.5 on 15 August 2012—almost 8 years ago. It was time to let go of .NET 4.0, even though we know that there are development teams still relying on .NET 4.0.

Developing software is always about finding a balance. Keeping older versions supported comes with costs, even for an open-source project. Or perhaps even *especially* for open-source projects where development takes place in free time and free time is precious.

For Simple Injector specifically, the introduction of .NET Standard introduced quite some complexity in the library, caused by the changing Reflection API in .NET Standard. This new Reflection API was later added to .NET 4.5, but that lead to the use of `#if` preprocessor directives, additional build outputs, and risk of introducing errors. This is complexity we wanted to get rid of, and a new major release is the place to do this. This always comes at the risk of frustrating developers that still maintain old systems, while still want to enjoy improvements in Simple Injector.

We're truly sorry if this frustrates your project, but we hope that Simple Injector v4 keeps serving you well until it's time for you to migrate to .NET 4.5 and beyond.


### Less first-chance exceptions

In recent years, Microsoft made some changes to Visual Studio that have impacted Simple Injector users. One of those changes is how Visual Studio handles first-chance exceptions by default.

When debugging an application, not all exceptions have to be dealt with. When a third-party or framework library catches an exception and continues, you can safely ignore that exception. Where older versions of Visual Studio didn't show these exceptions by default, newer versions of Visual Studio automatically stop the debugger and popup the exception window. This can be really confusing because it's not always immediately clear whether we're dealing with a first-chance exception or one that breaks your application.

I have wasted many hours myself because of this. Even Microsoft libraries themselves still throw exceptions that they recover from themselves.

In Simple Injector 4, there were quite some times that the library would throw an exception that it caught itself. This design worked well in the older versions of Visual Studio. But since stopping at first-chance exceptions is the norm now, this behavior is problematic for our users. Not only does it cause confusion, getting those constant exception popups during startup can be really annoying.

In v5 we changed the APIs that would throw and catch exceptions. They now follow the 'try-parse' pattern, where they return a Boolean. This does mean, however, a breaking change. In case you have a custom `IConstructorSelectionBehavior` or `IDependencyInjectionBehavior` implementation, you will have to change that implementation. 

It was unfortunately impossible for us to remove all first-chance exceptions from the library. There are still corner cases where exceptions are caught, most notably in the generics sub system. There are some corner cases where Simple Injector can't correctly determine generic type constraints, which means that it relies on the framework to communicate the existence of such constraint. Unfortunately, that part of the .NET Reflection API lacks a 'try-parse' API—we're stuck with catching exceptions. The chances, however, of you hitting this are very slim, because it only happens under very special conditions.


### Simplified registration of disposable components

When it comes to analyzing object graphs, Simple Injector always erred on the side of safety.

Lifestyle Mismatches are, by far, the most common DI pitfall to deal with. Detecting these mismatches is, therefore, something Simple Injector does for a very long time now. Simple Injector prevents you from accidentally injecting a short-lived dependency into a longer-lived consumer.

Simple Injector considers a Transient component's lifetime to be shorter than that of a Scoped component. But a single Scope could theoretically live for a very long time. If you wish, you could leave your scope open for the complete duration of a long-running operation or even for the duration of  the application, which would make an injected Transient live for as long as well. That's the reason that injection of a Transient into a Scoped was blocked by default.

In practice, however, Scopes are usually wrapped around a single (web) request, which makes their lifetime very limited and deterministic. In that case injecting a Transient into a Scoped is relatively risk-free.

On top of that, the Transient lifestyle also behaves quite differently compared to the Scoped lifestyle. Transient components are not tracked by Simple Injector and, therefore, can't be disposed of. You likely encountered the "{your class} is registered as transient but implements IDisposable" error before. Registering it as Scoped fixes the issue but would cause the Lifestyle Mismatch error when that registration contains Transient dependencies.

Because of the low risk of injecting a Transient into a Scoped, this strict behavior causes more confusion and frustration than that it prevents errors. That's why we decided to loosen up this behavior. This means that by default, Simple Injector v5 allows you to inject transients into Scoped components. That is, of course, unless you revert to the old behavior, which might still be useful when you're building applications that make use of long-living Scope instances. In that case, you can set `Container.Options.UseStrictLifestyleMismatchBehavior` to `true`.

### Performance, performance, performance

Simple Injector has historically always been one of the top performers when it comes to speed of resolving. Very early on we decided that performance is a feature and were able to have great performance even with new compelling features and verification features that were added.

At many places we're now at a practical limit of what's achieveble from a performance perspective. At other points performance can theoretically be improved, but it has no practical use, because you wouldn't notice the difference when running a real application.

There is one area, however, where performance could still be improved, which is startup time. Registering and verifying a big application can take a considerable amount of time. While running a performance analysis during the development of v5, we noticed a few hotspots that caused a considerable slow down during the registration phase. This was caused using a few very, unfortunate slow Reflection calls. After building an optimized POC, we noticed a performance boost of up to 60%, which is very significant, especially for big applications.

Unfortunately, this improvement meant we had to introduce a breaking change. We felt, however, the performance gain to be so significant, that we found it worth the risk. Fortunately, only few developers will be affected by this change. **You will only be affected by this breaking change if you've created your own lifestyles.** If that's the case, please review the release notes closely to see what we've changed.


### Asynchronous disposal

Another great new improvement is the ability for Simple Injector to asynchronously dispose registered components. In Simple Injector v4 the ASP.NET Core integration package made sure that asynchronous components were disposed at the end of a web request. But in v5 we integrated this into the core library. Not only did this simplify the ASP.NET integration package, it also means that asynchronous disposal is now available everywhere, for instance, when running background operations in ASP.NET Core, but actually for any type of application you're developing.

The ASP.NET Core v5 integration package automatically calls `Scope.DisposeAsync()` when a web request ends. In other cases, you will have to call `Scope.DisposeAsync()` manually.

**Please note that this feature is only available in the .NET 4.6.1, .NET Standard 2.0, and .NET Standard 2.1 versions of Simple Injector.**

### Metadata

The last big feature is the ability to inject a dependency's metadata into a consumer. In v5 you can now write this:

{{< highlight csharp >}}
public class Foo
{
    public Foo(DependencyMetadata<IDependency> metadata) { }
}

public class Bar
{
    public Bar(IList<DependencyMetadata<IDependency>> metadata) { }
}
{{< / highlight >}}

In this case, the class `X` is injected with a `DependencyMetadata<T>`, which is a new Simple Injector type. This metadata type gives access to the dependency's **InstanceProducer**, its implementation type, and allows the type to be resolved by calling **GetInstance()**.

Admittedly, this is a rather advanced feature that perhaps not many users will need. This feature is meant to be used inside infrastructure components (i.e. classes that are part of the Composition Root). Instructure components sometimes require more information about the dependency and need to be able to lazily resolve it. The feature is not meant to be used *outside* the Composition Root, because that would cause your application to take a dependency on Simple Injector, which is something we advise against.

For more information about this feature, see [the documentation](https://simpleinjector.org/advanced+metadata).

For a complete list of all the breaking changes, new features and bug fixes, please view the [release notes](https://github.com/simpleinjector/SimpleInjector/releases/tag/v5.0.0).
