---
title:	"The Tale of the Async Interfaces"
date:	2020-12-15
author: Steven
draft:	false
aliases:
    - 2020/12/the-tail-of-the-async-interfaces

---

> With the release of Simple Injector v5, I made an error of judgement. To support asynchronous disposal of the `Container` and `Scope` objects I added a dependency from the core library to the Microsoft.Bcl.AsyncInterfaces NuGet package. Unfortunately, this proved to be a very painful mistake. In this blog post I'll explain why I choose to take this dependency, why this was a mistake, and how this finally got fixed in [v5.2](https://github.com/simpleinjector/SimpleInjector/releases/tag/5.2.0).

## Double Trouble

<img align="right" width="40%" style="max-width:275px;" src="/images/diamondduck.png">

With the introduction of .NET Core 3, Microsoft added a new `IAsyncDisposable` interface as asynchronous counterpart of `IDisposable`. In order enable asynchronous disposal in older framework versions (i.e. .NET Core 2.0 and .NET 4.6.1), Microsoft published the [Microsoft.Bcl.AsyncInterfaces](https://www.nuget.org/packages/Microsoft.Bcl.AsyncInterfaces/) NuGet package. Where the package's `net461` and `netstandard2.0` targets contain assemblies that specify—among other things—the `IAsyncDisposable` interface, the `netstandard2.1` target assembly uses type forwarding to reference the framework's version.

The publication of the AsyncInterfaces NuGet packages allowed Simple Injector v5.0 to start using the `IAsyncDisposable` interface. This allowed Simple Injector to asynchronously dispose of classes implementing `IAsyncDisposable`. It also allowed users to dispose of both `Scope` and `Container` in an asynchronous fashion.

Historically, the Simple Injector core library hasn’t taken a dependency on an external NuGet package. And for good reason. Because external dependencies can cause conflicts, such as those pesky binding redirect issues that we’re all too familiar with. This is an unfortunate restriction and there have been many times I wished I could just pull in some external package to simplify my work. The very useful [System.Collections.Immutable](https://www.nuget.org/packages/System.Collections.Immutable) package is one such example.

Unfortunately, there is no easy way of allowing Simple Injector to asynchronously dispose of registered types without taking a dependency on AsyncInterfaces. This left me with two options:

1. Add this dependency to the core and implement asynchronous disposal directly in the core library
2. Create a new NuGet package (e.g., `SimpleInjector.Async`) that adds extensions that allow asynchronous disposal of `Scope` and `Container`.

I decided to go with the first option as the second would result in a less discoverable solution. The former option allows implementing `IAsyncDisposable` directly on `Container` and `Scope`; the latter would likely rely on extension methods, to allow something similar to the following: `await scope.AsAsyncDisposable().DisposeAsync();`.

This new feature was [implemented](https://github.com/simpleinjector/SimpleInjector/issues/791) and introduced in [Simple Injector v5](https://blog.simpleinjector.org/2020/06/simple-injector-v5/). This is when the trouble started.

## Diamonds are forever
Within a few days of the release of v5, developers [began](https://github.com/simpleinjector/SimpleInjector/issues/823) reporting binding redirect issues. This was caused by the multitude of versions that exist for the AsyncInterfaces package and its dependencies. Developers were using other libraries and framework parts that took a dependency on AsyncInterfaces, or its dependencies [System.Threading.Tasks.Extensions](https://www.nuget.org/packages/System.Threading.Tasks.Extensions) and [System.Runtime.CompilerServices.Unsafe](https://www.nuget.org/packages/System.Runtime.CompilerServices.Unsafe). When one of the used application dependencies references a different version of such package (and in particular when the package’s contained assembly has a different assembly version) binding redirect issues can appear.

In an ideal world, the NuGet package manager automatically solves binding redirect issues for us by adding the required plumbing in the application's configuration file. But for some reason, the package manager fails to do this. Instead, we have to manually add binding redirects to our configuration files. And this seems especially confusing with regards to AsyncInterfaces and its sub dependencies, as their assembly versions do not match the NuGet package version. The dependency chain of AsyncInterfaces seems to be in [the list of common troublemakers](https://nickcraver.com/blog/2020/02/11/binding-redirects/).

With the introduction of .NET Core, there’s the idea that binding redirect issues are thing of the past. A [more-recent bug report](https://github.com/simpleinjector/SimpleInjector/issues/823#issuecomment-726143424), however, demonstrated that this isn't always the case, demonstrating that these issues won't easily go away if we wait long enough.

The pain experienced with these dependencies can be solved by setting the correct binding redirects in the application’s configuration file. To help Simple Injector users, I started posting binding direct examples that developers could copy-paste to fix their problem. But even using these examples, developers struggled, and I recently got stuck with this on an application I was developing. Whatever binding redirects I tried, after analyzing the assembly versions of the used NuGet packages, the application would crash with a dreaded “Could not load file or assembly X or one of its dependencies” exception. This was the moment that I started to realize the gravity of this Diamond Dependency dilemma that AsyncInterfaces and his little helpers caused. Action was required. This instigator had to go.

> **Diamond Dependency**
>
> My application took a dependency on both Simple Injector and System.Collections.Immutable. Those two libraries, however, both depended (indirectly) on the previously mentioned CompilerServices.Unsafe. Simple Injector did so via AsyncInterfaces and Tasks.Extensions, while Immutable depended on CompilerServices.Unsafe via [System.Memory](https://www.nuget.org/packages/System.Memory). This is an example of a Diamond Dependency.
>
> A Diamond Dependency is a dependency chain of at least four libraries where library A (e.g. my application) depends on libraries B (Simple Injector) and C (Immutable). B and C than depend on the final library D (Unsafe). A problem emerges when those middle libraries require different versions of this final library D. This is called the Diamond Dependency Conflict.

## If it quacks like a duck
But removing the AsyncInterfaces dependency was easier said than done. Removing the asynchronous disposal feature was not an option; developers already depend on that feature, often—but not always —implicitly by using one of the ASP.NET Core integration packages. And we can certainly expect asynchronous disposal to become more common soon, long before the Diamond Dependency problem will disappear (i.e. before Simple Injector can drop support for .NET 4 and .NET Core 2).

Over the last 6 months I have conducted two experiments that tried duck typing to allow removing the AsyncInterfaces dependency from the core library. With duck typing, instead of depending on interfaces, you take a more dynamic approach where you accept any type that conforms to a certain signature. The C# compiler takes this approach in many places, for instance with the `foreach`, `using`, and `async using` keywords. Both trials were discontinued because of the complexity they would introduce into the library, especially when taking performance into consideration. But after I recently experienced the seriousness of the situation myself, I knew removing the party crasher was the only viable solution to this problem. And so, I had to release the Quacken!

After multiple days of trying, failing, testing, improving, [asking](https://stackoverflow.com/questions/65200662/), things started to quack like duck. The meat and potatoes of the implementation is inside the `Scope` class (which now is littered with compiler directives). `Scope` now does the following:

* It tries to figure out if a tracked instance implements an interface named "System.IAsyncDisposable". Whether that interface is The Real Thing (tm) or just some self-defined surrogate is irrelevant. If the instance implements that interface, it is stored for disposal, as would happen for 'normal' `IDisposable` instances.
* During asynchronous disposal of the `Scope`, the `Scope` will invoke the instance’s `DisposeAsync` method. The returned object (typically `Task` or `ValueTask`) will be awaited.

Of course, performance must be taken into consideration, considering that Reflection calls are slow. Such a performance penalty would perhaps be acceptable during disposing of the `Container`, but certainly not when disposing of a `Scope`, as an application might create and dispose of thousands of `Scope` instances per second. And so `Scope` implements the following caching:

* Whether a checked type implements `IAsyncDisposable` or not. This way only one call to `Type.GetInterfaces().Where(i => i.FullName == "System.IAsyncDisposable")` is required.
* When the `IAsyncInterface` is detected for the first time, it will be stored internally. This allows any subsequent checks to call `asyncDisposableType.IsAssignableFrom(type)` instead of the slower `GetInstances().Where(...)` again.
* The call to `IAsyncDisposable.DisposeAsync().AsTask()` is compiled using expression trees, just as the rest of Simple Injector does for object composition under the covers. This makes calling `DisposeAsync` (almost) as fast as a native interface call. The .NET Standard 2.1 version, btw, completely skips all this duck typing nonsense and just natively calls `IAsyncDisposable` because, as I mentioned previously, with .NET Core 3 that interface is recognized natively.

The greatest disadvantage of this approach, from a user's perspective, is that I had to remove the `DisposeAsync` methods from `Container` and `Scope` in the pre-.NET Standard 2.1 builds. Because not only did the removal of AsyncInterfaces mean no reference to `IAsyncDisposable`, it also removed the reference to `ValueTask`, which the `IAsyncDisposable.DisposeAsync` method returns. For a while I played with the idea of the other builds to have an `DisposeAsync` method that would simply return `Task`. This would allow developers to use C#'s `async using` syntax on `Container` and `Scope`. But I quickly realized that the different signature of the `DisposeAsync` method (the return type is part of the signature) would cause `MissingMethodExceptions`.

To prevent incompatible signatures, while still allowing both the `Container` and `Scope` to be disposed of asynchronously, methods with completely different names needed to be added. This is why you'll find `DisposeContainerAsync()` and `DisposeScopeAsync()` methods on `Container` and `Scope` respectively. I'm the first to agree that this is bats-ugly, but it’s the best I could come up with.

On the flip side, however, because of the use of duck typing, Simple Injector can now support asynchronous disposal *on all of its builds*. Where previously asynchronous disposal was only supported on the `net461` and `netstandard2.0` builds of Simple Injector, with the introduction of Simple Injector v5.2, asynchronous disposal is supported on `net45` and `netstandard1.0` as well. Although its not possible to reference AsyncInterfaces' `IAsyncDisposable` in your application, you can simply define `System.IAsyncDisposable` somewhere in your application, and it just works. Here's an example:

``` c#
namespace System
{
    public interface IAsyncDisposable
    {
        Task DisposeAsync();	
    }
}
```

Even though the official `IAsyncDisposable` interface exposes `ValueTask` rather than `Task`, Simple Injector accepts this alternative definition anyway. As long as the interface is called "System.IAsyncDisposable" and there's a method named "DisposeAsync" which either returns `Task` or `ValueTask`, everything will just run smoothly. This allows you to start using asynchronous disposal until you can migrate to your code base to .NET Core 3 or .NET 5.

All the sweat and tears I poured over my keyword in the past weeks to get this fixed are now dried up and materialized in the Simple Injector code [v5.2](https://github.com/simpleinjector/SimpleInjector/releases/tag/5.2.0) code base. This will certainly not fix all your binding redirect issues but will at least ensure that Simple Injector is not amplifying the problem any longer.

Happy injecting.