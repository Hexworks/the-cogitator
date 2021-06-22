---
excerpt: "Having items is nice, but let's improve on that by adding a new game mechanic: hunger!"
title: "How To Make a Roguelike: #13 Food and Hunger"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #13 Food and Hunger"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/05/23/how-to-make-a-roguelike-food-and-hunger.html
updated_at: 2021-04-15
---

> Now, that we are able to pick up items and take a look at them in the inventory we can start to add more useful loot, and some new mechanics as well. In this article we're going to create a hunger system and add the corresponding item type: food.

## Implementing a Hunger System

Let's think about how hunger works. For humans like you and me hunger works like this (over-simplified):

If we do nothing, just lie in a bed we still burn a flat amount of energy (calories). This is called BMR, or Basal Metabolic Rate. If we *do* something, like walking around, lifting weights, etc we burn more energy. We can roughly calculate the amount of calories we burn for each of these activities. All of the above reduce our energy levels. If we reach zero we die. What we can do against this is *eating*. When we eat, it adds a flat amount of energy to our energy reserves. Ideally we *burn* as much as we *eat* so we can stay well fed! Now let's see how we can go about implementing such system in our game.

## Adding Energy Level

First, we're going to create some new *attributes* which will hold our current energy levels, and the amount of energy stored in specific items. `NutritionalValue` will represent the calorie count of an item:

```kotlin
package com.example.cavesofzircon.attributes

import org.hexworks.amethyst.api.base.BaseAttribute

data class NutritionalValue(val energy: Int) : BaseAttribute()
```

and the corresponding item *type* is `Food`:

```kotlin
package com.example.cavesofzircon.attributes.types

import com.example.cavesofzircon.attributes.NutritionalValue
import com.example.cavesofzircon.extensions.GameEntity
import com.example.cavesofzircon.extensions.tryToFindAttribute

interface Food : Item

val GameEntity<Food>.energy: Int
    get() = tryToFindAttribute(NutritionalValue::class).energy
```

The *energy level* of an `Entity` can be represented by the `EnergyLevel` *attribute*:

```kotlin
package com.example.cavesofzircon.attributes

import org.hexworks.amethyst.api.base.BaseAttribute
import org.hexworks.cobalt.databinding.api.extension.createPropertyFrom

class EnergyLevel(
    initialEnergy: Int,
    val maxEnergy: Int
) : BaseAttribute() {

    var currentEnergy: Int
        get() = currentValueProperty.value
        set(value) {
            currentValueProperty.value = Math.min(value, maxEnergy)
        }

    private val currentValueProperty = createPropertyFrom(initialEnergy)

}
```

and we can also add a *trait* which we can use for easy access of this attribute:

```kotlin
package com.example.cavesofzircon.attributes.types

import com.example.cavesofzircon.attributes.EnergyLevel
import com.example.cavesofzircon.extensions.GameEntity
import org.hexworks.amethyst.api.entity.EntityType

interface EnergyUser : EntityType

val GameEntity<EnergyUser>.energyLevel: EnergyLevel
    get() = findAttribute(EnergyLevel::class).get()
```

So far so good. Now we can start adding foods to our game. And in order to eat them we're gonna need a *message*, `Eat` which looks like this:

```kotlin
package com.example.cavesofzircon.messages

import com.example.cavesofzircon.attributes.types.EnergyUser
import com.example.cavesofzircon.attributes.types.Food
import com.example.cavesofzircon.extensions.GameEntity
import com.example.cavesofzircon.world.GameContext

data class Eat(
    override val context: GameContext,
    override val source: GameEntity<EnergyUser>,
    override val target: GameEntity<Food>
) : EntityAction<EnergyUser, Food>
```

Now that we're at it let's add another message that we're going to use to when we expend energy. This can be *digging*, *moving* or simply *existing*:

