---
excerpt: "Our game is almost complete now, and the only thing which is missing is a Victory and a Lose screen. Let's add them now!"
title: "How To Make a Roguelike: #19 Win and Lose Conditions"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #19 Win and Lose Conditions"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/07/01/how-to-make-a-roguelike-win-and-lose-conditions.html
updated_at: 2021-06-11
---

> Our game has everything it needs: items, monsters, exploration, hunger and experience levels. To make it complete we need to add proper *win* and *lose* conditions as well. Let's make it happen now!

## The Sorry State of the Endgame

In one of the earlier articles we've created a `WinView` and a `LoseView` but we're not using them yet. If the *player* dies nothing happens and we can't interact with the world anymore.

We also can't pick up all the *zircon*s lying around because the inventory quickly fills up.

Moreover there is no *exit* so we are forever stuck in the *Caves of Zircon* Let's start fixing this by reworking how we collect *Zircons*.

## Collecting Zircons

As we've stated on the *Help Dialog* the goal of the game is to collect as many *Zircons* as possible. A *Zircon* doesn't take up much space so it would make sense for the *player* to have a separate pouch only for them. Let's add this to our game!

First of all we're going to need an attribute, `ZirconCounter` for this:

```kotlin
package com.example.cavesofzircon.attributes

import com.example.cavesofzircon.extensions.toStringProperty
import org.hexworks.amethyst.api.base.BaseAttribute
import org.hexworks.cobalt.databinding.api.binding.bindPlusWith
import org.hexworks.cobalt.databinding.api.extension.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.Component

data class ZirconCounter(
        private val zirconCountProperty: Property<Int> = createPropertyFrom(0)
) : BaseAttribute(), DisplayableAttribute {

    var zirconCount: Int by zirconCountProperty.asDelegate()

    override fun toComponent(width: Int): Component {
        val zirconProp = createPropertyFrom("Zircons: ")
                .bindPlusWith(zirconCountProperty.toStringProperty())
        return Components.header()
                .withText(zirconProp.value)
                .withSize(width, 1)
                .build().apply {
                    textProperty.updateFrom(zirconProp)
                }
    }
}
```

then the usual type, which is `ZirconHolder` in this case:

```kotlin
package com.example.cavesofzircon.attributes.types

import com.example.cavesofzircon.attributes.ZirconCounter
import com.example.cavesofzircon.extensions.GameEntity
import org.hexworks.amethyst.api.entity.EntityType

interface ZirconHolder : EntityType

val GameEntity<ZirconHolder>.zirconCounter: ZirconCounter
    get() = findAttribute(ZirconCounter::class).get()
```

Now we can make the `Player` have this type:

```kotlin
object Player : BaseEntityType(
        name = "player"
), Combatant, ItemHolder, EnergyUser, EquipmentHolder, ExperienceGainer, ZirconHolder
```

As for the *systems* this is going to be a bit interesting. What we want to do with *zircons* is to add them to our pouch (`ZirconCounter`) instead of adding it to the inventory. So what we'll do is to make our new `ZirconGatherer` *facet* accept `PickItemUp` events but will only *consume* them if the item in question is a *zircon*. Otherwise it'll return `Pass` and `ItemPicker` gets a chance to do its own work.

Let's see how can we implement it. First of all we're going to move `findTopItem` from `ItemPicker` to `World` since we'll be using it in `ZirconGatherer` as well:

```kotlin
// add this to World.kt
import com.example.cavesofzircon.attributes.types.Item
import com.example.cavesofzircon.extensions.filterType
import org.hexworks.cobalt.datatypes.extensions.flatMap

fun findTopItem(position: Position3D) =
        fetchBlockAt(position).flatMap { block ->
            Maybe.ofNullable(block.entities.filterType<Item>().firstOrNull())
        }
```

Now implementing `ZirconGatherer` becomes simpler:

```kotlin
package com.example.cavesofzircon.systems

import com.example.cavesofzircon.attributes.ZirconCounter
import com.example.cavesofzircon.attributes.types.Zircon
import com.example.cavesofzircon.attributes.types.ZirconHolder
import com.example.cavesofzircon.attributes.types.zirconCounter
import com.example.cavesofzircon.extensions.whenTypeIs
import com.example.cavesofzircon.functions.logGameEvent
import com.example.cavesofzircon.messages.PickItemUp
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.Pass
import org.hexworks.amethyst.api.Response
import org.hexworks.amethyst.api.base.BaseFacet

object ZirconGatherer : BaseFacet<GameContext, PickItemUp>(PickItemUp::class, ZirconCounter::class) {

    override suspend fun receive(message: PickItemUp): Response {
        val (context, source, position) = message
        var response: Response = Pass
        val world = context.world
        world.findTopItem(position).map { item ->               // 1
            source.whenTypeIs<ZirconHolder> { zirconHolder ->   // 2
                if (item.type == Zircon) {                      // 3
                    zirconHolder.zirconCounter.zirconCount++    // 4
                    world.removeEntity(item)                    // 5
                    logGameEvent("$zirconHolder picked up a Zircon!", ZirconGatherer)
                    response = Consumed
                }
            }
        }
        return response
    }
}
```

