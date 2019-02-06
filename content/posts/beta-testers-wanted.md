---
title:	"Wanted: Beta Testers"
date:	2015-06-29
author: Steven
draft:	false
---

With Simple Injector 3 we are breaking with the past and working hard to simplify the API to better represent our ideals and, of course, the name of our library. This means that we are introducing breaking changes in v3 that will undoubtedly impact any developer migrating from v2.

The driver for making these breaking changes is that areas of the API have evolved over time and in so doing have grown confusing (e.g. `RegisterOpenGeneric` and `RegisterManyForOpenGeneric`). In the pursuit of keeping Simple Injector simple we felt obligated to improve the consistency of the API. These decisions have not been taken lightly because we hate breaking your code. Our driving force is, however, a simpler Simple Injector for everyone.

We are doing our best to make the transition as seamless as possible and we need feedback on this effort. We are looking for Simple Injector users to test the beta of Simple Injector 3 and tell us what they think.

If you can help then please upgrade your NuGet packages to [3.0.0-beta4](https://www.nuget.org/packages/SimpleInjector/3.0.0-beta4) and try to compile your code (it will probably fail!). Our main strategy is to guide users with [obsolete messages](https://msdn.microsoft.com/en-us/library/aa664623%28v=vs.71%29.aspx) that should be presented as descriptive compiler errors. Once you have fixed any errors please call Verify() on your container, (as you are hopefully already doing) and test your application to ensure everything resolves as expected. To discuss any element of beta testing please join us on [gitter](https://simpleinjector.org/chat). We would usually expect the upgrade process to take no more than a half hour.

*Please note that we don’t expect you to do our testing for us. Simple Injector has an extensive suite of unit tests and we don’t expect to have introduced many new bugs.*

Here’s a list of the most prominent breaking changes we are introducing:

* Calling `Verify()` will now automatically diagnose the container’s configuration for common configuration mistakes and exceptions will be thrown. We felt that not enough developers were explicitly calling `Analyzer.Analyze()` to check for diagnostic warnings, leading to a common source of bugs. So we decided to integrate this into Verification in the hope that this will guide our users to the pit of success. The old Verification behaviour can be triggered by calling `Verify(VerificationOption.VerifyOnly)`.
* Even if you don’t call `Verify()`, the v3 the container will always check for [lifestyle mismatches](https://simpleinjector.org/dialm) when resolving an instance and will throw an exception if there is a mismatch in the object graph. This more strict behaviour can be suppressed globally but (for obvious reasons) we advise against doing so.
* The `RegisterSingle` methods have been renamed to `RegisterSingleton` as we felt there was some ambiguity in the name `RegisterSingle`, especially in combination with methods like `RegisterAll`, `RegisterCollection` and `RegisterManyForOpenGeneric`. We considered removing these method completely, but the benefits of doing so did not compare favourably to the pain of fixing this as a breaking change.
* The `RegisterAll` methods have been renamed to `RegisterCollection`. Just as with the `RegisterSingle` methods, new developers experienced confusion in their naming. The new name expresses more clearly what the methods do.
* The `RegisterManyForOpenGeneric` extension methods from the `SimpleInjector.Extensions` namespace have been replaced with new `Register` and `RegisterCollection` overloads on the `Container`. When doing one-to-one mappings, `Register(Type, IEnumerable<Assembly>)` can be used, and when registering collections `RegisterCollection(Type, IEnumerable<Assembly>)` can be used.
* The `RegisterSingleDecorator` extension methods have been removed. Decorators can be registered by calling `RegisterDecorator` while supplying the `Lifestyle.Singleton`.
* The `RegisterOpenGeneric` extension methods have been removed. The `Container.Register` methods have been extended to accept open generic types. This removes the superficial difference between the registration of open generic types and other registrations. (This difference originally had a technical background, but we allowed the internal difference to effect the user experience.) Do note that there is a behavioural difference between `RegisterOpenGeneric` and `Container.Register`. `RegisterOpenGeneric` registers each generic type as fall-back, which means it will apply that type if no other registration exists for that type. Conversely, `Container.Register` will not register a fall-back, it will, instead, test for overlapping registrations. If the fall-back behaviour is required, the new `RegisterConditional` methods can be used. The new `RegisterConditional` methods allow supplying a delegate that allows signaling a registration as fall-back registration.
* `IConstructorVerificationBehavior` and `IConstructorVerificationBehavior` have been merged and replaced with `IDependencyInjectionBehavior`.

*In the majority of cases the compiler error messages should guide you through the migration. If you find a scenario that is unclear or it takes time to figure out please let us know. We want to make the migration as seamless as possible.*

Besides the above list of breaking changes, we have some compelling new features that may be of interest:

* A `Lifestyle.Scoped` property was added to simplify registration of scoped lifestyles, since in most applications you would typically only use one specific scoped lifestyle. You can now use `Register<IService, Impl>(Lifestyle.Scoped)` instead of having to call `RegisterPerWebRequest<IService, Impl>()` for example. This also simplifies reuse in your composition root, when the same configuration is reused over multiple application types (such as MVC and WCF).
* As noted above `RegisterConditional` methods are added to the `Container` to allow registering types based on contextual information such as information about the consumer in which they are injected.
* Batch registration made easier by adding overloads of the `Register` method that accept either a collection of types or a collection of assemblies.
* As explained above, the `Register` methods now accept open generic types.
* The container now implements `IDisposable`. This allows cached singletons to be disposed when the container gets disposed.
* `RegisterCollection(Type, IEnumerable<Registration>)` now accepts open generic service types as well.