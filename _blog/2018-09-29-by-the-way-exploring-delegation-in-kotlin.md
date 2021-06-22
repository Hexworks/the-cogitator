---
excerpt: "Kotlin has an interesting keyword, 'by' which can be used for delegation. There is a lot of confusion around it so in this article we'll clean that up."
title:  "By the way - exploring delegation in Kotlin"
short_title:  "By the way - exploring delegation in Kotlin"
tags: [kotlin]
comments: true
updated_at: 2018-09-29
---

> Kotlin has an interesting keyword: `by` which can be used for delegation. There is a lot of confusion around it so in this article we'll clean that up.

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

```kotlin
class Rectangle(val width: Int, val height: Int) {
    fun area() = width * height
}

class Window(val bounds: Rectangle) {
    // Delegation
    fun area() = bounds.area()
}
```

As we have seen above Kotlin supports *delegation of an interface* and *delegation of accessors for a property*.
Let's take a look at how they work in detail.

## Delegation of an interface

For this exercise we'll use `Position`:

```kotlin
data class Position(val x: Int, val y: Int)
```

`Positionable`:

```kotlin
/**
 * Represents an object which has a [Position] on a 2D plane.
 */
interface Positionable {

    /**
     * Returns the [Position] of this [Positionable].
     */
    fun getPosition(): Position

    /**
     * Sets the [Position] of this [Positionable]
     */
    fun setPosition(position: Position)
}
```

and `DefaultPositionable` which is just an implementation of `Positionable`:

```kotlin
class DefaultPositionable(private var position: Position) : Positionable {

    override fun getPosition() = position

    override fun setPosition(position: Position) {
        this.position = position
    }
}
```

The above functionality can be used without the `by` keyword like this:

```kotlin
class RectNoBy(val width: Int, val height: Int, private val positionable: Positionable) : Positionable {

    override fun getPosition(): Position {
        return positionable.getPosition()
    }

    override fun setPosition(position: Position) {
        positionable.setPosition(position)
    }
}
```

This involves a lot of boilerplate code as seen in the above example. The purpose of the `by` keyword is to do away
with this kind of drudgery, therefore *it allows object composition to achieve the same code reuse as inheritance*:

```kotlin
/**
 *  `Positionable by DefaultPositionable(position)` says: "Let's implement `Positionable` by using
 *  `DefaultPositionable` as the implementation, thanks.
 */
class Rect(val width: Int, val height: Int, position: Position) : Positionable by DefaultPositionable(position)
```

