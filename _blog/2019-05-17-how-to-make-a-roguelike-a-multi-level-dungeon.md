---
excerpt: "Now that we have real combat let's expand the explorable dungeon to all the levels we have generated!"
title: "How To Make a Roguelike: #9 A Multi-level Dungeon"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #9 A Multi-level Dungeon"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/04/17/how-to-make-a-roguelike-a-multi-level-dungeon.html
updated_at: 2019-05-17
---

> We have added multiple levels to our dungeon previously they are just not accessible yet. Let's add some stairs
to our game and some UI upgrades so we can see the stats of our adventurer!

## Connecting the Levels

Let's think about how we can connect our levels. There are a lot of known algorithms for this, but we're going to
choose a very simple one:

1. Let the `current level` be the top level
1. Pick a random *position* on the `current level` where the tile is a `floor` (we can only place stairs on floors, not on walls)
2. Check whether the tile *below* the selected *position* is empty as well
3. If it is we add *stairs down* on the top level, and *stairs up* on the level below
4. Otherwise we retry step 1.
5. When we successfully add a stairs pair we decrement the `current level` and continue from step 1.
6. When we reach the bottom level we're done (`current level == 0`)

This algorithm is linear in complexity (`O(n)`) so it will be fine for our purposes. 

> There is one theoretical problem though: what happens if we can't pick a position from the two levels we want to connect
which are both *floors*? In this case we want to *reject* the level below and regenerate it. This is what happens in
most procedurally generated games (including Dwarf Fortress!), but the chances are for this in our case are astronomically
low, so implementing it is left to the reader as an exercise.

## Adding New Tiles

The first thing we need is a set of new tiles which we'll use to represent our tiles, `STAIRS_UP` and `STAIRS_DOWN`:

```kotlin
// Add these to GameTileRepository
val STAIRS_UP = Tiles.newBuilder()
        .withCharacter('<')
        .withForegroundColor(GameColors.ACCENT_COLOR)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()

val STAIRS_DOWN = Tiles.newBuilder()
        .withCharacter('>')
        .withForegroundColor(GameColors.ACCENT_COLOR)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()
```

and add the corresponding entities as well.

Add these to `EntityTypes.kt`:
```kotlin
object StairsDown : BaseEntityType(
        name = "stairs down")

object StairsUp : BaseEntityType(
        name = "stairs up")
```

`EntityFactory`:
```kotlin
import org.hexworks.cavesofzircon.attributes.types.StairsDown
import org.hexworks.cavesofzircon.attributes.types.StairsUp

fun newStairsDown() = newGameEntityOfType(StairsDown) {
    attributes(EntityTile(GameTileRepository.STAIRS_DOWN),
            EntityPosition())
}

fun newStairsUp() = newGameEntityOfType(StairsUp) {
    attributes(EntityTile(GameTileRepository.STAIRS_UP),
            EntityPosition())
}
```

and `GameBlockFactory`:

```kotlin
fun stairsDown() = GameBlock.createWith(EntityFactory.newStairsDown())

fun stairsUp() = GameBlock.createWith(EntityFactory.newStairsUp())
```

## Placing Stairs

Now that we have our *blocks* in place we just have to connect the levels with them. For this we're going to add
a function first which generates a `Sequence` of random positions for a given level:

