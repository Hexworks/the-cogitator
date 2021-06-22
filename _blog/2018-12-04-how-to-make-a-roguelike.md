---
excerpt: This article is the start of a tutorial series which will teach you how to write a roguelike game.
title: "How To Make a Roguelike"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2018/12/04/how-to-make-a-roguelike.html
updated_at: 2018-12-04
---

> This tutorial series is loosely based on [Trystan's Awesome Roguelike Tutorial](http://trystans.blogspot.com/2016/01/roguelike-tutorial-00-table-of-contents.html).
  Go check it out if you want to use Java instead.
  
## Introduction

If you are reading this it means that you are probably planning of writing a game of some sort.
Writing games is not only fun and useful if you are just starting out as a programmer but even
if you have a lot of coding experience under your belt and you want to learn a new language.

The problem is that if you want to write one you'll have to learn how to create 3D graphics,
how to use a complex game engine, and all sorts of related things...are you not?

## Roguelikes to the rescue

If we take a look at the definition of a Roguelike:

> Roguelike is a subgenre of role-playing video game characterized by a dungeon crawl through procedurally generated levels, turn-based gameplay, tile-based graphics, and permanent death of the player character.

it turns out that it has some inherent features which make writing one easy and fun!
This tutorial series is therefore about writing *your own roguelike game* from scratch.

## Language and library choices

For this tutorial we'll use the [Kotlin](https://kotlinlang.org/) programming language.
"Why not use C++, Java or Python?" you might ask. The reason is that

- Low-level languages like C++ make it much harder to focus on writing actual game mechanics
  and you quickly get bogged down with memory management, complex language features and such.
  For a roguelike raw performance is not nearly as important as if you were writing an AAA title.
- While Java has tons of libraries for this purpose it is a bit outdated even with the new improvements
  with Java 10 and 11.
- Python is a nice language, but its dynamic nature makes it harder do reason about your code
  as your codebase grows. Apart from that it is also more difficult to write code which runs on
  a multitude of platforms (including the browser).

## Why Kotlin then?

Kotlin is a pragmatic language and its expressive power is on-par with Python. The language in
fact feels like as if you were combining the best features of Java and Python: it keeps the static
nature, but lets you write code fast with type inference, extension methods and other features.

For example this Python code:

```python
class Tile:
  pass

def write_tile(tile: Tile):
  print("Tile: " + str(tile))

write_tile(Tile())
```

looks like this in Kotlin:

```kotlin
class Tile

fun writeTile(tile: Tile) =
    println("Tile: $tile")

writeTile(Tile())
```

pretty similar, huh? You also get multiplatform capabilities with Kotlin, so the code you write can be
run on the JVM, in the browser, and on native platforms as a binary.

To top it all off, you can use *any* Java library, since Kotlin gives you seamless interoperability.

> I've written about this topic on my blog [here](http://the-cogitator.com/2017/05/19/kotlin-is-the-new-java.html).
  Take a look if you are interested.

## Game Libraries

For this game we're gonna use [Zircon](https://github.com/Hexworks/zircon), which is a Tile engine,
the [Amethyst](https://github.com/Hexworks/amethyst) Entity Attribute System and some useful features
form the [Cobalt](https://github.com/Hexworks/cobalt) library, like data binding and the `EventBus`.

> Note that I've chosen these libraries because I'm familiar with them and I also think that they are
  the best fit for the problem at hand. Disclaimer: I work on those libraries so I might be biased
  but you'll see if they work out for you or not.
  
## Other things we need

For this tutorial we're gonna need some basic [Git](https://git-scm.com/) and [Gradle](https://gradle.org/)
knowledge. I'll explain these on the way so you needn't worry about them for now.

As for our development environment I highly recommend the [Intellij IDEA Community Edition](https://www.jetbrains.com/idea/download).
It is not only *free* but it is the best Kotlin ide you can get.
  
Now that we are all set, let's start coding! In the next article we'll set up our project
and start working on our game right away!
