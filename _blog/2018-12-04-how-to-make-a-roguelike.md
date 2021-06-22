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
updated_at: 2020-11-20
---

> This tutorial series is loosely based on [Trystan's Awesome Roguelike Tutorial](http://trystans.blogspot.com/2016/01/roguelike-tutorial-00-table-of-contents.html).
  Go check it out if you want to use Java instead.
  
> #### Table of Contents
>
> Here are the links to all the current tutorial articles if you want to skip the introduction
> or want to continue where you left off previously.
> 
> 1. [#1 Project Setup](https://hexworks.org/posts/tutorials/2018/12/12/how-to-make-a-roguelike-project-setup.html)
> 2. [#2 Views, Screens, Inputs](https://hexworks.org/posts/tutorials/2018/12/28/how-to-make-a-roguelike-views-screens-inputs.html)
> 3. [#3 Generating Random Caves](https://hexworks.org/posts/tutorials/2019/01/05/how-to-make-a-roguelike-generating-random-caves.html)
> 4. [#4 The Player](https://hexworks.org/posts/tutorials/2019/02/13/how-to-make-a-roguelike-the-player.html)
> 5. [#5 Exploring The Cave](https://hexworks.org/posts/tutorials/2019/02/28/how-to-make-a-roguelike-exploring-the-cave.html)
> 6. [#6 Entity Interactions](https://hexworks.org/posts/tutorials/2019/03/14/how-to-make-a-roguelike-entity-interactions.html)
> 7. [#7 Stationary Monsters](https://hexworks.org/posts/tutorials/2019/03/21/how-to-make-a-roguelike-stationary-monsters.html)
> 8. [#8 Combat and Damage](https://hexworks.org/posts/tutorials/2019/04/02/how-to-make-a-roguelike-combat-and-damage.html)
> 9. [#9 A Multi-level Dungeon](https://hexworks.org/posts/tutorials/2019/04/17/how-to-make-a-roguelike-a-multi-level-dungeon.html)
> 10. [#10 Vision and Fog of War](https://hexworks.org/posts/tutorials/2019/04/27/how-to-make-a-roguelike-vision-and-fog-of-war.html)
> 11. [#11 Wandering Monsters](https://hexworks.org/posts/tutorials/2019/05/07/how-to-make-a-roguelike-wandering-monsters.html)
> 12. [#12 Items and Inventory](https://hexworks.org/posts/tutorials/2019/05/15/how-to-make-a-roguelike-items-and-inventory.html)
> 13. [#13 Food and Hunger](https://hexworks.org/posts/tutorials/2019/05/23/how-to-make-a-roguelike-food-and-hunger.html)
> 14. [#14 Displaying Stats](https://hexworks.org/posts/tutorials/2019/06/14/how-to-make-a-roguelike-displaying-stats.html)
> 15. [#15 Weapons and Armor](https://hexworks.org/posts/tutorials/2019/06/21/how-to-make-a-roguelike-weapons-and-armor.html)
> 16. [#16 Aggressive Monsters](https://hexworks.org/posts/tutorials/2019/06/26/how-to-make-a-roguelike-aggressive-monsters.html)
> 17. [#17 Experience and Leveling Up](https://hexworks.org/posts/tutorials/2019/06/28/how-to-make-a-roguelike-experience-and-leveling-up.html)
> 18. [#18 Help and Examine Dialogs](https://hexworks.org/posts/tutorials/2019/06/30/how-to-make-a-roguelike-help-and-examine-dialogs.html)
> 19. [#19 Win and Lose Conditions](https://hexworks.org/posts/tutorials/2019/07/01/how-to-make-a-roguelike-win-and-lose-conditions.html)
> 20. [#20 Wrapping Up](https://hexworks.org/posts/tutorials/2019/07/02/how-to-make-a-roguelike-wrapping-up.html)
  
## Introduction

If you are reading this it means that you are probably planning to write a game of some sort.
Writing games is not only fun and useful if you are just starting out as a programmer but even
if you have a lot of coding experience under your belt, and you want to learn a new language.

The problem is that if you want to write one you'll have to learn how to create 3D graphics,
how to use a complex game engine, and all sorts of related things...are you not?

## Roguelikes to the rescue

If we take a look at the definition of a Roguelike:

> Roguelike is a subgenre of role-playing video game characterized by a dungeon crawl through procedurally
> generated levels, turn-based gameplay, tile-based graphics, and permanent death of the player character.

it turns out that it has some inherent features that make writing one easy and fun!
This tutorial series is therefore about writing *your own roguelike game* from scratch.

## Language and library choices

For this tutorial we'll use the [Kotlin](https://kotlinlang.org/) programming language.
"Why not use C++, Java or Python?" you might ask. The reasons are:

- Low-level languages like C++ make it much harder to focus on writing actual game mechanisms
  and you quickly get bogged down with memory management, complex language features and such.
  For a roguelike raw performance is not nearly as important as if you were writing an AAA title.
- While Java has tons of libraries for this purpose it is a bit outdated even with the new improvements.
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

> I've written about this topic on my blog [here](http://the-cogitator.com/posts/blog/2017/05/19/kotlin-is-the-new-java.html).
  Take a look if you are interested.

## Game Libraries

For this game we're gonna use [Zircon](https://github.com/Hexworks/zircon), which is a Tile engine,
the [Amethyst](https://github.com/Hexworks/amethyst) SEA (Systems, Entities, Attributes) library, and some useful features
form the [Cobalt](https://github.com/Hexworks/cobalt) library, like data binding and the `EventBus`.

> Note that I've chosen these libraries because I'm familiar with them, and I also think that they are
  the best fit for the problem at hand. Disclaimer: I work on those libraries, so I might be biased,
  but you'll see if they work out for you or not.
  
## Other things we need

For this tutorial we're gonna need some basic [Git](https://git-scm.com/) and [Gradle](https://gradle.org/)
knowledge. I'll explain these on the way, so you needn't worry about them for now.

As for our development environment I highly recommend the [Intellij IDEA Community Edition](https://www.jetbrains.com/idea/download).
It is not only *free*, but it is the best Kotlin IDE you can get.
  
Now we are all set, let's start coding! In the next article we'll set up our project
and start working on our game right away!
