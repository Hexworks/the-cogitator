---
layout: post
title:  "By the way - exploring delegation in Kotlin"
excerpt: "Kotlin has an interesting keyword, 'by' which can be used for delegation. There is a lot of confusion around it so in this article we'll clean that up."
comments: true
---

<div id="tldr">
Kotlin has an interesting keyword: `by` which can be used for delegation. There is a lot of confusion around it so in this article we'll clean that up.
</div>

## A primer on delegation

If we check the [keyword reference page](https://kotlinlang.org/docs/reference/keyword-reference.html) of Kotlin `by` is listed there
and it says that it is capable of two different things:

> `by`
>  - delegates the implementation of an interface to another object
>  - delegates the implementation of accessors for a property to another object

I think this is the source of most confusion so after reading this article you'll be able to see that these two things are essentially
the same in *Kotlin*.

Before we do that we should clarify what delegation is in the first place. By perusing the related [Wikipedia page](https://en.wikipedia.org/wiki/Delegation_pattern)
we can find the following definition:

> In software engineering, the delegation pattern is an object-oriented design pattern that allows object composition
 to achieve the same code reuse as inheritance.

I think this is a pretty accurate description and it also comes with a Kotlin code example!

{% gist b24a5ea2f347f3eac2a0596b03a367d1 %}

As we have seen above Kotlin supports *delegation of an interface* and *delegation of accessors for a property*.
Let's take a look at how they work in detail.

## Delegation of an interface

For this exercise we'll use `Position`:

{% gist 245e7bc68c775219075065cb206dc199 %}

`Positionable`:

{% gist 81eab43dbb6f72c3b94f05059cf9f843 %}

and `DefaultPositionable` which is just an implementation of `Positionable`:

{% gist 10691118c7d5986dea20082e1575e4c1 %}

The above functionality can be used without the `by` keyword like this:

{% gist 30b231622edbe41efa013aca6c37eff1 %}

This involves a lot of boilerplate code as seen in the above example. The purpose of the `by` keyword is to do away
with this kind of drudgery, therefore *it allows object composition to achieve the same code reuse as inheritance*:

{% gist 8326cf6c2a544ddea79f8366cc62cd81 %}

So in a nutshell: with `by` we get [Item 16](https://github.com/mgp/book-notes/blob/master/effective-java-2nd-edition.markdown#item-16-favor-composition-over-inheritance)
from Effective Java for free. Pretty nifty, huh?

### Taking a look under the hood

Now that we know how it works on the outside let's check out the internals. This is really important as we'll see later.
This is the Java code which will be generated from the `Rect` implementation above:

> You can do this yourself in IDEA by using `Tools > Kotlin > Show Kotlin Bytecode` and then clicking `Decompile`.

{% gist c4444ed570a47220842070aabbfaf790 %}

So what Kotlin does is that it takes our delegate, `DefaultPositionable`, puts it in the resulting class as a `private final`
field named `$$delegate_0` and adds implementations for all the methods of `Positionable` and in them it calls our delegate.

If by eyeballing the above code you had guessed that we can have multiple delegates in a class you'd be right!
Let's do just that by extracting `width` and `height` into its own delegate:

{% gist 23868b009b9ff6ac76e2c80471fd892c %}

With `Sizable` we can have a class which has the same functionality as `Rect` but with only delegates:

{% gist cc572a7dd2e95c0852f0058524558e9a %}

The resulting Java code is this:

{% gist 30f3d827d4d921ace9e36547bb38d33a %}

> This also means that with delegation we can unit test our building blocks in isolation (`DefaultPositionable` and
`DefaultSizable` here) which is a significant improvement over inheritance in itself.

So it turns out that with the `by` keyword we have a really powerful alternative to inheritance and the only limitation
is that we have to use `interface`s on the right hand side of `by`: Kotlin can only delegate to interfaces.

> This also means that Kotlin supports [Mixin](https://en.wikipedia.org/wiki/Mixin)s or 
> [Multiple inheritance](https://docs.oracle.com/javase/tutorial/java/IandI/multipleinheritance.html).
> 
> Remember that multiple inheritance comes in 3 "flavors": inheritance of state, implementation and type.
> Java originally only supported multiple inheritance of type (with interfaces), and with the advent of
> Java 8 it added support for multiple inheritance of implementation (with default methods).
> By using the `by` keyword we can have multiple inheritance of *state* as well!

### Variations of interface delegation

There are multiple ways of using interface delegation and not all of them work in a way as you would expect.
`Rect` in fact could have been defined with all of the examples below to similar effect but slight differences
in utility:

{% gist 8f9fcea767fd9900100be94186fb34b2 %}

`RectWithConstructor` lets us pass custom implementations of `Positionable` to our *Rect*. With `RectWithProperty`
*we can have access to our delegate* within our *Rect* and with `RectWithDefaultValue` we can achieve both
while being able to just pass a `Position` to the constructor if we are fine with the default value:

{% gist 55101f6f6a0b221de6e6f31719bbb295 %}

`RectWithFunction` is a bit of an outlier and it is there to demonstrate that we can also have functions producing
delegates for us. This will be useful later when we'll try to understand property delegation.

> Note that `RectWithDefaultValue` achieves multiple inheritance of state as well.

### The pitfall of interface delegation

You might have wondered about what happens when we can not only pick an implementation for our delegate but
change it at runtime:

{% gist 13f80e8c371f42bd23516ea511e21d31 %}

This looks good, but if we take a peek at the generated Java code we'll see a discrepancy:

{% gist cd3bfef23eea0ee4b1f8764d81c1f60f %}

**Yep**, if we make our delegate a `var` and change it at runtime it won't change the actual delegate. It will happily
set the value of the `positionable` property and leave `$$delegate_0` unchanged. This can lead to **pretty nasty**
problems if you forget this so my advice is not to do this.

> If you try to hack around this by renaming `positionable` to `$$delegate_0` it won't work, you'll get a
> compiler error detailing a JVM name clash.

## Property delegation

You might have seen code like this before:

{% gist 5b9ea2fb7dcdecceeb7cdb1ca20c9fc3 %}

and wondered about why we have a keyword for seemingly unrelated things. To fully understand how this works
we need to take a look at the difference between *properties* and *fields*.
 
### Properties and Fields
 
In Java we have **fields** and they look like this:

{% gist 288fcb84c6e25459bd6fd5151b6f07e7 %}

There are no methods, no strings attached, just our solitary field. If we write the same class in Kotlin while it
might look similar:

{% gist b8255ce5ef58aab31ec1510be2d773e8 %}

the Java code generated from it will be a bit different:

{% gist 634bcd65ed1efc294db8a521b085ee62 %}

So here is the *difference*: **fields** are just solitary slots which store data, **properties** on the other hand
have *getters* and if we declared them as a `var` they will also have *setters*. This difference is very important
and we'll see how this makes it possible to have not only interface delegates but property delegates as well!

There are cases when the Kotlin compiler won't even declare a backing field for a property. For example with this
Kotlin code:

{% gist b47ecf97509132b4a72ade9c5338bfab %}

the generated Java code will only contain a method, since the value of `foo` can't be changed and it is pre-determined:

{% gist 6558d25dca2f1b39adf4e874372bf268 %}

> There are more to properties but I won't explain them here, but the Kotlin docs is rather verbose about this topic.
> You can check it out [here](https://kotlinlang.org/docs/reference/properties.html).

### Peeking at an actual property delegate

With this knowledge we have everything to understand how property delegation works.
If we take a look [at the docs](https://kotlinlang.org/docs/reference/delegated-properties.html) for this topic 
(which I recommend reading thoroughly) we can see this:

> There are certain common kinds of properties, that, though we can implement them manually every time we need them,
> would be very nice to implement once and for all, and put into a library. Examples include:
>  
> - lazy properties: the value gets computed only upon first access;
> - observable properties: listeners get notified about changes to this property;
> - storing properties in a map, instead of a separate field for each property.
> 
> To cover these (and other) cases, Kotlin supports delegated properties:

```kotlin

```

Let's revisit the example above which has a `lazy` property delegate:

{% gist 5ce9ef5f91620fdc20f182a994a08854 %}

If we take a look at what the `lazy` function does:

{% gist dd8b9a729da4f15aa4b6479e3e459e00 %}

we can see that it returns a `SynchronizedLazyImpl` which is a class that implements `Lazy`:

{% gist 52125c1c03bfaf9b31a7687d8acc1edd %}

`Lazy` also comes with an extension function:

{% gist 975f8b2aaa803e70d6ca6bc572db36e5 %}

which is there to implement the `getValue` operator.

> You can read more about operator overloading [here](https://kotlinlang.org/docs/reference/operator-overloading.html).

The main point here is that delegating to a property uses the exact same mechanisms behind the scenes as interface
delegation. The differences which cause the confusion are the `getValue`/`setValue` operators which we need to implement
within our property delegates but *not because of how delegation works, but because how overriding property access operators
work*.

Calling `lazy` to delegate to an instance of `Lazy` is the same on the code level as calling `fetchPositionable()` in the example
above to delegate to an instance of `Positionable`.


### How to write our own delegate

Kotlin gives us two interfaces to use for this purpose: `ReadOnlyProperty` and `ReadWriteProperty`.

{% gist 3a1618e227b726937a3d89a3ac3f3110 %}

This further reinforces the similarity between the two kinds of delegation. Let's take a look at how we can implement
a property which is not initialized during object construction but at a later time but still won't let us read
`null` values from it:

{% gist 2bf26217a968a4d9366b44f8d643868a %}

> Note that this class is in the Kotlin stdlib, but it is not well known that it is there.

We can also add a factory method to produce it:

{% gist 72a14026d93365be2b05175e09a02aa9 %}

and we can use it like this:

{% gist baddd89fd36a8f3725e991f06b8db52d %}

### Taking a look at the Java code

If we decompile the Java byte code from `SomeClass` above we'll see this:

{% gist 30baa121176ef59fb4f28b573165a56c %}

Looks strangely familiar, no? Kotlin generates a *field* for our delegate (`someValue$delegate`) and uses it
when either the getter or the setter of our property is called.

## Summary

By having *properties* instead of *fields* property delegation can work out of the box because *properties*
come with getters and setters so Kotlin can provide custom implementations for them when we want to
user property delegates.

With interface delegation we can have all three kinds of multiple inheritance: state, implementation, and type.
It also lets us use composition without the boilerplate.

And also we now understand perfectly what the documentation says about delegation:
With *interface delegation* we **delegate the implementation of an interface to another object**
and with *property delegation* we **delegate the implementation of accessors for a property to another object**.

Now, with all this new knowledge under our belt let's **go forth, and Kode on!**
































