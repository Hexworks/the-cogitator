---
excerpt: "Since we have combat, monsters, and fog of war, now is the time to add items to our game!"
title: "How To Make a Roguelike: #12 Items and Inventory"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #12 Items and Inventory"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/05/15/how-to-make-a-roguelike-items-and-inventory.html
---

> We have almost everything for our game. We can explore the cave, fight with monsters, traverse levels, but
there is an important piece which is missing: **items**. In this session we'll add items, and the corresponding
*inventory* into our game.

> For this article we need to update our Maven dependencies for the libraries we use:
> Overwrite these in gradle.properties
>
> cobalt_version=2019.1.0-PREVIEW
> amethyst_version=2019.1.0-PREVIEW
> zircon_version=2019.1.0-PREVIEW


## Between a Rock and a Hard Place

*Items* in a roguelike are very important. They can be used to open doors (*keys*), eaten to fight hunger (*food*)
or gathered to sell them later on (*treasures*). Let's start introducing items to our game by adding *rocks* which
we can pick up...more specifically **Zircons**!.

First of all let's think about how we would represent *items*. What's true for all *items* is that they have a
`Tile` just like the *player* and a *bat*. Because we plan to have an *inventory* as well it seems like a good idea
to have *icons* for our *items* as well. We're going to use `GraphicalTile`s for this as you'll see later.

Since we already have an `EntityTile` let's start by adding an `ItemIcon` *attribute*:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.amethyst.api.Attribute
import org.hexworks.zircon.api.data.GraphicalTile

data class ItemIcon(val iconTile: GraphicalTile) : Attribute
```

With this we can create an interface for our `Item`s and also add some *extension properties* which will help when
using them:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.EntityTile
import org.hexworks.cavesofzircon.attributes.ItemIcon
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.zircon.api.data.GraphicalTile
import org.hexworks.zircon.api.data.Tile

interface Item : EntityType

val GameEntity<Item>.tile: Tile
    get() = findAttribute(EntityTile::class).get().tile

val GameEntity<Item>.iconTile: GraphicalTile
    get() = findAttribute(ItemIcon::class).get().iconTile
```

And we can also add a `typealias` for `GameEntity<Item>` to our `TypeAliases.kt` file:

```kotlin
import org.hexworks.cavesofzircon.attributes.types.Item

typealias GameItem = GameEntity<Item>
```

so that we can refer to game items in a convenient way.
As for our *Zircon* we just have to add a new `EntityType` to our game:

```kotlin
// Add this to EntityTypes.kt

object Zircon : BaseEntityType(
        name = "Zircon",
        description = "A small piece of Zircon. Its value is unfathomable."), Item
```

As usual we're going to add a new `Tile` for this in our `GameTileRepository` and `GameColors`:

```kotlin
// put this in GameColors
val ZIRCON_COLOR = TileColors.fromString("#dddddd")

// and this in GameTileRepository
val ZIRCON  = Tiles.newBuilder()
        .withCharacter(',')
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .withForegroundColor(GameColors.ZIRCON_COLOR)
        .buildCharacterTile()
```

All our items will also be `Entity` objects so to create a *Zircon* we just have to add it to our `EntityFactory`:

```kotlin
import org.hexworks.cavesofzircon.attributes.ItemIcon
import org.hexworks.cavesofzircon.attributes.types.Zircon
import org.hexworks.zircon.api.GraphicalTilesetResources
import org.hexworks.zircon.api.Tiles

fun newZircon() = newGameEntityOfType(Zircon) {
    attributes(ItemIcon(Tiles.newBuilder()
            .withName("white gem")
            .withTileset(GraphicalTilesetResources.nethack16x16())
            .buildGraphicTile()),
            EntityPosition(),
            EntityTile(GameTileRepository.ZIRCON))
}
```

What's interesting here is that we're using the graphical *tiles* from the *Nethack Tileset*. These come from the
Zircon distribution and you can mix and match *CP437* and *Graphical Tiles* as you see fit!

## Adding Zircons to our Game

Now that we have the *entities* we just need to add them to the game world so that our adventurer can pick them up.
Let's scatter them randomly throughout the levels.

