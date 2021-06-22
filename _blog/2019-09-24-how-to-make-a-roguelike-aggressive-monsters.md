---
excerpt: "We have loot lying around in the dungeon, but it is kinda lame. Let's create a new type of monster which will carry these!"
title: "How To Make a Roguelike: #16 Aggressive Monsters"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #16 Aggressive Monsters"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/06/26/how-to-make-a-roguelike-aggressive-monsters.html
updated_at: 2019-09-24
---

> Strolling around a dungeon and looting items randomly lying on the ground is only so much fun.
It would be much better if we could at least fight to get them, right? Let's add *aggressive monsters*
to our game which carry valuable items and attack the *player* on sight!

## Designing an Aggressive Enemy

What we're going to implement in this session is a monster which attacks the player on sight. The logic
will be this: If the monster sees the *player* starts moving towards him but as soon as it loses sight
it goes back to just wandering around randomly. This is kinda dumb, but easy to implement.

Now let's think about which kind of monster might have this behavior. It is a *Zombie*!

Let's add the necessary *color*:

```kotlin
// add this to GameColors

val ZOMBIE_COLOR = TileColors.fromString("#618358")
```

the *tile* for it:
 
 ```kotlin
// Add this to GameTileRepository

val ZOMBIE = Tiles.newBuilder()
        .withCharacter('z')
        .withForegroundColor(GameColors.ZOMBIE_COLOR)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()
```

and the new `EntityType`:

```kotlin
// add this to EntityTypes.kt

object Zombie : BaseEntityType(
        name = "zombie"), Combatant, ItemHolder
```

Now let's see what will we need for this creature. We have `Inventory`, `Wanderer`, and all the *system*s and
*attribute*s in place so the only thing we need is the `if I see it I attack it otherwise I just wanter` behavior.
For this we're going to need a new *system* and a way to create a path from the *zombie* to the *player*.

## A Hunter-Seeker Monster

First of all we need to add a function to `World` which can determine whether two *entities* can see each other:

```kotlin
import kotlin.math.abs

fun whenCanSee(looker: GameEntity<EntityType>, target: GameEntity<EntityType>, fn: (path: List<Position>) -> Unit) { // 1
    looker.findAttribute(Vision::class).map { (radius) ->       // 2
        val level = looker.position.z
        if (looker.position.isWithinRangeOf(target.position, radius)) {  // 3
            val path = LineFactory.buildLine(looker.position.to2DPosition(), target.position.to2DPosition())  // 4
            if (path.none { isVisionBlockedAt(Positions.from2DTo3D(it, level)) }) { // 5
                fn(path.positions().toList().drop(1))
            }
        }
    }
}

private fun Position3D.isWithinRangeOf(other: Position3D, radius: Int): Boolean {
    return this.isUnknown().not()
            && other.isUnknown().not()
            && this.z == other.z
            && abs(x - other.x) + abs(y - other.y) <= radius
}
```

In this function we:

1. Supply the two entities and a callback which will only be called with the *path* if there is line of sight
   between them
2. We try to find a `Vision` attribute
3. Determine whether the *position* of the *target* `Entity` is within range
4. Then we build a path if it is
5. And if the vision is not blocked along that path we call our callback

This might not be the most efficient way to do this, but it is very easy to implement.

With this in place the `HunterSeeker` *behavior* is rather straightforward to implement:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.base.BaseBehavior
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.commands.MoveTo
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.extensions.position
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.zircon.api.Positions

object HunterSeeker : BaseBehavior<GameContext>() {

    override fun update(entity: GameEntity<out EntityType>, context: GameContext): Boolean {
        val (world, _, _, player) = context
        var hunted = false
        world.whenCanSee(entity, player) { path ->  // 1
            entity.executeCommand(MoveTo(
                    context = context,
                    source = entity,
                    position = Positions.from2DTo3D(path.iterator().next(), player.position.z)))    // 2
            hunted = true
        }
        return hunted   // 3
    }
}
```

Here we:

1. Determine whether the `entity` can see the `player` or not
2. If yes it moves in the direction of the *player*
3. Then we return whether we `hunted` in this turn or not.

"Why do we need this `hunted` variable?" you might ask. The reason is that now we're going to combine
two behaviors to form the `if I see it I attack it otherwise I just wanter` behavior!

Let's see how to make it happen by adding our new Zombie `Entity` to the `EntityFactory`:

```kotlin
import org.hexworks.cavesofzircon.attributes.types.Zombie
import org.hexworks.cavesofzircon.systems.HunterSeeker

fun newZombie() = newGameEntityOfType(Zombie) {
    attributes(BlockOccupier,
            EntityPosition(),
            EntityTile(GameTileRepository.ZOMBIE),
            Vision(10),
            CombatStats.create(
                    maxHp = 25,
                    attackValue = 8,
                    defenseValue = 4),
            Inventory(2).apply {
                addItem(newRandomWeapon())
                addItem(newRandomArmor())
            },
            EntityActions(Attack::class))
    facets(Movable, Attackable, ItemDropper, LootDropper, Destructible)
    behaviors(HunterSeeker or Wanderer)
}
```

> We could argue that a *zombie* should have *Equipment* which they use against us, but we know that
Zombies are mindless and lost all their ability to use items.

So as you can see we re-used a lot of prior functionality including `Inventory` and made our *zombie* have
a random *weapon* and *armor*. What's interesting here is `behaviors(HunterSeeker or Wanderer)`. What happens
here is that *Amethyst* will try to do the `HunterSeeker` *behavior* and if it returns `false` (the *zombie* can't
see the player) it will do `Wanderer` instead!

Now we just need to remove the random items lying around in the world by deleting `addWeapons` and `addArmor`
from `GameBuilder` and add `addZombies` instead. For this we just remove `WEAPONS_PER_LEVEL` and `ARMOR_PER_LEVEL`
from `GameConfig` and add `ZOMBIES_PER_LEVEL` instead:

```kotlin
const val ZOMBIES_PER_LEVEL = 3
```

Then the code in `GameBuilder` will be this:

```kotlin
import org.hexworks.cavesofzircon.GameConfig.ZOMBIES_PER_LEVEL

// ...

fun buildGame(): Game {

    // ...
    addZombies()

    // ...
    return game
}

private fun addZombies() = also {
    repeat(world.actualSize().zLength) { level ->
        repeat(ZOMBIES_PER_LEVEL) {
            EntityFactory.newZombie().addToWorld(level)
        }
    }
}

// ...
```

Now let's see how this looks in the game!

![Killing Zombies](/assets/img/killing_zombies.gif)

Now I wouldn't say that the process is entirely hygienic but at least now we can loot items from monsters!

## Conclusion

In this article we explored how we can add aggressive monsters to our game by combining *behaviors*. We've
reused a lot of the functionality we already had so we didn't have to write much code either!

Next up is another roguelike staple: **experience and leveling up**!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `16_AGGRESSIVE_MONSTERS` tag.
