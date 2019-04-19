---
excerpt: "Now that we can interact with the World, the next logical step is to add monsters to it!"
title: "How To Make a Roguelike: #7 Stationary Monsters"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #7 Stationary Monsters"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/03/21/how-to-make-a-roguelike-stationary-monsters.html
---

> Last time we added interactions between our entities and taught the Player how to *dig*. Now it
is time to add one of the most important things to our game: **monsters**!

## Implementing our First Monster

Monsters are an integral part of any roguelike game, and they come in all varieties. They might
be able to patrol the dungeon, pick up items, use them, and even perform magic. Thanks to the
*SEA* model we use it is easy to mix and match these things, but let's start with something
very simple: a *stationary monster*. In our case this will be a simple *fungus* creature which
just sits in the cave.

For this we're going to need a new `EntityType`, `Fungus`:

```kotlin
// add this to EntityTypes.kt
object Fungus : BaseEntityType(
        name = "fungus")
```

Since this is a completely new `Entity` in our game we'll also need to add a tile for it
with a color:

```kotlin

// Add this to GameColors
val FUNGUS_COLOR = TileColors.fromString("#85DD1B")


// Add this to GameTileRepository
val FUNGUS = Tiles.newBuilder()
        .withCharacter('f')
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .withForegroundColor(GameColors.FUNGUS_COLOR)
        .buildCharacterTile()
```

Then a factory method for it is trivial:

```kotlin
import org.hexworks.cavesofzircon.attributes.types.Fungus

fun newFungus() = newGameEntityOfType(Fungus) {
    attributes(BlockOccupier,
            EntityPosition(),
            EntityTile(GameTileRepository.FUNGUS))
    facets()
    behaviors()
}
```

Now in order to see fungi in our game we only need to create some when we build the `Game`.
Let's now think about how we're gonna control how many fungi we add. It is always a good idea
to separate configuration from actual game code because in this case we don't mix them
and setting a simple variable such as `fungi per level` can be done in a config file. We already
have `GameConfig` so let's add it there:

```kotlin
// add this to GameConfig
const val FUNGI_PER_LEVEL = 15
```

Then we modify the `GameBuilder` to accommodate fungi. 

> It is always a good idea to create abstractions for common functionality to make the code simpler.
Now we're going to perform a little [Refactoring](https://en.wikipedia.org/wiki/Code_refactoring)
with this goal.

Let's now add an extension function to `GameEntity` which will add it to our game world. We create
an extension function instead of a regular one because this will keep our code fluent and readable
as you'll see in the code:

```kotlin
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.zircon.api.Positions
import org.hexworks.zircon.api.data.Size
import org.hexworks.zircon.api.data.impl.Size3D
import org.hexworks.cavesofzircon.extensions.GameEntity


// add this to GameBuilder
private fun <T : EntityType> GameEntity<T>.addToWorld(  // 1
        atLevel: Int,                                   // 2
        atArea: Size = world.actualSize().to2DSize()): GameEntity<T> {  // 3
    world.addAtEmptyPosition(this,
            offset = Positions.default3DPosition().withZ(atLevel),      // 4 
            size = Size3D.from2DSize(atArea))                           // 5
    return this
}
```

Here we:

1. Add this extension method to **any** `GameEntity` and we use the `T` generic type
   parameter to preserve the type in the return value to out function
