---
layout: post
title:  "The Nature of Nothing in Kotlin"
excerpt: For someone who comes from the Java world the concept of Nothing might be confusing. In this article, I'll try to clean that up with some practical examples to boot.
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

{% gist 06706a4e4b6aa1c3fffa0092482f09a6 %}

In Kotlin, however, we already have a type which is returned from functions which don't declare a return type: [`Unit`](https://kotlinlang.org/docs/reference/functions.html#unit-returning-functions).

{% gist 3f5a995d91b52182c281424757dfa97e %}

Although the documentation says it corresponds to `void` there is an important difference.
`Unit` has an actual instance, that's why we can assign it to a variable, like in the example above:

{% gist 154558b50fa85cadd7b85d91a2c17b9c %}

So what good `Nothing` does for us? Let's consider this function which does not return at all:

{% gist e212a555b4da323cac665f69155f885a %}

This function would have returned `Unit` if it hadn't thrown an `Exception`. Now if you guess that this is exactly the
case of returning `Nothing` you are right. Let's look at the actual implementation of it:

{% gist ea790c881e10b9fa0911f5bb9248794a %}

Since it has a private constructor and it is not called from the class itself this means that there can't be any
instances of `Nothing`. `Nothing` is also a bottom type which means that it is a subtype of every other type (including `Unit`).

## Signaling exceptional conditions with `Nothing`

Now you might begin to wonder why all this stuff is useful? Let's say that we have a function which interoperates with Java
classes which might return `null`:

{% gist d9c336a408127bf2134c63ec3f4732d6 %}

If we call this in our code we have to do something about the nullability if we want a simple non-nullable `String`
out of this:

{% gist ea25f2d93fdf18ccbdddb9d2ef5f73b3 %}

But wait... This doesn't compile because `throwException` returns `Unit` and we want to get a `String`. Let's try redefining
`throwException` in a more useful way:

{% gist a65c6fedd849874609d14e77718d3204 %}

Our code now compiles. Why is that? The answer is that since `Nothing` is a subtype of every other class in Kotlin,
the compiler can accept it, and because `throwException` is never going to return successfully, its assignment will never
happen. In short: `Nothing` is **magic**. Better yet now you can start using it to explicitly state that a function
will always throw an `Exception`.

Understanding how `Nothing` works might be a bit hard but you can develop a nice intuition for it if you imagine `Nothing`
as something which is the opposite of `Any`. `Any` is the supertype of everything (just like `Object` in Java) and `Nothing`
is the subtype of everything.

## Representing `null`

Now that you know how `Nothing` works you might start feeling warm and fuzzy...until you try writing a function like this:

{% gist 884a0ba4b5242b97135c0544fc38d86f %}

If you try this with the previous example it no longer compiles. Why is that? Let's see if there is a value which can be
returned from a function which has the `Nothing?` return type:

{% gist 6f367272c8a5cc06613fa77b67789932 %}

Yes. The only value which can be returned from this method is `null`. This also means that `null` has the type of `Nothing?`. 
Although this is hardly useful in our day-to-day work now at least we understand it.

## Algebraic data types

[Algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type) are sometimes very useful and you might
already be using them even if you don't know it (a linked list for example). In Kotlin we can use `Nothing` if we
want to implement them to great effect. Let's take a look at a good example, `Either`:

{% gist af19dc65f93577a77038b981045bd685 %}

`Either` represents a value which might be either `Left` or `Right`. `Left` traditionally means an exceptional condition
and `Right` usually is just a normal value but `Either` can't hold both values. We represent this concept with `Nothing`.
The right type of `Left` and the left type of `Right` is `Nothing`. 
We can then use `Either` for example as a return type instead of throwing exceptions:

{% gist 0abe7bcf1245004e547eb0cce1fd196c %}

Note that if you replace `Nothing` with `Unit` or any other type it simply won't compile, but here it represents what
it was made for: nothingness.

Another fine example of using `Nothing` is a tree structure:

{% gist 06d8075e5f721418a1f90101d8e44be4 %}

Here `Nothing` represents a missing left or right `Node` in a `Tree`.

## Conclusion

Although `Nothing` might not be intuitive at first, once we learn how it works it quickly becomes a useful asset
in our toolkit. We can use it to indicate nothingness in data structures, to signal the inevitability of `Exception`s
being thrown or even `null`.

Now that we have all this under our belt let's *go forth* and Kode on!

