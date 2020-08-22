---
title:  "Exploring Kotlin: Useful Standard Library Functions"
short_title:  "Exploring Kotlin: Useful Standard Library Functions"
tags: [kotlin]
excerpt: Kotlin comes with a lot of useful functions like let, apply, with or also. Less is written about what
         comes with the collections, ranges, and other packages of the standard library. In this article we'll explore them.
comments: true
---

> As [others have written about this before](http://beust.com/weblog/2015/10/30/exploring-the-kotlin-standard-library/)
Kotlin comes with a lot of handy functions like `let`, `apply`, `with` or `also`.
Less is written about what comes with the `collections`, `ranges`, and other packages of the standard library.
I think that a lot more can be done using just the Kotlin standard library, so let's explore it in depth!

## Tuples

Kotlin comes with `Pair` and `Triple` which are basic generic *tuples*:

```kotlin
Pair("foo", "bar")

Triple(1, "wom", "bat")
```

Java does not have them so you might ask why are these useful?

Ever wanted to [return two values](https://discuss.kotlinlang.org/t/multiple-return-types-from-function/664) from a function?
With `Pair`s this is rather easy to accomplish, you just have to use it as a return type. What's more useful is that we can 
use [destructuring](https://kotlinlang.org/docs/reference/multi-declarations.html) to split a `Pair` into two values:

```kotlin
fun someFunction(): Pair<String, Int> = Pair("foo", 1)

// ...

val (foo, one) = someFunction() // foo will be the first, one will be the second value of a Pair
```

We can put tuples to good use when we work with `Map`s as well.

`Pair` comes with an useful [`infix`](https://kotlinlang.org/docs/reference/functions.html#infix-notation) function, `to`
which lets us create a `Pair` like this:

```kotlin
1 to 3
```

Then we can use this syntax to create `Map`s in a much more readable way:

```kotlin
mapOf(1 to 3, 4 to 2)
```

If you come from Java these might be a bit odd at first, but once you get used to it using `infix` functions and tuples will become second nature!

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

```kotlin
listOf(1 to 2, 2 to 3, 5 to 1).toMap()
```

In addition to `map`, `filter` and `reduce` we also have `mapValues`, `mapKeys`, `filterKeys` and `filterValues` for `Map`s.

`mapValues` and `mapKeys` will create new `Map`s when we call them. They are useful when we have a `Map` and we only
want to transform either the *keys* or the *values*. The `filter` variants follow the same logic but with filtering.
We can also combine them:

```kotlin
val dirtyData = mapOf("1" to "foo", "2" to "bar", "baz" to "qux")

val cleanData = dirtyData
        .filterKeys { it.toIntOrNull() != null }
        .mapKeys { it.key.toInt() }

// {1=foo, 2=bar}
```

If we want to perform operations other than these we can turn our `Map` into a collection by calling `toList` or `asIterable`.

These operations might be a bit odd if you come from Java, but after a while it makes sense if you start to think about `Map`s as a sequence of key-value `Pair`s.

### Conversions

In addition to `toList`, `toMap` and other conversion functions there are also some specialized ones which are defined
on only some selected types like `Byte`, `Boolean` or `Char`. For example we can turn a `List` of `Byte`s to a `ByteArray`
like this:

```kotlin
listOf<Byte>(1, 2, 3).toByteArray()
```

There is a `to*Array` defined for each primitive type. They all return a corresponding `*Array` type which is optimized.

## Immutable Collections

> Note that these collections in Kotlin are not immutable in fact, it is only the interface which does not allow mutation.
 There are some pitfalls to this. Take a look at my [other article about the topic](http://the-cogitator.com/2017/10/02/kotlin-pitfalls-and-how-to-avoid-them.html#java-interop-with-unmodifiable-collections)
 where I explain this problem.

Immutable collections are perfect for functional programming since every operation defined on them returns a new version
of the old collection without changing it.
This also means that they are safe from a concurrency perspective since we don't need locks to work with them.
A problem though is that we lose operations like `removeAll` or `retainAll`.

Luckily most operations which work with mutable collections have an immutable counterpart.
`plus` and `minus` work like `add` and `remove` and we also have `subtract`, `union` and `intersect`. They work like
`removeAll`, `addAll` and `retainAll`:

```kotlin
val mySet = setOf(1, 2, 3, 4, 5, 6)

val union = mySet.union(setOf(7, 8, 9))

// [1, 2, 3, 4, 5, 6, 7, 8, 9]

val intersection = mySet.intersect(setOf(3, 4, 5, 11, 12))

// [3, 4, 5]

val difference = mySet.subtract(setOf(1, 2, 3))

// [4, 5, 6]
```

### Drop and take

We can also work with collections in a way *offset* and *limit* works in *RDBMS*es. `drop` will return with a `List`
without the first `n` elements:

```kotlin
val myList = listOf(1, 2, 3, 4, 5, 6)

myList.drop(4)

// [5, 6]
```

`dropLast` works in the same way but drops elements from the end:

```kotlin

myList.dropLast(4)

// [1, 2]
```

There is also `dropWhile` and `dropLastWhile` which drops elements until a certain condition is met:

```kotlin
myList.dropWhile { it < 5 }

// [5, 6]

myList.dropLastWhile { it > 3 }

// [1, 2, 3]
```

For all of the above functions there is a `take` variant which works like `drop` but it *takes* elements:

```kotlin
myList.take(3)

// [1, 2, 3]

myList.takeLast(3)

// [4, 5, 6]

myList.takeWhile { it < 5 }

// [1, 2, 3, 4]

myList.takeLastWhile { it > 5 }

// [6]
```

If you come from LISP these functions might be familiar to you: they are like `first` and `rest` in Clojure for example.

In Kotlin you can define `first` as `take(1)` and `rest` as `drop(1)`.

### Calculating distinct values

These are useful but sometimes we only want to pick *distinct* values. We can do so by calling `distinct` or
`distinctBy`. With `distinctBy` we can write our own selector function:

```kotlin
val listWithDuplicates = listOf(1, 1, 2, 2, 3, 4)

listWithDuplicates.distinct()

// [1, 2, 3, 4]

val chars = listOf("a", "A", "b", "B", "c")

chars.distinctBy { it.toLowerCase() }

// [a, b, c]
```

### Grouping collections

We've already seen ways we can turn `Map`s to `List`s but can we do it the other way around? The answer is yes.
Kotlin comes with `groupBy`, `associate` and `associateBy` which lets us split our `List`s in different ways.

`groupBy` separates our `List` into multiple `List`s grouped by keys, so the result is a [multimap](https://en.wikipedia.org/wiki/Multimap).
We only need to provide a key selector function to do so. In this example we group a `List` of `Int`s into
even and odd groups:

```kotlin
val items = listOf(1, 2, 3, 4)

items.groupBy { if (it % 2 == 0) "even" else "odd" }

// {odd=[1, 3], even=[2, 4]}
```

`associate` is different in a way that it **transforms** each element to a key-value pair and if multiple values map to the same key
only the last one is returned. In our previous list of `Int`s which are sorted this will effectively give us the greatest
odd and even numbers:

```kotlin
items.associate { (if (it % 2 == 0) "greatest_even" else "greatest_odd") to it }

// {greatest_odd=3, greatest_even=4}
```

A variant to `associate` is `associateBy` which does not transform the original values but takes a *key selector*
function. If multiple elements would have the same key only the last one is added to the resulting `Map`. This is
an example which does the same as the previous one but with `associateBy`:

```kotlin
items.associateBy { if(it % 2 == 0) "greatest_even" else "greatest_odd" }

// {greatest_odd=3, greatest_even=4}
```

`partition` is a special transformation function which groups to only a `Pair` of `List`s based on the result of
a predicate:

```kotlin
val items = listOf(1, 2, 3, 4)

val x = items.partition { it % 2 == 0 }

// ([2, 4], [1, 3])
```

### Joining collections

We've seen how we can split things but let's see what we have for *join*ing them!

`zip` will work exactly like the zipper of your trousers: it zips two `List`s into a `List` of `Pair`s:

```kotlin
val names = listOf("Jon", "John", "Jane")
val ages = listOf(23, 32, 28)

names.zip(ages)

// [(Jon, 23), (John, 32), (Jane, 28)]
```

We can be a bit more sophisticated if we provide a transform function to `zip`: 

```kotlin
data class User(val name: String, val age: Int)

val names = listOf("Jon", "John", "Jane")
val ages = listOf(23, 32, 28)

names.zip(ages, { name, age -> User(name, age)})

// [User(name=Jon, age=23), User(name=John, age=32), User(name=Jane, age=28)]
```

`zipWithNext` will pair each element with the next:

```kotlin
val nodes = listOf("A", "B", "C", "D")

val x = nodes.zipWithNext()

// [(A, B), (B, C), (C, D)]
```

and it can also take a transform function:

```kotlin
data class Node(val value: String, val edges: List<String>)

val nodes = listOf("A", "B", "C", "D")

nodes.zipWithNext { node, edge -> Node(node, listOf(edge))}

// [Node(value=A, edges=[B]), Node(value=B, edges=[C]), Node(value=C, edges=[D])]
```

Sometimes we want to transform our collections to a `String` representation. This is useful if we want to log
the contents of them for example. For this purpose we have `joinTo` which takes an `Appendable` and some extra
arguments (like a separator) and returns the `Appendable` with the contents of the collection appended:

```kotlin
val numbers = listOf(1, 2, 3, 4)

val stringBuilder = StringBuilder()

val x = numbers.joinTo(stringBuilder, ", ")

// 1, 2, 3, 4
```

Since joining to a `String` is so common we also have `joinToString`:

```kotlin
val numbers = listOf(1, 2, 3, 4)

numbers.joinToString(", ")

// 1, 2, 3, 4
```

## The takeaway

As you have seen from the previous examples Kotlin collections work a bit differently from Java ones. When we use Java we have the Stream API which lets us perform most of these operations but they come at a price: we need to convert between streams and collections, and they also come with more boilerplate.

Kotlin does not differentiate between streams and collections. All of the above funtions are defined for `Iterable`s, `Collection`s, `Map`s or `List`s. This lets us write programs more fluently and at the end of the day we'll end up with more readable code, by doing less work.

Immutable collections are an added bonus: they help us write code which is less error-prone, without the need to write more.
And since we can't mutate them we'll have less concurrency issues, like race conditions or deadlocks.

## Honorable mentions

The examples above are far from exhaustive but there are some interesting functions which are really useful sometimes.
For example we have the most commonly used `String` transformations as extension functions:

```kotlin
"hello, world".capitalize()

// Hello, world

"HELLO, WORLD".toLowerCase().capitalize()

// Hello, world

"good bye".toUpperCase()

// GOOD BYE

"234343423434234".toBigInteger()

"2342342342.23423423".toBigDecimal()
```

There are also `Range`s which are very useful for iteration. We can create them directly from numbers with
useful `infix` operations:

```kotlin
(0 until 10).forEach {
    print("$it ")
}

// 0 1 2 3 4 5 6 7 8 9

(0 .. 10).forEach {
    print("$it ")
}

// 0 1 2 3 4 5 6 7 8 9 10

(10 downTo 0).forEach {
    print("$it ")
}

// 10 9 8 7 6 5 4 3 2 1 0

(10 downTo 0 step 2).forEach {
    print("$it ")
}

// 10 8 6 4 2 0
```

## Conclusion

We've only scratched the surface with the examples above but I hope that you now have an idea about what the Kotlin
stdlib has to offer. I strongly encourage you to open your IDE and take a look at these functions from the source.
They are documented very well so you can get started in no time.
