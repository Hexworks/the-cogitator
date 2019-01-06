---
excerpt: For someone who comes from the Java world the concept of Nothing might be confusing. In this article, I'll try to clean that up with some practical examples to boot.
title:  "The Nature of Nothing in Kotlin"
short_title:  "The Nature of Nothing in Kotlin"
tags: [kotlin]
comments: true
---
<div id="tldr">
For someone who comes from the Java world the concept of Nothing might be confusing. In this article, I'll try to clean that up with some practical examples to boot.
</div>

## What is `Nothing`?

Simply put, `Nothing` is a [bottom type](https://en.wikipedia.org/wiki/Bottom_type):

> In type theory, a theory within mathematical logic, the bottom type is the type that has no values.
> A function whose return type is bottom cannot return any value.

So this means that `Nothing` is something like `Void` in Java, and the opposite of `Any` in Kotlin, no?
Well, not quite. In Java, when a method has no return value we can declare its return type as `void`:

```java
public void someMethod() {
    // do nothing
}
```

In Kotlin, however, we already have a type which is returned from functions which don't declare a return type: [`Unit`](https://kotlinlang.org/docs/reference/functions.html#unit-returning-functions).

```kotlin
fun fnWithNoExplicitReturnType() {
    // returns Unit implicitly.
}

fun fnWithExplicitReturnType(): Unit {
    // same as above, with implicit return of Unit
}

fun fnWithExplicitReturn(): Unit {
    return Unit // same as above
}

// can be called like this
val unit: Unit = fnWithNoExplicitReturnType()
```

Although the documentation says it corresponds to `void` there is an important difference.
`Unit` has an actual instance, that's why we can assign it to a variable, like in the example above:

```kotlin
/**
 * The type with only one value: the Unit object. This type corresponds to the `void` type in Java.
 */
public object Unit {
    override fun toString() = "kotlin.Unit"
}
```

So what good `Nothing` does for us? Let's consider this function which does not return at all:

```kotlin
fun throwException() {
    throw RuntimeException("Oops...")
}  
```

This function would have returned `Unit` if it hadn't thrown an `Exception`. Now if you guess that this is exactly the
case of returning `Nothing` you are right. Let's look at the actual implementation of it:

```kotlin
/**
 * Nothing has no instances. You can use Nothing to represent "a value that never exists": for example,
 * if a function has the return type of Nothing, it means that it never returns (always throws an exception).
 */
public class Nothing private constructor()
```

Since it has a private constructor and it is not called from the class itself this means that there can't be any
instances of `Nothing`. `Nothing` is also a bottom type which means that it is a subtype of every other type (including `Unit`).

## Signaling exceptional conditions with `Nothing`

Now you might begin to wonder why all this stuff is useful? Let's say that we have a function which interoperates with Java
classes which might return `null`:

```kotlin
fun interact(): String? {
    // ...
}
```

If we call this in our code we have to do something about the nullability if we want a simple non-nullable `String`
out of this:

```kotlin
val result: String = interact() ?: throwException()
```

But wait... This doesn't compile because `throwException` returns `Unit` and we want to get a `String`. Let's try redefining
`throwException` in a more useful way:

```kotlin
fun throwException(): Nothing {
    throw RuntimeException("Oops...")
}

// this suddenly compiles
val result: String = interact() ?: throwException()
```

Our code now compiles. Why is that? The answer is that since `Nothing` is a subtype of every other class in Kotlin,
the compiler can accept it, and because `throwException` is never going to return successfully, its assignment will never
happen. Better yet now you can start using it to explicitly state that a function
will always throw an `Exception`.

Understanding how `Nothing` works might be a bit hard but you can develop a nice intuition for it if you imagine `Nothing`
as something which is the opposite of `Any`. `Any` is the supertype of everything (just like `Object` in Java) and `Nothing`
is the subtype of everything.

## Representing `null`

Now that you know how `Nothing` works you might start feeling warm and fuzzy...until you try writing a function like this:

```kotlin
fun throwException(): Nothing? {
    throw RuntimeException("Oops...")
}
```

If you try this with the previous example it no longer compiles. Why is that? Let's see if there is a value which can be
returned from a function which has the `Nothing?` return type:

```kotlin
fun returnNull(): Nothing? = null

val nullValue : Nothing? = returnNull()
```

Yes. The only value which can be returned from this method is `null`. This also means that `null` has the type of `Nothing?`. 
Although this is hardly useful in our day-to-day work now at least we understand it.

## Algebraic data types

[Algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type) are sometimes very useful and you might
already be using them even if you don't know it (a linked list for example). In Kotlin we can use `Nothing` if we
want to implement them to great effect. Let's take a look at a good example, `Either`:

```kotlin
sealed class Either<out L, out R> {

    data class Left<out L>(val value: L) : Either<L, Nothing>()
    
    data class Right<out R>(val value: R) : Either<Nothing, R>()

    fun isRight() = this is Right<R>

    fun isLeft() = this is Left<L>
}

fun <L> left(a: L) = Either.Left(a)
fun <R> right(b: R) = Either.Right(b)
```

`Either` represents a value which might be either `Left` or `Right`. `Left` traditionally means an exceptional condition
and `Right` usually is just a normal value but `Either` can't hold both values. We represent this concept with `Nothing`.
The right type of `Left` and the left type of `Right` is `Nothing`. 
We can then use `Either` for example as a return type instead of throwing exceptions:

```kotlin
fun parseInt(str: String) : Either<Exception, Int> {
    return str.toIntOrNull()?.let {
        right(it)
    } ?: left(NumberFormatException())
}

val result: Either<Exception, Int> = parseInt("xul")
println(result)

// Left(value=java.lang.NumberFormatException)
```

Note that if you replace `Nothing` with `Unit` or any other type it simply won't compile, but here it represents what
it was made for: nothingness.

Another fine example of using `Nothing` is a tree structure:

```kotlin
sealed class Tree<out T> {
    class Node<T>(val left: Tree<T>, val right: Tree<T>) : Tree<T>()
    class Leaf<T>(val value: T) : Tree<T>()
    object Empty : Tree<Nothing>()
}

val tree = Node(
              Leaf("Foo"),
              Node(Leaf("baz"), Empty))
```

Here `Nothing` represents a missing left or right `Node` in a `Tree`.

## Conclusion

Although `Nothing` might not be intuitive at first, once we learn how it works it quickly becomes a useful asset
in our toolkit. We can use it to indicate nothingness in data structures, to signal the inevitability of `Exception`s
being thrown or even `null`.

Now that we have all this under our belt let's *go forth* and Kode on!

