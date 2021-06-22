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
updated_at: 2021-03-15
---

> The last time around we connected the levels of our dungeon effectively adding a new *dimension* to our game. This is great, but it gets boring fast, as we can see everything! Let's improve the situation by adding a *vision system* and *Fog of War* to our game!

## Augmenting our Entities

Let's think about how a *vision system* is supposed to work. First of all each `Entity` that has *vision* should have some kind of `Attribute` which holds their sight radius. There should also be some operation which determines what our `Entity` can see around it. Finally I think we can agree that most creatures don't have roentgen sight, so some other `Attribute` which tells us whether something blocks vision or not would also be useful. Let's see how we can go about implementing this!

The first thing to add is (you guessed right) an `Attribute` that we can use to hold the vision radius of our entities:

```kotlin
package com.example.cavesofzircon.attributes

import org.hexworks.amethyst.api.base.BaseAttribute

data class Vision(val radius: Int) : BaseAttribute()
```

and a *flag* `Attribute` which determines whether an `Entity` *blocks vision* or not:

```kotlin
package com.example.cavesofzircon.attributes

import org.hexworks.amethyst.api.base.BaseAttribute

object VisionBlocker : BaseAttribute()
```

Seems straightforward, yes? Now if we think a bit about this, in our game `VisionBlocker` belongs to the walls only, and `Vision` is an attribute of the *player* since fungi don't have eyes (yet?). Let's augment `EntityFactory` with these:

```kotlin
// edit EntityFactory with these changes

import com.example.cavesofzircon.attributes.Vision
import com.example.cavesofzircon.attributes.VisionBlocker

fun newWall() = newGameEntityOfType(Wall) {
    attributes(
        EntityPosition(),
        BlockOccupier,
        EntityTile(GameTileRepository.WALL),
        VisionBlocker   // 1
    )
    facets(Diggable)
}

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
        EntityPosition(),
        EntityTile(GameTileRepository.PLAYER),
        EntityActions(Dig::class, Attack::class),
        CombatStats.create(
            maxHp = 100,
            attackValue = 10,
            defenseValue = 5
        ),
        Vision(9)       // 2
    )
    behaviors(InputReceiver)
    facets(Movable, CameraMover, StairClimber, StairDescender)
}
```

Here we:
1. Add the *flag* `Attribute` to the `newWall` function
2. And set our *player* up with a vision radius of 9. This means that the *player* can see
   `9` tiles far around them.
   
Now let's add a shorthand extension property to `AnyGameEntity` to be able to tell whether it blocks vision:

```kotlin
// put these to EntityExtensions.kt

import com.example.cavesofzircon.attributes.VisionBlocker

val AnyGameEntity.blocksVision: Boolean
    get() = this.findAttribute(VisionBlocker::class).isPresent
```

Now we have the data, but there is no behavior yet. What we want to able to do is to tell whether vision is blocked at a given *position* or not, and also to be able to draw a circle around an entity highlighting the visible tiles. The logical place to put this is `World`:

```kotlin
import com.example.cavesofzircon.extensions.blocksVision
import org.hexworks.zircon.api.data.Position
import com.example.cavesofzircon.attributes.Vision
import org.hexworks.zircon.api.shape.EllipseFactory
import org.hexworks.zircon.api.shape.LineFactory

fun isVisionBlockedAt(pos: Position3D): Boolean {
    return fetchBlockAt(pos).fold(whenEmpty = { false }, whenPresent = {    // 1
        it.entities.any(GameEntity<EntityType>::blocksVision)               // 2
    })
}

fun findVisiblePositionsFor(entity: GameEntity<EntityType>): Iterable<Position> {
    val centerPos = entity.position.to2DPosition()                  // 3
    return entity.findAttribute(Vision::class).map { (radius) ->    // 4
        EllipseFactory.buildEllipse(                                // 5
            fromPosition = centerPos,
            toPosition = centerPos.withRelativeX(radius).withRelativeY(radius)
        )
            .positions
            .flatMap { ringPos ->
                val result = mutableListOf<Position>()
                val iter = LineFactory.buildLine(centerPos, ringPos).iterator() // 6
                do {
                    val next = iter.next()
                    result.add(next)
                } while (iter.hasNext() &&
                    isVisionBlockedAt(Position3D.from2DPosition(next, entity.position.z)).not()
                )                                                               // 7
                result
            }
    }.orElse(listOf())                                                          // 8
}
```

> `EllipseFactory` and `LineFactory` come as part of Zircon but they are abstract enough that we can use them in all kinds of situations like you can see here.

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

