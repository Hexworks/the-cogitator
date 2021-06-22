---
excerpt: "We can kill a lot of monsters now but we don't gain anything else apart from the loot. Let's add leveling to our game!"
title: "How To Make a Roguelike: #17 Experience and Leveling Up"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #17 Experience and Leveling Up"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/06/28/how-to-make-a-roguelike-experience-and-leveling-up.html
updated_at: 2021-05-29
---

> Killing *zombies*, *fungi* and *bats* are fun, especially when we get loot and food from them. What's still missing is *gaining experience levels*, so we're going to add it now.

## Designing Experience Levels

If you have ever played an RPG before you might be familiar with the concept. After the *player* gains a significant amount of experience they level up which means that they can pick some benefit which will make the *player* stronger, then we go back to accumulating *experience*. Each level is harder to achieve than the previous one, and there is usually a level cap.

For our game we're going to implement something simple. The amount of experience an `Entity` gains will
be calculated in the following way: 

```
(target hp + target attack value + target defense value) - attacker's current level * 2
```

This will make the gainer get more *xp* if the target is more powerful and less *xp* with each
level.

The threshold for leveling up is calculated in the following way:

```kotlin
if current xp > current level ^ 1.5 * 20 then level up
```

So we will need more and more xp with each level. As an added bonus we'll also make the entity which levels up heal a bit! Let's take a look at how we can implement this.

## Accumulating Experience

First of all we're going to need an *attribute* which stores our current XP and level:

```kotlin
package com.example.cavesofzircon.attributes

import com.example.cavesofzircon.extensions.toStringProperty
import org.hexworks.amethyst.api.base.BaseAttribute
import org.hexworks.cobalt.databinding.api.binding.bindPlusWith
import org.hexworks.cobalt.databinding.api.extension.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property
import org.hexworks.zircon.api.Components

data class Experience(
        val currentXPProperty: Property<Int> = createPropertyFrom(0),
        val currentLevelProperty: Property<Int> = createPropertyFrom(1)
) : BaseAttribute(), DisplayableAttribute {

    var currentXP: Int by currentXPProperty.asDelegate()
    var currentLevel: Int by currentLevelProperty.asDelegate()

    override fun toComponent(width: Int) = Components.vbox()
            .withSize(width, 3)
            .build().apply {
                val xpLabel = Components.label()
                        .withSize(width, 1)
                        .build()
                val levelLabel = Components.label()
                        .withSize(width, 1)
                        .build()

                xpLabel.textProperty.updateFrom(createPropertyFrom("XP:  ")
                        .bindPlusWith(currentXPProperty.toStringProperty()))

                levelLabel.textProperty.updateFrom(createPropertyFrom("Lvl: ")
                        .bindPlusWith(currentLevelProperty.toStringProperty()))

                addComponent(Components.textBox(width)
                        .addHeader("Experience", false))
                addComponent(xpLabel)
                addComponent(levelLabel)
            }
}
```

We start at level 1 by default and zero experience.

Then we create a new *entity type* which will make it easier to access the necessary *attributes*:

```kotlin
package com.example.cavesofzircon.attributes.types

import com.example.cavesofzircon.attributes.CombatStats
import com.example.cavesofzircon.attributes.Experience
import com.example.cavesofzircon.extensions.GameEntity
import org.hexworks.amethyst.api.entity.EntityType

interface ExperienceGainer : EntityType

val GameEntity<ExperienceGainer>.experience: Experience
    get() = findAttribute(Experience::class).get()

val GameEntity<ExperienceGainer>.combatStats: CombatStats
    get() = findAttribute(CombatStats::class).get()
```

And assign it to the *player*:

```kotlin
object Player : BaseEntityType(
        name = "player"
), Combatant, ItemHolder, EnergyUser, EquipmentHolder, ExperienceGainer
```

Since we know that we'll only gain experience from combat we also make `ExperienceGainer` have access to `CombatStats`.

When the *player* gains a level we're going to open up a dialog and for this we're going to use an *event*:

```kotlin
package com.example.cavesofzircon.events

import org.hexworks.cobalt.events.api.Event

data class PlayerGainedLevel(override val emitter: Any) : Event
```

> We could argue, that other creatures might be able to gain a level, but this `Event` is not for *Amethyst*, but for *Zircon*. We only display dialogs for the *player*, thus the name of this class.

What we have to do before we can make this system work is to add a new *message*, `EntityDestroyed`:

```kotlin
package com.example.cavesofzircon.messages

import com.example.cavesofzircon.extensions.GameEntity
import com.example.cavesofzircon.extensions.GameMessage
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.entity.EntityType

data class EntityDestroyed(
        override val context: GameContext,
        override val source: GameEntity<EntityType>,
        val destroyer: GameEntity<EntityType>
) : GameMessage
```

Which we use in `Destructible` to signal that an *entity* was destroyed:

