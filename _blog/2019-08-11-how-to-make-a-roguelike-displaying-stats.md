---
excerpt: "We have a lot of information about our character, but we can't see them. Let's add some visualization for them!"
title: "How To Make a Roguelike: #14 Displaying Stats"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #14 Displaying Stats"
series: coz
comments: true
canonical_url: https://hexworks.org/posts/tutorials/2019/06/14/how-to-make-a-roguelike-displaying-stats.html
---

> We know a lot of things about our character. We know how hungry we are, what items
we have and the amount of hp we have. The problem is that we can't see these values
on the screen, so in this article we're going to add them!

## A Primer on Data Binding

In this article we're going to use a feature which comes with Zircon called *data binding*.
This might be familiar to you if you have used some fat client library, like JavaFX or SWT
but we're going to take a look at it now.

In a nutshell *data binding* is responsible for *binding* the *data* stored in variables
to other variables. For this purpose we can use **properties**:

```kotlin
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property

val myVal: Property<Int> = createPropertyFrom(1)

println(myVal.value) // 1
```

This code above creates a `Property` object with a `value` of `1`. So as we can see a `Property`
is just a *wrapper* for an actual `value`, but it adds some handy features which we can use to
perform complex data binding operations. The basis of data binding is listening to changes:

```kotlin
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property

val myVal: Property<Int> = createPropertyFrom(1)

myVal.onChange { changeEvent ->
    println("old: ${changeEvent.oldValue}, new: ${changeEvent.newValue}")
}

myVal.value = 2

// old: 1, new: 2
```

