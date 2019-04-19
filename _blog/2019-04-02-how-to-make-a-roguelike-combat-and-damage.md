---
excerpt: "Having monsters in our world asks for one thing: real combat. Let's work on that a bit!"
title: "How To Make a Roguelike: #8 Combat and Damage"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #8 Combat and Damage"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/04/02/how-to-make-a-roguelike-combat-and-damage.html
---

> We already have monsters in our game but hitting them just results in an instagib. Now we're going
to improve this by adding real *combat* to our game!

## Our Fight Club

How *combat* works is a central question in most games having it. This tutorial is not about game design
though so we'll go with a very simple model: the damage amount is a random number from 1 to the attacker's
attack value minus the defenders defense value. We know the drill, so let's create an `Attribute` first which
will hold our `CombatStats`:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.amethyst.api.Attribute
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property

data class CombatStats(val maxHpProperty: Property<Int>,    // 1
                       val hpProperty: Property<Int> = createPropertyFrom(maxHpProperty.value),  // 2
                       val attackValueProperty: Property<Int>,
                       val defenseValueProperty: Property<Int>): Attribute { // 3
                       
    val maxHp: Int by maxHpProperty.asDelegate()    // 4
    var hp: Int by hpProperty.asDelegate()          // 5
    val attackValue: Int by attackValueProperty.asDelegate()
    val defenseValue: Int by defenseValueProperty.asDelegate()

    companion object {

        fun create(maxHp: Int, hp: Int = maxHp, attackValue: Int, defenseValue: Int) =  // 6
                CombatStats(
                        maxHpProperty = createPropertyFrom(maxHp),
                        hpProperty = createPropertyFrom(hp),
                        attackValueProperty = createPropertyFrom(attackValue),
                        defenseValueProperty = createPropertyFrom(defenseValue))
    }
}                       
```

> Wait, what is this `Property` thing? A `Property` wraps a value and augments it with features like
*data binding* and the ability to listen to its changes. We'll take a look at these features soon
enough!

Here we
1. Need to know our maximum health as well
2. And we set the initial health to be equal to it
3. We implement `Attribute` as usual
4. `asDelegate` needs some explanation: it uses the `by` keyword to delegate fetching the value
   of the above fields (`maxHp`, `hp`, `attackValue` and `defenseValue`) to the respective properties.
   I've written about the topic of delegation [here](http://the-cogitator.com/posts/blog/2018/09/29/by-the-way-exploring-delegation-in-kotlin.html)
   if you're interested. The *TL;DR* is that `Property` has a `getValue` and a `setValue` function
   so it can be used to represent a field like `maxHp`
5. `hp` is a `var` because we're going to change it
6. It is quite common to have *factory methods* on classes like this. We need it here because by using it
   we keep the logic which is necessary for creating `CombatStats` within the class so its users will just
   have to supply the necessary parameters and nothing else

Now let's create a new interface which will be used by our `Combatant`s:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.CombatStats
import org.hexworks.cavesofzircon.extensions.GameEntity

interface Combatant : EntityType    // 1

val GameEntity<Combatant>.combatStats: CombatStats  // 2
    get() = findAttribute(CombatStats::class).get()
```

So why did we do this?

1. First, this interface extends `EntityType` so we can use it to mark any `Entity` we create later on
   as a `Combatant` which lets us write
2. the extension property `combatStats` which just fetches the `CombatStats` of a `GameEntity` which
   has the `EntityType` `Combatant`.
   
This code above is useful because our code handling combat can be much more expressive. Instead of having
to do `entity.findAttribute(CombatStats::Class).get()` every time we want to access `CombatStats` we
can just do `entity.combatStats` which is more readable. We'll see how this works soon but until then
let's make our *player* and the *fungus* a `Combatant`:

```kotlin
object Player : BaseEntityType(
        name = "player"), Combatant

object Fungus : BaseEntityType(
        name = "fungus"), Combatant
```

As we've previously added `Attack` let's see how that will change to incorporate our new `Combatant` type:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.cavesofzircon.attributes.types.Combatant
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.world.GameContext

data class Attack(override val context: GameContext,
                  override val source: GameEntity<Combatant>,
                  override val target: GameEntity<Combatant>) : EntityAction<Combatant, Combatant>

