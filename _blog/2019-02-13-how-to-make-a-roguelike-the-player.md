---
excerpt: Our cave is ready to explore, so let's add a player to it!
title: "How To Make a Roguelike: #4 The Player"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #4 The Player"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/02/13/how-to-make-a-roguelike-the-player.html
updated_at: 2019-02-13
---

> With our cave in place the next thing to do is to add a Player to move around the cave.
For this we'll use [Amethyst](https://github.com/Hexworks/amethyst), a very simple ECS(ish)
library. We'll also upgrade our Zircon version which now has more useful support for component
events!

## Updating Zircon

Since the last article Zircon received a major upgrade to input handling which will make our life
much easier in the long run, so let's start by updating our `gradle.properties` file with the
new versions:

```groovy
version=2019.0.1-PREVIEW

kotlin_version=1.3.21

cobalt_version=2018.1.1-PREVIEW
amethyst_version=2019.0.4-PREVIEW
zircon_version=2019.0.14-PREVIEW
```

In order for this to work we'll need to change some of our views which had buttons on them: `StartView`,
`WinView` and `LoseView`. The change is minor, but very important: a new kind of event was added to input
handling in Zircon: `ComponentEvent`s. So now we can listen to component activation events on all our
components like this:

```kotlin
myComponent.onComponentEvent(ComponentEventType.ACTIVATED) { // 1
    // 2
    Processed // 3
}
```

What happens here is:

1. We add a listener for component events to our component. Here we listen to the `ACTIVATED` event. This either
   means that the user clicked an interactable component (`Button`, `CheckBox` or something similar) or the user
   pressed the activation key on a focused component ([Spacebar] right now).
2. We process the event
3. We return `Processed` which means that we handled the event

> Clicking is straightforward, but what about "pressing the activation key"? Well, this is similar to what you do
when you navigate in your browser using [Tab] and [Shift] + [Tab]. If you do so focus wanders around between the
controls in your browser an when you press [Spacebar] the control is activated. Try it! This works the same way
in Zircon.

> And what's that `Processed` thingy we return from our listener? Well, that's part of the event propagation model
which Zircon supports. You can read up on that topic [here](/zircon/docs/2018-11-21-input-handling).

Armed with this knowledge let's modify our event listeners in the aforementioned views.

`StartView`:

```kotlin
        startButton.onComponentEvent(ComponentEventType.ACTIVATED) {
            replaceWith(PlayView())
            close()
            Processed
        }
```

`WinView`:

```kotlin
        restartButton.onComponentEvent(ComponentEventType.ACTIVATED) {
            replaceWith(PlayView())
            close()
            Processed
        }

        exitButton.onComponentEvent(ComponentEventType.ACTIVATED) {
            System.exit(0)
            Processed
        }
```

and finally `LoseView`:

```kotlin
        restartButton.onComponentEvent(ComponentEventType.ACTIVATED) {
            replaceWith(PlayView())
            close()
            Processed
        }

        exitButton.onComponentEvent(ComponentEventType.ACTIVATED) {
            System.exit(0)
            Processed
        }
```

If your IDE doesn't help you
these are the imports you will need to add to your views:

```kotlin
import org.hexworks.zircon.api.extensions.onComponentEvent
import org.hexworks.zircon.api.uievent.ComponentEventType
import org.hexworks.zircon.api.uievent.Processed
```

## Introducing Amethyst

