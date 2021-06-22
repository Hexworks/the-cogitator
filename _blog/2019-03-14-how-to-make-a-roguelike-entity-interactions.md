---
excerpt: "While walking through walls is fun, it is not a good game mechanism. Let's improve on that with Entity Interactions!"
title: "How To Make a Roguelike: #6 Entity Interactions"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #6 Entity Interactions"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/03/14/how-to-make-a-roguelike-entity-interactions.html
updated_at: 2019-03-14
---

> In the previous article we added cave exploration and scrolling, but there is not much going on
between the entities of our world. Let's improve on that by enabling the player to interact with
the world with its own Entity Actions!

## A World of Entities

In a traditional [Entity Component System](https://en.wikipedia.org/wiki/Entity_component_system) almost all
*things* can be entities. This is also true with [Amethyst](https://github.com/Hexworks/amethyst). In our case
there are two things in the world right now: **floors** and **walls**. Both of them are just simple `Block`s
in our game world. We don't want to interact with floors yet, but let's make *wall* an `Entity`!

Our `GameBlock` already supports adding entities to it, so let's upgrade it a bit so that we can create one
with an initial `Entity`:

```kotlin
// add this to the GameBlock class
companion object {

    fun createWith(entity: GameEntity<EntityType>) = GameBlock(
                    currentEntities = mutableListOf(entity))
}
```

> A quick refresher: [companion object](https://proandroiddev.com/a-true-companion-exploring-kotlins-companion-objects-dbd864c0f7f5)s
are singleton `object`s which are added to a regular class. They work in a similar way as the `static` keyword in Java, you can
call functions and access properties on them by using their parent class. Here we can invoke `create` in the following
way: `GameBlock.create()`. 

Now if we want a *wall* `Entity` we need to create a type for it first:

```kotlin
// put this in EntityTypes.kt
object Wall : BaseEntityType(
        name = "wall")
```

Apart from a type a *wall* also has a *position* and an attribute which tells us that it occupies a block (eg: you can't
just move into that block with the player). We already have `EntityPosition` so let's create our first **flag** attribute,
`BlockOccupier`:

```kotlin
package org.hexworks.cavesofzircon.attributes.flags

import org.hexworks.amethyst.api.Attribute

object BlockOccupier : Attribute
```

This is an `object` because we will only ever need a single instance of it. `BlockOccupier` works as a flag:
if an `Entity` has it the block is occupied, otherwise it is not. This will enable us to add walls, creatures
and other things to our dungeon which can't occupy the same space.

It is also useful to add an extension property to our `EntityExtensions` so that we can easily determine
for all of our entities whether they occupy a block or not:

```kotlin
import org.hexworks.cavesofzircon.attributes.flags.BlockOccupier

// put this in EntityExtensions.kt
val AnyGameEntity.occupiesBlock: Boolean
    get() = findAttribute(BlockOccupier::class).isPresent
```
Remember that `BlockOccupier` is just a flag so it is enough that we check for its presence.

We know that we'll check our `GameBlock`s for occupiers so let's start using the `occupiesBlock`
property right away in:

```kotlin
import org.hexworks.cavesofzircon.extensions.occupiesBlock

// add these extension properties to GameBlock
val occupier: Maybe<GameEntity<EntityType>>
    get() = Maybe.ofNullable(currentEntities.firstOrNull { it.occupiesBlock })  // 1
    
val isOccupied: Boolean
    get() = occupier.isPresent                                                  // 2
```

1. `occupier` will return the first `Entity` which has the `BlockOccupier` flag or an
   empty `Maybe` if there is none
2. Note how we tell whether a block is occupied by checking for the presence of an `occupier`

> You might ask why are we creating these extension properties instead of just checking for
the presence or absence of `Attribute`s. The answer is that by having these very expressive
properties we can write code which is really easy to read and interpret as you'll see later.

Now creating a *wall* `Entity` is pretty straightforward:

```kotlin
import org.hexworks.cavesofzircon.attributes.flags.BlockOccupier
import org.hexworks.cavesofzircon.attributes.types.Wall

// put this function to EntityFactory.kt
fun newWall() = newGameEntityOfType(Wall) {
    attributes(
            EntityPosition(),
            BlockOccupier,
            EntityTile(GameTileRepository.WALL))
}
```

Now to start using this we just make our `GameBlockFactory` create the *wall* blocks with the new factory method:

```kotlin
// put this in GameBlockFactory
fun wall() = GameBlock.createWith(EntityFactory.newWall())
```

There is one thing we need to fix. In the `WorldBuilder`'s smoothing algorithm we check whether a block is a floor.
This will no longer work because the `defaultTile` of a `GameBlock` is now always a floor tile so let's change it
to use `isEmptyFloor` instead:

```kotlin
// change this in WorldBuilder#smooth

if (block.isEmptyFloor) {
    floors++
} else rocks++
```

> Wait, what happened to our `defaultTile`? Well, the thing is that when we demolish a wall we want to see the default
tile in its place which is a floor in our case. Previously we did not have the ability to do this, but now we're
going to implement it.

If we run the program now there won't be any visible changes to gameplay. The reason is that although we added
the `BlockOccupier` flag to our game it is not used by anything...yet. Let's take a look at how can we go about
implementing the interaction between entities.

## Adding Entity Interactions

Let's stop for a second and think about how this works in most roguelike games. When the player is
idle nothing happens since updating the game world is bound to player movement. The usual solution
is that whenever we try to move into a new tile the game tries to figure out what to do.
If it is an enemy creature we *attack* it. If it is an empty tile we *move* into it. It is also possible
to move off of a ledge in which case the player usually suffers some *damage* or *dies*.
To sum it all up these are the steps we want to perform when the player presses a movement key:

- Check what's in the block *where* we want to move
- Take a look at what we *are able to do*
- Try each one on the target block and see what happens

Let's see how we can go about implementing this. We know that entities communicate with each other
by using `Command`s. So an `EntityAction` can be implemented as such:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.GameEntity

interface EntityAction<S : EntityType, T : EntityType> : GameCommand<S> { // 1

    val target: GameEntity<T>               // 2

    operator fun component1() = context     // 3
    operator fun component2() = source
    operator fun component3() = target
}
```

Our `EntityAction` is different from a regular `GameCommand` in a way that it also has a `target`.
So an `EntityAction` represents `source` trying to perform an action on `target`.

1. We have two generic type parameters, `S` and `T`. `S` is the `EntityType` of the `source`,
  `T` is the `EntityType` of the `target`. This will be useful later on as we'll see.
2. We save the reference to `target` in all `EntityAction`s
3. The `component1`, `component2` ... `componentN` methods implement [destructuring](https://kotlinlang.org/docs/reference/multi-declarations.html)
   in Kotlin. Since destructuring is positional as we've seen previously by implementing
   the `component*` functions we can control how an `EntityAction` can be destructured.
   In our case with these 3 [operator](https://kotlinlang.org/docs/reference/operator-overloading.html)
   functions we can destructure any `EntityAction`s like this:
   ```kotlin
   val (context, source, target) = entityAction 
   ```
   
> Kotlin supports *operator overloading*! This means that things like `+` and `-` can be overloaded
  just like in C++. You can read more about this [here](https://kotlinlang.org/docs/reference/operator-overloading.html).


Let's add an actual `EntityAction` to our project now! Our current problem is that we can walk through
walls, so let's solve that problem by adding a `Dig` action:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.world.GameContext

data class Dig(override val context: GameContext,
               override val source: GameEntity<EntityType>,
               override val target: GameEntity<EntityType>) : EntityAction<EntityType, EntityType>
```

This is rather simple. Here we supply `EntityType` as a type parameter for both `source` and
`target` because we don't want to limit who *can dig* and what *can be digged out*.

Now that we can create actions for our entities we need to enable them using it. In the *SEA* model
this means that we need a new `Attribute` which can be added to `Entity` objects which can perform
actions:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.amethyst.api.Attribute
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.commands.EntityAction
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.world.GameContext
import kotlin.reflect.KClass


class EntityActions(private vararg val actions: KClass<out EntityAction<out EntityType, out EntityType>>) // 1
    : Attribute {

    // 2
    fun createActionsFor(context: GameContext, source: GameEntity<EntityType>, target: GameEntity<EntityType>):
            Iterable<EntityAction<out EntityType, out EntityType>> {
        return actions.map {                                            
            try {
                it.constructors.first().call(context, source, target)   // 3
            } catch (e: Exception) {                                    // 4
                throw IllegalArgumentException("Can't create EntityAction. Does it have the proper constructor?")
            }
        }
    }
}
```

1. This `Attribute` is capable of holding classes of any kind of `EntityAction`. We use `vararg`
   here which is similar to how varargs work in Java: we can create the `EntityActions` object with
   any number of constructor parameters like this: `EntityActions(Dig::class, Look::class)`. We need
   to use the class objects (`KClass`) here instead of the actual `EntityAction` objects because each
   time we perform an action a new `EntityAction` has to be created. So you can think about `actions`
   here as templates.
2. This function can be used to create the actual `EntityAction` objects by using the given `context`,
   `source` and `target`
3. When we create the actions we just call the first constructor of the class and hope for the
   best. There is no built-in way in Kotlin (nor in Java) to make sure that a class has a specific
   constructor in compile time so that's why
4. We catch any exceptions and rethrow them here stating that the operation failed. We just have
   to remember that whenever we create an `EntityAction` it has a constructor for the 3 mandatory
   fields.

Now let's add a convenience function for actually trying the actions of an `Entity`:

```kotlin
import org.hexworks.amethyst.api.Consumed
import org.hexworks.cavesofzircon.attributes.EntityActions
import org.hexworks.cavesofzircon.world.GameContext

// put this in EntityExtensions.kt
fun AnyGameEntity.tryActionsOn(context: GameContext, target: AnyGameEntity): Response { // 1
    var result: Response = Pass                                         // 2
    findAttribute(EntityActions::class).map {                           // 3
        it.createActionsFor(context, this, target).forEach { action ->
            if (target.executeCommand(action) is Consumed) {            // 4
                result = Consumed
                return@forEach                                          // 5
            }
        }
    }
    return result
}
```

1. We define this function as an extension function on `AnyGameEntity`. This means that from
   now on we can call `tryActionsOn` on **any** of our entities!
2. We can only try the actions of an entity which has any, so we try to find the attribute.
3. if we find the attribute we just create the actions for our `context`/`source`/`target`
   combination
4. And we then execute the command on the `target` and if the command is `Consumed` it means that
5. We can break out of the `forEach` block.

> There are some interesting things going on here. First we call `createActionsFor` on `it`.
  What is this thing? When we create a lambda if it only has one parameter we can refer to it
  using the implicit `it` name. Read more about *it* (pun intended)
  [here](https://discuss.kotlinlang.org/t/it-keyword/6869). Also what's this funky `return@forEach`?
  This construct allows us to break out of the `forEach` loop. You can read more about this
  feature [here](https://kotlinlang.org/docs/reference/returns.html).
  
## Digging out Walls

Now that we have created a mechanism for our entities to perform actions on each other let's put it
to good use. We already have an action we can use (`Dig`), so we need to create the corresponding
`Facet`: `Diggable`. When we dig a block out we want to remove it from the `World` so let's add
this function to our `World` class:

```kotlin
    // add this to the World class
    fun removeEntity(entity: Entity<EntityType, GameContext>) {
        fetchBlockAt(entity.position).map {
            it.removeEntity(entity)
        }
        engine.removeEntity(entity)
        entity.position = Position3D.unknown()
    }
```

and then `Diggable` will look like this:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.commands.Dig
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.world.GameContext

object Diggable : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(Dig::class) { (context, _, target) ->
        context.world.removeEntity(target)  // 1
        Consumed                            // 2
    }
}
```

1. If the command we try to execute is `Dig` we remove the `target` `Entity` from our `World`
2. And then we return `Consumed` to state that we consumed the `Command` so it shouldn't be
   tried with other entities.
   
Now if we add the `Dig` action to our player `Entity`:

```kotlin
import org.hexworks.cavesofzircon.attributes.EntityActions
import org.hexworks.cavesofzircon.commands.Dig

// update this method in EntityFactory
fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
            EntityPosition(), 
            EntityTile(GameTileRepository.PLAYER),
            EntityActions(Dig::class))
    behaviors(InputReceiver)
    facets(Movable, CameraMover)
}
```

and make the walls `Diggable`:

```kotlin
import org.hexworks.cavesofzircon.systems.Diggable

// update newWall in EntityFactory with this
fun newWall() = newGameEntityOfType(Wall) {
    attributes(
            EntityPosition(),
            BlockOccupier,
            EntityTile(GameTileRepository.WALL))
    facets(Diggable)
}
```

and start the program we can still move through walls because we haven't added support for entity
actions in our `Movable` `Facet`. If you recall we've figured out before that entity actions are
bound to player movement in most roguelike games, so let's modify `Movable` now, to fix this:

```kotlin
import org.hexworks.cavesofzircon.extensions.tryActionsOn
import org.hexworks.cobalt.datatypes.extensions.map

world.fetchBlockAt(position).map { block -> // 1
    if (block.isOccupied) {
        result = entity.tryActionsOn(context, block.occupier.get()) // 2
    } else {
        if (world.moveEntity(entity, position)) { // 3
            result = Consumed
            if (entity.type == Player) {
                result = CommandResponse(MoveCamera(
                        context = context,
                        source = entity,
                        previousPosition = previousPosition))
            }
        }
    }
}
```

What changed is that

1. We will only do anything if there is a block at the given position. It is possible that there
   are no blocks at the edge of the map for example (if we want to move off the map)
2. If the block is occupied we try our actions on the block
3. Otherwise we do what we were doing before

What this will do is that if we bump into something which occupies the block the player will try to
dig it out, otherwise we'll just move to that block! Let's see it in action:

![Digging out walls](/assets/img/digging_out_walls.gif)

Wow, this is awesome!

## Conclusion

In this article we've learned how entities can interact with each other so now there is nothing stopping us
from adding new ways for changing our world which enriches the player experience. We've also learned how
everything can be an `Entity` and why this is very useful.

In the next article we'll add a new kind of `Entity`: **monsters**!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `6_ENTITY_INTERACTIONS` tag.




