First let's define a configuration value for our *zircons* in `GameConfig`:

```kotlin
const val ZIRCONS_PER_LEVEL = 20
``` 

then we can add a function for adding them in `GameBuilder`:

```kotlin
import org.hexworks.cavesofzircon.GameConfig.ZIRCONS_PER_LEVEL

private fun addZircons() = also {
    repeat(world.actualSize().zLength) { level ->
        repeat(GameConfig.ZIRCONS_PER_LEVEL) {
            EntityFactory.newZircon().addToWorld(level)
        }
    }
}
```

and call it in the `buildGame` function:

```kotlin
addZircons()
```

Let's see what we have:

![Finding Zircons](/assets/img/finding_zircons.gif)

So far so good! Now we just need a way to pick them up and store them! Let's introduce the `Inventory`:

## Adding an Inventory

An *inventory* is just an `Attribute` which is capable of storing *entities* which happen to be
`Item`s. A good *inventory* also has a size and some means to add/remove *items* from it. Let's take
a look at our implementation:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.amethyst.api.Attribute
import org.hexworks.cavesofzircon.extensions.GameItem
import org.hexworks.cobalt.datatypes.Identifier
import org.hexworks.cobalt.datatypes.Maybe

class Inventory(val size: Int) : Attribute {                        // 1

    private val currentItems = mutableListOf<GameItem>()    

    val items: List<GameItem>                               
        get() = currentItems.toList()

    val isEmpty: Boolean                                            // 2
        get() = currentItems.isEmpty()

    val isFull: Boolean                                             // 3
        get() = currentItems.size >= size

    fun findItemBy(id: Identifier): Maybe<GameItem> {       // 4
        return Maybe.ofNullable(items.firstOrNull { it.id == id })
    }

    fun addItem(item: GameItem): Boolean {                  // 5
        return if (isFull.not()) {
            currentItems.add(item)
        } else false
    }

    fun removeItem(entity: GameItem): Boolean {             // 6
        return currentItems.remove(entity)
    }
}
```

This implementation is very simple, it:

1. has a size
2. we can tell whether it is empty
3. or full
4. we can find an item in it
5. add new ones
6. and remove them

Now that we have an actual `Inventory` `Entity` let's add a a counterpart for `Item`: `ItemHolder` which represents
an `Entity` which has an `Inventory`:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.Inventory
import org.hexworks.cavesofzircon.extensions.GameItem
import org.hexworks.cavesofzircon.extensions.GameItemHolder

interface ItemHolder : EntityType

fun GameItemHolder.addItem(item: GameItem) = inventory.addItem(item)

fun GameItemHolder.removeItem(item: GameItem) = inventory.removeItem(item)

val GameItemHolder.inventory: Inventory
    get() = findAttribute(Inventory::class).get()
```

add a convenient `typealias` for `GameEntity<ItemHolder>` to `TypeAliases.kt`:

```kotlin
import org.hexworks.cavesofzircon.attributes.types.ItemHolder

typealias GameItemHolder = GameEntity<ItemHolder>
```

and make the *player* type implement it:

```kotlin
object Player : BaseEntityType(
        name = "player"), Combatant, ItemHolder
```

Let's make our *player* have an `Inventory` now so add it in the `newPlayer` function in `EntityFactory`:

```kotlin
// new import
import org.hexworks.cavesofzircon.attributes.Inventory

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
            Vision(9),
            EntityPosition(),
            BlockOccupier,
            CombatStats.create(
                    maxHp = 100,
                    attackValue = 10,
                    defenseValue = 5),
            EntityTile(GameTileRepository.PLAYER),
            EntityActions(Dig::class, Attack::class),
            Inventory(10))
    behaviors(InputReceiver)
    facets(Movable, CameraMover, StairClimber, StairDescender, Attackable, Destructible)
}
```

Now to be able to pick *items* up we just need to add a shortcut for it. Let's use the `p` button for this so whenever
the player stands on a *tile* which has *items* on it and presses `p` the item at the top should appear in the `Inventory`.
You know the drill by now: first we need to add a `Command` for this:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.cavesofzircon.attributes.types.ItemHolder
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.GameItemHolder
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.zircon.api.data.impl.Position3D

