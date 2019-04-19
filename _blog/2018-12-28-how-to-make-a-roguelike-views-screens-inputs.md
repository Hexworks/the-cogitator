---
excerpt: We gonna learn how to work with Views and Screens and also how to handle inputs from the user.
title: "How To Make a Roguelike: #2 Views, Screens, Inputs"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #2 Views, Screens, Inputs"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2018/12/28/how-to-make-a-roguelike-views-screens-inputs.html
---

> Having a skeleton project is nice, but we should start doing some actual work on our project.
  Let's start by creating the basic user interface elements and handling our player's interactions.
  
## Zircon basics

> It is **highly recommended** at this point to read the [Zircon Overview](/projects/zircon) and the
  [Zircon Crash Course](/zircon/docs/2018-07-18-a-zircon-crash-course) to familiarize yourself with Zircon.
  
If you open the `main.kt` file in IDEA you'll see the following code:

```kotlin
import org.hexworks.zircon.api.ColorThemes
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.Screens
import org.hexworks.zircon.api.SwingApplications
import org.hexworks.zircon.api.component.ComponentAlignment

fun main(args: Array<String>) {

    val grid = SwingApplications.startTileGrid() // 1
    val screen = Screens.createScreenFor(grid) // 2

    screen.addComponent(Components.header()
            .withText("Hello, from Caves of Zircon!")
            .withAlignmentWithin(screen, ComponentAlignment.CENTER)) // 3

    screen.applyColorTheme(ColorThemes.arc()) // 4
    screen.display() // 5

}
```