```

Oh wait! `Attack` will only work with entities which are `Combatant`s, so we can use our extension property!

So now we have to take a look at how our `Attackable` `Facet` will change. First let's think about what would happen
if we just add this `Facet` to all our entities which can get into combat...yep, they would attack each other, not
just the player so first let's create an extension property on `AnyGameEntity` so it is easy for us to determine
the `EntityType`:

```kotlin
// add this to EntityExtensions.kt

import org.hexworks.cavesofzircon.attributes.types.Player

val AnyGameEntity.isPlayer: Boolean
    get() = this.type == Player
```

Another useful thing our `Combatant`s can report about themselves is whether they are ready to be *destroyed*
(eg: their health is equal to or below zero). This can also be added to all `Combatant` entities:

```kotlin
// put this in EntityExtensions.kt

import org.hexworks.cavesofzircon.attributes.types.Combatant
import org.hexworks.cavesofzircon.attributes.types.combatStats

fun GameEntity<Combatant>.whenHasNoHealthLeft(fn: () -> Unit) { // 1
    if (combatStats.hp <= 0) {  // 2
        fn()                    // 3
    }
}
```

This function works like this:

1. We define it as an extension function on any `GameEntity` which has a `Combatant` type. We are passing a lambda
   function (`fn`) here, which will only be called when
2. The `hp` of our `Combatant` is equal or below zero. Otherwise the function does nothing.
3. If the condition holds we execute whatever the user of this function supplied as a lambda.

> While this function seems a bit more complex (why the lambda?) than what our intuition tells
us it should be but we'll see soon how this greatly enhances the readability of our final `Attackable` `Facet`.

One final thing to add before we modify `Attackable` is a new command, `Destroy` which "asks" an `Entity` do
destroy itself if the combat system determines that it is time to go:

> Why are we separating destruction? The reason is that there are a bunch of reasons why this is useful. One
of them is having an *invulnerability potion* or an *indestructible* perk which will prevent an `Entity` from
being destroyed even in the most dire circumstances (like reaching zero *hp*). More on this later!

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.world.GameContext

data class Destroy(override val context: GameContext,
                   override val source: GameEntity<EntityType>, // 1
                   val target: GameEntity<EntityType>,  // 2
                   val cause: String = "natural causes.") : GameCommand<EntityType> // 3
```

1. `source` is the *destroyer*
2. `target` is the `Entity` being destroyed
3. we also supply a `cause` which tells us why the `Entity` is being destroyed. 

> Note that we're not dealing with `Combatant`s here! The reason is that it is possible
  to *destroy* something without combat. Think about having a spell like `Erase` which
  will just erase something from existence.

With the new things in place `Attackable` looks like this:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.Pass
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.types.combatStats
import org.hexworks.cavesofzircon.commands.Attack
import org.hexworks.cavesofzircon.commands.Destroy
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.isPlayer
import org.hexworks.cavesofzircon.extensions.whenHasNoHealthLeft
import org.hexworks.cavesofzircon.world.GameContext

object Attackable : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(Attack::class) { (context, attacker, target) ->

        if (attacker.isPlayer || target.isPlayer) { // 1

            val damage = Math.max(0, attacker.combatStats.attackValue - target.combatStats.defenseValue)    // 2
            val finalDamage = (Math.random() * damage).toInt() + 1  // 3
            target.combatStats.hp -= finalDamage                    // 4

            target.whenHasNoHealthLeft {    // 5

                target.executeCommand(Destroy(  // 6
                        context = context,
                        source = attacker,
                        target = target,
                        cause = "a blow to the head"))
            }

            Consumed    // 7
        } else Pass
    }
}
```

This is how it works:

1. First we make sure that either the `attacker` or the `target` is a player so monsters won't kill each other
2. We calculate the `damage`. We can do this without fetching `Attribute`s manually because `source` and `target`
   in `Attack` are `Combatant`s.
3. We add `1` to `damage` to calculate the `finalDamage` to make sure that there are no super-strong entities we
   have no hope of damaging
4. When the `finalDamage` is determined we just decrement the `hp` of the `target`
5. And when it has no health left
6. We try to `Destroy` it
7. Then we return `Consumed` or `Pass` if no player was involved 

Now let's see the `CombatStats` we're gonna give to the *player* and the *fungus*:

```kotlin
import org.hexworks.cavesofzircon.attributes.CombatStats