```kotlin
package com.example.cavesofzircon.systems

import com.example.cavesofzircon.messages.Destroy
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.Response
import org.hexworks.amethyst.api.base.BaseFacet
import com.example.cavesofzircon.functions.logGameEvent
import com.example.cavesofzircon.messages.EntityDestroyed

object Destructible : BaseFacet<GameContext, Destroy>(Destroy::class) {
    override suspend fun receive(message: Destroy): Response {
        val (context, destroyer, target, cause) = message
        context.world.removeEntity(target)
        destroyer.receiveMessage(EntityDestroyed(context, target, destroyer))
        logGameEvent("$target dies $cause.", Destructible)
        return Consumed
    }
}
```

With these in place we can implement our *facet* which will be responsible for the experience calculation whenever an *entity* destroys something:

```kotlin
package com.example.cavesofzircon.systems

import com.example.cavesofzircon.attributes.CombatStats
import com.example.cavesofzircon.attributes.types.ExperienceGainer
import com.example.cavesofzircon.attributes.types.combatStats
import com.example.cavesofzircon.attributes.types.experience
import com.example.cavesofzircon.events.PlayerGainedLevel
import com.example.cavesofzircon.extensions.attackValue
import com.example.cavesofzircon.extensions.defenseValue
import com.example.cavesofzircon.extensions.isPlayer
import com.example.cavesofzircon.extensions.whenTypeIs
import com.example.cavesofzircon.functions.logGameEvent
import com.example.cavesofzircon.messages.EntityDestroyed
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.Response
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.zircon.internal.Zircon
import kotlin.math.min

object ExperienceAccumulator : BaseFacet<GameContext, EntityDestroyed>(EntityDestroyed::class) {
    override suspend fun receive(message: EntityDestroyed): Response {
        val (_, defender, attacker) = message
        attacker.whenTypeIs<ExperienceGainer> { experienceGainer ->                                         // 1
            val xp = experienceGainer.experience
            val stats = experienceGainer.combatStats
            val defenderHp = defender.findAttribute(CombatStats::class).map { it.maxHp }.orElse(0)          // 2
            val amount = (defenderHp + defender.attackValue + defender.defenseValue) - xp.currentLevel * 2  // 3
            if (amount > 0) {
                xp.currentXP += amount
                while (xp.currentXP > Math.pow(xp.currentLevel.toDouble(), 1.5) * 20) {                     // 4
                    xp.currentLevel++
                    logGameEvent("$attacker advanced to level ${xp.currentLevel}.", ExperienceAccumulator)
                    stats.hpProperty.value = min(stats.hp + xp.currentLevel * 2, stats.maxHp)               // 5
                    if (attacker.isPlayer) {
                        Zircon.eventBus.publish(PlayerGainedLevel(ExperienceAccumulator))                   // 6
                    }
                }
            }

        }
        return Consumed
    }
}
```

Here we:

1. make sure that the entity which destroys the other is an `ExperienceGainer`
2. Then we find out the `hp` of the defender. We will use `0` if it doesn't have `hp`!
3. Then we do the experience calculation
4. And after we assigned it we check whether we gained a level or not
5. If we did we heal a bit
6. And if the attacker was the *player* we send the event

Now all we need to do to make this system work is add the corresponding *attribute* and *facet* to the *player*:

```kotlin
// modify this in EntityFactory

import com.example.cavesofzircon.systems.ExperienceAccumulator
import com.example.cavesofzircon.attributes.Experience

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
            Experience(),
            // ... )
    // ...
    facets(ExperienceAccumulator, // ...)
}
```

Now if we start killing things we can see that experience is gained and our *displayable* *experience* *attribute* automagically appears among the *player* stats.

![Gaining Experience](/assets/img/gaining_experience.gif)

## Displaying a Level Up Dialog

It is nice that we report that the *player* leveled up but we should also display a dialog where the *player* can choose which stat to increase. Since we've created quite a few dialogs already let's thing about how to create an abstraction which makes it easy to create a new one. Let's think about how a dialog looks like. In most of the cases it has some *content* and a *close button*. Let's create a *fragment* which we'll use for the closing operation:

```kotlin
package com.example.cavesofzircon.view.dialog

import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.component.Container
import org.hexworks.zircon.api.component.Fragment
import org.hexworks.zircon.api.component.modal.Modal
import org.hexworks.zircon.internal.component.modal.EmptyModalResult

class CloseButtonFragment(modal: Modal<EmptyModalResult>, parent: Container) : Fragment {

    override val root = Components.button().withText("Close")
            .withAlignmentWithin(parent, ComponentAlignment.BOTTOM_RIGHT)
            .build().apply {
                onActivated {
                    modal.close(EmptyModalResult)
                }
            }
}
```

This is rather straightforward. We create a `Button` which we align to the bottom right, and when it is activated we just close the `modal`.

Then the `Dialog` itself will be a `ModalFragment` which can optionally contain a *close button*:

```kotlin
package com.example.cavesofzircon.view.dialog

import com.example.cavesofzircon.GameConfig
import org.hexworks.zircon.api.builder.component.ModalBuilder
import org.hexworks.zircon.api.component.Container
import org.hexworks.zircon.api.component.modal.Modal
import org.hexworks.zircon.api.component.modal.ModalFragment
import org.hexworks.zircon.api.screen.Screen
import org.hexworks.zircon.internal.component.modal.EmptyModalResult

abstract class Dialog(
        private val screen: Screen,
        withClose: Boolean = true
) : ModalFragment<EmptyModalResult> {

    abstract val container: Container                           // 1

    final override val root: Modal<EmptyModalResult> by lazy {  // 2
        ModalBuilder.newBuilder<EmptyModalResult>()
                .withComponent(container)
                .withParentSize(screen.size)
                .withCenteredDialog(true)                       // 3
                .build().also {
                    if (withClose) {                            // 4
                        container.addFragment(CloseButtonFragment(it, container))
                    }
                    container.theme = GameConfig.THEME
                }
    }
}
```

What this does is that:

1. It defines a `container` property which will be supplied when we implement `Dialog`
2. The `root` is calculated lazily to make sure that `container` is available when `root`
   is accessed for the first time
3. It centers the dialog
4. And adds the *close fragment* if we want it

With the new `Dialog` class now we can implement the `LevelUpDialog` itself:

```kotlin
package com.example.cavesofzircon.view.dialog

import com.example.cavesofzircon.attributes.CombatStats
import com.example.cavesofzircon.attributes.Vision
import com.example.cavesofzircon.attributes.types.Player
import com.example.cavesofzircon.extensions.GameEntity
import com.example.cavesofzircon.extensions.tryToFindAttribute
import com.example.cavesofzircon.functions.logGameEvent
import org.hexworks.zircon.api.ComponentDecorations.box
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.graphics.BoxType
import org.hexworks.zircon.api.screen.Screen
import org.hexworks.zircon.internal.component.modal.EmptyModalResult

class LevelUpDialog(
        screen: Screen,
        player: GameEntity<Player>
) : Dialog(screen, false) {

    override val container = Components.vbox()                                          // 1
            .withDecorations(box(title = "Ding!", boxType = BoxType.TOP_BOTTOM_DOUBLE))
            .withSize(30, 15)
            .build().apply {
                val stats = player.tryToFindAttribute(CombatStats::class)
                val vision = player.tryToFindAttribute(Vision::class)

                addComponent(Components.textBox(27)
                        .addHeader("Congratulations, you leveled up!")
                        .addParagraph("Pick an improvement from the options below:"))   // 2

                addComponent(Components.button()                                        // 3
                        .withText("Max HP")
                        .build().apply {
                            onActivated {
                                stats.maxHpProperty.value += 10
                                logGameEvent("You look healthier.", this)
                                root.close(EmptyModalResult)
                            }
                        })


                addComponent(Components.button()
                        .withText("Attack")
                        .build().apply {
                            onActivated {
                                stats.attackValueProperty.value += 2
                                logGameEvent("You look stronger.", this)
                                root.close(EmptyModalResult)
                            }
                        })

                addComponent(Components.button()
                        .withText("Defense")
                        .build().apply {
                            onActivated {
                                stats.defenseValueProperty.value += 2
                                logGameEvent("You look tougher.", this)
                                root.close(EmptyModalResult)
                            }
                        })

                addComponent(Components.button()
                        .withText("Vision")
                        .build().apply {
                            onActivated {
                                vision.radius++                                         // 4
                                logGameEvent("You look more perceptive.", this)
                                root.close(EmptyModalResult)
                            }
                        })
            }
}
```

With this dialog we:

1. Create a nice *vbox*
2. Congratulate the *player*
3. Add a *button* for all 4 stats we can improve
4. Make `radius` in `Vision` a `var` so we can modify it

To display this we just need to listen to the `PlayerGainedLevel` *event* in `PlayView`:

```kotlin
import import com.example.cavesofzircon.events.PlayerGainedLevel
import com.example.cavesofzircon.view.dialog.LevelUpDialog

// ...

class PlayView(private val game: Game = GameBuilder.defaultGame()) : BaseView() {

    // ...

    init {

        // ...
        Zircon.eventBus.subscribeTo<PlayerGainedLevel> {
            screen.openModal(LevelUpDialog(screen, game.player))
            KeepSubscription
        }
        
        // ...
    }
}
```

Now let's see how this new feature works:

![Leveling Up](/assets/img/leveling_up.gif)

As you can see after choosing *Attack* we get `+2` to our *attack* stat and a nice log message. Congratulations, you've just made an experience system!

## Conclusion

In this article we added a simple *experience system* to our game complete with stat boosts and levels. As a fun addition we can now make our monsters gain experience as well, since the code we've written is universal. Apart from these we also learned how to abstract away creating dialogs which will be very useful later on!

In the next session we're going to add *help* and *examine* dialogs!

Until then go forth and *kode on*!
 
> The code of this article can be found in commit #17.