```kotlin
package com.example.cavesofzircon.messages

import com.example.cavesofzircon.attributes.types.EnergyUser
import com.example.cavesofzircon.extensions.GameEntity
import com.example.cavesofzircon.extensions.GameMessage
import com.example.cavesofzircon.world.GameContext

data class Expend(
    override val context: GameContext,
    override val source: GameEntity<EnergyUser>,
    val energy: Int
) : GameMessage
``` 

Now let's see how we can *consume* (pun intended) these messages. For `Eat`ing it seems straightforward to add a `DigestiveSystem`:

```kotlin
package com.example.cavesofzircon.systems

import com.example.cavesofzircon.attributes.EnergyLevel
import com.example.cavesofzircon.attributes.types.energy
import com.example.cavesofzircon.attributes.types.energyLevel
import com.example.cavesofzircon.extensions.isPlayer
import com.example.cavesofzircon.functions.logGameEvent
import com.example.cavesofzircon.messages.Eat
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.Response
import org.hexworks.amethyst.api.base.BaseFacet

object DigestiveSystem : BaseFacet<GameContext, Eat>(Eat::class, EnergyLevel::class) {
    override suspend fun receive(message: Eat): Response {
        val (_, entity, food) = message
        entity.energyLevel.currentEnergy += food.energy
        val verb = if (entity.isPlayer) {
            "You eat"
        } else "The $entity eats"
        logGameEvent("$verb the $food.", DigestiveSystem)
        return Consumed
    }
}
```

This seems simple enough. Whenever an `Entity` which is an `EnergyUser` `Eat`s something the *energy* contained in the `Food` is added to their `currentEnergy`.

and for `Expend` ing we will add a new kind of `System`: an `Actor`! So what's an *actor* and why do we need it? Simply put an `Actor` is a `System` which is both a `Facet` and a `Behavior`. In this case we'll add an actor named `EnergyExpender` which implements the behavior we were talking about before: it `Expend`s energy on its own (basal metabolic rate) but it also `Expend`s energy when we do something hard like digging. Before we add this *system* let's add a helper function to `EntityExtensions.kt` which we're going to use:

```kotlin
import kotlin.reflect.full.isSubclassOf

inline fun <reified T : EntityType> AnyGameEntity.whenTypeIs(fn: (Entity<T, GameContext>) -> Unit) {
    if (this.type::class.isSubclassOf(T::class)) {
        fn(this as Entity<T, GameContext>)
    }
}
```

then we can add `EnergyExpender`:

> We'll have to update our *Amethyst* version here from `2021.0.1-RELEASE` to `2021.0.3-RELEASE` because the `BaseActor` class was missing some members in the previous version. You can do this by modiying `gradle.properties`.

```kotlin
package com.example.cavesofzircon.systems

import com.example.cavesofzircon.attributes.EnergyLevel
import com.example.cavesofzircon.attributes.types.EnergyUser
import com.example.cavesofzircon.attributes.types.energyLevel
import com.example.cavesofzircon.extensions.GameEntity
import com.example.cavesofzircon.extensions.whenTypeIs
import com.example.cavesofzircon.messages.Destroy
import com.example.cavesofzircon.messages.Expend
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.Consumed
import org.hexworks.amethyst.api.Response
import org.hexworks.amethyst.api.base.BaseActor
import org.hexworks.amethyst.api.entity.Entity
import org.hexworks.amethyst.api.entity.EntityType
import kotlin.reflect.KClass

object EnergyExpender : BaseActor<GameContext, Expend>(Expend::class, EnergyLevel::class) {

    override val messageType: KClass<Expend> = Expend::class

    override suspend fun receive(message: Expend): Response {
        val (context, entity, energy) = message
        entity.energyLevel.currentEnergy -= energy              // 1
        checkStarvation(context, entity, entity.energyLevel)    // 2
        return Consumed
    }

    override suspend fun update(entity: Entity<EntityType, GameContext>, context: GameContext): Boolean {
        entity.whenTypeIs<EnergyUser> {                         // 3
            entity.receiveMessage(
                Expend(                                         // 4
                    context = context,
                    source = it,
                    energy = 2
                )
            )
        }
        return true
    }


    private suspend fun checkStarvation(
        context: GameContext,
        entity: GameEntity<EntityType>,
        energyLevel: EnergyLevel
    ) {
        if (energyLevel.currentEnergy <= 0) {                   // 5
            entity.receiveMessage(
                Destroy(                                        // 6
                    context = context,
                    source = entity,
                    target = entity,
                    cause = "starvation"
                )
            )
        }
    }
}
```

