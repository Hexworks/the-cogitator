---
excerpt: "Our character can loot food, but there are no weapons nor armor in our game yet. Let's create them."
title: "How To Make a Roguelike: #15 Weapons and Armor"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #15 Weapons and Armor"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2019/06/21/how-to-make-a-roguelike-weapons-and-armor.html
updated_at: 2019-09-02
---

> Adding wearable items like weapons and armor is a staple of all roguelike games and *Caves of Zircon* is no different.
In this article we'll figure out how to add them to our game.

## Implementing Combat Items

Let's think about how we can implement *weapons* and *armor* and how they should work!
First of all, we're going to keep it simple and say that each combat item can have an *attack* and a *defense*
value which we'll just add to our base values when calculating combat results. We also make it have a *type*
which we can display to the player as additional information:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.Component

data class ItemCombatStats(val attackValue: Int = 0,
                           val defenseValue: Int = 0,
                           val combatItemType: String) : DisplayableAttribute {

    override fun toComponent(width: Int): Component {
        return Components.textBox()
                .withContentWidth(width)
                .addParagraph("Type: $combatItemType", withNewLine = false)
                .addParagraph("Attack: $attackValue", withNewLine = false)
                .addParagraph("Defense: $defenseValue", withNewLine = false)
                .build()
    }

}
```

We make this a `DisplayableAttribute` so we can display it on a details screen later.
We're also going to need `CombatItem` as a base type for *weapon* and *armor*:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

interface CombatItem : Item
```

`Armor` itself:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

import org.hexworks.amethyst.api.entity.Entity
import org.hexworks.cavesofzircon.attributes.ItemCombatStats
import org.hexworks.cavesofzircon.world.GameContext

interface Armor : CombatItem

val Entity<Armor, GameContext>.attackValue: Int
    get() = findAttribute(ItemCombatStats::class).get().attackValue

val Entity<Armor, GameContext>.defenseValue: Int
    get() = findAttribute(ItemCombatStats::class).get().defenseValue
```

and a `Weapon` type which we can use for easier implementation of
the equip mechanics:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

import org.hexworks.cavesofzircon.attributes.ItemCombatStats
import org.hexworks.cavesofzircon.extensions.GameEntity

interface Weapon : CombatItem

val GameEntity<Weapon>.attackValue: Int
    get() = findAttribute(ItemCombatStats::class).get().attackValue

val GameEntity<Weapon>.defenseValue: Int
    get() = findAttribute(ItemCombatStats::class).get().defenseValue
```

As with `InventoryHolder` we're going to add a new entity type, `EquipmentHolder`:

```kotlin
package org.hexworks.cavesofzircon.attributes.types

import org.hexworks.amethyst.api.entity.EntityType
import org.hexworks.cavesofzircon.attributes.Equipment
import org.hexworks.cavesofzircon.attributes.Inventory
import org.hexworks.cavesofzircon.extensions.GameCombatItem
import org.hexworks.cavesofzircon.extensions.GameEquipmentHolder

interface EquipmentHolder : EntityType

fun GameEquipmentHolder.equip(inventory: Inventory, item: GameCombatItem): GameCombatItem {
    return equipment.equip(inventory, item)
}

val GameEquipmentHolder.equipment: Equipment
    get() = findAttribute(Equipment::class).get()
```

and the corresponding aliases:

```kotlin
// add these to TypeAliases.kt
import org.hexworks.cavesofzircon.attributes.types.CombatItem
import org.hexworks.cavesofzircon.attributes.types.EquipmentHolder

typealias GameCombatItem = GameEntity<CombatItem>

typealias GameEquipmentHolder = GameEntity<EquipmentHolder>
```

With these in place we can implement the `Equipment` *attribute*:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.cavesofzircon.attributes.types.Armor
import org.hexworks.cavesofzircon.attributes.types.Weapon
import org.hexworks.cavesofzircon.attributes.types.attackValue
import org.hexworks.cavesofzircon.attributes.types.defenseValue
import org.hexworks.cavesofzircon.attributes.types.iconTile
import org.hexworks.cavesofzircon.extensions.GameCombatItem
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.cavesofzircon.extensions.whenTypeIs
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.Component