> Wait, what is this `flatMap` thing? What happens here anyway? What you see here is a bit of functional programming. What `flatMap` does is that it flattens all the collections into one big collection from what we return from the `takeWhile`. So for example `listOf(1, 2).flatMap { listOf(it, it) }` will return `[1, 1, 2, 3]` in contrast to `listOf(1, 2).map { listOf(it, it) }` which would return [[1, 1], [2, 2]]

Now that we have the means to determine what the *player* can see, let's add a Fog of War effect which the *player* can reveal as they explore the cave!

## Adding the Fog of War

Let's think about how *Fog of War* works. When we start the game everything is shrouded in darkness and only a small part is visible around the *player*. As we traverse the cave each step reveals some of the shadows and it will stay visible.

> We could have a Field of View as well, but personally I've always found it annoying so we'll stick with Fog of War only for this tutorial.

We're going to implement it in a very simple way. By default we'll cover everything with a shroud. Then the world will have a *behavior* that'll reveal the shroud using the player's *vision* attribute! Let's start by adding the necessary *tile* and *color*:

```kotlin
// add this to GameColors:
val UNREVEALED_COLOR = TileColor.fromString("#090909")

// and this to GameTileRepository:
val UNREVEALED = Tile.newBuilder()
    .withCharacter(' ')
    .withBackgroundColor(GameColors.UNREVEALED_COLOR)
    .buildCharacterTile()
```

For the `FogOfWar` itself we're going to use the `TOP` tile of our `Block`s, so let's add this to the `GameBlock`:

```kotlin
init {
    top = GameTileRepository.UNREVEALED
    updateContent()
}
```

> Technically we could make an entity called `Shorud` and add it to all `Block`s but it would create hundreds of thousands of new objects. This is not ideal, so we'll optimize a bit.

Now if we start the game, we'll see the map is covered with Fog of War, so our next job is to create a new `Behavior` that will autonomously reveal the shroud around the player on each update:

```kotlin
package com.example.cavesofzircon.systems

import com.example.cavesofzircon.builders.GameTileRepository
import com.example.cavesofzircon.extensions.position
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.base.BaseBehavior
import org.hexworks.amethyst.api.entity.Entity
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.zircon.api.data.Position3D

object FogOfWar : BaseBehavior<GameContext>() {
    override suspend fun update(entity: Entity<EntityType, GameContext>, context: GameContext): Boolean {
        val (world, _, _, player) = context
        world.findVisiblePositionsFor(player).forEach { pos ->
            world.fetchBlockAt(
                Position3D.create(
                    x = pos.x,
                    y = pos.y,
                    z = player.position.z
                )
            ).map { block ->
                block.top = GameTileRepository.EMPTY
            }
        }
        return true
    }
}
```

How this works is that it simply finds the visible positions around the player and sets the `top` tile of the block to `EMPTY` effectively revealing the tile underneath.

Now to make this work we're going to add this to our `EntityTypes`:

```kotlin
object FOW : BaseEntityType(
    name = "Fog of War"
)
```

and this to `EntityFactory`:


```kotlin
import com.example.cavesofzircon.attributes.types.FOW

fun newFogOfWar() = newGameEntityOfType(FOW) {
    behaviors(FogOfWar)
}
```

Now this fog of war entity won't belong to a Block, and it will have no position. In order to see this in our game we have to have a way of adding a *world entity* to the `World`:

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

    world.addWorldEntity(EntityFactory.newFogOfWar())

    return Game.create(
        player = player,
        world = world
    )
}
```

If you look at this now you'll still see the blackness. The reason for this is that the `Player` haven't moved yet and no update happened to our game world, so the shroud still encompasses the whole map.

An easy solution for this is to simply make an initializing update to our world. For this we can add the update in the `PlayView` at the end of the `init` block, we just have to make sure that we use a key that we can't type:

```kotlin
game.world.update(
    screen, KeyboardEvent(
        type = KeyboardEventType.KEY_TYPED,
        key = "",
        code = KeyCode.DEAD_GRAVE
    ), game
)
```

Now let's see what we created:

![Revealing the Shroud](/assets/img/revealing_the_shroud.gif)

## Conclusion

In this tutorial we've added a very simple *vision system* to our game which we put to good use by implementing a Fog of War effect. In the next article we're going to add *wandering monsters* which will definitely lead to some jump scares!

Until then go forth and *kode on*!
 
> The code of this article can be found in commit #10.
