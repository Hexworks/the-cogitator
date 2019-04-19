---
excerpt: Now, we generate an actual dungeon, or rather a cave we can explore in our game.
title: "How To Make a Roguelike: #3 Generating Random Caves"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #3 Generating Random Caves"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/01/05/how-to-make-a-roguelike-generating-random-caves.html
---

> Now, that we have set up our Views it is time to start using the `GameArea` and generate caves
which the player can explore! I'd also like to mention that it is useful most of the time if we
check whether there are never versions of the libraries we're using. In our case Zircon was
updated since the last article, so let's open `gradle.properties` and update `zircon_version`
to `2019.0.9-PREVIEW`!
  
## Cleaning up Our PlayView

We have already used `Component`s in the previous article but now we're gonna design an UI for our
game before we start generating our game world, so let's get started!

First of all let's remove the `winButton` and the `loseButton` from our `PlayView`. It was fun while it
lasted but now we want to get some serious work done:

```kotlin
import org.hexworks.zircon.api.ColorThemes
import org.hexworks.zircon.api.mvc.base.BaseView

class PlayView : BaseView() {

    override val theme = ColorThemes.arc()

    override fun onDock() {

        
    }
}
```

If you take a look at our `View`s you'll see that we set a `theme` in all of them by hand so it is time to move
this into our own config file. We'll also configure some other things while we're at it:

```kotlin
object GameConfig {

    // game
    const val DUNGEON_LEVELS = 2 // 1

    // look & feel
    val TILESET = CP437TilesetResources.rogueYun16x16() // 2
    val THEME = ColorThemes.zenburnVanilla() // 3
    const val SIDEBAR_WIDTH = 18
    const val LOG_AREA_HEIGHT = 8 // 4

    // sizing
    const val WINDOW_WIDTH = 80
    const val WINDOW_HEIGHT = 50

    val WORLD_SIZE = Sizes.create3DSize(WINDOW_WIDTH, WINDOW_HEIGHT, DUNGEON_LEVELS)

    fun buildAppConfig() = AppConfigs.newConfig() // 5
            .enableBetaFeatures()
            .withDefaultTileset(TILESET)
            .withSize(Sizes.create(WINDOW_WIDTH, WINDOW_HEIGHT))
            .build()

}
```

> Hey, isn't **globals bad**? Well, shared **mutable** state is bad, but in this config file everything is
immutable, so it can't cause problems for us. We could have used a `json` file or something similar instead,
but we'll keep this for simplicity's sake.

So what happens here is that we:

1. create a constant for the number of levels we're gonna have in our dungeon
2. choose a tileset which is better for the dungeon vibe we are trying to achieve
3. pick a theme which has more eerie colors than the previous one
4. configure the width of our sidebar and the height of our log area which we'll add momentarily
5. add a function which builds an `AppConfig` for us based on these settings. This can be used
   for building our `Application`
   
Now we just modify our `main.kt` a bit to use this new config:

```kotlin
import org.hexworks.cavesofzircon.view.StartView
import org.hexworks.zircon.api.SwingApplications

fun main(args: Array<String>) {

    val application = SwingApplications.startApplication(GameConfig.buildAppConfig())

    application.dock(StartView())

}
```

and now all is left to do is to add the main UI components to our `PlayView`.

## Using Panels

`Panel` is a `Component` which we can add to our `View`s and they can hold other `Components`. This means that
if we add a bunch of `Button`s and other controls to a `Panel` and move the `Panel` around, its child controls
will move with it!

Let's add a sidebar and our new theme to the `PlayView` now:

```kotlin
import org.hexworks.cavesofzircon.GameConfig
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.mvc.base.BaseView

class PlayView : BaseView() {

    override val theme = GameConfig.THEME

    override fun onDock() {

        val sidebar = Components.panel()
                .withSize(GameConfig.SIDEBAR_WIDTH, GameConfig.WINDOW_HEIGHT)
                .wrapWithBox()
                .build()

        screen.addComponent(sidebar)
    }
}
```

If you run it you'll see something like this:

![Sidebar](/assets/img/sidebar.png)

Nice, isn't it? Now let's add a `LogArea` at the bottom as well:

```kotlin
val logArea = Components.logArea()
        .withTitle("Log") // 1
        .wrapWithBox()  // 2
        .withSize(GameConfig.WINDOW_WIDTH - GameConfig.SIDEBAR_WIDTH, GameConfig.LOG_AREA_HEIGHT)
        .withAlignmentWithin(screen, ComponentAlignment.BOTTOM_RIGHT)  // 3
        .build()

screen.addComponent(logArea)
```

Let's take a look at what happens here:

1. We set the title of our `LogArea` to "Log"
2. We wrap it with a box. This is an important step because `title` is only visible if our component is wrapped
3. We align the log within our `Screen` to the `BOTTOM_RIGHT`. We can either align *within* a component or
   *around* a component. In this case we chose the former.

Now, let's start up and bask in the "warm" colors of our UI:

![warm colors](/assets/img/warm_colors.png)

So far so good, right? The problem is that we still don't see a **game** so let's explore how to create one.

## The Game Component

Zircon comes with a `GameComponent` and the corresponding `GameArea`. We can use these two to display our game
world in an easy to use manner: the `GameComponent` is just a component, like the `Panel` and the `LogArea` we
used above. It is special because we can add a `GameArea` to it which holds the in-memory representation of our
world.

A `GameArea` consists of `Block`s which are very similar to blocks in Minecraft. They have 6 sides (top, bottom, left, right, front, back)
and they can also hold internal content (called layers). A `Block` contains `Tile`s so we can re-use everything
we had so far.

This setup clearly separates the **storing** of game data from **displaying** it so we won't end up with
convoluted render logic!

The `GameArea` supports **top down oblique** projection, like in this example:

![top down oblique](/assets/img/top_down_oblique.gif)

and simple **top down** projection:

![top down](/assets/img/top_down.gif)

We'll use the latter here.

## Setting Up the Game

In order to have a `GameArea` with generated caves we're gonna need some plumbing. What I'm talking about is colors
tiles and blocks. It is a good idea to have a separate class for all of these so let's create a `GameColors` class
where we'll put all the colors we use in our game:

### Using Colors

```kotlin
import org.hexworks.zircon.api.TileColors

object GameColors {
    val WALL_FOREGROUND = TileColors.fromString("#75715E")
    val WALL_BACKGROUND = TileColors.fromString("#3E3D32")

    val FLOOR_FOREGROUND = TileColors.fromString("#75715E")
    val FLOOR_BACKGROUND = TileColors.fromString("#1e2320")
}
```