As you can see with a change listener we can do anything we want when the value of a property changes. Since Kotlin
supports [property delegation](https://kotlinlang.org/docs/reference/delegated-properties.html) we can also make
a regular **Kotlin** property out of our data binding `Property`:

```kotlin
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property

val myProp: Property<Int> = createPropertyFrom(1)

val myVal: Int by myProp.asDelegate()

myProp.value = 2

println(myVal) // 2
```

This is very useful already but there is more! We don't have to use these change listeners all the time because
`Property` comes with handy binding operations:

```kotlin
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property

val myProp: Property<Int> = createPropertyFrom(1)

val myOtherProp = createPropertyFrom(2)

myOtherProp.bind(myProp)

println(myOtherProp.value)  // 1

myProp.value = 3

println(myOtherProp.value)  // 3

myOtherProp.value = 4

println(myProp.value) // 4
```

So as you can see `bind` creates a **two-way** binding. The changes to a property are reflected in the other.
This is good, but there is more! We can create `Binding`s as well:

```kotlin
import org.hexworks.cobalt.databinding.api.binding.Binding
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.expression.concat

val firstName = createPropertyFrom("John")
val lastName = createPropertyFrom("Doe")
val fullName: Binding<String> = firstName.concat(" ").concat(lastName)

firstName.value = "Jane"

println(fullName.value) // Jane Doe
```

Here we define `fullName` as a `Binding` which is calculated by concatenating `firstName` and `lastName`
with a space between them!

We can combine these functions in any way we want and we can also bind the contents of our Zircon `Component`s
using this feature!

There are also operations defined for numbers, like `add` or `multiply`:

```kotlin
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.expression.add

val first = createPropertyFrom(1)
val second = createPropertyFrom(2)

val sum = first.add(second)

first.value = 3

println(sum.value) // 5
```

`Boolean`s:

```kotlin
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.expression.not

val first = createPropertyFrom(false)

val negation = first.not()

first.value = true

println(negation.value) // false
```

and there are conversions:

```kotlin
import org.hexworks.cobalt.databinding.api.createPropertyFrom

val num = createPropertyFrom(1)

val boolValue = createPropertyFrom(false)

boolValue.updateFrom(num) { value -> // updateFrom creates a one-way binding
    value != 0
}

println(boolValue.value) // true

num.value = 0

println(boolValue.value) // false
```

## Displaying Attributes

Now armed with this knowledge let's think about how we can easily display the `Attributes` we have.
The ones we are interested in are `CombatStats` and `EnergyLevel`. There are a lot of options to
consider here and we're going to go for the simplest: Adding a new interface named `DisplayableAttribute`:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.amethyst.api.Attribute
import org.hexworks.zircon.api.component.Component

interface DisplayableAttribute : Attribute {

    fun toComponent(width: Int): Component

}
```

What this does is that the implementors will have to write a `toComponent` function which will create
a Zircon `Component` out of an `Attribute`.

For our implementations we're going to add an extension function to `Property`:

```kotlin
// put this in PropertyExtensions.kt
package org.hexworks.cavesofzircon.extensions

import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.property.Property

fun Property<Int>.toStringProperty(): Property<String> {
    val intProp = this
    return createPropertyFrom("").apply {
        this.updateFrom(intProp) {
            it.toString()
        }
    }
}
```

The above code is a good example of augmenting existing functionality where we can't modify the source.
In our case the code for `Property` is part of Zircon, but we want to add a function which converts a
`Property<Int>` to a `Property<String>`. This function above does exactly that. This is also a testament
to the power of extension functions! Now back to our attributes...

First, we are going to modify `CombatStats` which already has the proper properties (pun intended) in
place:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.cavesofzircon.extensions.toStringProperty
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.expression.concat
import org.hexworks.cobalt.databinding.api.property.Property
import org.hexworks.zircon.api.Components

data class CombatStats(val maxHpProperty: Property<Int>,
                       val hpProperty: Property<Int> = createPropertyFrom(maxHpProperty.value),
                       val attackValueProperty: Property<Int>,
                       val defenseValueProperty: Property<Int>) : DisplayableAttribute {
    val maxHp: Int by maxHpProperty.asDelegate()
    var hp: Int by hpProperty.asDelegate()
    val attackValue: Int by attackValueProperty.asDelegate()
    val defenseValue: Int by defenseValueProperty.asDelegate()

    override fun toComponent(width: Int) = Components.vbox()
            .withSize(width, 5)                     // 1
            .build().apply {
                val hpLabel = Components.label()    // 2
                        .withSize(width, 1)
                        .build()
                val attackLabel = Components.label()
                        .withSize(width, 1)
                        .build()
                val defenseLabel = Components.label()
                        .withSize(width, 1)
                        .build()

                hpLabel.textProperty.updateFrom(createPropertyFrom("HP:  ")     // 3
                        .concat(hpProperty.toStringProperty())                  // 4
                        .concat("/").concat(maxHpProperty))

                attackLabel.textProperty.updateFrom(createPropertyFrom("Att: ")
                        .concat(attackValueProperty.toStringProperty()))

                defenseLabel.textProperty.updateFrom(createPropertyFrom("Def: ")
                        .concat(defenseValueProperty.toStringProperty()))

                addComponent(Components.textBox()       // 5
                        .withContentWidth(width)
                        .addHeader("Combat Stats"))
                addComponent(hpLabel)
                addComponent(attackLabel)
                addComponent(defenseLabel)

            }

    // ...
}
```

Here we:

1. Create a `VBox` which automtaically aligns its children from top to bottom
2. Create `Label`s for each stat we want to display (hp, attack, defense)
3. Start updating their `textProperty` from our `Attribute`s
4. Use the previously created `toStringProperty` function
5. Add all the labels to our `VBox`

`EnergyLevel` is pretty similar:

```kotlin
package org.hexworks.cavesofzircon.attributes

import org.hexworks.cavesofzircon.extensions.toStringProperty
import org.hexworks.cobalt.databinding.api.createPropertyFrom
import org.hexworks.cobalt.databinding.api.expression.concat
import org.hexworks.zircon.api.Components

class EnergyLevel(initialEnergy: Int,
                  val maxEnergy: Int) : DisplayableAttribute {

    // ...

    override fun toComponent(width: Int) = Components.vbox()
            .withSize(width, 5)                         // 1
            .build().apply {
                val hungerLabel = Components.label()    // 2
                        .withSize(width, 1)
                        .build()

                hungerLabel.textProperty.updateFrom(currentValueProperty.toStringProperty() // 3
                        .concat("/").concat(maxEnergy))

                addComponent(Components.textBox()   // 4
                        .withContentWidth(width)
                        .addHeader("Hunger"))
                addComponent(hungerLabel)

            }
}
```

We:

1. Create a `VBox`
2. Create a `Label`
3. Bind it to our current/max energy
4. And add the `Label` to the box

## Putting It Together

Now we have our *displayable properties* in place we just need to add them to our game. For this we're going
to create a `Fragment` which will contain the stats:

```kotlin
package org.hexworks.cavesofzircon.view.fragment

import org.hexworks.cavesofzircon.attributes.DisplayableAttribute
import org.hexworks.cavesofzircon.attributes.types.Player
import org.hexworks.cavesofzircon.extensions.GameEntity
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.Fragment

class PlayerStatsFragment(
        width: Int,
        player: GameEntity<Player>) : Fragment {

    override val root = Components.vbox()
            .withSize(width, 30)    // 1
            .withSpacing(1)         // 2
            .build().apply {
                addComponent(Components.header().withText("Player"))                // 3
                player.attributes.toList().filterIsInstance<DisplayableAttribute>() // 4
                        .forEach {
                            addComponent(it.toComponent(width))                     // 5
                        }
            }
}
```

Which:

1. Is composed of a `VBox`
2. Which has a spacing of `1` between its children
3. Has a `Header` with the text `"Player"`
4. And we filter all the `player`'s attributes for `DisplayableAttribute`
5. And call `toComponent` on them

As for displaying this we just add this to `PlayView`:

```kotlin
package org.hexworks.cavesofzircon.view

import org.hexworks.cavesofzircon.GameConfig
import org.hexworks.cavesofzircon.blocks.GameBlock
import org.hexworks.cavesofzircon.events.GameLogEvent
import org.hexworks.cavesofzircon.view.fragment.PlayerStatsFragment
import org.hexworks.cavesofzircon.world.Game
import org.hexworks.cavesofzircon.world.GameBuilder
import org.hexworks.cobalt.events.api.subscribe
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.GameComponents
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.data.Tile
import org.hexworks.zircon.api.extensions.onKeyboardEvent
import org.hexworks.zircon.api.game.ProjectionMode
import org.hexworks.zircon.api.mvc.base.BaseView
import org.hexworks.zircon.api.uievent.KeyboardEventType
import org.hexworks.zircon.api.uievent.Processed
import org.hexworks.zircon.internal.Zircon

class PlayView(private val game: Game = GameBuilder.defaultGame()) : BaseView() {

    // ...

    override fun onDock() {

        // ...

        val sidebar = Components.panel()
                .withSize(GameConfig.SIDEBAR_WIDTH, GameConfig.WINDOW_HEIGHT)
                .wrapWithBox()
                .build()

        sidebar.addFragment(PlayerStatsFragment(
                width = sidebar.contentSize.width,
                player = game.player))

        // ...
    }
}
```

As you can see by using `Fragment`s this is rather straightforward.

Let's see what we created:

![displaying stats](/assets/img/displaying_stats.gif)

As you can see as we get hit by the *bat* our `hp` decreases, and after eating our `energy`
is restored!

## Conclusion

We've added as stats panel with the added bonus that it auto-updates without having to write a lot
of code. We've learned that data binding is a powerful tool if used in the right place.
In the next article we're going to add some new items into our game: `weapons` and `armor`!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `14_DISPLAYING_STATS` tag.
