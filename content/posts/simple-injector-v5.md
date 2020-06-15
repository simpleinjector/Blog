---
title:	"Simple Injector v5 - Virus edition"
date:	2020-06-11
author: Steven
draft:	false
---

It's been over three years since we released Simple Injector 4.0. The number of features that mean a bumping the major version number have been piling up on the backlog, and so we started work on the next major release a few months ago. And it's finally here! We've removed legacy methods, improved performance, fixed bugs, added features, and continued to push the library towards a [stratgey of best-practice](https://simpleinjector.org/principles+best-practices).

**There are quite a few breaking changes, which will likely impact you when migrating from v4 to v5. There are two changes in particular that you should be aware of: the handling of unregistered concrete types and auto-verification. Please read on to understand what has changed and why.**

In this blog post we describe the most prominent changes and their rational, starting with how v5 stops resolving unregistered concrete types.

### Unregistered concrete types are no longer resolved.

Simple Injector has always promoted best practices and this is an evolving process. Over the years, for instance, we figured out it was better to:

- require an active scope for resolving scoped instances
- collections to be registered, even if empty
- container-uncontrolled collections to be wrapped with transient decorators solely

These insights where gained during the development process and for each we decided to introduce a breaking change because we felt the breaking change was worth it.

Resolving unregistered concrete types is a similar case. While the ability to resolve and inject unregistered types can be an appealing concept, we have noticed that developers are often tripped over this behavior. In the past, we have introduced new verification features (such as the [Short-Circuited Dependencies diagnostic warning](https://simpleinjector.org/diasc)) to help reduce issues but errors still occur.

Changing this behavior has been long on my radar and is something I discussed with the other contributors even before the release of v4, over three years ago. Unfortunately, at that time, we were too close to the release of v4 and needed more time to assess the impact to our users. That's why we posponed the change to v5. Instead, we introduced a switch in v4 that allowed disabling this behavior and started promoting disabling this behavior in the documentation. This would allow new users and new projects to use the new settings and existing users to migrate at their own pace.

With the introduction of v5, we flipped the switch, meaning that resolution of unregistered concrete types is now disabled by default. We advise you keep the default behavior as-is and ensure you register all concrete types directly. If your current application heavily depends on unregistered concrete types being resolved, you can resore the old behavior by setting `Container.Options.ResolveUnregisteredConcreteTypes` to `true`.

For more details, please see [the documentation](https://simpleinjector.org/ructd).

### The container is now automatically verified when first resolved.

Just as it's a good idea to explicitly register all types up front, we have learned that it is a good idea to trigger container verification automatically on the first resolve.

Many developers using Simple Injector don't realize its full potential and forget to call `Container.Verify()`. We often see developers run into problems that a call to `Verify()` would have prevented. This is why in v5 we decided to automatically trigger full verification including diagnostics when the very first registration is resolved.

This wasn't an easy call, though, and is a **severe breaking change**. It's severe, because the change in behavior can be easily overlooked.

**When upgrading to Simple Injector v5, check whether your code base deliberately skips verification because of performance concerns.** And when this is the case, you should suppress auto-verification. 

There are two likely scenarios where you would want to suppress verification:

- **Running integration tests where each test creates and configures a new `Container` instance.** In that case verifying the container in each test might cause the integration test suite to slow down considerably because `Verify()` is a costly operation.
- **Running a large application where start-up time is important.** For big applications, the verification process could take up a considerate amount of time. In such a case you would prevent the application from verifying on startup, and instead move the `Verify` call to a unit/integration test. That allows fast application start-up.

Disabling auto-verification can be done by setting `Container.Options.EnableAutoVerification` to `false`.

### No more .NET 4.0

We have dropped support for .NET 4.0, **the minimum supported versions are now .NET 4.5 and .NET Standard 1.0.**

.NET 4.0 was released on 12 April 2010, which is more than a decade ago. It has been superseded with .NET 4.5 on 15 August 2012—now almost 8 years ago. It's time to let go of .NET 4.0, even though there may be some development teams still relying on it.

Developing software is always about finding a balance. Keeping support for older versions supported comes with costs—even for an open-source project. Perhaps even *especially* for open-source projects where development is done in free time (and free time is precious).

The introduction of support for .NET Standard introduced some complexity in the library, caused by the changing Reflection API. This new Reflection API was later added to .NET 4.5, but that lead to the use of `#if` preprocessor directives, additional build outputs and risk of introducing errors. This is complexity we wanted to get rid of, and a new major release is the time to do so. This comes with the risk of frustrating developers that maintain old applications and still want to enjoy improvements in Simple Injector. We're truly sorry if this frustrates your project, and hope that Simple Injector v4 serves you well until you can migrate to .NET 4.5 and beyond.

### Less first-chance exceptions

More recently, Microsoft made some changes to Visual Studio that have impacted Simple Injector users. One of those changes is how Visual Studio handles first-chance exceptions by default.

When debugging an application, not all exceptions have to be dealt with. When a third-party or framework library catches an exception and continues, you can safely ignore that exception. Where older versions of Visual Studio didn't show these exceptions by default, newer versions of Visual Studio automatically stop the debugger and popup the exception window. This can be really confusing because it's not always immediately clear whether the first-chance exception is being dealt with by the library component or is one that breaks your application.

I have wasted many hours because of this. Even Microsoft libraries throw exceptions that they recover from themselves!

In Simple Injector 4, there are times where the library would throw an exception that it caught elsewhere and handled itself. This design worked well in the older versions of Visual Studio. But since stopping at first-chance exceptions is the new norm, the behavior is problematic for our users. Not only does it cause confusion, getting those constant exception popups during startup can be really annoying.

In v5 we changed the APIs that would throw and catch exceptions. They now follow the 'try-parse' pattern and return a `boolean`. This does mean, however, it's a breaking change. In case you have a custom `IConstructorSelectionBehavior` or `IDependencyInjectionBehavior` implementation, you will need to change your implementation. 

It was impossible for us to completely remove all first-chance exceptions from the library and there are still edge cases where exceptions are caught—most notably in the generics sub system. There are some edge cases where Simple Injector can't correctly determine generic type constraints, which means that it relies on the framework to communicate the existence of such a constraint. This part of the .NET Reflection API lacks a 'try-parse' API and we're stuck with catching exceptions. The chances, however, of you hitting this are very slim, because it only happens under very special conditions.

### Simplified registration of disposable components

When it comes to analyzing object graphs, Simple Injector always erred on the side of safety.

Lifestyle Mismatches are, by far, the most common DI pitfall to deal with. Detecting these mismatches is something Simple Injector had done for a very long time now. Simple Injector prevents you from accidentally injecting a short-lived dependency into a longer-lived consumer.

Simple Injector considers a Transient component's lifetime to be shorter than that of a Scoped component. But a single Scope could theoretically live for a very long time. If you wish, you could leave your scope open for the complete duration of a long-running operation or even for the duration of the application, which would make an injected Transient live for as long as well. This is the reason the injection of a Transient into a Scoped was blocked by default and reported by the Diagnostics sub system.

In practice, however, Scopes are usually wrapped around a single (web) request, which makes their lifetime very limited and deterministic. In these cases, injecting a Transient into a Scoped is relatively risk-free.

Additionally, the Transient lifestyle also behaves quite differently compared to the Scoped lifestyle. Transient components are not tracked by Simple Injector and, therefore, can't be disposed of. You likely encountered the "{your class} is registered as transient but implements IDisposable" error before. Registering it as Scoped fixes the issue but would cause the Lifestyle Mismatch error when that registration contains Transient dependencies.

Because of the low risk of injecting a Transient into a Scoped, this strict behavior causes more confusion and frustration than that it prevents errors and is why we have decided to relax this behavior. By default, Simple Injector v5 allows you to inject transients into Scoped components. If you would prefer to revert to the old behavior you can set `Container.Options.UseStrictLifestyleMismatchBehavior` to `true`.

### Performance, performance, performance

Simple Injector has historically always been one of the top performers when it comes to speed of resolving. Very early on we decided that performance is a feature and were able to have great performance while adding new features.

We're now at a practical limit of what's achievable from a performance perspective. There are areas where performance can theoretically be improved, but it has no practical use, because you wouldn't notice the difference when running a real application.

There is one area, however, where performance could still be improved, and this is startup time. As I discussed above, registering and verifying a big application can take a considerable amount of time. While running a performance analysis during the development of v5, we noticed a few hotspots that caused a considerable slowdown in performance during the registration phase. These were due to using some of the slower Reflection calls. After building an optimized POC, we noticed a performance boost of up to 60%, which is very significant, especially for big applications.

Unfortunately, this improvement meant we had to introduce a breaking change. But the performance gain is significant, and we felt it worth the risk. Only few developers will be affected by this change. **You will only be affected by this breaking change if you've created your own lifestyles.** If this is the case, please review the release notes closely to see what we've changed.


### Asynchronous disposal

Another great improvement is the ability for Simple Injector to asynchronously dispose registered components. The ASP.NET Core integration package in Simple Injector v4 made sure that components implementing `IAsyncDisposable` were disposed at the end of a web request. This, however, only worked within the context of a web request, and only components could be disposed that implemented both `IAsyncDisposable` *and* `IDisposable`, which might not always be the case.

In v5 we have now integrated this feature into the core library. Not only did this simplify the ASP.NET integration package and remove the `IDisposable` limitation, it also means that asynchronous disposal is available everywhere. For example, when running background operations in ASP.NET Core, or when running a Console Application.

The v5 ASP.NET Core integration package automatically calls `Scope.DisposeAsync()` when a web request ends. In other cases, you will need to call `Scope.DisposeAsync()` manually.

**Please note that this feature is only available in the .NET 4.6.1, .NET Standard 2.0, and .NET Standard 2.1 versions of Simple Injector.**

### Metadata

The last big new feature is the ability to inject a dependency's metadata into a consumer. In v5 you can now write this:

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

The class `X` will be receive a `DependencyMetadata<T>`, which is a new Simple Injector type. This metadata gives access to the dependency's **InstanceProducer**, its implementation type, and allows the type to be resolved by calling **GetInstance()**.

Admittedly, this is a rather advanced feature that not many users will need. It's meant to be used inside infrastructure components (i.e. classes that are part of the Composition Root). Instructure components sometimes require more information about the dependency and need to be able to lazily resolve it. The feature is *not* meant to be used *outside* the Composition Root, because that would cause your application to take a dependency on Simple Injector, which is something we advise against.

For a more elaborate example of this this feature, see [the documentation](https://simpleinjector.org/advanced+metadata).

For a complete list of all the breaking changes, new features and bug fixes, please view the [release notes](https://github.com/simpleinjector/SimpleInjector/releases/tag/v5.0.0).