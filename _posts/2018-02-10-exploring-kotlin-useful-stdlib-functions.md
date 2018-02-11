---
layout: post
title:  "Exploring Kotlin: Useful Standard Library Functions"
excerpt: As others have written about this before
         Kotlin comes with a cornucopia of useful functions like let, apply, with or also. Less is written about what
         comes with the collections, ranges, and other packages of the standard library. In this article we'll explore them.
comments: true
---
<div id="tldr">
As <a href="http://beust.com/weblog/2015/10/30/exploring-the-kotlin-standard-library/">others have written about this before</a>
Kotlin comes with a cornucopia of useful functions like <code>let</code>, <code>apply</code>, <code>with</code>
 or <code>also</code>. Less is written about what comes with the <code>collections</code>, <code>ranges</code>,
 and other packages of the standard library. In this article we'll explore them.
</div>

## Tuples

Kotlin comes with `Pair` and `Triple` which are basic generic *tuples*:

{% gist 4e2c9ce8f402c13abbfed0b1c23c8c11 %}


These can be used in a lot of ways like implementing [multiple return types](https://discuss.kotlinlang.org/t/multiple-return-types-from-function/664),
or [destructuring](https://kotlinlang.org/docs/reference/multi-declarations.html) but most of the time we'll use them
for `Map`s.

`Pair` comes with an useful [`infix`](https://kotlinlang.org/docs/reference/functions.html#infix-notation) function, `to`
which lets us create one like this:

{% gist 5b514f7472945bb7f227c8e4c3e6edf2 %}

Then we can use this syntax to create `Map`s in a much more readable way:

{% gist 7e7c7026239636bbcbb5572215d9edd3 %}

## Collections

If you have worked with the Kotlin stdlib for a while you probably bumped into a bunch of library functions which
are either improvements over the Java versions or new additions by Kotlin.
You have [`map`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/-map/index.html),
[`filter`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/filter.html),
[`reduce`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/reduce.html) for example which are defined on
`Iterable` objects and there are a bunch of others which are defined on immutable `List`s, or `Set`s like `plus` or `minus`.

> Note that from now on we'll talk about operations which are defined for *immutable* collections.

Creating collections never have been easier. We have `listOf`, `setOf`, `mapOf` and even `arrayOf` to create the
corresponding collection.

What's interesting is that most of these operations are also defined for `Map`s.

> Note that a `Map` in Kotlin does not implement the `Collection` interface, but defines some operations for `Map`s
  which are counterparts for the ones in `Collection`.
 
### Maps
Kotlin treats `Map`s as collections of `Pair`s and we can create `Map`s from any collection which holds them:

{% gist 179dfb6340d411eeb3a2fc72682d0035 %}

In addition to `map`, `filter` and `reduce` we also have `mapValues`, `mapKeys`, `filterKeys` and `filterValues` for `Map`s.

`mapValues` and `mapKeys` will create new `Map`s when we call them. They are useful when we have a `Map` and we only
want to transform either the *keys* or the *values*. The `filter` variants follow the same logic but with filtering.
We can also combine them:

{% gist 163ddf242c3de018fdd68a71347ae3be %}

If we want to perform operations other than these we can turn our `Map` into a collection by calling `toList` or `asIterable`.

### Conversions

In addition to `toList`, `toMap` and other conversion functions there are also some specialized ones which are defined
on only some selected types like `Byte`, `Boolean` or `Char`. For example we can turn a `List` of `Byte`s to a `ByteArray`
like this:

{% gist d19e7bee50ae38748ef7bf265fbf9dbe %}

There is a `to*Array` defined for each primitive type. They all return a corresponding `*Array` type which is optimized.

### Immutable Collections

Immutable collections are perfect for functional programming since every operation defined on them returns a new version
of the old collection without changing it.
This also means that they are safe from a concurrency perspective since we don't need locks to work with them.
A problem though is that we lose operations like `removeAll` or `retainAll`.

Luckily most operations which work with mutable collections have an immutable counterpart.
`plus` and `minus` work like `add` and `remove` and we also have `subtract`, `union` and `intersect`. They work like
`removeAll`, `addAll` and `retainAll`:

{% gist b0ad20c2b19dbf9d83d1b85e8fa35fc5 %}

We can also work with collections in a way *offset* and *limit* works in *RDBMS*es. `drop` will return with a `List`
without the first `n` elements:

{% gist 34b7c0f519bce275d06c89fddaad8456 %}

`dropLast` works in the same way but drops elements from the end:

{% gist 54f023bbc936aa9a1efcfd15f47a3923 %}

There is also `dropWhile` and `dropLastWhile` which drops elements until a certain condition is met:

{% gist 7c99a6dededb84f25892e99449f1d149 %}

For all of the above functions there is a `take` variant which works like `drop` but it *takes* elements:

{% gist e6a0aa4626102277d92a660b53bae3fb %}

These are useful but sometimes we only want to pick *distinct* values. We can do so by calling `distinct` or
`distinctBy`. With `distinctBy` we can write our own selector function:

{% gist 6844d7634ef43caf64af5a3dd1c2d0db %}

We've already seen ways we can turn `Map`s to `List`s but can we do it the other way around? The answer is yes.
Kotlin comes with `groupBy`, `associate` and `associateBy` which lets us split our `List`s in different ways.

`groupBy` separates our `List` into multiple `List`s grouped by keys, so the result is a [multimap](https://en.wikipedia.org/wiki/Multimap).
We only need to provide a key selector function to do so. In this example we group a `List` of `Int`s into
even and odd groups:

{% gist 20b1fbe084deb90cd59554a8cdf581d4 %}

`associate` is different in a way that it **transforms** each element to a key-value pair and if multiple values map to the same key
only the last one is returned. In our previous list of `Int`s which are sorted this will effectively give us the greatest
odd and even numbers:

{% gist ccbd89372b7c7b72ab639cb50e214f0f %}

A variant to `associate` is `associateBy` which does not transform the original values but takes a *key selector*
function. If multiple elements would have the same key only the last one is added to the resulting `Map`. This is
an example which does the same as the previous one but with `associateBy`:

{% gist 4cf7c4c5a629380d76bc561051b1cab7 %}

`partition` is a special transformation function which groups to only a `Pair` of `List`s based on the result of
a predicate:

{% gist a40b991a3ed82b655755ff8bc535e2a1 %}

We've seen how we can split things but let's see what we have for *join*ing them!

`zip` will work exactly like the zipper of your trousers: it zips two `List`s into a `List` of `Pair`s:

{% gist eec2c7a7e643ad94a0b9e03d54864d98 %}

We can be a bit more sophisticated if we provide a transform function to `zip`: 

{% gist f4d7001ae235d820194c84af8f32d054 %}

`zipWithNext` will pair each element with the next:

{% gist 422e248fa5dd0135856adf8258b4ac4f %}

and it can also take a transform function:

{% gist 42008d3abdcd641a008fb78ecdda805c %}

Sometimes we want to transform our collections to a `String` representation. This is useful if we want to log
the contents of them for example. For this purpose we have `joinTo` which takes an `Appendable` and some extra
arguments (like a separator) and returns the `Appendable` with the contents of the collection appended:

{% gist a1966be18c80a57e75ca08170ef74cd9 %}

Since joining to a `String` is so common we also have `joinToString`:

{% gist 7faaa546dd4560140bb8795a9ec15694 %}

## Honorable mentions

The examples above are far from exhaustive but there are some interesting functions which are really useful sometimes.
For example we have the most commonly used `String` transformations as extension functions:

{% gist 011ec0eed630df022be4c9ca301ebfd9 %}

There are also `Range`s which are very useful for iteration. We can create them directly from numbers with
useful `infix` operations:

{% gist 50bb0da8237ce05317525a4cb17d6ff1 %}

## A real life example

These code snippets are nice but you might be wondering how a real life use case looks like. Let's see how we can
process some log information with what we have learned so far:

{% gist 8c0df247c4ce912a8a668c843db8d3cf %}

## Conclusion

We've only scratched the surface with the examples above but I hope that you now have an idea about what the Kotlin
stdlib has to offer. I strongly encourage you to open your IDE and take a look at these functions from the source.
They are documented very well so you can get started in no time.