class Equipment(initialWeapon: GameEntity<Weapon>,
                initialArmor: GameEntity<Armor>) : DisplayableAttribute {

    private val weaponProperty: Property<GameEntity<Weapon>> = createPropertyFrom(initialWeapon)    // 1
    private val armorProperty: Property<GameEntity<Armor>> = createPropertyFrom(initialArmor)

    val attackValue: Int                                                                            // 2
        get() = weaponProperty.value.attackValue + armorProperty.value.attackValue

    val defenseValue: Int
        get() = weaponProperty.value.defenseValue + armorProperty.value.defenseValue

    val armorName: String
        get() = armorProperty.value.name

    val weaponName: String
        get() = weaponProperty.value.name

    private val weapon: GameEntity<Weapon> by weaponProperty.asDelegate()
    private val armor: GameEntity<Armor> by armorProperty.asDelegate()

    private val weaponStats: String
        get() = " A: ${weapon.attackValue} D: ${weapon.defenseValue}"

    private val armorStats: String
        get() = " A: ${armor.attackValue} D: ${armor.defenseValue}"

    fun equip(inventory: Inventory, combatItem: GameCombatItem): GameCombatItem {   // 3
        combatItem.whenTypeIs<Weapon> {
            return equipWeapon(inventory, it)
        }
        combatItem.whenTypeIs<Armor> {
            return equipArmor(inventory, it)
        }
        throw IllegalStateException("Combat item is not Weapon or Armor.")
    }

    private fun equipWeapon(inventory: Inventory, newWeapon: GameEntity<Weapon>): GameCombatItem {
        val oldWeapon = weapon
        inventory.removeItem(newWeapon)
        inventory.addItem(oldWeapon)
        weaponProperty.value = newWeapon
        return oldWeapon
    }

    private fun equipArmor(inventory: Inventory, newArmor: GameEntity<Armor>): GameCombatItem {
        val oldArmor = armor
        inventory.removeItem(newArmor)
        inventory.addItem(oldArmor)
        armorProperty.value = newArmor
        return oldArmor
    }

    override fun toComponent(width: Int): Component {       // 4
        val weaponIcon = Components.icon().withIcon(weaponProperty.value.iconTile).build()
        val weaponNameLabel = Components.label()
                .withText(weaponName)
                .withSize(width - 2, 1)
                .build()
        val weaponStatsLabel = Components.label()
                .withText(weaponStats)
                .withSize(width - 1, 1)
                .build()

        val armorIcon = Components.icon().withIcon(armorProperty.value.iconTile).build()
        val armorNameLabel = Components.label()
                .withText(armorName)
                .withSize(width - 2, 1)
                .build()
        val armorStatsLabel = Components.label()
                .withText(armorStats)
                .withSize(width - 1, 1)
                .build()

        weaponProperty.onChange {
            weaponIcon.iconProperty.value = weapon.iconTile
            weaponNameLabel.textProperty.value = weapon.name
            weaponStatsLabel.textProperty.value = weaponStats
        }

        armorProperty.onChange {
            armorIcon.iconProperty.value = armor.iconTile
            armorNameLabel.textProperty.value = armor.name
            armorStatsLabel.textProperty.value = armorStats
        }

        return Components.textBox()
                .withContentWidth(width)
                .addHeader("Weapon", withNewLine = false)
                .addInlineComponent(weaponIcon)
                .addInlineComponent(weaponNameLabel)
                .commitInlineElements()
                .addInlineComponent(weaponStatsLabel)
                .commitInlineElements()
                .addNewLine()
                .addHeader("Armor", withNewLine = false)
                .addInlineComponent(armorIcon)
                .addInlineComponent(armorNameLabel)
                .commitInlineElements()
                .addInlineComponent(armorStatsLabel)
                .commitInlineElements()
                .build()
    }

}
```

Here we:

1. Create a `Property` for both *weapon* and *armor* and initialize both with initial values
2. Add shorthands for `attackValue` and `defenseValue`. Note that both types can have attack
   and defense values to make it more interesting
3. Add a function to *equip* a *game combat item* from a given *inventory*
4. And an implementation for `toComponent`. This might seem a lot of code but we just create
   `Component`s here for displaying this `Attribute` on the sidebar, there is not much logic in there
   
## Adding Weapons and Armor   

Now that we have the mechanisms in place we just need to add some actual items to our game which
we can later find and equip. For this we're going to need some new *tiles*:

```kotlin
// add these to GameTileRepository
import org.hexworks.zircon.api.color.ANSITileColor