> Kotlin `Sequence`s are similar to `Iterable`s in Java. Using them we can *generate* values lazily. They work
in a similar way to how *generators* work in Python. The difference is that `yield` is not a keyword in Kotlin,
but a function which can be used in `Sequence` builders. You can read more about how `Sequence`s work in Kotlin
[here](https://blog.kotlin-academy.com/effective-kotlin-use-sequence-for-bigger-collections-with-more-than-one-processing-step-649a15bb4bf).
If you are a visual type (like me) [here](https://typealias.com/guides/kotlin-sequences-illustrated-guide/) is a
*Visual Guide to Kotlin Sequences*.

```kotlin
import kotlin.random.Random

// put this in WorldBuilder

private val depth = worldSize.zLength

private fun generateRandomFloorPositionsOn(level: Int) = sequence { // 1
    while (true) {      // 2
        var pos = Position3D.unknown()                      // 3
        while (pos.isUnknown()) {                           // 4
            val candidate = Positions.create3DPosition(     // 5
                    x = Random.nextInt(width - 1),
                    y = Random.nextInt(depth - 1),
                    z = level)
            if (blocks[candidate].isEmptyFloor()) {         // 6
                pos = candidate
            }
        }
        yield(pos)                                          // 7
    }
}

private fun GameBlock?.isEmptyFloor(): Boolean {        // 8
    return this?.isEmptyFloor ?: false
}
```

Here we:

1. use the `sequence` builder function which lets us construct a new `Sequence`
2. Don't worry, with `while(true)` we just say that this sequence is infinite. In theory this could lead into problems
   but in practice the chance to generate 2 levels with perfectly overlapping walls is **astronomically** low
3. set the resulting position to `unknown()`. `unknown()` is an implementation of the [Null Object Pattern](https://en.wikipedia.org/wiki/Null_object_pattern)
4. we iterate while our resulting position is still the null object
5. and construct a random position within the level
6. which is an empty floor
7. then if we find one we `yield` it. `yield` is very similar to the `yield` keyword in Python, but here it is a function.
   This function comes from the `sequence` builder and it will lazily *yield* the next value of the `Sequence`
8. This is a convenience function which implements a nice Kotlin trick: defining functions on *nullable* types!
   In our example the *nullable* type is `GameBlock?` and it lets us call the `isEmptyFloor` function on a potentially
   `null` value. If we can't find the block in `blocks` this function will default to `false`
   
> You might be wondering why do we define extension functions on `GameBlock?` in `WorldBuilder`. This pattern is very useful
and common in Kotlin: by defining functions like this we make our code much more readable, while keeping the global namespace
clean (the `private` extension function will only be visible in `WorldBuilder`).

Next, we define a function which can *connect* two levels with *stairs*:

```kotlin
private fun connectRegionDown(currentLevel: Int) {                      // 1
    val posToConnect = generateRandomFloorPositionsOn(currentLevel)     // 2
            .first { pos ->                                       // 3
                blocks[pos].isEmptyFloor() && blocks[pos.below()].isEmptyFloor()    // 4
            }
    blocks[posToConnect] = GameBlockFactory.stairsDown()                            // 5
    blocks[posToConnect.below()] = GameBlockFactory.stairsUp()

}

private fun Position3D.below() = copy(z = z - 1)
```

What happens here is that we:

1. create a new function which accepts the `current level`
2. creates a sequence for the random floor positions on `current level`
3. finds the first in the `Sequence`
4. where the block at the `current position` is an empty floor and the block below that as well
5. then sets the proper stairs to those blocks

Note that we use the *null safety operator* (`?:`) here so `posToConnect` will never be `null`. If we can't find
a position we just throw an exception. This is not the cleanest solution, but it is a pragmatic approach: the chance
to not be able to find a position which can be connected is astronomically low.

All that's left to do is to connect the levels now, when we make the caves:

```kotlin
// modify WorldBuilder with these
fun makeCaves(): WorldBuilder {
    return randomizeTiles()
            .smooth(8)
            .connectLevels() // 1
}

private fun connectLevels() = also {
    (height - 1).downTo(1).forEach(::connectRegionDown) // 2
}
```

Here:

1. we add `connectLevels` to our `makeCaves` function
2. which just iterates the levels from top to bottom and connects them

> Hey, what is this funky `::connectRegionDown` thing? It is the same as if we were typing this:
`forEach { connectRegionDown(it) }`. It is just a function pointer which we can also use in Java 8, but
Kotlin lets us reference the function pointers of an object as well, not just a class. Typing `::connectRegionDown`
is the same as typing `this::connectRegionDown`.

Now if we start our game we'll see something like this:

![Stairs down](/assets/img/stairs_down.gif)

This is great, but nothing happens if we move onto the stairs. This is because we haven't defined the way we're going
to interact with it! Let's make it happen.

## Traversing the Stairs

So now we have stairs in place but we can't go up or down. Let's think about how we would like that to work. In some
games there is an interaction key (like the `Spacebar`) which can be used to interact with whatever is at a given tile.
The problem with this is that if there are multiple things to interact with a menu must be displayed from which the
user can choose.

According to the seminal UX book "Don't Make Me Think" it is better to have more steps in a process
which are straightforward than having a few complex steps so we're going to have explicit keyboard shortcuts to our
commands.

> The book is a *very good read* I'd recommend reading it for anybody who works with UIs. It can be found
[here](https://www.amazon.com/Dont-Make-Me-Think-Usability/dp/0321344758).

Since we already have `w`, `a`, `s` and `d` as our movement keys it is a good idea if we assign some keys which
are around them for `up`/`down` traversal of stairs. Let's use `r` for `up` and `f` for `down`!

First of all we're going to need two commands for this. `MoveUp`:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.world.GameContext

data class MoveUp(override val context: GameContext,
                  override val source: GameEntity<EntityType>) : GameCommand<EntityType>
```

and `MoveDown`:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.world.GameContext

data class MoveDown(override val context: GameContext,
                    override val source: GameEntity<EntityType>) : GameCommand<EntityType>
```

With this in place all we have to do is to refactor `InputReceiver` a bit:

> There is an *awesome* book on the topic of *Refactoring* by *Martin Fowler* I recommend reading [it](https://www.martinfowler.com/books/refactoring.html).

```kotlin
// new imports
import org.hexworks.cavesofzircon.attributes.types.Player
import org.hexworks.cobalt.logging.api.LoggerFactory
import org.hexworks.zircon.api.data.impl.Position3D
import org.hexworks.cavesofzircon.commands.MoveDown
import org.hexworks.cavesofzircon.commands.MoveUp

object InputReceiver : BaseBehavior<GameContext>() {

    private val logger = LoggerFactory.getLogger(this::class)

    override fun update(entity: GameEntity<out EntityType>, context: GameContext): Boolean {
        val (_, _, uiEvent, player) = context
        val currentPos = player.position
        if (uiEvent is KeyboardEvent) {
            when (uiEvent.code) {
                KeyCode.KEY_W -> player.moveTo(currentPos.withRelativeY(-1), context) // 1
                KeyCode.KEY_A -> player.moveTo(currentPos.withRelativeX(-1), context)
                KeyCode.KEY_S -> player.moveTo(currentPos.withRelativeY(1), context)
                KeyCode.KEY_D -> player.moveTo(currentPos.withRelativeX(1), context)
                KeyCode.KEY_R -> player.moveUp(context)
                KeyCode.KEY_F -> player.moveDown(context)
                else -> {
                    logger.debug("UI Event ($uiEvent) does not have a corresponding command, it is ignored.")
                }
            }
        }
        return true
    }

    private fun GameEntity<Player>.moveTo(position: Position3D, context: GameContext) { // 2
        executeCommand(MoveTo(context, this, position))
    }

    private fun GameEntity<Player>.moveUp(context: GameContext) {   // 3
        executeCommand(MoveUp(context, this))
    }

    private fun GameEntity<Player>.moveDown(context: GameContext) {
        executeCommand(MoveDown(context, this))
    }
}
```

Here we:

1. Instead of introducing a lot of ifs we *augment* the `player` which we know has a type of `GameEntity<Player>`
2. With *extension functions* for moving the player in all 4 directions
3. And up or down

This makes our code much more readable while still retaining the simplicity of the code.
As for the actual climbing we're going to add the corresponding `Facet`s as well; `StairClimber` and `StairDescender`:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.types.StairsUp
import org.hexworks.cavesofzircon.blocks.GameBlock
import org.hexworks.cavesofzircon.commands.MoveUp
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.position
import org.hexworks.cavesofzircon.functions.logGameEvent
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.cobalt.datatypes.extensions.map

object StairClimber : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(MoveUp::class) { (context, player) ->
        val world = context.world
        val playerPos = player.position
        world.fetchBlockAt(playerPos).map { block ->
            if (block.hasStairsUp) {    // 1
                logGameEvent("You move up one level...")
                world.moveEntity(player, playerPos.withRelativeZ(1))    // 2
                world.scrollOneUp()                                     
            } else {
                logGameEvent("You jump up and try to reach the ceiling. You fail.") // 3
            }
        }
        Consumed
    }

    private val GameBlock.hasStairsUp: Boolean  // 
        get() = this.entities.any { it.type == StairsUp }
}
```

What we do here is:

1. We check whether the block at the *player*'s position has stairs up
2. If yes, we move the entity up and also scroll the world up.
3. Otherwise we just add some flavor text to the log
4. `hasStairsUp` will only be used by `StairClimber` so there is no need to pollute the global
   namespace with it so we just add the *extension property* here
   
`StairDescender` is very similar:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.types.StairsDown
import org.hexworks.cavesofzircon.blocks.GameBlock
import org.hexworks.cavesofzircon.commands.MoveDown
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.position
import org.hexworks.cavesofzircon.functions.logGameEvent
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.cobalt.datatypes.extensions.map

object StairDescender : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(MoveDown::class) { (context, player) ->
        val world = context.world
        val playerPos = player.position
        world.fetchBlockAt(playerPos).map { block ->
            if (block.hasStairsDown) {
                logGameEvent("You move down one level...")
                world.moveEntity(player, playerPos.withRelativeZ(-1))
                world.scrollOneDown()
            } else {
                logGameEvent("You search for a trapdoor, but you find nothing.")
            }
        }
        Consumed
    }

    private val GameBlock.hasStairsDown: Boolean
        get() = this.entities.any { it.type == StairsDown }
}
```

> Wait, isn't this a *horrible* case of code duplication? They look **very** similar so we should create an abstraction
for this! Well, not so fast. One of the most useful virtue of a `System` is that it is self-contained and has *strong*
cohesion. We *could* create an abstraction but then we can't make them work in a different way (going up needing ladders,
going down needing ropes for example). With these two systems we might have more code, but it is easy to read, reason
about and test.

Now all that is left is to give our *player* the ability to go up and down:

```kotlin
// put this in EntityFactory
import org.hexworks.cavesofzircon.systems.StairClimber
import org.hexworks.cavesofzircon.systems.StairDescender

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
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

And take a look at what we've created:

![Moving Down](/assets/img/moving_down.gif)

We've *literally* added a new dimension to our game!

## Conclusion

In this article we've connected our levels with stairs and also added the ability to our player to traverse them.
We made the implementation generic so we can also add `StairClimber` and `StairDescender` to any entity which can move!

The only problem is that it is very easy to find our way down if there is no fog of war so in the next article we'll be
adding it to our game.

Until then go forth and *kode on*!
 
> The code of this article can be found under the `9_MULTI_LEVEL_DUNGEON` tag.
