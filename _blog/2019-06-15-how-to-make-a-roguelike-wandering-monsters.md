---
excerpt: "Now that we have Fog of War, let's hide something beneath it: a wondering monster!"
title: "How To Make a Roguelike: #11 Wandering Monsters"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #11 Wandering Monsters"
series: coz
comments: true
canonical_url: https://hexworks.org/posts/tutorials/2019/05/07/how-to-make-a-roguelike-wandering-monsters.html 
---

> Now that we have Fog of War in place which we can reveal let's make it a bit more exciting by
adding some wandering monsters into the mix which can jump on us...or rather bump into us! This
will also be the first instance of *real* combat: it won't be just beating things to a pulp
like we did with fungi, because these creatures will hit back!

## A Wild Bat Appears

So let's think a bit about what we want to add. It should be something really simple which is easy to
code and will serve as a good first example. One of the stereotypical cave dwellers is
a *bat* so let's add one to our game!

A *bat* is a very simple creature, it has a few hit points, it doesn't bite hard and it just flies
around randomly. Let's see how we can implement it.

First of all, let's add a new color for our bat to `GameColors`:

```kotlin
val BAT_COLOR = TileColors.fromString("#2348b2")
```

and this to `GameTileRepository`:

```kotlin
val BAT = Tiles.newBuilder()
        .withCharacter('b')
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .withForegroundColor(GameColors.BAT_COLOR)
        .buildCharacterTile()
```

We also need a new `EntityType` so let's add this to `EntityTypes.kt`:

```kotlin
object Bat : BaseEntityType(
        name = "bat"), Combatant
```

We're all set!

## The Wanderer Behavior

With the plumbing in place now we can define a new behavior which we can call `Wanderer`. What this `Behavior` will
do is it moves the entity around picking a random neighboring position. Simple enough, right? Let's see:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.base.BaseBehavior
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.commands.MoveTo
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.extensions.position
import org.hexworks.cavesofzircon.extensions.sameLevelNeighborsShuffled
import org.hexworks.cavesofzircon.world.GameContext

object Wanderer : BaseBehavior<GameContext>() {

    override fun update(entity: GameEntity<EntityType>, context: GameContext): Boolean {
        val pos = entity.position
        if (pos.isUnknown().not()) {        // 1
            entity.executeCommand(MoveTo(   // 2
                    context = context,
                    source = entity,
                    position = pos.sameLevelNeighborsShuffled().first()))   // 3
            return true
        }
        return false
    }
}
```

Here we:

1. Check whether the `Entity` has a valid `Position`
2. If yes, we send it the `MoveTo` command
3. Using a random neighboring position

Simple enough!

> You might be wondering why don't we just move the bat by hand. Why do we send a command to itself?
The answer is that it might be possible that a bat has some state which prevents it from moving like
being frozen or asleep! In this case the `Movable` facet might not even be present on the bat! By
using commands we delegate all these problems to whatever system handles it thus keeping the separation
of concerns and our code clean.

Now with our new `Wanderer` *system* in place we can finally create a *bat* `Entity`:

```kotlin
// new imports
import org.hexworks.cavesofzircon.attributes.types.Bat
import org.hexworks.cavesofzircon.systems.Wanderer

fun newBat() = newGameEntityOfType(Bat) {
    attributes(BlockOccupier,                   // 1
            EntityPosition(),
            EntityTile(GameTileRepository.BAT),
            CombatStats.create(                 // 2
                    maxHp = 5,
                    attackValue = 2,
                    defenseValue = 1),
            EntityActions(Attack::class))       // 3
    facets(Movable, Attackable, Destructible)   // 4
    behaviors(Wanderer)                         // 5
}
```

So the definition of a *bat* is:

1. It occupies a block so we can't move to the same tile as a *bat*
2. It has the usual attributes, but not too much hp/attack/defense
3. It can only *attack* (no digging sorry)
4. It can be moved, attacked and destroyed
5. And it will *wander* on its own

Now to make this work we just need to add *bats* to our world. This goes to `GameConfig`

```kotlin
const val BATS_PER_LEVEL = 10
```

and this one goes to `GameBuilder`:

```kotlin
// new imports
import org.hexworks.cavesofzircon.GameConfig.BATS_PER_LEVEL

// add this to buildGame

fun buildGame(): Game {

    // ...
    addFungi()
    addBats()

    // ...
}

// and this to GameBuilder
private fun addBats() = also {
    repeat(world.actualSize().zLength) { level ->
        repeat(BATS_PER_LEVEL) {
            EntityFactory.newBat().addToWorld(level)
        }
    }
}
```

Now if we start the game we'll notice that the bat can move to the same tile as the *player* and it can't hit us either!
Why is that? Well, we didn't make the *player* `Attackable`, `Destroyable` and it is not a `BlockOccupier`! Let's fix this:

```kotlin
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
            EntityActions(Dig::class, Attack::class))
    behaviors(InputReceiver)
    facets(Movable, CameraMover, StairClimber, StairDescender, Attackable, Destructible)
}
```

Now if we start the game everything fits together nicely:

![Killing Bats](/assets/img/killing_bats.gif)

Now this is much more interactive than stomping fungi!

## Conclusion

As you can see with all the `System`s and `Attribute`s we have we can really start to be productive because the
code we've written in the previous articles are reusable and cohesive. Here we reused `Movable`, `Attackable`,
`Destructible`, and also the `EntityActions` to produce a completely new entity with only a little coding on
our part.

Next we're going to take this to the next level by adding *items* and *inventory* to our game!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `11_WANDERING_MONSTERS` tag.