data class PickItemUp(override val context: GameContext,
                      override val source: GameItemHolder,                  // 1
                      val position: Position3D) : GameCommand<ItemHolder>
```

Here:

1. We used the `ItemHolder` [trait](https://en.wikipedia.org/wiki/Trait_(computer_programming)) here so this `Command`
   will only work for entities which have inventories. This is similar to what we did with `Combatant` previously where
   we made sure that only those entities can start a combat which have the necessary facets for it.
   
Now that we have a `Command` for picking items up we just need to add the appropriate `Behavior` which consumes it. For this
we need to add a helper function to our `EntityExtensions.kt` which we can use to filter the entities for a given type:

```kotlin
// new imports
import org.hexworks.amethyst.api.entity.Entity
import org.hexworks.amethyst.api.entity.EntityType
import kotlin.reflect.full.isSuperclassOf

inline fun <reified T : EntityType> Iterable<AnyGameEntity>.filterType(): List<Entity<T, GameContext>> {
    return filter { T::class.isSuperclassOf(it.type::class) }.toList() as List<Entity<T, GameContext>>
}
```

With this we can check the items of a `Block` for a given type (`Item` in our case) in the following code:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.types.Item
import org.hexworks.cavesofzircon.attributes.types.addItem
import org.hexworks.cavesofzircon.commands.PickItemUp
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.filterType
import org.hexworks.cavesofzircon.extensions.isPlayer
import org.hexworks.cavesofzircon.functions.logGameEvent
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.cavesofzircon.world.World
import org.hexworks.cobalt.datatypes.Maybe
import org.hexworks.cobalt.datatypes.extensions.flatMap
import org.hexworks.cobalt.datatypes.extensions.map
import org.hexworks.zircon.api.data.impl.Position3D

object ItemPicker : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(PickItemUp::class) { (context, itemHolder, position) ->
        val world = context.world
        world.findTopItem(position).map { item ->       // 1
            if (itemHolder.addItem(item)) {             // 2
                world.removeEntity(item)                // 3
                val subject = if (itemHolder.isPlayer) "You" else "The $itemHolder" // 4
                val verb = if (itemHolder.isPlayer) "pick up" else "picks up"
                logGameEvent("$subject $verb the $item.")
            }
        }
        Consumed
    }

    private fun World.findTopItem(position: Position3D) =
            fetchBlockAt(position).flatMap { block ->       // 5
                Maybe.ofNullable(block.entities.filterType<Item>().firstOrNull())
            }
}
```

`ItemPicker` consumes `PickItemUp` commands and

1. It tries to find the top item on a block and
2. If there was an item there it tries to add it to the inventory of the item holder
3. If that succeeded it removes the item from the world
4. Then prints a nice log message
5. Here what `flatMap` does is that it unwraps the `block` from the `Maybe` (it does nothing if the Maybe
   is empty) and re-wraps it int another `Maybe` which holds our item. Note that this can also be empty if
   there were no `Item`s in the `Block`.
   
Now that we have the `Command` and the `Behavior` ready we just need to add the `p` shortcut to `InputReceiver`:


```kotlin
// new import
import org.hexworks.cavesofzircon.commands.PickItemUp

override fun update(entity: GameEntity<out EntityType>, context: GameContext): Boolean {
    val (_, _, uiEvent, player) = context
    val currentPos = player.position
    if (uiEvent is KeyboardEvent) {
        when (uiEvent.code) {
            KeyCode.KEY_W -> player.moveTo(currentPos.withRelativeY(-1), context)
            KeyCode.KEY_A -> player.moveTo(currentPos.withRelativeX(-1), context)
            KeyCode.KEY_S -> player.moveTo(currentPos.withRelativeY(1), context)
            KeyCode.KEY_D -> player.moveTo(currentPos.withRelativeX(1), context)
            KeyCode.KEY_R -> player.moveUp(context)
            KeyCode.KEY_F -> player.moveDown(context)
            KeyCode.KEY_P -> player.pickItemUp(currentPos, context)     // 1
            else -> {
                logger.debug("UI Event ($uiEvent) does not have a corresponding command, it is ignored.")
            }
        }
    }
    return true
}

private fun GameEntity<Player>.pickItemUp(position: Position3D, context: GameContext) {     // 2
    executeCommand(PickItemUp(context, this, position))
}

// ...
```

