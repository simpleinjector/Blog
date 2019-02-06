---
title:	"Simple Injector v4 has been released"
date:	2017-03-31
author: Steven
draft:	false
---

For the last months we’ve been working on the next major release of Simple Injector, and it is finally here. We have removed legacy methods, simplified working with the library, and fixed many bugs and quirks.

**In contrast to the impact that v3 had for developers, we expect most developers to update without having to make any code changes when upgrading from the latest v3.x to v4.0.** There are quite some breaking changes through, but most of them are in more specialized parts of the library that you use when extending Simple Injector, such as writing custom Lifestyles, which is something most developers don’t do.

Our goal has always been to let the API guide you as much as possible through the breaking changes and how to fix them. In most cases removed parts of the API still exist, but are marked with [Obsolete(error: true)] attribute with expressive messages that explain what to do instead. This will cause your compiler to show a compilation error with (hopefully) a clear message describing the action to take. This should make it easier for you to migrate from v3.x to v4.0.

**Before you upgrade to v4.0, please make sure you upgrade to the latest v3.x version of Simple Injector first.**

With the release of v4.0 we moved to [.NET Standard](https://blogs.msdn.microsoft.com/dotnet/2016/09/26/introducing-net-standard/) in favour of PCL. This means we removed support for PCL in version 4. Since most new platforms embrace the new .NET Standard, this shouldn’t be a problem. As long as your platform supports .NET Standard, Simple Injector v4 will run happily.

### New Features

With this release we introduced many small and big simplifications to the API, some of which are:

* The integration of the common `LifetimeScopeLifestyle` and `ExecutionContextScopeLifestyle` as part of the core library. These lifestyles have been renamed to the more obvious `ThreadScopedLifestyle` and `AsyncScopedLifestyle`, and the old SimpleInjector.Extensions.* NuGet packages have been deprecated.
* The deprecation of framework-specific lifestyles `WebApiRequestLifestyle` and `AspNetRequestLifestyle` in favor of the new built-in `AsyncScopedLifestyle`.
* The automatic and transparent reuse of registrations for classes that are registered with multiple interfaces. Simple Injector detected these kinds of problems, calling them [Torn Lifestyles](https://simpleinjector.org/diatl), but in Simple Injector v4 we completely removed this problem altogether, making it something the user hardly ever has to think about.
* Several overloads added to simplify common scenarios.

On top of that, we removed some small parts of the API that could cause ambiguity and could lead to hidden, hard to detect errors. In some situations, e.g. when making conditional registrations, the user was able to make decisions on the service type of the consuming component, but this was unreliable, because such component could be registered with multiple interfaces. This could make the conditional registration invalid, where it was impossible for Simple Injector to warn the user about this. We removed these ambiguous properties and force the user to use the property containing the implementation type instead. We added some convenient extension methods on System.Type to make it easier to extract an abstraction from such implementation type, namely: `IsClosedTypeOf<T>`, `GetClosedTypeOf<T>` and `GetClosedTypesOf<T>`.

We improved the Diagnostic sub system once more. The biggest improvement is the detection of [Short Circuited Dependencies](https://simpleinjector.org/diasc). This is something that we were doing since v2, but there were situations in the past where Short Circuited Dependencies weren’t detected. We fixed that in this release.

For a complete list of all the breaking changes, new features and bug fixes, please view the [release notes](https://github.com/simpleinjector/SimpleInjector/releases/tag/v4.0).