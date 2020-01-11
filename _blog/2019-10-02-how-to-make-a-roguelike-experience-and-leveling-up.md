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
---

> Killing *zombies*, *fungi* and *bats* are fun, especially when we get loot and food from them. What's
still missing is *gaining experience levels*, so we're going to add it now.

## Designing Experience Levels

If you have ever played an RPG before you might be familiar with the concept. After the *player* gains
a significant amount of experience they level up which means that they can pick some benefit which will
make the *player* stronger, then we go back to accumulating *experience*. Each level is harder to
achieve than the previous one, and there is usually a level cap.

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

So we will need more and more xp with each level. As an added bonus we'll also make the entity which
levels up heal a bit! Let's take a look at how we can implement this.

## Accumulating Experience

First of all we're going to need an *attribute* which stores our current XP and level:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.expression.concat
import org.hexworks.cobalt.databinding.api.property.Property
import org.hexworks.zircon.api.Components

data class Experience(val currentXPProperty: Property<Int> = createPropertyFrom(0),
                      val currentLevelProperty: Property<Int> = createPropertyFrom(1)) : DisplayableAttribute {

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
                        .concat(currentXPProperty))

                levelLabel.textProperty.updateFrom(createPropertyFrom("Lvl: ")
                        .concat(currentLevelProperty))

                addComponent(Components.textBox()
                        .withContentWidth(width)
                        .addHeader("Experience", false))
                addComponent(xpLabel)
                addComponent(levelLabel)
            }
}
```

We start at level 1 by default and zero experience.

Then we create a new *entity type* which will make it easier to access the necessary *attributes*:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.CombatStats
import org.hexworks.cavesofzircon.attributes.Experience
import org.hexworks.cavesofzircon.extensions.GameEntity

interface ExperienceGainer : EntityType

val GameEntity<ExperienceGainer>.experience: Experience
    get() = findAttribute(Experience::class).get()

val GameEntity<ExperienceGainer>.combatStats: CombatStats
    get() = findAttribute(CombatStats::class).get()

```

And assign it to the *player*:

```kotlin
object Player : BaseEntityType(
        name = "player"), Combatant, ItemHolder, EnergyUser, EquipmentHolder, ExperienceGainer
```

Since we know that we'll only gain experience from combat we also make `ExperienceGainer` have
access to `CombatStats`.

When the *player* gains a level we're going to open up a dialog and for this we're going to use an *event*:

```kotlin
package org.hexworks.cavesofzircon.events

import org.hexworks.cobalt.events.api.Event

object PlayerGainedLevel : Event
```

> We could argue, that other creatures might be able to gain a level, but this `Event` is not for
*Amethyst*, but for *Zircon*. We only display dialogs for the *player*, thus the name of this class.

What we have to do before we can make this system work is to add a new *command*, `EntityDestroyed`:

```kotlin
package org.hexworks.cavesofzircon.commands

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.world.GameContext

data class EntityDestroyed(override val context: GameContext,
                           override val source: GameEntity<EntityType>,
                           val destroyer: GameEntity<EntityType>) : GameCommand<EntityType>
```

Which we use in `Destructible` to signal that an *entity* was destroyed:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.commands.Destroy
import org.hexworks.cavesofzircon.commands.EntityDestroyed
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.functions.logGameEvent
import org.hexworks.cavesofzircon.world.GameContext

object Destructible : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(Destroy::class) { (context, destroyer, target, cause) ->
        context.world.removeEntity(target)
        destroyer.executeCommand(EntityDestroyed(context, target, destroyer))
        logGameEvent("$target dies $cause.")
        Consumed
    }
}
```

With these in place we can implement our *facet* which will be responsible for the experience calculation
whenever an *entity* destroys something:

```kotlin
package org.hexworks.cavesofzircon.systems