Here we:

1. Just add a new key handler for `KEY_P`
2. And a helper method which fires the `PickItemUp` command. `PickItemUp` only accepts `ItemHolder`s but we
   augmented the `Player` type with it so everything works just fine.

Now if we add the `ItemPicker` *facet* to our player entity:

```kotlin
// new import
import org.hexworks.cavesofzircon.systems.ItemPicker

fun newPlayer() = newGameEntityOfType(Player) {

    // ...

    facets(/* ... */ ItemPicker)
}
```

we're able to pick up items in our game:

![Looting Zircons](/assets/img/looting_zircons.gif)

**Wow**, that's nice! But how do we check the *inventory*?

## Displaying the Inventory

Creating a nice UI is not an easy task especially if you have no experience in designing one. Luckily Zircon comes
with a bunch of `Component`s we can use to simplify it. Let's think about how we would display an inventory.
A tabular display will probably suffice with a row for each item. In a row we would display the *icon*, the *name*
of the item and a `Button` we can press to drop the *item*. We can also add a *title* and a *header* as a bonus!
Let's see how we can go about implementing it.

First, we're gonna need a *row* `Fragment` for the items:

> A `Fragment` is just a wrapper for a `Component` which lets us re-use our UI elements. It differs from a simple
`Container` like a `Panel` in a way that a `Fragment` is its own class we can instantiate, and it can also have
its own internal state. Since the `Fragment` is a wrapper it needs some component to wrap. This is called its
`root` which can be *any* component as you'll see later.

```kotlin
package org.hexworks.cavesofzircon.view.fragment

import org.hexworks.cavesofzircon.attributes.types.iconTile
import org.hexworks.cavesofzircon.extensions.GameItem
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.Fragment
import org.hexworks.zircon.api.graphics.Symbols

class InventoryRowFragment(width: Int, item: GameItem) : Fragment {     // 1

    val dropButton = Components.button()                        // 2
            .wrapSides(false)
            .withText("${Symbols.ARROW_DOWN}")                  // 3
            .build()

    override val root = Components.hbox()                       // 4
            .withSpacing(1)                                     // 5
            .withSize(width, 1)
            .build().apply {
                addComponent(Components.icon()                  // 6
                        .withIcon(item.iconTile))
                addComponent(Components.label()
                        .withSize(InventoryFragment.NAME_COLUMN_WIDTH, 1)   // 7
                        .withText(item.name))
                addComponent(dropButton)
            }
}
``` 

Where we

1. Implement `Fragment` and take the `width` and the `item` as a parameter
2. Add a `Button` for dropping the *item*
3. Make it have a nice *down arrow* as an icon
4. And we make the `root` a `HBox`
5. Which will add a spacing of one tile between its child components
6. And add the *icon* 
7. And the *name* to it.