So what is Amethyst, and why is it useful for us? Amethyst is a small library which takes
care of managing your game entities in a nutshell. It is similar to an [Entity Component System](https://en.wikipedia.org/wiki/Entity_component_system)
but it is a bit different in some important aspects.

"So what is a game entity anyway?", you might ask. A game entity can be anything which has its own behavior or state
in your game, form a goblin, to a wall, or even immaterial effects, like shadows. What's important about a game entity
is that it encapsulates all internal state of our goblin for example and it also contains all actions it can
perform. This means that by using this concept we can maintain the [Cohesion](https://en.wikipedia.org/wiki/Cohesion_(computer_science))
of our game objects while still retaining the flexibility.

So how does it work? Let's look at the *SEA*:

### Systems, Entities, Attributes

When working with an ECS, you'll probably see things like Entities, Components, and Systems. In Amethyst we have
Systems, Entities and Attributes. I'm not going into detail about the difference between ECS and SEA, but explain how
SEA works:

Let's say that we want to add a *goblin* to our game. If we think about how things can behave and change over time
we can conclude, that there are essentially two ways: internal, and external. For example a *goblin* can decide
that it wants to explore the dungeon, or just take a look around (**internal change**) or the player can decide to bash
the goblin to a pulpy mass with their club (**external change**). For this Amethyst has `System`s in place:

```kotlin
/**
 * A [System] is responsible for updating the internal state
 * ([Attribute]s) of an [Entity].
 */
interface System<C : Context>
```

There are several types of `System`s, and for our discussion here we need to know about `Behavior`:

```kotlin
/**
 * A [Behavior] is a [System] which performs actions autonomously
 * with entities whenever they are [update]d.
 */
interface Behavior<C : Context> : System<C> {

    fun update(entity: Entity<EntityType, C>, context: C): Boolean
}
```

which lets our *goblin* interact with the world, and `Facet`:

```kotlin
/**
 * A [Facet] is a [System] which performs actions based on the
 * [Command]s they receive.
 */
interface Facet<C : Context> : System<C> {

    fun executeCommand(command: Command<out EntityType, C>): Response
}
```

which lets whe **world** interact with our *goblin*.

> There is also `Actor` which combines `Facet` and `Behavior`. We'll talk about it later.

When a change happens over time to an entity (our *goblin* in this example) its state might change. To represent
this, Amethyst gives us `Attribute`s:

```kotlin
/**
 * An [Attribute] represents metadata about an entity which can
 * change over time.
 */
interface Attribute
```

An `Attribute` can be anything which you add to your entity, from health points to stomach contents. What's important
is that `Attribute`s should **never have** behavior, they are supposed to be dumb data structures.

On the other hand, `System`s should never have internal state. These two *important rules* allow us to compose `System`s
and `Attribute`s into `Entity` objects which are flexible, cohesive and safe to use.

The `Entity` itself is just a bag of `Attribute`s and `System`s:

```kotlin
interface Entity<out T : EntityType, C : Context> {

    val type: T

    val name: String

    val description: String
    
    val attributes: Sequence<Attribute>

    val facets: Sequence<Facet<C>>
    
    val behaviors: Sequence<Behavior<C>>
    
    fun sendCommand(command: Command<out EntityType, C>): Boolean

    fun executeCommand(command: Command<out EntityType, C>): Response

    fun update(context: C): Boolean

}
```

What's interesting here is `sendCommand`, `executeCommand` and `update`. It is not a coincidence that we have these in
`Facet` and `Behavior`! When an `Entity` receives a `Command` it will try to apply it to its `Facet`s, and when an `Entity`
is updated it lets its `Behavior`s interact with the world. What *world* means in this context is up to you, that's why
`update` takes a `context` object which can be anything. In our case it will contain our `World` for example.

Don't worry if this seems a bit complex, we'll see soon that the benefits of using such system outweigh the costs. 

So how do these entities work together? We have `Engine` for that which handles them so we don't have to
do it by hand:

```kotlin
interface Engine<T : Context> {

    /**
     * Adds the given [Entity] to this [Engine].
     */
    fun addEntity(entity: Entity<EntityType, T>)

    /**
     * Removes the given [Entity] from this [Engine].
     */
    fun removeEntity(entity: Entity<EntityType, T>)

    /**
     * Updates the [Entity] objects in this [Engine] with the given [context].
     */
    fun update(context: T)
}
```

As you can see the `Engine` is responsible for handling all your `Entity` objects and also for updating them.
We'll take a look at how this works in practice below, so prepare your keyboard!

## Enhancing our Game

The main goal of this tutorial session is to add a player to our `Game` which we can use to explore 
the dungeon, remember?

For this we're going to need a player `Entity` and a context object which is used when updating the 
game world. The `Context`
object is straightforward:

```kotlin
package org.hexworks.cavesofzircon.world

import org.hexworks.amethyst.api.Context
import org.hexworks.cavesofzircon.attributes.types.Player
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.zircon.api.screen.Screen
import org.hexworks.zircon.api.uievent.UIEvent

data class GameContext(val world: World, // 1
                       val screen: Screen,  // 2
                       val uiEvent: UIEvent, // 3
                       val player: GameEntity<Player>) : Context // 4
```

What we need for an update is:

1. The world itself
2. The `Screen` object which we can use to open dialogs and interact with the UI in general
3. The `UIEvent` which caused the update of the world (a key press for example)
4. The object representing the player. This is optional, but because we use the player in a lot of
   places it makes sense to add it here
   
> Wait, what is this funky \<Player\> thing? It is called generics which is supported by Kotlin. In case if you
come from a dynamically typed language like Python this can be hard to grasp at first. I recommend reading
about generics in general [here](https://www.oracle.com/technetwork/articles/java/juneau-generics-2255374.html) and
about Kotlin generics [here](https://kotlinlang.org/docs/reference/generics.html).

### Quick Detour: Typealiases and Extension Functions

> You can read up on type aliases [here](https://kotlinlang.org/docs/reference/type-aliases.html) and extension functions [here](https://kotlinlang.org/docs/reference/extensions.html)
if you need more information on the topic.

We're going to use a `typealias` soon. Let's explore what is a `typealias` and why is it useful!

A `typealias` can be used to give names (aliases) to otherwise complex or hard to read types. For example in our
game we will only use a single `Context` object which is `GameContext` so it makes sense to pre-fill the second
generic type parameter for `Entity` with our `GameContext` type so we don't have to type it out every time:

```kotlin
typealias GameEntity<T> = Entity<T, GameContext>
```

What's good is that we can now use `GameEntity` in our code anywhere where we would have used `Entity<T, GameContext>`.
One advantage of this is that our code gets more readable. This is a good thing in itself but there is another
reason to use `typealias` which we're going to exploit. We can define extension functions on type aliases!
This means that this code is perfectly valid:

```kotlin
fun GameContext.doSomething() {
    // do something
}
```

We'll use this in the future for our advantage.

> Note that `typealias` doesn't create a new type. It is just [syntactic sugar](https://en.wikipedia.org/wiki/Syntactic_sugar).

   
### Creating our First Entity   
   
So what's a `GameEntity` and `Player`? Let's take a look at them:

```kotlin
package org.hexworks.cavesofzircon.extensions

// put this in a file called TypeAliases.kt

import org.hexworks.amethyst.api.entity.Entity
import org.hexworks.cavesofzircon.world.GameContext

typealias AnyGameEntity = Entity<EntityType, GameContext>

typealias GameEntity<T> = Entity<T, GameContext>
```

Here we define `GameEntity` which is used to represent an `Entity` which uses *our* `GameContext` type
parameter, and also `AnyGameEntity` which can be used to define functions later for all objects which
are our game entities.

`Player` is an `EntityType`:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

// put this in a file called EntityTypes.kt

import org.hexworks.amethyst.api.base.BaseEntityType

object Player : BaseEntityType(
        name = "player")
```

Each `Entity` in *Amethyst* must have an `EntityType`. This is necessary for easy identification of
`Entity` objects. It looks like this:

```kotlin
interface EntityType : Attribute {

    val name: String

    val description: String
}
```

It is just a `name` and a `description`, and this type will also be added to our `Entity` as an `Attribute` later on.
We used the `BaseEntityType` base class as you can see in the above code which is handy if we don't want to type much.

There are some more things which our player needs. In fact all of our game entities will need them: a *position*
and a *tile*!

Let's create our first `Attribute`s now, `EntityPosition` and `EntityTile`!

`EntityPosition` will hold the current position of an `Entity`:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.amethyst.api.Attribute
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.zircon.api.data.impl.Position3D

class EntityPosition(initialPosition: Position3D = Position3D.unknown()) : Attribute { // 1
    private val positionProperty = createPropertyFrom(initialPosition) // 2

    var position: Position3D by positionProperty.asDelegate() // 3
}

```

This is just 3 lines of code, but there are some new concepts introduced here:

1. We add `initialPosition` as a constructor parameter to our class and its default value is `unknown`.
   What's this? `Position3D` comes from Zircon and can be used to represent a point in 3D space (as we have
   discussed before), and `unknown` impelments the [Null Object Pattern](https://en.wikipedia.org/wiki/Null_object_pattern)
   for us.
2. Here we create a private `Property` from the `initialPosition`. What's a `Property` you might ask? Well, it
   is used for data binding. A `Property` is a wrapper for a value which can change over time. It can be bound
   to other `Property` objects so their values change together and you can also add change listeners to them.
   `Property` comes from the [Cobalt library](https://github.com/Hexworks/cobalt) we use and it works in
   a very similar way as properties work in [JavaFX](https://docs.oracle.com/javafx/2/binding/jfxpub-binding.htm).
3. We create a *Kotlin delegate* from our `Property`. This means that `position` will be accessible to the outside
   world as if it was a simple field, but it takes its value from our `Property` under the hood.
   I've written more about delegation [here](http://the-cogitator.com/posts/blog/2018/09/29/by-the-way-exploring-delegation-in-kotlin.html)
   if you want more information on the topic.

`EntityTile` is an `Attribute` which holds the `Tile` of an `Entity` we use to display it in our world:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.amethyst.api.Attribute
import org.hexworks.zircon.api.Tiles
import org.hexworks.zircon.api.data.Tile

data class EntityTile(val tile: Tile = Tiles.empty()) : Attribute

```

We are using `Tiles.empty()` as a default value here.
   
Now we're going to employ a little Kotlin trick and create some properties on `AnyGameObject` which will supply the
position and tile of any of our entities!

```kotlin
package org.hexworks.cavesofzircon.extensions

// put this in a file called EntityExtensions.kt

import org.hexworks.amethyst.api.Attribute
import org.hexworks.cavesofzircon.attributes.EntityPosition
import org.hexworks.cavesofzircon.attributes.EntityTile
import org.hexworks.cobalt.datatypes.extensions.map
import org.hexworks.cobalt.datatypes.extensions.orElseThrow
import org.hexworks.zircon.api.data.Tile
import kotlin.reflect.KClass

var AnyGameEntity.position // 1
    get() = tryToFindAttribute(EntityPosition::class).position // 2
    set(value) { // 3
        findAttribute(EntityPosition::class).map {
            it.position = value
        }
    }

val AnyGameEntity.tile: Tile
    get() = this.tryToFindAttribute(EntityTile::class).tile

// 4
fun <T : Attribute> AnyGameEntity.tryToFindAttribute(klass: KClass<T>): T = findAttribute(klass).orElseThrow {
    NoSuchElementException("Entity '$this' has no property with type '${klass.simpleName}'.")
}
```  

Here we

1. Create an extension property (works the same way as an extension function) on `AnyGameEntity`.
2. Define a getter for it which tries to find the `EntityPosition` attribute in our `Entity` and throws
   and exception if the `Entity` has no position.
3. We also define a setter for it which sets the `Property` we defined before   
4. Define a function which implements the "try to find or throw an exception" logic for both of our
   properties.

> Typealiases, extension properties...why do we employ all this magic? If you have used an ECS library
before you might have had the feeling that working with them is cumbersome and the resulting program
won't be readable in a way that it expresses the intent properly. What we do here is plumbing which
we can use later to write actual game code which is readable and easy to understand. In other words
this is the same concept as [Plumbing and Porcelain](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)
in Git.

With this change now we can easily access the *position* and *tile* of any of our entities like this:

```kotlin

myEntity.position
myEntity.tile

```

This makes reading and writing code which works with entities much easier.

So far so good, but we still don't see the player! Let's create a `Tile` for the player and a
factory object for creating entities now.

For the player `Tile` we're going to add a nice color to `GameColors`:

```kotlin
val ACCENT_COLOR = TileColors.fromString("#FFCD22")
```

and the tile itself to `GameTileRepository`:

```kotlin
    val PLAYER = Tiles.newBuilder()
            .withCharacter('@')
            .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
            .withForegroundColor(GameColors.ACCENT_COLOR)
            .buildCharacterTile()
```

For creating the actual player `Entity` we'll have a factory object:

```kotlin
package org.hexworks.cavesofzircon.builders

import org.hexworks.amethyst.api.Entities
import org.hexworks.amethyst.api.builder.EntityBuilder
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.EntityPosition
import org.hexworks.cavesofzircon.attributes.EntityTile
import org.hexworks.cavesofzircon.attributes.types.Player
import org.hexworks.cavesofzircon.world.GameContext

fun <T : EntityType> newGameEntityOfType(type: T, init: EntityBuilder<T, GameContext>.() -> Unit) =
        Entities.newEntityOfType(type, init) // 1

object EntityFactory { // 2

    fun newPlayer() = newGameEntityOfType(Player) { // 3
        attributes(EntityPosition(), EntityTile(GameTileRepository.PLAYER)) // 4
        behaviors()
        facets()
    }
}
```

> If you are wondering what the dot (`.`) means in this funky code: `EntityBuilder<T, GameContext>.() -> Unit`
  then read on the topic of **receiver**s [here](https://stackoverflow.com/questions/45875491/what-is-a-receiver-in-kotlin). 

We're going to use `EntityFactory` in the future to create all our `Entity` objects. What happens here is:

1. We add a function which calls `Entities.newEntityOfType` and pre-fills the generic type parameter for
   `Context` with `GameContext`.
2. We define our factory as an `object` since we'll only ever have a single instance of it.
3. We add a function for creating a `newPlayer` and call `newGameEntityOfType` with our previously created
   `Player` type.
4. We specify our `Attribute`s, `Behavior`s and `Facet`s. We only have `Attributes` so far though.


### Firing up the Engine

So now that we have the means of creating `Entity` objects we're going to put the `Engine` somewhere
which will take care of them. For now this will belong to `World`. We'll also enable our `GameBlock`s
to hold onto entities since most of the time we'll be interested in checking whether a given block
has an `Entity` or not. Let's take a look at what we'll need to change:

The `GameBlock` now contains entities:

```kotlin
package org.hexworks.cavesofzircon.blocks

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.builders.GameTileRepository
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.extensions.tile
import org.hexworks.zircon.api.data.BlockSide
import org.hexworks.zircon.api.data.Tile
import org.hexworks.zircon.api.data.base.BlockBase

class GameBlock(private var defaultTile: Tile = GameTileRepository.FLOOR,
                private val currentEntities: MutableList<GameEntity<EntityType>>
                 = mutableListOf()) // 1
    : BlockBase<Tile>() {

    val isFloor: Boolean
        get() = defaultTile == GameTileRepository.FLOOR

    val isWall: Boolean
        get() = defaultTile == GameTileRepository.WALL
        
    val isEmptyFloor: Boolean // 2
            get() = currentEntities.isEmpty()     

    val entities: Iterable<GameEntity<EntityType>> // 3
        get() = currentEntities.toList()

    override val layers: MutableList<Tile> // 4
        get() {
            val entityTiles = currentEntities.map { it.tile }
            val tile = when {
                entityTiles.contains(GameTileRepository.PLAYER) -> GameTileRepository.PLAYER // 5
                entityTiles.isNotEmpty() -> entityTiles.first() // 6
                else -> defaultTile // 7
            }
            return mutableListOf(tile)
        }

    fun addEntity(entity: GameEntity<EntityType>) { // 8
        currentEntities.add(entity)
    }

    fun removeEntity(entity: GameEntity<EntityType>) { // 9
        currentEntities.remove(entity)
    }

    override fun fetchSide(side: BlockSide): Tile {
        return GameTileRepository.EMPTY
    }
}
```

> If you are puzzled by what these `get()`s are in the code then feel free to peruse
the relevant [documentation](https://kotlinlang.org/docs/reference/properties.html) page.
We'll use getters like this throughout the tutorial.

What we changed is:

1. We added `currentEntities` which is just a mutable list of `Entity` objects which is
   empty by default
2. We add a property which tells whether this block is just a floor (similar to `isWall`)
3. Exposed a getter for entities which takes a snapshot (defensive copy) of the current
   entities and returns them. We do this because we don't want to expose the internals of
   `GameBlock` which would make `currentEntities` mutable to the outside world
4. Incorporated our entities to how we display a block by
5. Checking if the player is at this block. If yes it is displayed on top
6. Otherwise the first `Entity` is displayed if present
7. Or the default tile if not
8. We expose a function for adding an `Entity` to our block
9. And also for removing one

In our `World` class we're going to need to add functions for adding entities, and also
for finding empty positions where we can add them. The upgraded `World` class looks like this:

```kotlin
package org.hexworks.cavesofzircon.world

import org.hexworks.amethyst.api.Engine
import org.hexworks.amethyst.api.Engines
import org.hexworks.amethyst.api.entity.Entity
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.blocks.GameBlock
import org.hexworks.cavesofzircon.builders.GameBlockFactory
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.extensions.position
import org.hexworks.cobalt.datatypes.Maybe
import org.hexworks.cobalt.datatypes.extensions.fold
import org.hexworks.cobalt.datatypes.extensions.map
import org.hexworks.zircon.api.Positions
import org.hexworks.zircon.api.builder.game.GameAreaBuilder
import org.hexworks.zircon.api.data.Tile
import org.hexworks.zircon.api.data.impl.Position3D
import org.hexworks.zircon.api.data.impl.Size3D
import org.hexworks.zircon.api.game.GameArea

class World(startingBlocks: Map<Position3D, GameBlock>,
            visibleSize: Size3D,
            actualSize: Size3D) : GameArea<Tile, GameBlock> by GameAreaBuilder.newBuilder<Tile, GameBlock>()
        .withVisibleSize(visibleSize)
        .withActualSize(actualSize)
        .withDefaultBlock(DEFAULT_BLOCK)
        .withLayersPerBlock(1)
        .build() {

    private val engine: Engine<GameContext> = Engines.newEngine() // 1

    init {
        startingBlocks.forEach { pos, block ->
            setBlockAt(pos, block)
            block.entities.forEach { entity ->
                engine.addEntity(entity) // 2
                entity.position = pos // 3
            }
        }
    }

    /**
     * Adds the given [Entity] at the given [Position3D].
     * Has no effect if this world already contains the
     * given [Entity].
     */
    fun addEntity(entity: Entity<EntityType, GameContext>, position: Position3D) { // 4
        entity.position = position
        engine.addEntity(entity)
        fetchBlockAt(position).map {
            it.addEntity(entity)
        }
    }

    fun addAtEmptyPosition(entity: GameEntity<EntityType>, // 5
                           offset: Position3D = Positions.default3DPosition(),
                           size: Size3D = actualSize()): Boolean {
        return findEmptyLocationWithin(offset, size).fold(
                whenEmpty = { // 6
                    false
                },
                whenPresent = { location ->  // 7
                    addEntity(entity, location)
                    true
                })

    }

    /**
     * Finds an empty location within the given area (offset and size) on this [World].
     */
    fun findEmptyLocationWithin(offset: Position3D, size: Size3D): Maybe<Position3D> { // 8
        var position = Maybe.empty<Position3D>()
        val maxTries = 10
        var currentTry = 0
        while (position.isPresent.not() && currentTry < maxTries) {
            val pos = Positions.create3DPosition(
                    x = (Math.random() * size.xLength).toInt() + offset.x,
                    y = (Math.random() * size.yLength).toInt() + offset.y,
                    z = (Math.random() * size.zLength).toInt() + offset.z)
            fetchBlockAt(pos).map {
                if (it.isEmptyFloor) {
                    position = Maybe.of(pos)
                }
            }
            currentTry++
        }
        return position
    }

    companion object {
        private val DEFAULT_BLOCK = GameBlockFactory.floor()
    }
}
```

What we changed:

1. We added the `Engine` to the world which handles our entities. We could have used
   dependency inversion here, but this is not likely to change in the future so we're
   keeping it simple.
2. Also added the `Entities` in the starting blocks to our engine
3. Saved their position
4. Added a function for adding new entities
5. Added a function for adding an `Entity` at an empty position. This function needs
   a little explanation though. What happens here is that we try to find and empty
   position in our `World` within the given bounds (`offset` and `size`). Using this
   function we can limit the search for empty positions to a single level or multiple
   levels, and also within a given level. This will be very useful later.
6. If we didn't find an empty position, then we return with `false` indicating that
   we were not successful
7. Otherwise we add the `Entity` at the position which was found.   
8. This function performs a random serach for an empty position. To prevent seraching
   endlessly in a `World` which has none, we limit the maximum number of tries to 10.

This was our biggest change so far but now we are able to add entities to our
blocks and find empty positions within our `World`!

### Updating our Game

Our `Game` object was relatively simple so far. It only took a `World` as a parameter in its constructor
and we had a handy factory method for this which created a `World` for us.
Now it will also take our player `Entity` we just created:

```kotlin
package org.hexworks.cavesofzircon.world

import org.hexworks.cavesofzircon.attributes.types.Player
import org.hexworks.cavesofzircon.extensions.GameEntity

class Game(val world: World,
           val player: GameEntity<Player>) {

    companion object {

        fun create(player: GameEntity<Player>,
                   world: World) = Game(
                world = world,
                player = player)
    }
}
```

`World` will also come from the outside, we remove the usage of the `WorldBuilder` from here. What
we're using here is called the [Dependency Inversion Principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle)
which is very useful for creating abstraction boundaries in our program.

> The TL;DR for DIP is this: By stating **what** we need (the `World` here) but not
**how** we get it we let the outside world decide how to provide it for us. This is also called
"Wishful Thinking". This kind of *dependency inversion* lets the users of our program *inject* any kind of object
which corresponds to the `World` contract. For example we can create an in-memory world, one which is stored in
a database or one which is generated on the fly. `Game` won't care! This is in stark contrast to what we had before:
an explicit instantiation of `World` by using the `WorldBuilder`.

It seems that we are going to need a class which encapsulates creating a new `Game` from scratch,
and which can be used from all these views, the `GameBuilder`:

```kotlin
package org.hexworks.cavesofzircon.world

import org.hexworks.cavesofzircon.GameConfig
import org.hexworks.cavesofzircon.GameConfig.WORLD_SIZE
import org.hexworks.cavesofzircon.attributes.types.Player
import org.hexworks.cavesofzircon.builders.EntityFactory
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.zircon.api.Positions
import org.hexworks.zircon.api.Sizes
import org.hexworks.zircon.api.data.impl.Size3D

class GameBuilder(val worldSize: Size3D) { // 1

    private val visibleSize = Sizes.create3DSize( // 2
            xLength = GameConfig.WINDOW_WIDTH - GameConfig.SIDEBAR_WIDTH,
            yLength = GameConfig.WINDOW_HEIGHT - GameConfig.LOG_AREA_HEIGHT,
            zLength = 1)

    val world = WorldBuilder(worldSize) // 3
            .makeCaves()
            .build(visibleSize = visibleSize)

    fun buildGame(): Game {

        prepareWorld()

        val player = addPlayer()

        return Game.create(
                player = player,
                world = world)
    }

    private fun prepareWorld() = also { // 4
        world.scrollUpBy(world.actualSize().zLength)
    }

    private fun addPlayer(): GameEntity<Player> { 
        val player = EntityFactory.newPlayer() // 5
        world.addAtEmptyPosition(player, // 6
                offset = Positions.default3DPosition().withZ(GameConfig.DUNGEON_LEVELS - 1), // 7
                size = world.visibleSize().copy(zLength = 0)) // 8
        return player
    }

    companion object {

        fun defaultGame() = GameBuilder(
                worldSize = WORLD_SIZE).buildGame()
    }
}
```

> Hey, what's this `also` thingy? `also` here is a [scoping function](https://kotlinlang.org/docs/reference/scope-functions.html).
You can read more about them [here](https://kotlinlang.org/docs/reference/scope-functions.html). The TL;DR is that
`also` creates a scope where we can invoke functions on the scoped object (the new `GameBlock` here) and after that
the scoped object is returned. This is very useful for initializing objects after creation as you can see here.

Here we:

1. Take the size of the `World` as a parameter
2. We define the **visible** size which is our viewport of the world
3. We build our `World` here as part of the `Game`
4. `prepareWorld` can be called with method chaining here, since `also` will return the `GameBuilder` object
5. We create the *player* entity here since we're going to pass it as a parameter to other objects
6. We immediately add the *player* to the `World` which takes an `offset` and a `size` as a parameter
7. `offset` determines the position where the search for empty positions will start. Here we specify that
   the top level will be searched starting at (0, 0)
8. And we also determine that we should search only the throughout the viewport. This ensures that the *player*
   will be visible on the screen when we start the game

We also need to modify the constructor of our `PlayView` to not create a `Game`, but use our builder instead:

```kotlin
import org.hexworks.cavesofzircon.world.GameBuilder

// ...

class PlayView(private val game: Game = GameBuilder.defaultGame()) : BaseView()
```

Now if we run our program we'll something like this:

![Caves and Player](/assets/img/caves_with_player.png)

You might be thinking that we've written all this code for displaying a `@` on the screen, but what we really
did is the [plumbing for our porcelain](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain).

## Conclusion

**Well done!** Now we have a Player and we've also laid the groundwork to add even more things to
our game easily. In the follow up articles we'll add player movement and also some
player actions, like digging into our game!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `4_THE_PLAYER` tag.