import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.Response
import org.hexworks.amethyst.api.base.BaseFacet
import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.CombatStats
import org.hexworks.cavesofzircon.attributes.types.ExperienceGainer
import org.hexworks.cavesofzircon.attributes.types.combatStats
import org.hexworks.cavesofzircon.attributes.types.experience
import org.hexworks.cavesofzircon.commands.EntityDestroyed
import org.hexworks.cavesofzircon.events.PlayerGainedLevel
import org.hexworks.cavesofzircon.extensions.GameCommand
import org.hexworks.cavesofzircon.extensions.attackValue
import org.hexworks.cavesofzircon.extensions.defenseValue
import org.hexworks.cavesofzircon.extensions.isPlayer
import org.hexworks.cavesofzircon.extensions.whenTypeIs
import org.hexworks.cavesofzircon.functions.logGameEvent
import org.hexworks.cavesofzircon.world.GameContext
import org.hexworks.cobalt.datatypes.extensions.map
import org.hexworks.zircon.internal.Zircon
import kotlin.math.min

object ExperienceAccumulator : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(EntityDestroyed::class) { (_, defender, attacker) ->
        attacker.whenTypeIs<ExperienceGainer> { experienceGainer ->   // 1
            val xp = experienceGainer.experience
            val stats = experienceGainer.combatStats
            val defenderHp = defender.findAttribute(CombatStats::class).map { it.maxHp }.orElse(0)            // 2
            val amount = (defenderHp + defender.attackValue + defender.defenseValue) - xp.currentLevel * 2    // 3
            if (amount > 0) {
                xp.currentXP += amount
                while (xp.currentXP > Math.pow(xp.currentLevel.toDouble(), 1.5) * 20) {                 // 4
                    xp.currentLevel++
                    logGameEvent("$attacker advanced to level ${xp.currentLevel}.")
                    stats.hpProperty.value = min(stats.hp + xp.currentLevel * 2, stats.maxHp)           // 5
                    if (attacker.isPlayer) {
                        Zircon.eventBus.publish(PlayerGainedLevel)                                      // 6
                    }
                }
            }

        }
        Consumed
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

import org.hexworks.cavesofzircon.systems.ExperienceAccumulator
import org.hexworks.cavesofzircon.attributes.Experience

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
            Experience(),
            // ... )
    // ...
    facets(ExperienceAccumulator, // ...)
}
```

Now if we start killing things we can see that experience is gained and our *displayable* *experience* *attribute*
automagically appears among the *player* stats.

![Gaining Experience](/assets/img/gaining_experience.gif)

## Displaying a Level Up Dialog

It is nice that we report that the *player* leveled up but we should also display a dialog where the *player*
can choose which stat to increase. Since we've created quite a few dialogs already let's thing about how to
create an abstraction which makes it easy to create a new one. Let's think about how a dialog looks like.
In most of the cases it has some *content* and a *close button*. Let's create a *fragment* which we'll use for
the closing operation:

```kotlin
package org.hexworks.cavesofzircon.view.dialog

import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.component.Container
import org.hexworks.zircon.api.component.Fragment
import org.hexworks.zircon.api.component.modal.Modal
import org.hexworks.zircon.api.extensions.onComponentEvent
import org.hexworks.zircon.api.uievent.ComponentEventType
import org.hexworks.zircon.api.uievent.Processed
import org.hexworks.zircon.internal.component.modal.EmptyModalResult

class CloseButtonFragment(modal: Modal<EmptyModalResult>, parent: Container) : Fragment {

