---
title: Ignore Null Strategies
date: "2000-01-02"
---

> There's no such thing as a null strategy!

Here is a simple refactoring/micro-pattern that can help remove a little complexity from classes that use passed-in behaviours/strategies/handlers/whatever you want to call them. [^n]
The examples given here may seem rather abstract or “enterprisey”, but it’s something that comes up repeatedly when developing software.

### The problem: Checking strategies for nullness

Below is a very simplified example of a consumer (the class `ProducerExample`) of the `IProducer` strategy. The consumer explicitly checks if the strategy is null before invoking it, and if it *is* null, it produces some default value instead.

This extra null check gets in the way of what the class is actually doing and introduces an extra code path that is not wanted (or needed).

<script src="https://gist.github.com/Porges/0a079f80ec0b90c4aff9b23ddc3b89c8.js?file=0-producer.cs"></script>

(The producer here is deliberately simple—in a real codebase this could take any number of arguments.)

One situation where this kind of code might appear is when a previously-untested class is being tested for the first time. In this case, dummy values might be inserted in order that `null` can be passed as the strategy at test time and you’ll still get results instead of a `NullReferenceException`. This kind of setup is dangerous, as inadvertently passing `null` in the production code will end up generating these dummy values! 💥

Here's a very similar example, using a strategy that doesn't have a result (which I'll call a “handler”). This is checked for nullness before it is invoked for its side-effects:

<script src="https://gist.github.com/Porges/0a079f80ec0b90c4aff9b23ddc3b89c8.js?file=1-handler.cs"></script>

### Solution: Don't check strategies for nullness!

Instead of explicitly checking for nullness, assume that the strategy is not null (you can enforce this in the constructor) and migrate any “default action” into a new implementation of the strategy.

Then, anywhere you were previously passing in `null`, you can instead pass an instance of your shiny new implementation.

For the producer example, we can move the default implementation into a new class, and remove the null check:

<script src="https://gist.github.com/Porges/0a079f80ec0b90c4aff9b23ddc3b89c8.js?file=2-producer2.cs"></script>

(The `[NotNull]` annotation here is something supported by Resharper via the [Resharper.Annotations](https://www.nuget.org/packages/JetBrains.Annotations) package.)


If we need to change the result of the producer depending on what the consumer is, we can use a variation on this, and create an implementation that stores any value we want:

<script src="https://gist.github.com/Porges/0a079f80ec0b90c4aff9b23ddc3b89c8.js?file=3-producer2-2.cs"></script>


In the “handler” case, we can create a [‘null object’ implementation](https://en.wikipedia.org/wiki/Null_Object_pattern) that does nothing, since we don't need to produce a result:

<script src="https://gist.github.com/Porges/0a079f80ec0b90c4aff9b23ddc3b89c8.js?file=4-handler2.cs"></script>

With the refactored code, the consuming classes are cleaner (less code, lower cyclomatic complexity), and we have extracted a “default” implementation, which could potentially be used by other consumers.

At the same time, we have created some useful additional implementations that we can use in unit tests! The “Default” or “Constant” producers are useful when providing canned data to classes that are being tested, and “Null” handlers are useful when ignoring part of the behaviour of a class in order to test other parts.

---

#### Variation: Function-oriented implementation

For single-method interfaces such as those above we can replace them with delegates. This can lead to much cleaner code.

For producers:

<script src="https://gist.github.com/Porges/0a079f80ec0b90c4aff9b23ddc3b89c8.js?file=5-producer-functional.cs"></script>

And for handlers:

<script src="https://gist.github.com/Porges/0a079f80ec0b90c4aff9b23ddc3b89c8.js?file=6-handler-functional.cs"></script>

I have yet to explore this style in-depth myself, but it seems promising. (I would avoid using pure `Func<T, ...>` as it doesn't give any indication of what the intention of the code is.)

The nice thing about delegates is that they will implicitly convert any compatible lambda; so if you need a one-off implementation in test code, you can write it directly in your test method, and not have to create an entirely new class.

[^n]: Here I'm going to use strategy in a broad sense, to basically include any behaviour that is injected into a class. Other people might call these collaborators (although I'd apply that to the implementations of the strategy and not the strategy abstraction itself), or something different.
