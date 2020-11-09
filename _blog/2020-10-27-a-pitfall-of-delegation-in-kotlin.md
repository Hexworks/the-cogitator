---
excerpt: "Delegation is a great tool that can help a lot with reducing boilerplate. It is not free of its pitfalls however as we'll see in this article."
title: "A Pitfall of Delegation in Kotlin"
tags: [kotlin, delegation]
author: addamsson
short_title: "A Pitfall of Delegation in Kotlin"
comments: true
---

> Delegation is a great tool that can help a lot with reducing boilerplate. It is not free of its pitfalls however as we'll see in this article.

## What's a Mixin?

I've been working on a library recently that takes advantage of composition. What I've found is that in a lot of the cases
inheritance becomes a brittle construct after a while, so I started using *mixins*.

A *mixin* is a piece of behavior that provides an `interface` that other classes can implement, like this:

```kotlin
interface Movable {

    val position: Position

    fun moveTo(position: Position)

    fun moveBy(position: Position) = moveTo(this.position + position)

}
```

for this example we'll also need a `Position` implementation:

```kotlin
data class Position(
        val x: Int,
        val y: Int
) {
    operator fun plus(other: Position): Position {
        return Position(x + other.x, y + other.y)
    }

    operator fun minus(other: Position): Position {
        return Position(x - other.x, y - other.y)
    }
}
```

In order to call this construct a *mixin* we need a concrete implementation of `Movable` that can be delegated to by its implementors:

```kotlin
class MovableImpl(
        initialPosition: Position = Position(0, 0)
) : Movable {

    override var position: Position = initialPosition
        private set

    override fun moveTo(position: Position) {
        this.position = position
    }

}
```

> Note that we can also have *mixin*s that don't have *state* in them. Those mixins can be implemented by using an interface
> only where we provide implementations for all functions. Such a *mixin* is called a **trait**.

Now, we can try this out and see that it works as intended:

```kotlin
val movable = MovableImpl()

movable.moveTo(Position(3, 4))
println(movable.position)
// Position(x=3, y=4)

movable.moveBy(Position(1, 2))
println(movable.position)
// Position(x=4, y=6)
```

All that's left to do is to start using this in an implementation class. In our case we're working on an UI library
that has UI `Component`s. These components can be moved around so we need to use the `Movable` *mixin*:

```kotlin
interface Component : Movable

class ComponentImpl(
        initialPosition: Position = Position(0, 0)
) : Component, Movable by MovableImpl(initialPosition)
```

We can see that our `ComponentImpl` didn't have to override any of the methods in `Movable` as we delegate the
implementation of the `Movable` interface to `MovableImpl`.  Let's see how it works:

```kotlin
val component = ComponentImpl()

component.moveTo(Position(3, 4))
println(component.position)
// Position(x=3, y=4)
component.moveBy(Position(1, 2))
println(component.position)
// Position(x=4, y=6)
```

Congratulations! You've written and used a *mixin*!

All is good...or is it?

## The Problem

Now let's take a look at a case where this construct will break down. Let's introduce a new type of `Component`,
a `Container`:

```kotlin
interface Container: Component {
    val children: List<Component>
}
```

A `Container` is a regular `Component` in most aspects except it has `children`. This is very common in UI systems.
For example in *HTML* a `div` can be considered a container and we can add `span` elements to it. A `span` is a regular
`Component` as it can't have children.

Implementing a `Container` is pretty similar to implementing a `Component`.

> Our implementation will take its `children` as a constructor parameter to maintain brevity.

```kotlin
class ContainerImpl(
        initialPosition: Position = Position(0, 0), 
        override val children: List<Component> = listOf()
) : Container, Movable by MovableImpl(initialPosition)
```

Now we have a problem though. Moving the `Container` around won't move its `children` so we'll have to override the `moveTo` function.

We know that `Movable` has a default implementation for `moveBy` that will call `moveTo`, so we don't have to override that:

```kotlin
class ContainerImpl(
        initialPosition: Position = Position(0, 0),
        override val children: List<Component> = listOf()
) : Container, Movable by MovableImpl(initialPosition) {

    override fun moveTo(position: Position) {
        val diff = position - this.position
        children.forEach {
            it.moveBy(diff)
        }
        moveBy(diff)
    }
}
```

Now let's see what happens if we run the previous example with a `Container` that has a child:

```kotlin
val component = ComponentImpl()
val container = ContainerImpl(children = listOf(component))

println()
println("Positions after moveTo:")
container.moveTo(Position(3, 4))
println(container.position)
println(component.position)

println()
println("Positions after moveBy:")
container.moveBy(Position(1, 2))
println(container.position)
println(component.position)
```

What will this print? Let's see:

```
Positions after moveTo:
Position(x=3, y=4)
Position(x=3, y=4)

Positions after moveBy:
Position(x=4, y=6)
Position(x=3, y=4)
```

**Whoops!** What happened here? 

When we created an override for `moveTo` we implemented a custom logic for moving the `children` of the `Component`. This works properly,
but it doesn't move the `children` when we call `moveBy`.

The problem is that `moveBy` will indeed call `moveTo` but that `moveTo` implementation comes from `MovableImpl` since we delegated to it and
it has no knowledge of our custom `moveBy` implementation!

## Solutions

This problem is a very subtle one. We need some special circumstances for this to happen at all. What also doesn't help us is that it is not
intuitive since there is no explicit code path to follow.

In our case the best is not to use *delegation* here since we need to use our custom implementation and only delegate by hand:

```kotlin
class ContainerImpl(
        initialPosition: Position = Position(0, 0),
        override val children: List<Component> = listOf(),
) : Container, Movable {

    override var position: Position = initialPosition
        private set

    override fun moveTo(position: Position) {
        val diff = position - this.position
        children.forEach {
            it.moveBy(diff)
        }
        this.position = position
    }
}
```

another option is to keep the *delegation* but override `moveBy` as well to delegate to the proper `moveTo`:

```kotlin
class ContainerImpl(
        initialPosition: Position = Position(0, 0),
        override val children: List<Component> = listOf(),
        private val movable: Movable = MovableImpl(initialPosition)
) : Container, Movable by movable {

    override fun moveTo(position: Position) {
        val diff = position - this.position
        children.forEach {
            it.moveBy(diff)
        }
        movable.moveTo(position)
    }

    override fun moveBy(position: Position) = moveTo(this.position + position)
}
```

> This latter option is useful if we have state or other methods in our *mixin*. In our case we have state (`position`).

## Conclusion

Delegation is not a simple topic and it is a prime example for the old adage:

> With great power, comes great responsibility.

Whenever we decide to use it we have to put great care into making sure that it works properly.

If you're interested in the topic take a look at [another article](https://the-cogitator.com/posts/blog/2018/09/29/by-the-way-exploring-delegation-in-kotlin.html) I wrote where we explore the concept of delegation in depth.

Now let's go forth and **kode on**!