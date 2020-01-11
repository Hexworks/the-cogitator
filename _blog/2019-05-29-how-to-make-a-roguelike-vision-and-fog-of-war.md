---
excerpt: "Having stairs to lower levels is nice but it is no fun if everything is visible at once. Let's add a vision system!"
title: "How To Make a Roguelike: #10 Vision and Fog of War"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #10 Vision and Fog of War"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/04/27/how-to-make-a-roguelike-vision-and-fog-of-war.html 
---

> The last time around we connected the levels of our dungeon effectively adding a new *dimension* to
our game. This is great, but it gets boring fast, as we can see everything! Let's improve the situation
by adding a *vision system* and *Fog of War* to our game!

## Augmenting our Entities

Let's think about how a *vision system* is supposed to work. First of all each `Entity` which has *vision*
should have some kind of `Attribute` which holds their vision radius. There should also be some operation which
determines what our `Entity` can see around it. Finally I think we can agree that most creatures don't have roentgen
sight, so some other `Attribute` which tells us whether something blocks vision or not would also be useful.
Let's see how we can go about implementing this!

The first thing to add is (you guessed right) an `Attribute` which we can use to hold the vision radius
of our entities:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.amethyst.api.Attribute

data class Vision(val radius: Int): Attribute
```

and a *flag* `Attribute` which determines whether an `Entity` *blocks vision* or not:

```kotlin
package org.hexworks.cavesofzircon.attributes.flags

import org.hexworks.amethyst.api.Attribute

object VisionBlocker : Attribute
```

Seems straightforward, yes? Now if we think a bit about this, in our game `VisionBlocker` belongs to
the walls only, and `Vision` is an attribute of the *player* since fungi don't have eyes (yet?).
Let's augment `EntityFactory` with these:

```kotlin
// new imports
import org.hexworks.cavesofzircon.attributes.flags.VisionBlocker
import org.hexworks.cavesofzircon.attributes.Vision

fun newWall() = newGameEntityOfType(Wall) {
    attributes(
            VisionBlocker,  // 1
            EntityPosition(),
            BlockOccupier,
            EntityTile(GameTileRepository.WALL))
    facets(Diggable)
}

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
            Vision(9),  // 2
            EntityPosition(),
            CombatStats.create(
                    maxHp = 100,
                    attackValue = 10,
                    defenseValue = 5),
            EntityTile(GameTileRepository.PLAYER),
            EntityActions(Dig::class, Attack::class))
    behaviors(InputReceiver)
    facets(Movable, CameraMover, StairClimber, StairDescender)
}
```

Here we:
1. Add the *flag* `Attribute` to the `newWall` function
2. And set our *player* up with a vision radius of 9. This means that the *player* can see
   `9` tiles far around them.
   
Now let's add a shorthand extension property to `AnyGameEntity` to be able to tell whether it
blocks vision:

```kotlin
// put this to EntityExtensions.kt
import org.hexworks.cavesofzircon.attributes.flags.VisionBlocker

val AnyGameEntity.blocksVision: Boolean
    get() = this.findAttribute(VisionBlocker::class).isPresent
```

Now we have the data, but there is no behavior yet. What we want to able to do is to tell whether vision is
blocked at a given *position* or not, and also to be able to draw a circle around an entity highlighting the
visible tiles. The logical place to put this is `World`:

```kotlin
import org.hexworks.cavesofzircon.attributes.Vision
import org.hexworks.cavesofzircon.extensions.blocksVision
import org.hexworks.zircon.api.data.Position
import org.hexworks.zircon.api.shape.EllipseFactory
import org.hexworks.zircon.api.shape.LineFactory

fun isVisionBlockedAt(pos: Position3D): Boolean {
    return fetchBlockAt(pos).fold(whenEmpty = { false }, whenPresent = {    // 1
        it.entities.any(GameEntity<EntityType>::blocksVision)   // 2
    })
}