// add this to the player entity's attributes
CombatStats.create(
        maxHp = 100,
        attackValue = 10,
        defenseValue = 5)
        
// and this to the fungus entity's attributes        
CombatStats.create(
        maxHp = 10,
        attackValue = 0,
        defenseValue = 0)      
```

So our player has `100` max hp, `10` attack and `5` defense value. Our fungus can't attack
so it only has hp, which is `10`. Let's keep it simple for now.

Now if we start the game and attack something...then nothing happens. This is because no one handles
`Destroy` yet! Let's add `Destructible`:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.commands.Destroy
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.world.GameContext

object Destructible : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(Destroy::class) { (context, attacker, target, cause) ->
        context.world.removeEntity(target)
        Consumed
    }
}
```

and make *fungus* have it

```kotlin
import org.hexworks.cavesofzircon.systems.Attackable
import org.hexworks.cavesofzircon.systems.Destructible

// add this to the fungus entity:
facets(Attackable, Destructible)
```

Now if we start up our game we can see that the *fungi* are dying a bit more slowly:

![Real combat](/assets/img/real_combat.gif)

## Logging combat messages

Fighting *fungi* is pure fun but we don't really know what happens when we bump them. Let's start using the
`LogArea` which we've added to our `PlayView`. There is a question though: *how do we have access to it from
our entities?* We could put it into the `GameContext` but then it will start to grow until it becomes a god
object having a reference to everything. We can be smarter than that by using *Zircon's* `EventBus` instead.
This is how it is going to work:

1. We create an event object which holds our game messages
2. We start listening to that event in our `PlayView`
3. We send that message whenever something important happens in our game

Let's start by adding an `Event`:

```kotlin
package org.hexworks.cavesofzircon.events

import org.hexworks.cobalt.events.api.Event

data class GameLogEvent(val text: String) : Event
```

> Using the `EventBus` supplied by *Zircon* is a good idea when we want to bridge the gap between the game world
handled by *Amethyst* and the UI itself which is managed by *Zircon*.

Now let's add a global *function* with which we can send game log events:

```kotlin
// put this in a file called Functions.kt
package org.hexworks.cavesofzircon.functions

import org.hexworks.cavesofzircon.events.GameLogEvent
import org.hexworks.zircon.internal.Zircon

fun logGameEvent(text: String) {                    // 1
    Zircon.eventBus.publish(GameLogEvent(text))     // 2
}
```

Here we

1. Create a global function which can be accessed from anywhere
2. And send a `GameLogEvent` using *Zircon*'s `EventBus` with the given `text`

Now let's augment `Attackable`:

```kotlin
import org.hexworks.cavesofzircon.functions.logGameEvent

target.combatStats.hp -= finalDamage

logGameEvent("The $attacker hits the $target for $finalDamage!")
```

and `Destructible` with some funky log messages:

```kotlin
import org.hexworks.cavesofzircon.functions.logGameEvent

context.world.removeEntity(target)

logGameEvent("$target dies after receiving $cause.")
```

Now we just have to start listening to `GameLogEvent` in our `PlayView` to make this work:

```kotlin
import org.hexworks.zircon.internal.Zircon
import org.hexworks.cobalt.events.api.subscribe
import org.hexworks.cavesofzircon.events.GameLogEvent

screen.addComponent(logArea)

Zircon.eventBus.subscribe<GameLogEvent> { (text) -> // 1
    logArea.addParagraph(                           // 2
            paragraph = text,
            withNewLine = false,                    // 3
            withTypingEffectSpeedInMs = 10)        // 4
}
```

Here we:

1. `subscribe` to the `GameLogEvent` message.
2. And we add a paragraph to our `LogArea`
3. Without adding a new line
4. But with some funky typing effect!

Let's see it in action:

![Log messages](/assets/img/log_messages.gif)

Well, that's **really funky!**

## Conclusion

In this article we added real combat with *combat stats* and also learned how to use extension properties
and generic type parameters to our advantage. We also started using *Zircon's* eventing capabilities to
greatly simplify how we interact with the UI to display messages!

In the next article we're going to create some stairs we can use to climb further down to the lower levels
of our *dungeon*! We'll also learn how to easily display our previously created *combat stats* on the UI.

Until then go forth and *kode on*!
 
> The code of this article can be found under the `8_COMBAT_AND_DAMAGE` tag.