> Note that Zircon uses the [Builder Pattern](https://en.wikipedia.org/wiki/Builder_pattern) heavily. What you see
above with the `Components.header()` call is the creation of one. With *builders* it is customary to use
[Method Chaining](https://en.wikipedia.org/wiki/Method_chaining) where each function call on a builder will
return the builder itself so you can call another method on it.

What happens here is we

1. Create a new `Application` and start rendering a `TileGrid`. A `TileGrid` is just a 2D grid which contains
   `Tile`s. In our case we'll use [CP437 Tiles](https://en.wikipedia.org/wiki/Code_page_437) because they are
   easy to work with.
2. We create a `Screen` for our grid. A `Screen` works like a `TileGrid` but it also supports adding UI components
   like buttons, text boxes, and so on. What's also important to know is that you can attach multiple `Screen`s to
   a single `TileGrid` so it is an easy way to navigate between different game screens. Note that only one `Screen`
   can be displayed (use the `display` method to do so).
   > Read more about `Screen`s [here](/zircon/docs/2018-08-18-a-primer-on-screens)
3. We add a `Component` to our `Screen` which is just a simple `Header` in our case.
4. We apply a `ColorTheme` to the `Screen`. Zircon comes with numerous color themes which you can choose from. Take a look
   at the `ColorThemes` class to see them.
5. We display the `Screen` to make it visible.

The above approach works for really simple applications but when you start to write more complex user interfaces you'll feel
the need to start using a more robust approach. Luckily Zircon comes with `View`s which implement the *view* part
of the [Model-View-Controller](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) pattern.

> TL;DR: a View is a re-usable class which holds the UI elements of your app's current screen.

## Creating our first View

Let's start using them by replacing our skeleton code with a `StartView`:

> I suggest putting all our `View`s in a single package, to keep it organized: `view`. In Kotlin all classes, interfaces and
objects go to their own file just as you would do it with Java. So if you come from the Python world keep this in mind.

```kotlin
import org.hexworks.zircon.api.ColorThemes
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.graphics.BoxType
import org.hexworks.zircon.api.mvc.base.BaseView

class StartView : BaseView() {

    override val theme = ColorThemes.arc()

    override fun onDock() {
        val msg = "Welcome to Caves of Zircon."
        val header = Components.textBox() // a text box can hold headers, paragraphs and list items
                .withContentWidth(msg.length) // the width of the content of this text box
                .addHeader(msg) // we add a header
                .addNewLine() // and a new line
                .withAlignmentWithin(screen, ComponentAlignment.CENTER) // and align it to center
                .build() // finally we build the component
        val startButton = Components.button()
                // we align the button to the bottom center of our header
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_CENTER)
                .withText("Start!") // its text is "Start!"
                .wrapSides(false) // we don't want to wrap this button with [ and ]
                .withBoxType(BoxType.SINGLE) // but we want a box around it
                .wrapWithShadow() // and some shadow
                .wrapWithBox()
                .build()
        screen.addComponent(header)
        screen.addComponent(startButton)
    }
}
```

> To create `View`s you'll need to extend the `BaseView` class. This implementation of `View`s was inspired by
  how [TornadoFX](https://edvin.gitbooks.io/tornadofx-guide/content/part1/3.%20Components.html) works.

When using `View`s we don't need to bother by setting up `Screen`s, a `View` contains one to use for displaying our
`Component`s. There are two methods which we can override: `onDock` and `onUndock`.

A `View` can be `dock`ed to an `Application` and it can also be replaced with another `View`. When a `View` is
docked `onDock` is called so it is the proper place to set up how the UI should look like. Conversely if a `View`
is undocked `onUndock` is called. In the code above we add a `Header` and a `Button` to our `View`.

"But how do I `dock` a `View` then?" you might ask. The answer is simple: the `Application` (which was created in the
skeleton code) supports this. Let's modify our `main.kt` to display our `StartView`:

```kotlin
import org.hexworks.cavesofzircon.view.StartView
import org.hexworks.zircon.api.SwingApplications

fun main(args: Array<String>) {

    val application = SwingApplications.startApplication()

    application.dock(StartView())

}
```

Let's see what we created! In IDEA you should see a green triangle on the left side of your editor:

![Start button](/assets/img/coz_2_start.png)

By clicking it you can start up your game. This is very handy during development when you don't want to constantly
rebuild your project from the command line. You should see something like this:

![StartView](/assets/img/coz_2_start_view.png)

**Congratulations!** You have created your first `View`!

## Handling user input

Zircon supports handling user input in multiple ways. Luckily it adds this support to `Component`s as well so we can
easily make the `Button` we added to our `View` interactive. Let's navigate to a new `PlayView` when the player
clicks "Start!". First we add a new `PlayView`:

```kotlin
import org.hexworks.zircon.api.ColorThemes
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.graphics.BoxType
import org.hexworks.zircon.api.mvc.base.BaseView

class PlayView : BaseView() {

    override val theme = ColorThemes.arc()

    override fun onDock() {

        val loseButton = Components.button()
                .withAlignmentWithin(screen, ComponentAlignment.LEFT_CENTER)
                .withText("Lose!")
                .wrapSides(false)
                .withBoxType(BoxType.SINGLE)
                .wrapWithShadow()
                .wrapWithBox()
                .build()

        val winButton = Components.button()
                .withAlignmentWithin(screen, ComponentAlignment.RIGHT_CENTER)
                .withText("Win!")
                .wrapSides(false)
                .withBoxType(BoxType.SINGLE)
                .wrapWithShadow()
                .wrapWithBox()
                .build()

        screen.addComponent(loseButton)
        screen.addComponent(winButton)

    }
}
```

then modify the `Button` in `StartView` to navigate to this new `View` when clicked:

```kotlin
startButton.onMouseReleased {
    replaceWith(PlayView()) // 1
    close()  // 2
}
```

What you see here is a lambda which might be a familiar concept from both Python and Java. It is the same as if
you would have written:

```kotlin
startButton.onMouseReleased ({
    replaceWith(PlayView())
    close()
})
```

but in Kotlin if the last parameter to a function is a lambda you can skip the parentheses.

What happens here is we

1. replace `StartView` with `PlayView` and
2. `close` our `StartView` since it is no longer used

It is important that you always `close` the resources which you are not gonna use again to prevent memory leaks.

> You can use the mouse to click these `Button`s or you can also use the `Tab` key to focus the next component
  and `Shift`+`Tab` to focus the previous one. You can *activate* a focused `Component` by pressing `Spacebar`.
  
Let's start our app and see what it does:

![View navigation](/assets/img/coz_2_navigation.gif)

Nice! Let's add a `WinView`:

```kotlin
import org.hexworks.zircon.api.ColorThemes
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.graphics.BoxType
import org.hexworks.zircon.api.kotlin.onMouseReleased
import org.hexworks.zircon.api.mvc.base.BaseView

class WinView : BaseView() {

    override val theme = ColorThemes.arc()

    override fun onDock() {
        val msg = "You won!"
        val header = Components.textBox()
                .withContentWidth(30)
                .addHeader(msg)
                .withAlignmentWithin(screen, ComponentAlignment.CENTER)
                .build()
        val restartButton = Components.button()
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_LEFT)
                .withText("Restart")
                .wrapSides(false)
                .wrapWithBox()
                .withBoxType(BoxType.SINGLE)
                .build()
        val exitButton = Components.button()
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_RIGHT)
                .withText("Quit")
                .wrapSides(false)
                .wrapWithBox()
                .withBoxType(BoxType.SINGLE)
                .build()

        restartButton.onMouseReleased {
            replaceWith(PlayView())
            close()
        }

        exitButton.onMouseReleased {
            System.exit(0)
        }

        screen.addComponent(header)
        screen.addComponent(restartButton)
        screen.addComponent(exitButton)
    }
}
```

and a `LoseView`:

```kotlin
import org.hexworks.zircon.api.ColorThemes
import org.hexworks.zircon.api.Components
import org.hexworks.zircon.api.component.ComponentAlignment
import org.hexworks.zircon.api.graphics.BoxType
import org.hexworks.zircon.api.kotlin.onMouseReleased
import org.hexworks.zircon.api.mvc.base.BaseView

class LoseView : BaseView() {

    override val theme = ColorThemes.arc()

    override fun onDock() {
        val msg = "Game Over"
        val header = Components.textBox()
                .withContentWidth(30)
                .addHeader(msg)
                .withAlignmentWithin(screen, ComponentAlignment.CENTER)
                .build()
        val restartButton = Components.button()
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_LEFT)
                .withText("Restart")
                .wrapSides(false)
                .wrapWithBox()
                .withBoxType(BoxType.SINGLE)
                .build()
        val exitButton = Components.button()
                .withAlignmentAround(header, ComponentAlignment.BOTTOM_RIGHT)
                .withText("Quit")
                .wrapSides(false)
                .wrapWithBox()
                .withBoxType(BoxType.SINGLE)
                .build()

        restartButton.onMouseReleased {
            replaceWith(PlayView())
            close()
        }

        exitButton.onMouseReleased {
            System.exit(0)
        }

        screen.addComponent(header)
        screen.addComponent(restartButton)
        screen.addComponent(exitButton)
    }
}
```

and wire them together with `PlayView`:

```kotlin
loseButton.onMouseReleased { 
    replaceWith(LoseView())
    close()
}

winButton.onMouseReleased {
    replaceWith(WinView())
    close()
}
```

Now you have a working game which you can both *win* and *lose*!

## Conclusion

What we achieved is rather simplistic sure, but every project has to start with setting up the basic building blocks.
We've also learned a lot about `View`s, `Component`s and how the work together to form a UI.

In the next article we'll start to get our hands real dirty: we'll add the `GameComponent` with our own `World` implementation
which will be able to display the caves which we'll generate by our own hands!

Until then go forth and *kode on*!
 
> The code of this article can be found under the `2_VIEWS_SCREENS_INPUTS` tag.