fun findVisiblePositionsFor(entity: GameEntity<EntityType>): Iterable<Position> {
    val centerPos = entity.position.to2DPosition()  // 3
    return entity.findAttribute(Vision::class).map { (radius) ->    // 4
        EllipseFactory.buildEllipse(    // 5
                fromPosition = centerPos,
                toPosition = centerPos.withRelativeX(radius).withRelativeY(radius))
                .positions()
                .flatMap { ringPos ->
                    val result = mutableListOf<Position>()
                    val iter = LineFactory.buildLine(centerPos, ringPos).iterator() // 6
                    do {
                        val next = iter.next()
                        result.add(next)
                    } while (iter.hasNext() &&
                            isVisionBlockedAt(Position3D.from2DPosition(next, entity.position.z)).not()) // 7
                    result
                }
    }.orElse(listOf()) // 8
}
```

> `EllipseFactory` and `LineFactory` come as part of Zircon but they are abstract enough that we can
use them in all kinds of situations like you can see here.

This seems a bit complex so let's see what happens here!

1. We use `fold` which lets us choose what happens when the `Maybe` is *empty* and when *present*.
2. When *present* we find out whether `any` of its entities are blocking vision
3. Then in `findVisiblePositionsFor` we start with the *position* of the `Entity`
4. Try to find its `Vision` `Attribute`
5. Create a ring around the `Entity` using its radius
6. Then we try to draw a *line* from the center position to all of the ring positions
7. And only take the positions from the *line* while the vision is not blocked at that *point*. Note
   that we use `do/while` here since we need to take *one more* position after we find a vision-blocking
   `Tile`. Otherwise we would not be able to see the walls!
8. We return an empty list if the `Entity` has no `Vision` `Attribute`

> Wait, what is this `flatMap` thing? What happens here anyway? What you see here is a bit of functional
programming. What `flatMap` does is that it flattens all the collections into one big collection from
what we return from the `takeWhile`. So for example `listOf(1, 2).flatMap { listOf(it, it) }` will
return `[1, 1, 2, 3]` in contrast to `listOf(1, 2).map { listOf(it, it) }` which would return [[1, 1], [2, 2]]

Now that we have the means to determine what the *player* can see, let's add a Fog of War effect which
the *player* can reveal as they explore the cave!

## Adding the Fog of War

Let's think about how *Fog of War* works. When we start the game everything is shrouded in darkness and only
a small part is visible around the *player*. As we traverse the cave each step reveals some of the shadows
and it will stay visible.

> We could have a Field of View as well, but personally I've always found it annoying so we'll stick with
Fog of War only for this tutorial.

Fog of War is going to be an `Entity`, but it is special: it does not interact with anything else, it is
also not part of the world it is just a passive shroud which gets revealed where the *player* moves. We
call this a "world" `Entity`. It is omnipresent, and it has no locality, like our entities so far.

Let's add a type for it for starters:

```kotlin
// add this to EntityTypes.kt
object FogOfWarType : BaseEntityType()
```

and we're also going to need a `Tile` for displaying the shroud:

```kotlin
// add this to GameColors:
val UNREVEALED_COLOR = TileColors.fromString("#000000")

// and this to GameTileRepository:
val UNREVEALED = Tiles.newBuilder()
        .withCharacter(' ')
        .withBackgroundColor(GameColors.UNREVEALED_COLOR)
        .buildCharacterTile()
```

For the `FogOfWar` itself we're going to use the `GameArea`'s layering feature. This works in a way that we
can create a regular Zircon `Layer` and push it *over* a level in a `GameArea`, so the code will be rather
simple:

```kotlin
package org.hexworks.cavesofzircon.entities

import org.hexworks.amethyst.api.base.BaseEntity
import org.hexworks.cavesofzircon.attributes.types.FogOfWarType
import org.hexworks.cavesofzircon.builders.GameTileRepository
import org.hexworks.cavesofzircon.extensions.position
import org.hexworks.cavesofzircon.world.Game
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.zircon.api.Layers
import org.hexworks.zircon.api.graphics.Layer
import java.util.concurrent.ConcurrentHashMap

class FogOfWar(game: Game) : BaseEntity<FogOfWarType, GameContext>(FogOfWarType) { // 1

    private val world = game.world
    private val player = game.player
    private val size = game.world.actualSize()

    private val fowPerLevel = ConcurrentHashMap<Int, Layer>().also { fows -> // 2
        repeat(size.zLength) { level ->         // 3
            val fow = Layers.newBuilder()       // 4
                    .withSize(size.to2DSize())
                    .build()
                    .fill(GameTileRepository.UNREVEALED)
            fows[level] = fow                   // 5
            world.pushOverlayAt(fow, level)     // 6
        }
    }

    init {
        updateFOW() // 7
    }

    override fun update(context: GameContext): Boolean {    // 8
        updateFOW()
        return true
    }
    
    private fun updateFOW() {
        world.findVisiblePositionsFor(player).forEach {     // 9
            fowPerLevel[player.position.z]?.setTileAt(it, GameTileRepository.EMPTY) // 10
        }
    }
}
```

It works in a way that:

1. We extend `BaseEntity` by hand
2. Create a new `Map` which will hold the shroud `Layer`s for each *level*
3. And for each level
4. Create a new `Layer` which has the same size as a *level* and fill it with the `UNREVEALED` `Tile`
5. Save it to our fog of wars map
6. And push it over our `World` at the given *level*
7. And then we update the Fog of War to reveal the tiles around the *player*
8. And we also update when the `World` is updated
9. Update works in a way that it finds each visible position for the *player*
10. And sets the `Tile` on the shroud layer to `EMPTY` for the currently visible positions.

Now to make this work we're going to add this to our `EntityFactory`:

```kotlin
import org.hexworks.cavesofzircon.entities.FogOfWar
import org.hexworks.cavesofzircon.world.Game

fun newFogOfWar(game: Game) = FogOfWar(game)
```

In order to see this in our game we have to have a way of adding a *world entity* to the `World`:

```kotlin
// add this to World
fun addWorldEntity(entity: Entity<EntityType, GameContext>) {
    engine.addEntity(entity)
}
```

and initialize it in the `GameBuilder`:

```kotlin
fun buildGame(): Game {

    prepareWorld()

    val player = addPlayer()
    addFungi()

    val game = Game.create(
            player = player,
            world = world)
    
    world.addWorldEntity(EntityFactory.newFogOfWar(game))
    
    return game
}
```

Now let's see what we created:

![Revealing the Shroud](/assets/img/revealing_the_shroud.gif)

## Conclusion

In this tutorial we've added a very simple *vision system* to our game which we put to good use
by implementing a Fog of War effect. In the next article we're going to add *wandering monsters*
which will definitely lead to some jump scares!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `10_VISION_AND_FOG_OF_WAR` tag.