val CLUB = Tiles.newBuilder()
        .withCharacter('(')
        .withForegroundColor(ANSITileColor.GRAY)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()

val DAGGER = Tiles.newBuilder()
        .withCharacter('(')
        .withForegroundColor(ANSITileColor.WHITE)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()

val SWORD = Tiles.newBuilder()
        .withCharacter('(')
        .withForegroundColor(ANSITileColor.BRIGHT_WHITE)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()

val STAFF = Tiles.newBuilder()
        .withCharacter('(')
        .withForegroundColor(ANSITileColor.YELLOW)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()

val JACKET = Tiles.newBuilder()
        .withCharacter('[')
        .withForegroundColor(ANSITileColor.GRAY)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()

val LIGHT_ARMOR = Tiles.newBuilder()
        .withCharacter('[')
        .withForegroundColor(ANSITileColor.GREEN)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()

val MEDIUM_ARMOR = Tiles.newBuilder()
        .withCharacter('[')
        .withForegroundColor(ANSITileColor.WHITE)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()

val HEAVY_ARMOR = Tiles.newBuilder()
        .withCharacter('[')
        .withForegroundColor(ANSITileColor.BRIGHT_WHITE)
        .withBackgroundColor(GameColors.FLOOR_BACKGROUND)
        .buildCharacterTile()