Here we:

1. Find the top item at the given position
2. And if the entity is a `ZirconHolder`
3. And the item is a `Zircon`
4. We increment the counter
5. And remove the item from the *world*

Now to make this happen we modify the *player* entity to have the new *attribute* and *system*:

```kotlin
import com.example.cavesofzircon.attributes.ZirconCounter
import com.example.cavesofzircon.systems.ZirconGatherer

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
        // ...
        ZirconCounter()
    )
    // ..
    facets(ZirconGatherer, ItemPicker, // ...)
}
```

We only have to make sure that `ItemPicker` is **after** `ZirconGatherer` so it can remove the *zircons* before `ItemPicker` has a chance.

Let's see what we've produced:

![Gathering Zircons](/assets/img/gathering_zircons.gif)

So far so good. Next up is the addition of an *exit* where if we press `f` (move down one level) we win the game!

## Final Exit

The first thing we're going to need is a tile for the *exit*:

```kotlin
// add this to GameTileRepository

val EXIT = Tile.newBuilder()
        .withCharacter('+')
        .withForegroundColor(GameColors.ACCENT_COLOR)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()
```

We're using `+` for it. Next up is the `Exit` type for the *exit* entity:

```kotlin
// add this to EntityTypes.kt

object Exit : BaseEntityType(
        name = "exit"
)
```

and then the *exit* *entity* itself:

```kotlin
// add this to EntityFactory

import com.example.cavesofzircon.attributes.types.Exit

fun newExit() = newGameEntityOfType(Exit) {
    attributes(EntityTile(GameTileRepository.EXIT),
            EntityPosition())
}
```

Adding it to our game is as simple as modifying the `GameBuilder` a bit:

```kotlin
private fun addExit() = also {
    EntityFactory.newExit().addToWorld(0)
}
```

Then we just call `addExit()` from `buildGame` and now we can see it added to the dungeon:

![Finding the Exit](/assets/img/finding_the_exit.png)

Now that we have an actual *exit* we just need to make it enable the *player* to exit through it and win the game! To signal this condition we're going to add a new *event*:

```kotlin
package com.example.cavesofzircon.events

import org.hexworks.cobalt.events.api.Event

data class PlayerWonTheGame(
        val zircons: Int,
        override val emitter: Any
) : Event
```

which we can send from `StairDescender` when the *player* tries to use the exit:

```kotlin
package com.example.cavesofzircon.systems

import com.example.cavesofzircon.attributes.types.Exit
import com.example.cavesofzircon.attributes.types.Player
import com.example.cavesofzircon.attributes.types.StairsDown
import com.example.cavesofzircon.attributes.types.zirconCounter
import com.example.cavesofzircon.blocks.GameBlock
import com.example.cavesofzircon.events.PlayerWonTheGame
import com.example.cavesofzircon.extensions.position
import com.example.cavesofzircon.extensions.whenTypeIs
import com.example.cavesofzircon.functions.logGameEvent
import com.example.cavesofzircon.messages.MoveDown
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.Response
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.zircon.internal.Zircon

object StairDescender : BaseFacet<GameContext, MoveDown>(MoveDown::class) {

    override suspend fun receive(message: MoveDown): Response {
        val (context, source) = message
        val world = context.world
        val pos = source.position
        world.fetchBlockAt(pos).map { block ->
            when {
                block.hasStairsDown -> {
                    logGameEvent("You move down one level...", StairDescender)
                    world.moveEntity(source, pos.withRelativeZ(-1))
                    world.scrollOneDown()
                }
                block.hasExit -> source.whenTypeIs<Player> {    // 1
                    Zircon.eventBus.publish(PlayerWonTheGame(it.zirconCounter.zirconCount, StairDescender))
                }
                else -> logGameEvent("You search for a trapdoor, but you find nothing.", StairDescender)
            }
        }
        return Consumed
    }

    private val GameBlock.hasStairsDown: Boolean
        get() = this.entities.any { it.type == StairsDown }

    private val GameBlock.hasExit: Boolean                      // 2
        get() = this.entities.any { it.type == Exit }
}
```

1. We only check for the exit when the *entity* is indeed the *player*
2. And we add a handy extension property just like we did with `hasStairsDown`

We can listen for this event in `PlayView`:

```kotlin
import com.example.cavesofzircon.events.PlayerWonTheGame

Zircon.eventBus.subscribeTo<PlayerWonTheGame> {
    replaceWith(WinView(grid, it.zircons))
    DisposeSubscription
}
```

and we're going to reconfigure `WinView` a bit:

```kotlin
package com.example.cavesofzircon.view

import com.example.cavesofzircon.GameConfig
import com.example.cavesofzircon.world.GameBuilder
import org.hexworks.zircon.api.ColorThemes
import org.hexworks.zircon.api.ComponentDecorations.box
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.graphics.BoxType
import org.hexworks.zircon.api.grid.TileGrid
import org.hexworks.zircon.api.view.base.BaseView
import kotlin.system.exitProcess

class WinView(
        private val grid: TileGrid,
        private val zircons: Int
) : BaseView(grid, ColorThemes.arc()) {

    init {
        val msg = "You won!"
        val header = Components.textBox(GameConfig.WINDOW_WIDTH / 2)
                .addHeader(msg)
                .addNewLine()
                .addParagraph("Congratulations! You have escaped from Caves of Zircon!", withNewLine = false)
                .addParagraph("You've managed to find $zircons Zircons.")       // 1
                .withAlignmentWithin(screen, ComponentAlignment.CENTER)
                .build()
        val restartButton = Components.button()
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_LEFT)    // 2
                .withText("Restart")
                .withDecorations(box(BoxType.SINGLE))
                .build()
        val exitButton = Components.button()
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_RIGHT)   // 3
                .withText("Quit")
                .withDecorations(box(BoxType.SINGLE))
                .build()

        restartButton.onActivated {
            replaceWith(PlayView(grid, GameBuilder(
                    worldSize = GameConfig.WORLD_SIZE
            ).buildGame()))
        }

        exitButton.onActivated {
            exitProcess(0)
        }

        screen.addComponent(header)
        screen.addComponent(restartButton)
        screen.addComponent(exitButton)
    }
}

```

Here we:

1. Changed the message a bit to include how many *zircons* the player have managed to gather
2. Then we add a *restart* button which creates a new game when clicked
3. And a *quit* button which just stops the process

Let's see how we can win the game:

![Winning The Game](/assets/img/winning_the_game.png)

## Losing the Game

Losing is [fun](http://dwarffortresswiki.org/index.php/v0.34:Losing)! And it is also a bit easier to implement. First, we need an *event* here as well:

```kotlin
package com.example.cavesofzircon.events

import org.hexworks.cobalt.events.api.Event

data class PlayerDied(
        val cause: String,
        override val emitter: Any
) : Event
```

Then we add an extra condition to `Destructible` to check whether the *entity* being destroyed is the *player*:

```kotlin
// new imports
import com.example.cavesofzircon.events.PlayerDied
import com.example.cavesofzircon.extensions.isPlayer
import org.hexworks.zircon.internal.Zircon

// ...
destroyer.receiveMessage(EntityDestroyed(context, target, destroyer))
if (target.isPlayer) {
    Zircon.eventBus.publish(PlayerDied("You died $cause", Destructible))
}
// ...
```

and we start listening to the `PlayerDied` event in `PlayView`:

```kotlin
import com.example.cavesofzircon.events.PlayerDied

Zircon.eventBus.subscribeTo<PlayerDied> {
    replaceWith(LoseView(grid, it.cause))
    DisposeSubscription
}
```

And the `LoseView` also gets a facelift:

```kotlin
package com.example.cavesofzircon.view

import com.example.cavesofzircon.GameConfig.WORLD_SIZE
import com.example.cavesofzircon.world.GameBuilder
import org.hexworks.zircon.api.ColorThemes
import org.hexworks.zircon.api.ComponentDecorations.box
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.graphics.BoxType
import org.hexworks.zircon.api.grid.TileGrid
import org.hexworks.zircon.api.view.base.BaseView
import kotlin.system.exitProcess

class LoseView(
        private val grid: TileGrid,
        private val causeOfDeath: String
) : BaseView(grid, ColorThemes.arc()) {

    init {
        val msg = "Game Over"
        val header = Components.textBox(30)
                .addHeader(msg)
                .addParagraph(causeOfDeath)                 // 1
                .addNewLine()
                .withAlignmentWithin(screen, ComponentAlignment.CENTER)
                .build()
        val restartButton = Components.button()             // 2
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_LEFT)
                .withText("Restart")
                .withDecorations(box(BoxType.SINGLE))
                .build()
        val exitButton = Components.button()                // 3
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_RIGHT)
                .withText("Quit")
                .withDecorations(box(BoxType.SINGLE))
                .build()

        restartButton.onActivated {
            replaceWith(PlayView(grid, GameBuilder(
                    worldSize = WORLD_SIZE
            ).buildGame()))
        }

        exitButton.onActivated {
            exitProcess(0)
        }

        screen.addComponent(header)
        screen.addComponent(restartButton)
        screen.addComponent(exitButton)
    }
}
```

Here:
 
1. We also have some additional info (the cause of death)
2. A restart
3. And a Quit button

Let's see dying in action!

![Dying](/assets/img/dying.png)

Nice.

## Conclusion

**Congratulations**, you have a working game complete with win and lose conditions! In the next article we're going to *wrap up* the tutorial and we'll also discuss the possible next steps and also some of the sources which might help you in your future endeavors!
 
> The code of this article can be found in commit #18.