So as you can see we have both `receive` and `update` here. And what happens here is:

1. When we receive an `Expend` command we just subtract the `energy` from our `currentEnergy`
2. And check for starvation
3. In `update` we check whether our *entity* is an `EnergyUser` and in this case
4. We ask it to `Expend` 2 energy
5. And in `checkStarvation` we check whether our `currentEnergy` is equal to or less than `0`
6. And if yes we send the `Destroy` command to the *entity*

Now let's augment our *player* with this:

```kotlin
// modify EntityTypes.kt by adding EnergyUser
object Player : BaseEntityType(
    name = "player"
), Combatant, ItemHolder, EnergyUser
```

and the `EntityFactory`:

```kotlin
import com.example.cavesofzircon.attributes.EnergyLevel
import com.example.cavesofzircon.systems.EnergyExpender
import com.example.cavesofzircon.systems.DigestiveSystem

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
            // ...
            EnergyLevel(10, 10)         // add this attribute
    )
    behaviors(/* ... */, EnergyExpender)    // and add the actor to behaviors
    facets(/* ... */ EnergyExpender, DigestiveSystem)  // and facets!
}
```

> A small note here: *Amethyst* doesn't differentiate between *actors* and other systems. `BaseActor` is only provided as a convenience, you still have to add an *actor* to both *behaviors* and *facets* by hand. We'll provide an `actors()` function in the future though.

Now if you try the code this is what'll happen:

![Dying of Hunger](/assets/img/dying_of_hunger.gif)

So now **we have a lose condition**! We can also see that the log message is not very useful so let's improve it by editing the text in `Destructible`: to `logGameEvent("$target dies $cause.", Destructible)` and fix the messages in `Attackable` to `"after receiving a blow to the head"` and in `EnergyExpender` to `"because of starvation"`.

> It is also funny that the *player* just disappears from the screen when he dies and we can do nothing more. We'll add proper *lose* and *victory* conditions in a later article.

## Looting Bat Meat

Now that we can die of starvation it would make sense to add some means to prevent this from happening. First of all let's change the initial values of `EnergyLevel` to `EnergyLevel(1000, 1000)`. We just added `10` to see what happens if we run out of energy but in a realistic scenario we don't die after making 5 steps.

It is very unlikely in a dungeon that food is just laying around so what about combining what we learned in the previous article and we add *food* to our game which is *lootable* from corpses?

We already have some creatures which might be able to drop *food* so the *bat* is a very good candidate for this!

Let's start by adding a new *entity type* to `EntityTypes.kt`, `BatMeat`:

```kotlin
object BatMeat : BaseEntityType(
    name = "Bat meat",
    description = "Stringy bat meat. It is edible, but not tasty."
), Food
```

and the necessary color in `GameColors`:

```kotlin
val BAT_MEAT_COLOR = TileColor.fromString("#EA4861")
```

and the corresponding `Tile` in `GameTileRepository`:

```kotlin
val BAT_MEAT = Tile.newBuilder()
    .withCharacter('m')
    .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
    .withForegroundColor(GameColors.BAT_MEAT_COLOR)
    .buildCharacterTile()
```

Now we have everything to create the actual entity in `EntityFactory`:

```kotlin
import com.example.cavesofzircon.attributes.NutritionalValue
import com.example.cavesofzircon.attributes.types.BatMeat

fun newBatMeat() = newGameEntityOfType(BatMeat) {
    attributes(
        ItemIcon(
            Tile.newBuilder()
                .withName("Meatball")       // 1
                .withTileset(GraphicalTilesetResources.nethack16x16())
                .buildGraphicalTile()
        ),
        NutritionalValue(750),              // 2
        EntityPosition(),
        EntityTile(GameTileRepository.BAT_MEAT)
    )
}
```

Here we:

1. Add a nice icon which we'll display in the inventory
2. And we also make it **very** nutritious


## Dropping Loot

Now that we have some food we just need to figure out how the player can obtain it. The most common way of doing so is usually to make *creatures* just drop whatever loot they have when they die so we're going to stick with this method.

First of all we're going to make the *bat* an `ItemHolder`:
 
```kotlin
// modify EntityTypes.kt with this

object Bat : BaseEntityType(
    name = "bat"
), Combatant, ItemHolder
```

and create a new *facet*, `LootDropper` which will just drop all inventory items when an *entity* with one is destroyed:

```kotlin
package com.example.cavesofzircon.systems

import com.example.cavesofzircon.attributes.types.ItemHolder
import com.example.cavesofzircon.attributes.types.inventory
import com.example.cavesofzircon.extensions.position
import com.example.cavesofzircon.extensions.whenTypeIs
import com.example.cavesofzircon.messages.Destroy
import com.example.cavesofzircon.messages.DropItem
import com.example.cavesofzircon.world.GameContext
import org.hexworks.amethyst.api.Pass
import org.hexworks.amethyst.api.Response
import org.hexworks.amethyst.api.base.BaseFacet

object LootDropper : BaseFacet<GameContext, Destroy>(Destroy::class) {

    override suspend fun receive(message: Destroy): Response {
        val (context, _, target) = message              // 1
        target.whenTypeIs<ItemHolder> { entity ->       // 2
            entity.inventory.items.forEach { item ->
                entity.receiveMessage(DropItem(context, entity, item, entity.position))     // 3
            }
        }
        return Pass                                     // 4
    }
}
```

> We could argue that it is weird to have *bat*s run around with their own meat in their pockets but it doesn't make a difference from a gameplay perspective.

What happens here is that

1. On the `Destroy` message
2. When our *entity* is an `ItemHolder`
3. We just drop all items
4. Then we `Pass`. This last step is **very important** this ensures that the `Destroy` message is not consumed and it is 
   propagated to the next *system* which will be `Destructible` in our case
   
Now let's modify our *bat* entity to have all the new systems:

```kotlin
// modify EntityFactory.kt with these

import com.example.cavesofzircon.systems.LootDropper

fun newBat() = newGameEntityOfType(Bat) {
    attributes(
            // ...
            Inventory(1).apply {
                addItem(newBatMeat())                           // 1
            }
    )
    facets(/* ... */ LootDropper, ItemDropper, Destructible)    // 2
    // ...
}
```

In this change we:

1. Added an *inventory* to the bat with a new *bat meat*
2. And added the `LootDropper` facet **before** `Destructible`. This is important because
   when `LootDropper` `Pass`es, `Destructible` will get the `Destroy` *message*. If they were
   flipped `LootDropper` would not get the `Destroy` *message* because `Destructible` consumes
   it. Try it out!

> The eagle eyed might have noticed that we also added `ItemDropper`! This is because we've already implemented how dropping items work, so we can just reuse it by sending a `DropItem` message from `LootDropper`!
   
Let's see how this works in our game:

![Collecting Food](/assets/img/looting_food.gif)

Nice. We can now kill *bats* for their *food*, but we still can't eat them. What's left to do is to add a button in our inventory for eating, so first let's modify `InventoryRowFragment` a bit:

```kotlin
package org.hexworks.cavesofzircon.view.fragment

import org.hexworks.cavesofzircon.attributes.types.Food
import org.hexworks.cavesofzircon.attributes.types.iconTile
import org.hexworks.cavesofzircon.extensions.GameItem
import org.hexworks.cavesofzircon.extensions.whenTypeIs
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.Fragment
import org.hexworks.zircon.api.graphics.Symbols

class InventoryRowFragment(width: Int, item: GameItem) : Fragment {

    val dropButton = Components.button()    // 1
        .withDecorations()
        .withText("Drop")
        .build()

    val eatButton = Components.button()     // 2
        .withDecorations()
        .withText("Eat")
        .build()

    override val root = Components.hbox()
        .withSpacing(1)
        .withSize(width, 1)
        .build().apply {
            addComponent(
                Components.icon()
                    .withIcon(item.iconTile)
            )
            addComponent(
                Components.label()
                    .withSize(InventoryFragment.NAME_COLUMN_WIDTH, 1)
                    .withText(item.name)
            )
            addComponent(dropButton)
            item.whenTypeIs<Food> {     // 3
                addComponent(eatButton)
            }
        }

}
```

Here we:

1. Add proper text to the drop button (we just had an arrow previously)
2. Create a button for eating
3. And add it to our row only if the item is `Food`

Then we can add support for this in `InventoryFragment`:

```kotlin
// ...

class InventoryFragment(
    inventory: Inventory,
    width: Int,
    onDrop: (GameItem) -> Unit,
    onEat: (GameItem) -> Unit                   // 1
) : Fragment {

override val root = Components.vbox()
    .withSize(width, inventory.size + 1)
    .build().apply {
        // ...
        inventory.items.forEach { item ->
            val row = InventoryRowFragment(width, item)
            addFragment(row).apply {
                // ...
                row.eatButton.onActivated {     // 2
                    detach()
                    onEat(item)
                    Processed
                }
            }
        }
    }
}
```

Here:

1. First we add a callback for `onEat`
2. And Add a listener to the `eatButton` which uses `onEat`

And finally we modify the creation of the `InventoryFragment` in `InventoryInspector`:

```kotlin
import com.example.cavesofzircon.attributes.types.EnergyUser
import com.example.cavesofzircon.attributes.types.Food
import com.example.cavesofzircon.extensions.whenTypeIs
import com.example.cavesofzircon.messages.Eat

val fragment = InventoryFragment(
    inventory = itemHolder.inventory,       // 1
    width = DIALOG_SIZE.width - 3,
    onDrop = { item ->
        CoroutineScope(Dispatchers.Single).launch {                                                 
            itemHolder.receiveMessage(DropItem(context, itemHolder, item, position))                
        }
    },
    onEat = { item ->                       // 2
        CoroutineScope(Dispatchers.Single).launch {                                                 
            itemHolder.whenTypeIs<EnergyUser> { eater ->                    // 3
                item.whenTypeIs<Food> { food ->
                    itemHolder.inventory.removeItem(food)
                    itemHolder.receiveMessage(Eat(context, eater, food))    // 4
                }
            }
        }
    })
```

Here we:

1. Start using keyword arguments for easier readability
2. Add the `onEat` callback
3. Which checks whether the `itemHolder` is also an `EnergyUser`
4. And sends the event if an `EnergyUser` tries to eat `Food`

> Using `whenTypeIs` twice makes the code a bit less readable. It is left for the reader to improve it.

Now let's check what we've produced:

![Eating](/assets/img/eating.gif)

Nice.

## Conclusion

In this session we introduced *food* and *hunger* and we also re-used the *inventory* and the `ItemDropper` to spare some code. We're really making progress in our game, but now that we have a lot of stats it would be useful to display them so the player knows how hungry they are or what attack / defense values they have. We're going to do that in the next article!

Until then go forth and *kode on*!
 
> The code of this article can be found in commit #13.
