---
title:	"When should you use a container?"
date:	2015-12-06
author: Steven
draft:	false
aliases:
    - /2015/12
---

A DI container is a tool that allows constructing the graphs of classes that contain an application’s behaviour (a.k.a. components or [injectables](http://misko.hevery.com/2008/09/30/to-new-or-not-to-new/)). When you apply Dependency Injection in your systems the DI container can simplify the process of object construction and can, when used correctly, improve the maintainability of the start-up path (a.k.a. the [Composition Root](https://freecontent.manning.com/dependency-injection-in-net-2nd-edition-understanding-the-composition-root/)) of your application. But a DI container is not mandatory when you apply Dependency Injection.

Applying Dependency Injection without a DI container is called [Pure DI](https://blog.ploeh.dk/2014/06/10/pure-di/). When you use Pure DI you define the structure of your object graphs explicitly in code and this code is still centralized in the Composition Root just as it is when using a DI container. Dependency Injection does not discourage the use of the `new` keyword to construct components; it promotes the centralization of the use of the `new` keyword.

In [this](https://blog.ploeh.dk/2012/11/06/WhentouseaDIContainer/) article, [Mark Seemann](https://blog.ploeh.dk/) shows the advantage of Pure DI over using a container: with Pure DI the compiler can verify the object graph. Mark makes some good points that for smaller applications Pure DI can be more beneficial than the use of containers, while larger applications can take advantage of [convention over configuration](https://en.wikipedia.org/wiki/Convention_over_configuration) which can help a lot in making your Composition Root maintainable. Mark even [shows](https://blog.ploeh.dk/2014/06/03/compile-time-lifetime-matching/) how Pure DI can help in finding configuration mistakes like [Captive Dependencies](https://blog.ploeh.dk/2014/06/02/captive-dependency/).

The primary benefit of Pure DI is that it allows your code to fail fast (in this case the system fails at compile time). Detecting failures early is crucial when it comes to lowering development cost, because tracking down bugs is obviously much easier in a system that fails fast.

Although I do agree with Mark’s reasoning, it’s important to realize that Pure DI isn’t a silver bullet that detects all configuration mistakes. On the contrary, it’s quite easy to overlook problems such as Captive Dependencies as the Composition Root starts to grow. If you were to switch from a DI container to Pure DI and you were expecting your code to fail fast, you might be in for an unpleasant surprise when the first bugs appear. This can happen because the C# compiler can only do a few simple checks on your behalf, such as:

* Check whether the number of arguments supplied to a constructor match
* Check whether the types supplied to a constructor match

The compiler is unable to perform the following checks:

* Are null values supplied to constructors?
* Do constructor invocations fail?
* Are dependencies injected into a component with a longer lifetime (the so called Captive Dependencies)?
* Are dependencies that are expected to have a certain lifestyle created more than once for the duration of that lifetime? (Problems known as [Torn Lifestyle](https://simpleinjector.org/diatl) and [Ambiguous Lifestyle](https://simpleinjector.org/diaal))
* Are disposable components not disposed when they go out of scope?

All these issues are relatively easy to spot when the number of components in the application is really small, but once that number starts to grow it’s very easy to lose track. A really strict coding style within your Composition Root does help but can easily go wrong when a team of developers is maintaining the Composition Root (opposed to having one single DI expert who has a really close watch on these types of issues).

It’s hard to define a threshold in application size for when a DI container outweighs Pure DI. In the business systems I help create, we almost always use a DI container for the central application (which often is a web application), while using Pure DI for small (background) Windows Services since they typically use just a fraction of the total business layer. Once the Composition Root starts to grow, tools that can verify and diagnose the correctness of the Composition Root become extremely valuable.

It is unfortunate that most DI containers have a limited set of capabilities when it comes to verifying their configuration (causing your application to fail silently). Simple Injector deals with all the previously stated issues and more. In Simple Injector it’s just a matter of following [good practices](https://simpleinjector.org/howto+verify-the-container-s-configuration) and calling `Container.Verify()` once you have completed the configuration of the container. Verification of your configuration gives you an increased level of confidence that all known object graphs are wired correctly at application start-up. Simple Injector can give more certainty than Pure DI, while keeping the benefits of, among other things, convention over configuration.