2. `atLevel` will be used to supply the level at which we want to add the `Entity`
3. `atArea` specifies the size of the area at which we want to add the `Entity`
   this defaults to the actual size of the world (the whole level). this function
   returns the `GameEntity` which we called this function on which allows us to
   perform [Method Chaining](https://en.wikipedia.org/wiki/Method_chaining)
4. We call `addAtEmptyPosition` with the supplied level
5. and we set the size using the supplied `Size`

> It is a good practice to make small incremental improvements to our program whenever
we see a common usage pattern emerge or when we just realize that something can be
implemented in a better way. This is called [The Boy Scout Rule](https://medium.com/@biratkirat/step-8-the-boy-scout-rule-robert-c-martin-uncle-bob-9ac839778385).


Now the new functions to create the *player* and the *fungi* become simpler and more
readable:

```kotlin
import org.hexworks.cavesofzircon.GameConfig.FUNGI_PER_LEVEL
import org.hexworks.cavesofzircon.attributes.types.Player
import org.hexworks.cavesofzircon.builders.EntityFactory

private fun addPlayer(): GameEntity<Player> {
    return EntityFactory.newPlayer().addToWorld(
            atLevel = GameConfig.DUNGEON_LEVELS - 1,
            atArea = world.visibleSize().to2DSize())
}

private fun addFungi() = also {
    repeat(world.actualSize().zLength) { level ->
        repeat(FUNGI_PER_LEVEL) {
            EntityFactory.newFungus().addToWorld(level)
        }
    }
}
```

and we just need to add `addFungi` to the `buildGame` function:

```kotlin
fun buildGame(): Game {

    prepareWorld()

    val player = addPlayer()
    addFungi()

    return Game.create(
            player = player,
            world = world)
}
```

Now if we start the game we can see this:

![Adding fungi](/assets/img/adding_fungi.gif)

As you can see, now *fungi* are present, but we can't really do anything with them. They do occupy
a block so we can't walk over them, but there is not much else. Let's see how can we make them
**Attackable**! As you might have guessed we're going to need the usual `Command` + `Facet` pair
for this, `Attack` and `Attackable`. Since entities will perform `Attack` we make it an
`EntityAction`:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.world.GameContext

data class Attack(override val context: GameContext,
                  override val source: GameEntity<EntityType>,
                  override val target: GameEntity<EntityType>) : EntityAction<EntityType, EntityType>

```

and `Attackable` will accept this `Command`:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.commands.Attack
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.world.GameContext

object Attackable : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(Attack::class) { (context, attacker, target) ->
        context.world.removeEntity(target)
        Consumed
    }
}
```

This is going to be super simple now: when we `Attack` and `Entity` it is an instant kill, we just
remove the the `target` from the world altogether and report that we `Consumed` the `Command`.

Later we will add combat stats, experience, and things like that but this will do as a first implementation.

> It is a common pitfall which game developers fall into that they try to implement **everything**
in their game in the first version. This won't work in most of the cases. What is usually better if
we do something which is *good enough* and improve on it later. This is also what
[Agile Development](https://agilemanifesto.org/) is about.

Now let's add `Attackable` to our *fungus* entity and `Attack` as a possible `EntityAction` to our
*player* and see what happens!

```kotlin
import org.hexworks.cavesofzircon.commands.Attack
import org.hexworks.cavesofzircon.systems.Attackable

// modify these functions in EntityFactory
fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
            EntityPosition(),
            EntityTile(GameTileRepository.PLAYER),
            EntityActions(Dig::class, Attack::class))
    behaviors(InputReceiver)
    facets(Movable, CameraMover)
}

fun newFungus() = newGameEntityOfType(Fungus) {
    attributes(BlockOccupier,
            EntityPosition(),
            EntityTile(GameTileRepository.FUNGUS))
    facets(Attackable)
    behaviors()
}
```

Moving into *fungi* will now *destoy* them!

![Destroying fungi](/assets/img/attacking_fungi.gif)

**Nice!**

## Fungi Gardening

Having *fungi* in our World is nice but in a good game each creature has unique properties which
makes the game interesting. If we sit down and think about what *fungi* does the first thing which
comes into mind is that it is **Growing**! There should also be a limit to the maximum size a *fungus
colony* can spread. We'll define an `Attribute` for this. Let's take a look at how these things
work. First, we add a configuration property to maximum fungus spread:

```kotlin
// Add this to GameConfig
const val MAXIMUM_FUNGUS_SPREAD = 20
```

then we introduce an attribute which will hold our current colony size:

```kotlin
import org.hexworks.amethyst.api.Attribute
import org.hexworks.cavesofzircon.GameConfig

data class FungusSpread(
        var spreadCount: Int = 0,
        val maximumSpread: Int = GameConfig.MAXIMUM_FUNGUS_SPREAD) : Attribute

```

The `Growing` itself is a `Behavior`, since a *fungus* can grow on its own:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.base.BaseBehavior
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.FungusSpread
import org.hexworks.cavesofzircon.builders.EntityFactory
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.extensions.position
import org.hexworks.cavesofzircon.extensions.tryToFindAttribute
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.cobalt.datatypes.extensions.map
import org.hexworks.zircon.api.Sizes

object FungusGrowth : BaseBehavior<GameContext>(FungusSpread::class) { // 1

    override fun update(entity: GameEntity<out EntityType>, context: GameContext): Boolean {
        val world = context.world
        val fungusSpread = entity.tryToFindAttribute(FungusSpread::class)   // 2
        val (spreadCount, maxSpread) = fungusSpread                         // 3
        return if (spreadCount < maxSpread && Math.random() < 0.015) {      // 4
            world.findEmptyLocationWithin(
                    offset = entity.position
                            .withRelativeX(-1)
                            .withRelativeY(-1),
                    size = Sizes.create3DSize(3, 3, 0)).map { emptyLocation ->
                world.addEntity(EntityFactory.newFungus(fungusSpread), emptyLocation)   // 5
                fungusSpread.spreadcount++
            }
            true
        } else false
    }
}
```

Here:

1. We create a `Behavior` and supply `FungusSpread` as a mandatory `Attribute` to it
2. When `update` is called with an `entity` we try to find its `FungusSpread` `Attribute`. We know
   that it is there so we don't have to use the `findAttribute` method.
3. Destructuring works for `FungusSpread` because it is a `data class`
4. You can specify any probability here. It will have a direct effect on how often fungus spreads.
5. Note that we pass `fungusSpread` as a parameter to `newFungus` (which we'll modify momentarily)
   so that all fungi in the same fungus colony can share this `Attribute`. This makes sure that fungus
   won't spread all over the place and the size of a colony is controlled

> A quick refresher: [data classes](https://kotlinlang.org/docs/reference/data-classes.html) give
us a lot of useful features like destructuring, a useful `toString` method and more!

Finally we update the factory method of *fungus* to use the new things we created:

```kotlin
import org.hexworks.cavesofzircon.attributes.FungusSpread
import org.hexworks.cavesofzircon.systems.FungusGrowth


fun newFungus(fungusSpread: FungusSpread = FungusSpread()) = newGameEntityOfType(Fungus) { // 1
    attributes(BlockOccupier,
            EntityPosition(),
            EntityTile(GameTileRepository.FUNGUS),
            fungusSpread)   // 2
    facets(Attackable)
    behaviors(FungusGrowth) // 3
}
```

Our modifications are:

1. We added the `fungusSpread` as a parameter to `newFungus` and it also has a default value. This enables us
   to call it with a `FungusSpread` object when the fungus grows and use the default when we create the first
   one in the builder
2. We pass the `fungusSpread` parameter to our builder so it will use whatever we supplied instead of creating
   one by hand
3. We also add `FungusGrowth` which we created now

> Wait, isn't shared mutable state is the quintessence of evil and ruin? In most cases yes, but
here we *Amethyst* handles the updating of our entities so we're safe to tamper with state.

Now let's take a look at what we proudced!

![Fungus spread](/assets/img/fungus_spread.gif)

Wow, that's cool!

## Conclusion

Adding our first monster was easy, it seems that we're starting to reap the benefits of using a system which was designed
for this kind of usage. Upgrading our *fungus* was also straightforward! In the next article we're going to add some
real combat to our game with *hit points*, *damage* and *combat messages*!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `7_STATIONARY_MONSTERS` tag.




