    override val root = Components.button().withText("Close")
            .withAlignmentWithin(parent, ComponentAlignment.BOTTOM_RIGHT)
            .build().apply {
                onComponentEvent(ComponentEventType.ACTIVATED) {
                    modal.close(EmptyModalResult)
                    Processed
                }
            }
}
```

This is rather straightforward. We create a `Button` which we align to the bottom right,
and when it is activated we just close the `modal`.

Then the `Dialog` itself will be a `ModalFragment` which can optionally contain a *close button*:

```kotlin
package org.hexworks.cavesofzircon.view.dialog

import org.hexworks.cavesofzircon.GameConfig
import org.hexworks.zircon.api.builder.component.ModalBuilder
import org.hexworks.zircon.api.component.Container
import org.hexworks.zircon.api.component.modal.Modal
import org.hexworks.zircon.api.component.modal.ModalFragment
import org.hexworks.zircon.api.screen.Screen
import org.hexworks.zircon.internal.component.modal.EmptyModalResult

abstract class Dialog(private val screen: Screen,
                      withClose: Boolean = true) : ModalFragment<EmptyModalResult> {

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
                    container.applyColorTheme(GameConfig.THEME)
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
package org.hexworks.cavesofzircon.view.dialog

import org.hexworks.cavesofzircon.attributes.CombatStats
import org.hexworks.cavesofzircon.attributes.Vision
import org.hexworks.cavesofzircon.attributes.types.Player
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.extensions.tryToFindAttribute
import org.hexworks.cavesofzircon.functions.logGameEvent
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.extensions.onComponentEvent
import org.hexworks.zircon.api.graphics.BoxType
import org.hexworks.zircon.api.screen.Screen
import org.hexworks.zircon.api.uievent.ComponentEventType
import org.hexworks.zircon.api.uievent.Processed
import org.hexworks.zircon.internal.component.modal.EmptyModalResult

class LevelUpDialog(screen: Screen, player: GameEntity<Player>) : Dialog(screen, false) {

    override val container = Components.vbox()
            .withTitle("Ding!")
            .withSize(30, 15)
            .withBoxType(BoxType.TOP_BOTTOM_DOUBLE)     // 1
            .wrapWithBox()
            .build().apply {
                val stats = player.tryToFindAttribute(CombatStats::class)
                val vision = player.tryToFindAttribute(Vision::class)
                
                addComponent(Components.textBox()
                        .withContentWidth(27)
                        .addHeader("Congratulations, you leveled up!")
                        .addParagraph("Pick an improvement from the options below:"))   // 2

                addComponent(Components.button()                                        // 3
                        .withText("Max HP")
                        .build().apply {
                            onComponentEvent(ComponentEventType.ACTIVATED) {
                                stats.maxHpProperty.value += 10
                                logGameEvent("You look healthier.")
                                root.close(EmptyModalResult)
                                Processed
                            }
                        })


                addComponent(Components.button()
                        .withText("Attack")
                        .build().apply {
                            onComponentEvent(ComponentEventType.ACTIVATED) {
                                stats.attackValueProperty.value += 2
                                logGameEvent("You look stronger.")
                                root.close(EmptyModalResult)
                                Processed
                            }
                        })

                addComponent(Components.button()
                        .withText("Defense")
                        .build().apply {
                            onComponentEvent(ComponentEventType.ACTIVATED) {
                                stats.defenseValueProperty.value += 2
                                logGameEvent("You look tougher.")
                                root.close(EmptyModalResult)
                                Processed
                            }
                        })

                addComponent(Components.button()
                        .withText("Vision")
                        .build().apply {
                            onComponentEvent(ComponentEventType.ACTIVATED) {
                                vision.radius++                                 // 4 
                                logGameEvent("You look more perceptive.")
                                root.close(EmptyModalResult)
                                Processed
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
import import org.hexworks.cavesofzircon.events.PlayerGainedLevel
import org.hexworks.cavesofzircon.view.dialog.LevelUpDialog

// ...

class PlayView(private val game: Game = GameBuilder.defaultGame()) : BaseView() {

    // ...

    override fun onDock() {

        // ...
        Zircon.eventBus.subscribe<PlayerGainedLevel> {
            screen.openModal(LevelUpDialog(screen, game.player))
        }
        
        // ...
    }
}
```

Now let's see how this new feature works:

![Leveling Up](/assets/img/leveling_up.gif)

As you can see after choosing *Attack* we get `+2` to our *attack* stat and a nice log message.
Congratulations, you've just made an experience system!

## Conclusion

In this article we added a simple *experience system* to our game complete with stat boosts and
levels. As a fun addition we can now make our monsters gain experience as well, since the code
we've written is universal. Apart from these we also learned how to abstract away creating dialogs
which will be very useful later on!

In the next session we're going to add *help* and *examine* dialogs!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `17_EXPERIENCE_AND_LEVELING_UP` tag.