So in a nutshell: with `by` we get [Item 16](https://github.com/mgp/book-notes/blob/master/effective-java-2nd-edition.markdown#item-16-favor-composition-over-inheritance)
from Effective Java for free. Pretty nifty, huh?

### Taking a look under the hood

Now that we know how it works on the outside let's check out the internals. This is really important as we'll see later.
This is the Java code which will be generated from the `Rect` implementation above:

> You can do this yourself in IDEA by using `Tools > Kotlin > Show Kotlin Bytecode` and then clicking `Decompile`.

```kotlin
public final class Rect implements Positionable {
   private final int width;
   private final int height;
   // $FF: synthetic field
   private final DefaultPositionable $$delegate_0;

   public final int getWidth() {
      return this.width;
   }

   public final int getHeight() {
      return this.height;
   }

   public Rect(int width, int height, @NotNull Position position) {
      Intrinsics.checkParameterIsNotNull(position, "position");
      super();
      this.$$delegate_0 = new DefaultPositionable(position);
      this.width = width;
      this.height = height;
   }

   @NotNull
   public Position getPosition() {
      return this.$$delegate_0.getPosition();
   }

   public void setPosition(@NotNull Position position) {
      Intrinsics.checkParameterIsNotNull(position, "position");
      this.$$delegate_0.setPosition(position);
   }
}
```

So what Kotlin does is that it takes our delegate, `DefaultPositionable`, puts it in the resulting class as a `private final`
field named `$$delegate_0` and adds implementations for all the methods of `Positionable` and in them it calls our delegate.

If by eyeballing the above code you had guessed that we can have multiple delegates in a class you'd be right!
Let's do just that by extracting `width` and `height` into its own delegate:

```kotlin
interface Sizable {

    fun getWidth(): Int

    fun getHeight(): Int
}

class DefaultSizable(private val width: Int,
                     private val height: Int) : Sizable {

    override fun getWidth() = width

    override fun getHeight() = height
}
```

With `Sizable` we can have a class which has the same functionality as `Rect` but with only delegates:

```kotlin
class RectWithSizable(width: Int, height: Int, position: Position)
    : Positionable by DefaultPositionable(position),
        Sizable by DefaultSizable(width, height)
```

The resulting Java code is this:

```kotlin
public final class RectWithSizable implements Positionable, Sizable {
   // $FF: synthetic field
   private final DefaultPositionable $$delegate_0;
   // $FF: synthetic field
   private final DefaultSizable $$delegate_1;

   public RectWithSizable(int width, int height, @NotNull Position position) {
      Intrinsics.checkParameterIsNotNull(position, "position");
      super();
      this.$$delegate_0 = new DefaultPositionable(position);
      this.$$delegate_1 = new DefaultSizable(width, height);
   }

   @NotNull
   public Position getPosition() {
      return this.$$delegate_0.getPosition();
   }

   public void setPosition(@NotNull Position position) {
      Intrinsics.checkParameterIsNotNull(position, "position");
      this.$$delegate_0.setPosition(position);
   }

   public int getHeight() {
      return this.$$delegate_1.getHeight();
   }

   public int getWidth() {
      return this.$$delegate_1.getWidth();
   }
}
```

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

```kotlin
class RectWithConstructor(val width: Int,
                          val height: Int,
                          positionable: Positionable) : Positionable by positionable
            
class RectWithProperty(val width: Int,
                       val height: Int,
                       val positionable: Positionable) : Positionable by positionable

class RectWithDefaultValue(val width: Int,
                           val height: Int,
                           position: Position,
                           val positionable: Positionable = DefaultPositionable(position))
    : Positionable by positionable      
    
class RectWithFunction(val width: Int,
                       val height: Int,
                       position: Position,
                       positionable: Positionable = fetchPositionable(position)) : Positionable by positionable



fun fetchPositionable(position: Position): Positionable = DefaultPositionable(position)  
```

`RectWithConstructor` lets us pass custom implementations of `Positionable` to our *Rect*. With `RectWithProperty`
*we can have access to our delegate* within our *Rect* and with `RectWithDefaultValue` we can achieve both
while being able to just pass a `Position` to the constructor if we are fine with the default value:

```kotlin
val rect = RectWithDefaultValue(10, 20, Position(5, 6))
```

`RectWithFunction` is a bit of an outlier and it is there to demonstrate that we can also have functions producing
delegates for us. This will be useful later when we'll try to understand property delegation.

> Note that `RectWithDefaultValue` achieves multiple inheritance of state as well.

### The pitfall of interface delegation

You might have wondered about what happens when we can not only pick an implementation for our delegate but
change it at runtime:

```kotlin
class RectWithMutableDelegate(val width: Int,
                              val height: Int,
                              var positionable: Positionable) : Positionable by positionable
```

This looks good, but if we take a peek at the generated Java code we'll see a discrepancy:

```kotlin
public final class RectWithMutableDelegate implements Positionable {
   private final int width;
   private final int height;
   @NotNull
   private Positionable positionable;
   // $FF: synthetic field
   private final Positionable $$delegate_0;

   public final int getWidth() {
      return this.width;
   }

   public final int getHeight() {
      return this.height;
   }

   @NotNull
   public final Positionable getPositionable() {
      return this.positionable;
   }

   public final void setPositionable(@NotNull Positionable var1) {
      Intrinsics.checkParameterIsNotNull(var1, "<set-?>");
      this.positionable = var1;
   }

   public RectWithMutableDelegate(int width, int height, @NotNull Positionable positionable) {
      Intrinsics.checkParameterIsNotNull(positionable, "positionable");
      super();
      this.$$delegate_0 = positionable;
      this.width = width;
      this.height = height;
      this.positionable = positionable;
   }

   @NotNull
   public Position getPosition() {
      return this.$$delegate_0.getPosition();
   }

   public void setPosition(@NotNull Position position) {
      Intrinsics.checkParameterIsNotNull(position, "position");
      this.$$delegate_0.setPosition(position);
   }
}
```

**Yep**, if we make our delegate a `var` and change it at runtime it won't change the actual delegate. It will happily
set the value of the `positionable` property and leave `$$delegate_0` unchanged. This can lead to **pretty nasty**
problems if you forget this so my advice is not to do this.

> If you try to hack around this by renaming `positionable` to `\`$$delegate_0`\` it won't work, you'll get a
> compiler error detailing a JVM name clash.

## Property delegation

You might have seen code like this before:

```kotlin
class MyClass {

    val myValue by lazy { 
        "I am lazy"
    }
}
```

and wondered about why we have a keyword for seemingly unrelated things. To fully understand how this works
we need to take a look at the difference between *properties* and *fields*.
 
### Properties and Fields
 
In Java we have **fields** and they look like this:

```kotlin
public class SomeClass {
    
    public String someValue = "xul";
}
```

There are no methods, no strings attached, just our solitary field. If we write the same class in Kotlin while it
might look similar:

```kotlin
class SomeClass {
    
    var someValue: String = "xul"
}
```

the Java code generated from it will be a bit different:

```kotlin
public final class SomeClass {
   @NotNull
   private String someValue = "xul";

   @NotNull
   public final String getSomeValue() {
      return this.someValue;
   }

   public final void setSomeValue(@NotNull String var1) {
      Intrinsics.checkParameterIsNotNull(var1, "<set-?>");
      this.someValue = var1;
   }
}
```

So here is the *difference*: **fields** are just solitary slots which store data, **properties** on the other hand
have *getters* and if we declared them as a `var` they will also have *setters*. This difference is very important
and we'll see how this makes it possible to have not only interface delegates but property delegates as well!

There are cases when the Kotlin compiler won't even declare a backing field for a property. For example with this
Kotlin code:

```kotlin
class ClassWithProperty {

    val foo: String
        get() = "foo"
}
```

the generated Java code will only contain a method, since the value of `foo` can't be changed and it is pre-determined:

```kotlin
public final class ClassWithProperty {
   @NotNull
   public final String getFoo() {
      return "foo";
   }
}
```

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
class SomeClass {

    val someValue: String by lazy {
        "wombat"
    }
}
```

Let's revisit the example above which has a `lazy` property delegate:

```kotlin
class SomeClass {

    val someValue: String by lazy {
        "wombat"
    }
}
```

If we take a look at what the `lazy` function does:

```kotlin
public actual fun <T> lazy(initializer: () -> T): Lazy<T> = SynchronizedLazyImpl(initializer)
```

we can see that it returns a `SynchronizedLazyImpl` which is a class that implements `Lazy`:

```kotlin
public interface Lazy<out T> {
    /**
     * Gets the lazily initialized value of the current Lazy instance.
     * Once the value was initialized it must not change during the rest of lifetime of this Lazy instance.
     */
    public val value: T

    /**
     * Returns `true` if a value for this Lazy instance has been already initialized, and `false` otherwise.
     * Once this function has returned `true` it stays `true` for the rest of lifetime of this Lazy instance.
     */
    public fun isInitialized(): Boolean
}
```

`Lazy` also comes with an extension function:

```kotlin
public inline operator fun <T> Lazy<T>.getValue(thisRef: Any?, property: KProperty<*>): T = value
```

which is there to implement the `getValue` operator.

> You can read more about operator overloading [here](https://kotlinlang.org/docs/reference/operator-overloading.html).

The main point here is that delegating to a property uses the exact same mechanisms behind the scenes as interface
delegation. The differences which cause the confusion are the `getValue`/`setValue` operators which we need to implement
within our property delegates but *not because of how delegation works, but because how overriding property access operators
work*.

Calling `lazy()` to delegate to an instance of `Lazy` is the same on the code level as calling `fetchPositionable()` in the example
above to delegate to an instance of `Positionable`.


### How to write our own delegate

Kotlin gives us two interfaces to use for this purpose: `ReadOnlyProperty` and `ReadWriteProperty`.

```kotlin
interface ReadOnlyProperty<in R, out T> {
    operator fun getValue(thisRef: R, property: KProperty<*>): T
}

interface ReadWriteProperty<in R, T> {
    operator fun getValue(thisRef: R, property: KProperty<*>): T
    operator fun setValue(thisRef: R, property: KProperty<*>, value: T)
}
```

This further reinforces the similarity between the two kinds of delegation. Let's take a look at how we can implement
a property which is not initialized during object construction but at a later time but still won't let us read
`null` values from it:

```kotlin
class NotNullVar<T : Any> : ReadWriteProperty<Any?, T> {
    private var value: T? = null

    public override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return value ?: throw IllegalStateException("Property ${property.name} should be initialized before get.")
    }

    public override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        this.value = value
    }
}
```

> Note that this class is in the Kotlin stdlib, but it is not well known that it is there.

We can also add a factory method to produce it:

```kotlin
public fun <T : Any> notNull(): ReadWriteProperty<Any?, T> = NotNullVar()
```

and we can use it like this:

```kotlin
class SomeClass {

    var someValue: String by notNull()
}
```

### Taking a look at the Java code

If we decompile the Java byte code from `SomeClass` above we'll see this:

```kotlin
public final class SomeClass {
   // $FF: synthetic field
   static final KProperty[] $$delegatedProperties = new KProperty[]{(KProperty)Reflection.mutableProperty1(new MutablePropertyReference1Impl(Reflection.getOrCreateKotlinClass(SomeClass.class), "someValue", "getSomeValue()Ljava/lang/String;"))};
   @NotNull
   private final ReadWriteProperty someValue$delegate = NotNullVarKt.notNull();

   @NotNull
   public final String getSomeValue() {
      return (String)this.someValue$delegate.getValue(this, $$delegatedProperties[0]);
   }

   public final void setSomeValue(@NotNull String var1) {
      Intrinsics.checkParameterIsNotNull(var1, "<set-?>");
      this.someValue$delegate.setValue(this, $$delegatedProperties[0], var1);
   }
}
```

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