```

*Zircon* comes with some built-in colors in the `ANSITileColor` class we can use in our projects. This above
is an example for that. For simplicity's sake we're going to use the character `(` for our weapons and `[`
for armor.

For the *tile*s above we need to create the corresponding types:

```kotlin
// add these to EntityTypes.kt

object Dagger : BaseEntityType(
        name = "Rusty Dagger",
        description = "A small, rusty dagger made of some metal alloy."), Weapon

object Sword : BaseEntityType(
        name = "Iron Sword",
        description = "A shiny sword made of iron. It is a two-hand weapon"), Weapon

object Staff : BaseEntityType(
        name = "Wooden Staff",
        description = "A wooden staff made of birch. It has seen some use"), Weapon

object LightArmor : BaseEntityType(
        name = "Leather Tunic",
        description = "A tunic made of rugged leather. It is very comfortable."), Armor

object MediumArmor : BaseEntityType(
        name = "Chainmail",
        description = "A sturdy chainmail armor made of interlocking iron chains."), Armor

object HeavyArmor : BaseEntityType(
        name = "Platemail",
        description = "A heavy and shiny platemail armor made of bronze."), Armor

object Club : BaseEntityType(
        name = "Club",
        description = "A wooden club. It doesn't give you an edge over your opponent (haha)."), Weapon

object Jacket : BaseEntityType(
        name = "Leather jacket",
        description = "Dirty and rugged jacket made of leather."), Armor
```

And we should also modify `Player` to be an `EquipmentHolder` while we're at it:

```kotlin
object Player : BaseEntityType(
        name = "player"), Combatant, ItemHolder, EnergyUser, EquipmentHolder
```

Then we can create the actual combat item *entities* in `EntityFactory`:

```kotlin
// Add these to EntityFactory
import org.hexworks.cavesofzircon.attributes.ItemCombatStats
import org.hexworks.cavesofzircon.attributes.types.Club
import org.hexworks.cavesofzircon.attributes.types.Dagger
import org.hexworks.cavesofzircon.attributes.types.HeavyArmor
import org.hexworks.cavesofzircon.attributes.types.Jacket
import org.hexworks.cavesofzircon.attributes.types.LightArmor
import org.hexworks.cavesofzircon.attributes.types.MediumArmor
import org.hexworks.cavesofzircon.attributes.types.Staff
import org.hexworks.cavesofzircon.attributes.types.Sword

fun newDagger() = newGameEntityOfType(Dagger) {
    attributes(ItemIcon(Tiles.newBuilder()
            .withName("Dagger")
            .withTileset(GraphicalTilesetResources.nethack16x16())
            .buildGraphicTile()),
            EntityPosition(),
            ItemCombatStats(
                    attackValue = 4,
                    combatItemType = "Weapon"),
            EntityTile(GameTileRepository.DAGGER))
}

fun newSword() = newGameEntityOfType(Sword) {
    attributes(ItemIcon(Tiles.newBuilder()
            .withName("Short sword")
            .withTileset(GraphicalTilesetResources.nethack16x16())
            .buildGraphicTile()),
            EntityPosition(),
            ItemCombatStats(
                    attackValue = 6,
                    combatItemType = "Weapon"),
            EntityTile(GameTileRepository.SWORD))
}

fun newStaff() = newGameEntityOfType(Staff) {
    attributes(ItemIcon(Tiles.newBuilder()
            .withName("staff")
            .withTileset(GraphicalTilesetResources.nethack16x16())
            .buildGraphicTile()),
            EntityPosition(),
            ItemCombatStats(
                    attackValue = 4,
                    defenseValue = 2,
                    combatItemType = "Weapon"),
            EntityTile(GameTileRepository.STAFF))
}

fun newLightArmor() = newGameEntityOfType(LightArmor) {
    attributes(ItemIcon(Tiles.newBuilder()
            .withName("Leather armor")
            .withTileset(GraphicalTilesetResources.nethack16x16())
            .buildGraphicTile()),
            EntityPosition(),
            ItemCombatStats(
                    defenseValue = 2,
                    combatItemType = "Armor"),
            EntityTile(GameTileRepository.LIGHT_ARMOR))
}

fun newMediumArmor() = newGameEntityOfType(MediumArmor) {
    attributes(ItemIcon(Tiles.newBuilder()
            .withName("Chain mail")
            .withTileset(GraphicalTilesetResources.nethack16x16())
            .buildGraphicTile()),
            EntityPosition(),
            ItemCombatStats(
                    defenseValue = 3,
                    combatItemType = "Armor"),
            EntityTile(GameTileRepository.MEDIUM_ARMOR))
}

fun newHeavyArmor() = newGameEntityOfType(HeavyArmor) {
    attributes(ItemIcon(Tiles.newBuilder()
            .withName("Plate mail")
            .withTileset(GraphicalTilesetResources.nethack16x16())
            .buildGraphicTile()),
            EntityPosition(),
            ItemCombatStats(
                    defenseValue = 4,
                    combatItemType = "Armor"),
            EntityTile(GameTileRepository.HEAVY_ARMOR))
}

fun newClub() = newGameEntityOfType(Club) {
    attributes(ItemCombatStats(combatItemType = "Weapon"),
            EntityTile(GameTileRepository.CLUB),
            EntityPosition(),
            ItemIcon(Tiles.newBuilder()
                    .withName("Club")
                    .withTileset(GraphicalTilesetResources.nethack16x16())
                    .buildGraphicTile()))
}

fun newJacket() = newGameEntityOfType(Jacket) {
    attributes(ItemCombatStats(combatItemType = "Armor"),
            EntityTile(GameTileRepository.JACKET),
            EntityPosition(),
            ItemIcon(Tiles.newBuilder()
                    .withName("Leather jacket")
                    .withTileset(GraphicalTilesetResources.nethack16x16())
                    .buildGraphicTile()))
}
```

Now as you might have guessed, we just add `Equipment` to our *player* `Entity`:

```kotlin
import org.hexworks.cavesofzircon.attributes.Equipment

fun newPlayer() = newGameEntityOfType(Player) {
    attributes(
            // ...
            Equipment(
                    initialWeapon = newClub(),
                    initialArmor = newJacket()))

    // ...
}
```

Then if we start up our game we'll see our initial items on the sidebar:

![Initial Combat Items](/assets/img/initial_combat_items.png)

## Augmenting The Combat System

So far so good, but we not yet use our `Equipment` in combat! Let's take a look at how we can
fix this. First of all we're going to add some extensions to simplify how we determine the
attack / defense value of an `Entity`:

```kotlin
// add these to EntityExtensions.kt

import org.hexworks.cavesofzircon.attributes.CombatStats
import org.hexworks.cavesofzircon.attributes.Equipment
import org.hexworks.cavesofzircon.attributes.ItemCombatStats

val AnyGameEntity.attackValue: Int
    get() {
        val combat = findAttribute(CombatStats::class).map { it.attackValue }.orElse(0)
        val equipment = findAttribute(Equipment::class).map { it.attackValue }.orElse(0)
        val item = findAttribute(ItemCombatStats::class).map { it.attackValue }.orElse(0)
        return combat + equipment + item
    }

val AnyGameEntity.defenseValue: Int
    get() {
        val combat = findAttribute(CombatStats::class).map { it.defenseValue }.orElse(0)
        val equipment = findAttribute(Equipment::class).map { it.defenseValue }.orElse(0)
        val item = findAttribute(ItemCombatStats::class).map { it.defenseValue }.orElse(0)
        return combat + equipment + item
    }
```

Here we augment `AnyGameEntity` to have an `attackValue` and a `defenseValue` and we try to calculate
them from the `CombatStats` which we added previously, the `Equipment` and the `ItemCombatStats`.
This will calculate this properly for any entity even if it is the *player* or a sword. The fun
part is that this system makes it possible to turn the *player* into a `Weapon` and enhance its abilities with this!

Note that if the `Entity` has none of these *attributes* the attack/defense will correctly be `0`.

Now modifying `Attackable` to take into account these values becomes trivial:

```kotlin
import org.hexworks.cavesofzircon.extensions.attackValue
import org.hexworks.cavesofzircon.extensions.defenseValue

object Attackable : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command.responseWhenCommandIs(Attack::class) {

        val (context, attacker, target) = it

        if (attacker.isPlayer || target.isPlayer) {

            val damage = Math.max(0, attacker.attackValue - target.defenseValue)
            val finalDamage = (Math.random() * damage).toInt() + 1
            target.combatStats.hp -= finalDamage

            // ...

            Consumed
        } else Pass
    }
}
```

Here instead of accessing `combatStats` we just use `attackValue` and `defenseValue` which we just created.

## Finding Items

Now we have our *weapons* and *armor*, as well as our initial items but we can't find them in the game yet!
Let's add some random items to our world. Having swords and armor laying around it not very elegant, and we're
going to change this later but for now this is going to do the trick.

For this we're going to add functions to generate random items to our `EntityFactory`:

```kotlin
import kotlin.random.Random
import org.hexworks.cavesofzircon.attributes.types.Armor
import org.hexworks.cavesofzircon.attributes.types.Weapon
import org.hexworks.cavesofzircon.extensions.GameEntity

fun newRandomWeapon(): GameEntity<Weapon> = when (Random.nextInt(3)) {
    0 -> newDagger()
    1 -> newSword()
    else -> newStaff()
}

fun newRandomArmor(): GameEntity<Armor> = when (Random.nextInt(3)) {
    0 -> newLightArmor()
    1 -> newMediumArmor()
    else -> newHeavyArmor()
}
```

With this we can modify `GameBuilder` to incorporate some items laying on the floor, but first we add some
additional configuration to `GameConfig`:

```kotlin
const val WEAPONS_PER_LEVEL = 3
const val ARMOR_PER_LEVEL = 3
```

then we can make the change in `GameBuilder`:

```kotlin
import org.hexworks.cavesofzircon.GameConfig.ARMOR_PER_LEVEL
import org.hexworks.cavesofzircon.GameConfig.WEAPONS_PER_LEVEL

fun buildGame(): Game {

    // ...
    
    addWeapons()
    addArmor()

    // ...

    return game
}

private fun addWeapons() = also {
    repeat(world.actualSize().zLength) { level ->
        repeat(WEAPONS_PER_LEVEL) {
            EntityFactory.newRandomWeapon().addToWorld(level)
        }
    }
}

private fun addArmor() = also {
    repeat(world.actualSize().zLength) { level ->
        repeat(ARMOR_PER_LEVEL) {
            EntityFactory.newRandomArmor().addToWorld(level)
        }
    }
}
```

Now when we run around in the dungeon items can be picked up:

![Finding Combat Items](/assets/img/finding_combat_items.gif)

The problem is that we can't equip them yet. Let's make the change and add an *Equip* button to our *Inventory*!

## Equipping Weapons and Armor

The *equip item* functionality will be accessible from the *Inventory* screen.
First, let's modify `InventoryRowFragment` to have this change:

```kotlin
import org.hexworks.cavesofzircon.attributes.types.CombatItem

class InventoryRowFragment(width: Int, item: GameItem) : Fragment {

    // ...
    
    val equipButton = Components.button()
            .wrapSides(false)
            .withText("Equip")
            .build()
            
   override val root = Components.hbox()
               .withSpacing(1)
               .withSize(width, 1)
               .build().apply {
                    
                   // ...
               
                   item.whenTypeIs<CombatItem> {
                       addComponent(equipButton)
                   }
               }      
}   
```

and incorporate it in `InventoryFragment`:

```kotlin
import org.hexworks.zircon.api.component.VBox
import org.hexworks.cobalt.datatypes.Maybe
import org.hexworks.cobalt.datatypes.extensions.map
import org.hexworks.cavesofzircon.GameConfig

class InventoryFragment(inventory: Inventory,
                        width: Int,
                        private val onDrop: (GameItem) -> Unit,
                        private val onEat: (GameItem) -> Unit,
                        private val onEquip: (GameItem) -> Maybe<GameItem>) : Fragment { // 1

    override val root = Components.vbox()
            .withSize(width, inventory.size + 1)
            .build().apply {
                val list = this
                addComponent(Components.hbox()
                        .withSpacing(1)
                        .withSize(width, 1)
                        .build().apply {
                            addComponent(Components.label().withText("").withSize(1, 1))
                            addComponent(Components.header().withText("Name").withSize(NAME_COLUMN_WIDTH, 1))
                            addComponent(Components.header().withText("Actions").withSize(ACTIONS_COLUMN_WIDTH, 1))
                        })
                inventory.items.forEach { item ->
                    addRow(width, item, list)       // 2
                }
            }

    private fun addRow(width: Int, item: GameItem, list: VBox) {
        list.addFragment(InventoryRowFragment(width, item).apply {
            dropButton.onComponentEvent(ACTIVATED) {
                list.removeComponent(this.root)
                onDrop(item)
                Processed
            }
            eatButton.onComponentEvent(ACTIVATED) {
                list.removeComponent(this.root)
                onEat(item)
                Processed
            }
            equipButton.onComponentEvent(ACTIVATED) {
                onEquip(item).map { oldItem ->          // 3
                    list.removeComponent(this.root)
                    addRow(width, oldItem, list)
                }
                Processed
            }
        })
        list.applyColorTheme(GameConfig.THEME)
    }

    // ...
}
```

Here we:

1. Add a callback, `onEquip` to our class which takes a `GameItem` and returns a `Maybe` of `GameItem`.
   Why a `Maybe`? Because if we fail to equip an item (becuase it is not a combat item for example) there
   is no previously equipped item to return!
2. Create an `addRow` function which we use to add a row to our *inventory*. This is important because we'll
   call this again when a new item is equipped to put the old item into the list.
3. What we do when the *equip* button is clicked it that if the equip is successful (it returned an old item)
   we put it back to the list.


Then we modify `InventoryInspector` to swap the *equipment* when this happens:

```kotlin
import org.hexworks.cavesofzircon.attributes.types.CombatItem
import org.hexworks.cavesofzircon.attributes.types.EquipmentHolder
import org.hexworks.cavesofzircon.attributes.types.equip
import org.hexworks.cobalt.datatypes.Maybe
import org.hexworks.cavesofzircon.extensions.GameItem

object InventoryInspector : BaseFacet<GameContext>() {

    override fun executeCommand(command: GameCommand<out EntityType>) = command
            .responseWhenCommandIs(InspectInventory::class) { (context, itemHolder, position) ->

                // ...
                
                val fragment = InventoryFragment(
                        // ...
                        onEquip = { item ->
                            var result = Maybe.empty<GameItem>()
                            itemHolder.whenTypeIs<EquipmentHolder> { equipmentHolder -> // 1
                                item.whenTypeIs<CombatItem> { combatItem ->             // 2
                                    result = Maybe.of(equipmentHolder.equip(itemHolder.inventory, combatItem))  // 3
                                }
                            }
                            result  // 4
                        })
                // ...
            }
}
```

What happens here is that:

1. If the `itemHolder` is also an `EquipmentHolder`
2. And the `item` is a `CombatItem`
3. We equip it
4. Otherwise we just return an empty `Maybe`

Now if we start this up and take a look around we'll see the whole thing coming together:

![Equipping Items](/assets/img/equipping_items.gif)

## Conclusion

In this article we've added *weapons* and *armor* and also a simple *equipment* system in one
fell swoop. Now we can run around our dungeon and find actual items which we can use!

In the next article we're going to add a new type of monster which is agressive and attacks us
if it sees us! We'll also modify the code we have for *weapons* and *armor* so that they won't be
just lying around...we'll loot them from monsters!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `15_WEAPONS_AND_ARMOR` tag.