> A `HBox` is a *container* component which aligns the components you add to it
from left to right automatically and it also handles item removal. It looks like 
[this](https://cdn.discordapp.com/attachments/363754040103796737/576844557619167273/hbox.gif)
and it is very useful for implementing complex UI layouts.

Next up is the `InventoryFragment` which will display all rows, the header and the title. There are a lot of
schools for designing UI workflows, but we're not going to pick a specific method. Instead we're going to be
pragmatic: our UI elements will only know about the things which are *necessary* for them and the locality
of our operations will follow the same pattern. This means that the *inventory* will know that when we *drop*
an item it needs to remove a row, but anything else is not its concern, so we're going to have a callback
which we'll use from our `System` which handles dropping:

```kotlin
package org.hexworks.cavesofzircon.view.fragment

import org.hexworks.cavesofzircon.attributes.Inventory
import org.hexworks.cavesofzircon.extensions.GameItem
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.Fragment
import org.hexworks.zircon.api.extensions.onComponentEvent
import org.hexworks.zircon.api.uievent.ComponentEventType.ACTIVATED
import org.hexworks.zircon.api.uievent.Processed

class InventoryFragment(inventory: Inventory,
                        width: Int,
                        onDrop: (GameItem) -> Unit) : Fragment {

    override val root = Components.vbox()           // 1
            .withSize(width, inventory.size + 1)
            .build().apply {
                val list = this
                addComponent(Components.hbox()      // 2
                        .withSpacing(1)
                        .withSize(width, 1)
                        .build().apply {
                            addComponent(Components.label().withText("").withSize(1, 1))
                            addComponent(Components.header().withText("Name").withSize(NAME_COLUMN_WIDTH, 1))
                            addComponent(Components.header().withText("Actions").withSize(ACTIONS_COLUMN_WIDTH, 1))
                        })
                inventory.items.forEach { item ->                           // 3
                    addFragment(InventoryRowFragment(width, item).apply {
                        dropButton.onComponentEvent(ACTIVATED) {            // 4
                            list.removeComponent(this.root)                 // 5
                            onDrop(item)                                    // 6
                            Processed
                        }
                    })
                }
            }

    companion object {
        const val NAME_COLUMN_WIDTH = 15
        const val ACTIONS_COLUMN_WIDTH = 10
    }
}
```

Here we:

1. Create a `VBox` for holding the rows which is very similar to the `HBox` but it aligns its items from top to
   bottom instead
2. Add a header
3. And for each inventory item we add a row fragment
4. And an event listener which is activated whenever the `dropButton` gets activated
5. In that case we remove the whole row
6. And call the `onDrop` callback

Now we just need to think about how to open the inventory itself. For this we're going to add 2 new systems:
`InventoryInspector` and `ItemDropper`. Let's start with `ItemDropper` as it is very simple. For it we just need
a `Command`, `DropItem`:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.GameItem
import org.hexworks.cavesofzircon.extensions.GameItemHolder
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.zircon.api.data.impl.Position3D

data class DropItem(override val context: GameContext,
                    override val source: GameItemHolder,
                    val item: GameItem,
                    val position: Position3D) : GameCommand<EntityType>
```

and the `Facet` `ItemDropper`:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.types.inventory
import org.hexworks.cavesofzircon.attributes.types.removeItem
import org.hexworks.cavesofzircon.commands.DropItem
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.isPlayer
import org.hexworks.cavesofzircon.functions.logGameEvent
import org.hexworks.cavesofzircon.world.GameContext

object ItemDropper : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command
            .responseWhenCommandIs(DropItem::class) { (context, itemHolder, item, position) ->  // 1
                if (itemHolder.removeItem(item)) {              // 2
                    context.world.addEntity(item, position)     // 3
                    val subject = if (itemHolder.isPlayer) "You" else "The $itemHolder"
                    val verb = if (itemHolder.isPlayer) "drop" else "drops"
                    logGameEvent("$subject $verb the $item.")
                }
                Consumed
            }
}
```

In this `System` we:

1. Consume the `DropItem` command
2. Try to remove the item from the `itemHolder`
3. And if it was successful we add it back to the `World`

Inventory inspection is similar but it is a bit more involved, because we're going to use a *dialog* for it!
Let's start with a `Command`, `InspectInventory`:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.GameItemHolder
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.zircon.api.data.impl.Position3D

data class InspectInventory(override val context: GameContext,
                            override val source: GameItemHolder,
                            val position: Position3D) : GameCommand<EntityType>
```

and let's see how `InventoryInspector` looks like:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.GameConfig
import org.hexworks.cavesofzircon.attributes.types.inventory
import org.hexworks.cavesofzircon.commands.DropItem
import org.hexworks.cavesofzircon.commands.InspectInventory
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.view.fragment.InventoryFragment
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.Sizes
import org.hexworks.zircon.api.builder.component.ModalBuilder
import org.hexworks.zircon.api.component.ComponentAlignment.BOTTOM_LEFT
import org.hexworks.zircon.api.extensions.onComponentEvent
import org.hexworks.zircon.api.uievent.ComponentEventType.ACTIVATED
import org.hexworks.zircon.api.uievent.Processed
import org.hexworks.zircon.internal.component.modal.EmptyModalResult

object InventoryInspector : BaseFacet<GameContext>() {

    val DIALOG_SIZE = Sizes.create(33, 14)

    override fun executeCommand(command: GameCommand<out EntityType>) = command
            .responseWhenCommandIs(InspectInventory::class) { (context, itemHolder, position) ->

                val screen = context.screen

                val panel = Components.panel()      // 1
                    .withSize(DIALOG_SIZE)
                    .wrapWithBox(true)
                    .wrapWithShadow(true)
                    .withTitle("Inventory")
                    .build()

                val fragment = InventoryFragment(itemHolder.inventory, DIALOG_SIZE.width - 3) { item -> // 2
                    itemHolder.executeCommand(DropItem(context, itemHolder, item, position))            // 3
                }

                panel.addFragment(fragment)

                val modal = ModalBuilder.newBuilder<EmptyModalResult>()     // 4
                        .withParentSize(screen.size)
                        .withComponent(panel)
                        .build()

                panel.addComponent(Components.button()                      // 5
                        .withText("Close")
                        .withAlignmentWithin(panel, BOTTOM_LEFT)
                        .build().apply {
                            onComponentEvent(ACTIVATED) {
                                modal.close(EmptyModalResult)
                                Processed
                            }
                        })

                modal.applyColorTheme(GameConfig.THEME)
                screen.openModal(modal)
                Consumed
            }

}
```

This `Facet` works in a way that:

1. When we want to open the inventory it creates a `Panel`
2. Which holds our inventory fragment
3. And responds to the `onDrop` callback by sending the `DropItem` command to our entity
4. And uses a `Modal` for displaying our panel
5. And has a close button which will close the modal when clicked.

> A `Modal` is an object which will block all inputs to other components until it is closed. If you have
ever seen an `alert` dialog in a browser this concept will be familiar. `Modal`s can also return a value,
but here we use the built-in `EmptyModalResult` because we don't want to use this feature here.

Now we just have to add all this to our *player* entity:

```kotlin
// new imports
import org.hexworks.cavesofzircon.systems.InventoryInspector
import org.hexworks.cavesofzircon.systems.ItemDropper

fun newPlayer() = newGameEntityOfType(Player) {

    // ...
    
    facets(/* ... */ InventoryInspector, ItemDropper)
}
```

and augment our `InputReceiver` with the new shortcut for opening the inventory: the `i` key:

```kotlin
// new import
import org.hexworks.cavesofzircon.commands.InspectInventory

// ...

override fun update(entity: GameEntity<out EntityType>, context: GameContext): Boolean {
    val (_, _, uiEvent, player) = context
    val currentPos = player.position
    if (uiEvent is KeyboardEvent) {
        when (uiEvent.code) {
            KeyCode.KEY_W -> player.moveTo(currentPos.withRelativeY(-1), context)
            KeyCode.KEY_A -> player.moveTo(currentPos.withRelativeX(-1), context)
            KeyCode.KEY_S -> player.moveTo(currentPos.withRelativeY(1), context)
            KeyCode.KEY_D -> player.moveTo(currentPos.withRelativeX(1), context)
            KeyCode.KEY_R -> player.moveUp(context)
            KeyCode.KEY_F -> player.moveDown(context)
            KeyCode.KEY_P -> player.pickItemUp(currentPos, context)
            KeyCode.KEY_I -> player.inspectInventory(currentPos, context)
            else -> {
                logger.debug("UI Event ($uiEvent) does not have a corresponding command, it is ignored.")
            }
        }
    }
    return true
}

private fun GameEntity<Player>.inspectInventory(position: Position3D, context: GameContext) {
    executeCommand(InspectInventory(context, this, position))
}

// ...
```

Now let's see what we created:

![Dropping Items](/assets/img/dropping_items.gif)

Now, that's nice. We've just made our roguelike great again!

## Conclusion

In this session we added a whole *inventory* system into our game complete with *items* and a nice
inventory dialog. This is a nice start, so next we're going to add more items, and some new mechanics
to our game: *food* and *hunger*!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `12_ITEMS_AND_INVENTORY` tag.