> `object` here means that there is a single instance of `GameColors`, so it is effectively a [Singleton](https://en.wikipedia.org/wiki/Singleton_pattern).
Make sure not to hold shared mutable state in these objects since it will lead to errors. Here we just put configuration
in this `object`.

`TileColors` is a utility class which you can use to create `TileColor` objects which can be used to set foreground and
background colors for `Tile`s. `fromString` uses the semantics which you might know from CSS: A `#` followed by 2 digit
hexadecimal numbers corresponding to red, green, and blue. You can also use `TileColors.create` if you don't like this.

### A Factory for Tiles

Another thing we need is an `object` to hold our `Tile`s:

```kotlin
import org.hexworks.zircon.api.Tiles
import org.hexworks.zircon.api.data.CharacterTile
import org.hexworks.zircon.api.graphics.Symbols

object GameTileRepository {

    val EMPTY: CharacterTile = Tiles.empty()

    val FLOOR: CharacterTile = Tiles.newBuilder()
            .withCharacter(Symbols.INTERPUNCT) // 1
            .withForegroundColor(GameColors.FLOOR_FOREGROUND) // 2
            .withBackgroundColor(GameColors.FLOOR_BACKGROUND) // 3
            .buildCharacterTile() // 4

    val WALL: CharacterTile = Tiles.newBuilder()
            .withCharacter('#')
            .withForegroundColor(GameColors.WALL_FOREGROUND)
            .withBackgroundColor(GameColors.WALL_BACKGROUND)
            .buildCharacterTile()

}
```

This `object` serves as a [Factory](https://en.wikipedia.org/wiki/Factory_method_pattern) which we can use to create
`Tile` objects. Right now we only have `floor` and `wall`, but this is fine. `EMPTY` is an empty and transparent `Tile`.

What we do here is:

1. We set a character for our `Tile`. You can either type the character 
(make sure that it is in [Code Page 437](https://en.wikipedia.org/wiki/Code_page_437)) or use the `Symbols` object
which has shorthands for the supported characters.
2. We set the foreground color
3. And the background color
4. And finally build a `CharacterTile`. Zircon supports multiple kinds of `Tile`s (like a `GraphicTile` which holds
textures), but we use `CharacterTile` in this game.

I call this object `GameTileRepository` because it holds immutable `Tile` objects, so it is not a factory. A `Tile` in
Zircon is immutable, so it can be re-used any number of times throughout the project. This is useful because we can
spare a lot of memory by not creating a lot of new objects.

### Creating a Block

If we want to use the `GameArea` we can use the default `Block` implementation like this:

```kotlin
Blocks.newBuilder<Tile>().build()
```

but in this game we'll need a custom `Block` implementation because we'll extend this later to hold our game entities
as well, so let's create a `GameBlock`:

```kotlin
import org.hexworks.cavesofzircon.builders.GameTileRepository
import org.hexworks.zircon.api.data.BlockSide
import org.hexworks.zircon.api.data.Tile
import org.hexworks.zircon.api.data.base.BlockBase

class GameBlock(private var defaultTile: Tile = GameTileRepository.FLOOR) // 1
    : BlockBase<Tile>() { // 2

    val isFloor: Boolean
        get() = defaultTile == GameTileRepository.FLOOR // 3


    val isWall: Boolean
        get() = defaultTile == GameTileRepository.WALL


    override val layers
        get() = mutableListOf(defaultTile) // 4


    override fun fetchSide(side: BlockSide): Tile {
        return GameTileRepository.EMPTY // 5
    }
}
```

Here we
1. Set the `defaultTile` to be a `floor` by default. What you see in this line is a **constructor**:
   `GameBlock(private var defaultTile: Tile = GameTileRepository.floor())`.
   In Kotlin [constructors](https://kotlinlang.org/docs/reference/classes.html#constructors) are defined this way.
   We just supply a list of parameters and that's all. What `= GameTileRepository.floor()` does is that if you
   don't pass a `defaultTile` as an argument to this constructor, it will use this default value. You can read
   more about default parameter values [here](https://kotlinlang.org/docs/reference/functions.html#default-arguments).
2. We extend `BlockBase`. It takes a generic type parameter (you can read more about the topic [here](https://kotlinlang.org/docs/reference/generics.html))
   and we set it to `Tile` indicating that we can use any implementation of `Tile` here. Zircon has these base classes
   for a lot of things including `Tile`s to make it easier for you to create custom versions.
3. We add two propertyes (`isFloor` and `isWall`) which we can use to check whether a given `Block` is a wall or a floor.
   The syntax you see here: `get() = defaultTile == GameTileRepository.FLOOR` defines a getter for `isFloor` We can access
   this value by typing `block.isFloor` and whenever it is called it will do the comparison above. If you have programmed
   in C# this way of defining getters / setters might be familiar, but for those who haven't seen this before I'd suggest
   taking a look at the [related docs]().
4. We return a list containing our `defaultTile` when Zircon asks for `layers`. What this means that the content of
   our `Block` will be this single `Tile`. Think about blocks as if they were a box which you can put things (layers)
   into.
5. In this game we don't care about the sides of our blocks since we'll be looking at them from a top down view so
   we just return our `EMPTY` tile which is transparent. Whenever you use `Tiles.empty()` Zircon will ignore rendering
   it so you don't have to worry about it.
   
Since we already have a repository for `Tile`s it is straightforward to create a factory for our `Block`s as well:

```kotlin
import org.hexworks.cavesofzircon.blocks.GameBlock

object GameBlockFactory {

    fun floor() = GameBlock(GameTileRepository.FLOOR)

    fun wall() = GameBlock(GameTileRepository.WALL)

}
```

## Generating a dungeon

Now we have all the plumbing in place to start actually **generating** something. For this we're gonna need a *World*!
A world object is responsible for holding all our game data like levels, walls, floors, and things like that. For this
we are going to create a `GameArea` implementation. Luckily `GameArea` comes with a builder we can use to implement
our `World`:

```kotlin
import org.hexworks.cavesofzircon.blocks.GameBlock
import org.hexworks.cavesofzircon.builders.GameBlockFactory
import org.hexworks.zircon.api.builder.game.GameAreaBuilder
import org.hexworks.zircon.api.data.Tile
import org.hexworks.zircon.api.data.impl.Position3D
import org.hexworks.zircon.api.data.impl.Size3D
import org.hexworks.zircon.api.game.GameArea

class World(startingBlocks: Map<Position3D, GameBlock>, // 1
            visibleSize: Size3D,
            actualSize: Size3D) : GameArea<Tile, GameBlock> by GameAreaBuilder.newBuilder<Tile, GameBlock>() // 2
        .withVisibleSize(visibleSize) // 3
        .withActualSize(actualSize) // 4
        .withDefaultBlock(DEFAULT_BLOCK) // 5
        .withLayersPerBlock(1) // 6
        .build() {

    init {
        startingBlocks.forEach { pos, block ->
            setBlockAt(pos, block) // 7
        }
    }

    companion object {
        private val DEFAULT_BLOCK = GameBlockFactory.floor() // 8
    }
}
```

> That funky looking piece of code: `GameArea<Tile, GameBlock> by GameAreaBuilder.newBuilder<Tile, GameBlock>() ...`
may look confusing to you if you have used Java or Python before. The `by` keyword is used to do **delegation**.
I've written a deep-dive on this topic [here](http://the-cogitator.com/posts/blog/2018/09/29/by-the-way-exploring-delegation-in-kotlin.html).
The gist of it is that when you implement an interface (`GameArea<Tile, GameBlock>`) you can choose to use a
concrete implementation of that interface instead of implementing all its methods. Here we choose to implement `GameArea`
by delegating to another instance which we create with the `GameAreaBuilder`.

This looks a bit complex at first glance but we'll clear it up. This piece of code

1. A `World` object is about holding the world data in memory, but it is not about generating it, so we take the
   initial state of the world as a parameter.
2. We implement the `GameArea` which we'll use with the `GameComponent`
3. We set its `visibleSize`. This is the size of the area which will be visible on our screen
4. We set the `actualSize`. This is the size of the *whole* world which can be multiple times bigger than the visible part.
   `GameArea` supports scrolling so we'll be able to scroll through our caves soon
5. We set a default block for our world. The default block is used when there is no block present at a given position.
   This also means that a `GameArea` is sparse. It doesn't use arrays internally so you can have a world with an arbitrary
   size (even `Integer.MAX_VALUE`!)
6. We set the number of layers per block to 1. This means that within a `Block` we'll only use a single layer for displaying
   our tiles.
7. a `World` takes a `Map` of `GameBlock`s, so we need to add them to the `GameArea`. Where these blocks come from?
   We'll see soon enough wen we implement the `WorldBuilder`!
8. We set up a `DEFAULT_BLOCK` which is used whenever the renderer asks for a block at a position where there is no block
   currently. A `companion object` is similar to using `static` in java. Read more about it [here](https://kotlinlang.org/docs/reference/object-declarations.html#companion-objects).

Now that we have a `World` which we can use with our `GameComponent` we are going to need a builder for it.

Our builder will use a [cellular automata](http://roguebasin.roguelikedevelopment.org/index.php?title=Cellular_Automata_Method_for_Generating_Random_Cave-Like_Levels)
to generate organic-looking caves. The process is to fill the area with cave floors and walls at random then to smooth 
everything out by turning areas with mostly neighboring walls into walls and areas with mostly neighboring floors into floors.
Repeat the smoothing process a couple times and you have an interesting mix of cave walls and floors. Before we take a
look at the implementation let's add an **extension function** to `Position3D`.

> The `GameArea` uses the `Position3D` class to position `Block`s. a `Position3D` is similar to a simple `Position`
but it not only has `x` and `y` coordinates, but also a `z` level. When we generate our world it won't be a flat 2D
one, but one in 3 dimensions!

### Exploring extension functions

There are instances when you want to add something to an existing class, but you can't modify it for some reason.
When you are using a library written by someone else this is the case most of the time. For example we can't touch
`Position3D` because it comes with Zircon. In Java this is where you can use the [Decorator pattern](https://en.wikipedia.org/wiki/Decorator_pattern).
Luckily Kotlin makes this a lot easier by allowing us to define **extensions** instead. For our `WorldBuilder` we
want an *extension function* on `Position3D` which returns the positions around it on the same level. Let's create
a new file called `PositionExtensions.kt` and add this to it:

```kotlin
import org.hexworks.zircon.api.data.impl.Position3D

fun Position3D.sameLevelNeighborsShuffled(): List<Position3D> { // 1
    return (-1..1).flatMap { x ->
        (-1..1).map { y -> // 2
            this.withRelativeX(x).withRelativeY(y) // 3
        }
    }.minus(this).shuffled() // 4
}
```

> Putting extensions for a class `X` to a file called `XExtensions.kt` is the idiomatic way of defining extensions.

This is only a few lines of code but it introduces some concepts which we don't know yet:

1. We add the extension function to `Position3D`. We do it by defining a function not with a simple name, but
   by the format: `fun <target class>.<function name>: return type { // ...`.  You can read more about extensions
   [here](https://kotlinlang.org/docs/reference/extensions.html#extension-functions).
2. We use functional programming here. `flatMap` and `map` works in a similar way as you might got used to it in
   Java 8's Stream API. Take a look at the difference [here](http://the-cogitator.com/posts/blog/2017/05/19/kotlin-is-the-new-java.html#functional-programming).
3. When you write extension functions `this` will be bound to the class being extended. So `this` here
   will point to the `Position3D` instance on which `sameLevelNeighborsShuffled` is called.
4. `minus` here will remove `this` position from the `List` and return a new `List`. `shuffled` will also return
   a new list which contains the same elements but shuffled.
   
Armed with this extension function we can now implement the builder itself:

```kotlin
import org.hexworks.cavesofzircon.blocks.GameBlock
import org.hexworks.cavesofzircon.builders.GameBlockFactory
import org.hexworks.cavesofzircon.extensions.sameLevelNeighborsShuffled
import org.hexworks.zircon.api.Positions
import org.hexworks.zircon.api.data.impl.Position3D
import org.hexworks.zircon.api.data.impl.Size3D

class WorldBuilder(private val worldSize: Size3D) { // 1

    private val width = worldSize.xLength
    private val height = worldSize.zLength
    private var blocks: MutableMap<Position3D, GameBlock> = mutableMapOf() // 2

    fun makeCaves(): WorldBuilder { // 3
        return randomizeTiles()
                .smooth(8)
    }

    fun build(visibleSize: Size3D): World = World(blocks, visibleSize, worldSize) // 4

    private fun randomizeTiles(): WorldBuilder {
        forAllPositions { pos ->
            blocks[pos] = if (Math.random() < 0.5) { // 5
                GameBlockFactory.floor()
            } else GameBlockFactory.wall()
        }
        return this
    }

    private fun smooth(iterations: Int): WorldBuilder {
        val newBlocks = mutableMapOf<Position3D, GameBlock>() // 6
        repeat(iterations) {
            forAllPositions { pos ->
                val (x, y, z) = pos // 7
                var floors = 0
                var rocks = 0
                pos.sameLevelNeighborsShuffled().plus(pos).forEach { neighbor -> // 8
                    blocks.whenPresent(neighbor) { block -> // 9
                        if (block.isFloor) {
                            floors++
                        } else rocks++
                    }
                }
                newBlocks[Positions.create3DPosition(x, y, z)] = if (floors >= rocks) GameBlockFactory.floor() else GameBlockFactory.wall()
            }
            blocks = newBlocks // 10
        }
        return this
    }

    private fun forAllPositions(fn: (Position3D) -> Unit) { // 11
        worldSize.fetchPositions().forEach(fn)
    }

    private fun MutableMap<Position3D, GameBlock>.whenPresent(pos: Position3D, fn: (GameBlock) -> Unit) { // 12
        this[pos]?.let(fn)
    }
}
```

This might be the most complex piece of code we've written so far but the whole class still fits on the screen so it
might be easier to understand than it looks. Let's see:

1. We take the `worldSize` from the outside world. This is useful because later it can be parameterized.
2. We maintain a `Map` of `Block`s which we will use when we `build` the `World`
3. With `makeCaves` we create a [fluent interface](https://en.wikipedia.org/wiki/Fluent_interface) so that the users
   of `WorldBuilder` can use it in a similar manners as we build `Tile`s and `Component`s in Zircon.
4. When we `build` the `World` we take a visible size which will be used by the `GameArea`.
5. In Kotlin `if` is not a statement but an expression. This means that it returns a value so we can assign it
   to our `Map`.
6. We are going to need a new `Map` of blocks for our smoothing because we can't do it in place. Modifying the
   original `Map` would render our cellular automata algorithm useless because it needs to calculate the new state
   from the old state.
7. We create a 3D world, so we need not only `x` and `y`, but also `z`. What you see here is called **destructuring**.
   If you are not familiar with the concept, take a look at the docs [here](https://kotlinlang.org/docs/reference/multi-declarations.html).
8.  Here we iterate over a list of the current position and all its neighbors
9. And we only care about the positions which have a corresponding block (whey are not outside of the game world)
10. When we're done with smoothing we replace the old `Map` with the new one.
11. This is just a convenience function for iterating over all of the world's positions which I added as a demonstration
    of how functions with lambdas work. Here you can pass any
    function which takes a `Position3D` and returns `Unit` (`Unit` is the equivalent of Java's `Void`).
12. This function is an example of defining an extension function which takes a function as a parameter.
    What the header of the function means here is: 
    > Augment all `MutableMap`s which are holding `Position3D` to `GameBlock` mappings to have a function named
      "whenPresent" which takes a position and a function.
      
    That's right. This function will only be available on **any** `MutableMap` which has these generic type parameters.
    So this is in fact a form of [polymorphism](https://en.wikipedia.org/wiki/Polymorphism_(computer_science)).
    We'll use this feature of Kotlin in the coming articles so I wanted to put an useful example here so we can get
    familiar with the concept.

## Putting it together

Now all is left to do is to add a `Game` class which will hold all our game-specific data and can be passed around
in our application as a form of [Context Object](https://stackoverflow.com/questions/986865/can-you-explain-the-context-design-pattern/986947#986947):

```kotlin
import org.hexworks.cavesofzircon.GameConfig
import org.hexworks.zircon.api.data.impl.Size3D

class Game(val world: World) {

    companion object {

        fun create(worldSize: Size3D = GameConfig.WORLD_SIZE,
                   visibleSize: Size3D = GameConfig.WORLD_SIZE) = Game(WorldBuilder(worldSize)
                .makeCaves()
                .build(visibleSize))
    }
}
```

Wo with this code we can call `Game.create()` and we'll get a `Game` object which contains a fully generated
`World`! Let's wire this together with our `View`s. First we need to Modify `PlayView` to take a `Game` object
as a constructor parameter:

```kotlin
class PlayView(private val game: Game = Game.create()) : BaseView() {
 // ...
}
```

Note that constructors can also have default values for their parameters. With this change we say that "okay, take
a `Game` object as a constructor parameter, but if none is supplied, then just call `Game.create()` instead". With
this change we didn't break our existing code which is very useful in the long run!

Now to conclude coding for today we add the `GameComponent` to the `PlayView` to display our generated caves:

```kotlin
val gameComponent = GameComponents.newGameComponentBuilder<Tile, GameBlock>()
        .withGameArea(game.world)
        .withVisibleSize(game.world.visibleSize())
        .withProjectionMode(ProjectionMode.TOP_DOWN)
        .withAlignmentWithin(screen, ComponentAlignment.TOP_RIGHT)
        .build()

screen.addComponent(gameComponent)
```

This is the result:

![random caves](/assets/img/generated_caves.png)

## Conclusion

**Congratulations!** We've worked hard and managed to create the `World` where all things will happen in our game!
This is the first step to have something which is not only looking nice but also fun to play with. So in the next
article we'll add a *player* to our dungeon with which we can explore it.

Until then go forth and *kode on*!
 
> The code of this article can be found under the `3_GENERATING_RANDOM_CAVES` tag.

