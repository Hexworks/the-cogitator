---
excerpt: "Let's take a deep dive into how one can write HTML templates in Kotlin using functional programming."
title: "Functional Templating with Kotlin"
tags: [kotlin, functional-programming]
author: addamsson
short_title: "Functional Templating with Kotlin"
series: functional-templating
comments: true
published: false
---

> If you have used Kotlin on the web, chances are that you've tried generating HTML templates at some point. Let's take a deep dive into how this can be achieved with little to no library support using Functional Programming techniques.

## A kotlinx.html Crash Course

Before we get started let's take a look at what `kotlinx.html` is. In short this is a multiplatform library that can be used to generate html content. It also comes with a nice builder DSL to boot:

```kotlin
html {
    head {
        title(title)
    }
    body {
        div {
            +"content"
        }
    }
}
```

By looking at this code it becomes immediately apparent why `kotlinx.html` is more powerful than almost *any* templating library: it is all code. This means that there is no obscure syntax to learn and we don't have to install IDE plugins either. Syntax errors are also easily discovered since the Kotlin compiler will complain if we mess something up.

We can bring the power of Kotlin to bear to create a template using the constructs of the language itself including conditions, loops, and so on:

```kotlin
div {
    if (persons.isEmpty()) {
        p { +"There are no persons." }
    } else {
        ul {
            persons.forEach { person ->
                li {
                    +"${person.name}(${person.age})"
                }
            }
        }
    }
}
```

In order to generate the actual HTML content we just need to create a `TagConsumer` object out of an `Appendable` object (a `StringBuilder` for example). Within **the context** of a `TagConsumer` we can access the builder DSL and start building HTML:

```kotlin
fun main() {
    val sb = StringBuilder()
    val tagConsumer = sb.appendHTML()

    with(tagConsumer) {
        html {
            body {
                div {
                    if (persons.isEmpty()) {
                        p { +"There are no persons." }
                    } else {
                        ul {
                            persons.forEach { person ->
                                li {
                                    +"${person.name}(${person.age})"
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    println(sb.toString())
}
```

This example will output

```
<html>
  <body>
    <div>
      <p>There are no persons.</p>
    </div>
  </body>
</html>
```

as expected.

The cherry on top is that the code you see above can be written in a (multiplatform) common module to share it between the frontend and the backend. The things we are lacking are the templating features. `kotlinx.html` is just a DSL for generating HTML after all. Let's see how one would go about implementing them on top of this library.

## The Naive Implementation

Most templating libraries work in a way that we devise a data model, that is later bound to a template using special syntax:

```xml
<table>
  <tbody>
    <tr th:each="prod: ${allProducts}">
      <td th:text="${prod.name}">Oranges</td>
      <td th:text="${#numbers.formatDecimal(prod.price, 1, 2)}">0.99</td>
    </tr>
  </tbody>
</table>
```

In the example above we have some HTML markup and some syntax that refers to values stored in variables.

### Templates

In order to implement something like this all we need to do is to wrap the binding part in a class, provide a builder function and we can call it a day:

```kotlin
fun <T> content(builder: TagConsumer<*>.(T) -> Unit) = builder

abstract class Template<T> {

    abstract val root: TagConsumer<*>.(T) -> Unit // user overrides this

    fun render(data: T): String {
        val sb = StringBuilder()
        val tagConsumer = sb.appendHTML()
        root(tagConsumer, data) // we bind and render here
        return sb.toString() // and return the rendered HTML
    }
}
```

With this, implementing `Homepage` is rather straightforward:

```kotlin
class Homepage : Template<String>() {
    override val root = content<String> { title ->
        html {
            head {
                title(title)
            }
            body {
                div {
                    +"content"
                }
            }
        }
    }
}
```

So far so good, right?

### Partials

The next thing that usually comes up is partials or fragments: pieces of markup that you can include in multiple templates. A good example of this is the `head` block in a HTML document. Luckily for us this is easy to add to our model. We just create a common superclass for `Template` and `Partial`:

```kotlin
interface Includable<T> {

    val root: TagConsumer<*>.(T) -> Unit

    fun render(data: T): String {
        val sb = StringBuilder()
        val tagConsumer = sb.appendHTML()
        root(tagConsumer, data)
        return sb.toString()
    }
}
```

then we can implement both of them in a simple way:

```kotlin
abstract class Partial<T> : Includable<T>

abstract class Template<T> : Includable<T> {

    fun <U> TagConsumer<*>.include(includable: Includable<U>, data: U) {
        includable.root(this, data)
    }
}
```

We can now create `Head`:

```kotlin
object Head : Partial<String>() {
    override val root = content<String> { title ->
        head {
            title(title)
        }
    }
}
```

and include it in `Homepage` like this:

```kotlin
class Homepage : Template<String>() {
    override val root = content<String> {
        html {
            include(Head, "title")
            body {
                div {
                    +"content"
                }
            }
        }
    }
}
```

Well done, we have a template engine!

### Layouts

Now, that we can do what *Freemarker* does let's add the killer feature: **layouts**. A layout is just like a template but it has a *placeholder* in it that will hold some textual content (the contents of a template). This is a great feature for some libraries that support this because this way we don't have to always do those `include`s for the header and the footer and we can also switch wrappers for templates in a split second.

Let's see how can we implement this feature. This is how a `Layout` looks like:

```kotlin
abstract class Layout<T> { // A Layout is not Includable

    // we ditched root for a builder function that takes a template builder
    abstract fun <U> content(
        builder: TagConsumer<*>.(U) -> Unit,
        data: Pair<T, U>
    ): TagConsumer<*>.(T) -> Unit 

    // we still need to include stuff in our layout
    fun <U> TagConsumer<*>.include(includable: Includable<U>, data: U) {
        includable.root(this, data)
    }

    fun <U> render(
        consumer: TagConsumer<*>,
        template: TagConsumer<*>.(U) -> Unit,
        data: Pair<T, U>
    ) {
        // we create the layout builder, then we render it with our consumer
        content(template, data)(consumer, data.first)
    }

}
```

Now, this is a bit more complex than the code we've written before. First of all, a `Layout` is not `Includable` but it still retains the ability to *include* things. Rendering is a two-part operation: first, we create a *builder* function that is very similar to the `root` in a template using `content(template, data)`, then we do the actual rendering by calling it with the "regular" `TagConsumer`, just like we do with templates.

This ensures that both the `Layout` and its `Template` gets rendered.

Now let's see how we have to modify `Template` to incorporate this new logic:

```kotlin
abstract class Template<L, T>(
    private val layout: Layout<L> // the layout is a constructor parameter
) : Includable<Pair<L, T>> {

    // ...

    override fun render(data: Pair<L, T>): String {
        val sb = StringBuilder()
        val consumer = sb.appendHTML()

        layout.render(consumer, content { // we render the layout and pass the template builder
            root(consumer, data)
        }, data)
        
        return sb.toString()
    }
}
```


A layout is usually specified in a template by referring to its name. In our small library we'll pass the layout as a parameter to the template's constructor.

What changed is now the `render` function renders the layout and passes a template renderer as a parameter to it. We also need to apply a little hack using `content{}`. This is necessary because the layout only needs a template builder that accepts the template's parameter, not both parameters.

> If you have a clever idea for solving this hack feel free to tell me on [Discord](https://discord.com/invite/vSNgvBh).

Now our work is done. We have a fully functional multiplatform template engine with just a few classes, right?

## Going Functional

Now please think for a while and tell me when did you start to *cringe* when you were reading the code? I'd say it was probably around the time we started to implement layouts.

So what's the problem with all this? It solves our problem after all. As so many programmers out there I also enjoy writing witty code, but it is not always the solution. In fact unnecessary complexity can make our code hard to understand for other programmers.

First of all as you might have noticed this implementation became harder and harder to extend as we were writing it. It is *not scalable* in terms of development effort that is needed to add new features.

It was also *built* on top of assumptions that were based on things we've seen before: Freemarker, Thymeleaf, Jekyll, etc.

Last but not least we also assumed that classes are *necessary* to implement something like this. We introduced some abstractions that turned out to be a bit leaky. Where does `include` go? Who `render`s?

What if there is a simpler way to do all of this?

### Innovating Upon Our Design

There are some implementation details in our code that stand out in retrospect:

- Differentiating between a `Template` and a `Partial` is not necessary. They do the same, and the only difference is that a `Template` can `include` other `Includable`s. In fact `Partial` is redundant, we only care about how to *render* HTML and how to *bind* parameters.
- Using `Layout`s this way is *cringeworthy*. They add an unnatural twist to an otherwise simple rendering logic. Can they be implemented in another way?
- Using classes is the default for most programmers, but do we really **need** them? Do they add value here?

Now let's try to implement all of this by using some functional programming and see if it is better in any way.

### The Functional Approach

Before we take a look at the implementation let's think about *what* a template really is. For this let's explore a related concept: the **template method**.

> The template method is a method that defines the skeleton of an operation in terms of a number of high-level steps. [...] The intent of the template method is to define the overall structure of the operation, while allowing subclasses to refine, or redefine, certain steps.
> 
> -- Wikipedia

In our case the skeleton is the HTML structure we create and the redefinition of steps part is when we *bind* our data to add dynamism to it. One significant difference is that we need to have templates that can be nested into other templates.

> Note that the solution presented below is not purely functional since we mutate (append) to the `TagConsumer`. This approach was chosen because it was simpler to adapt the code to what `kotlinx.html` provides.

Luckily for us the `TagConsumer<*>` already takes care of that as it streams the tags we add into the underlying `Appendable` and the builder DSL supports this nested structure. From this point a *template* is nothing more than applying some *data* with a *function* to a `TagConsumer`:

```kotlin
fun <T> template(                                 // 1
    builder: TagConsumer<*>.(data: T) -> Unit     // 2
): TagConsumer<*>.(data: T) -> Unit = { data ->   // 3
    builder(data)                                    
}
```

> Note that this structure is called a [Higher-Order Function](https://en.wikipedia.org/wiki/Higher-order_function). A HOF is a function that either takes a function as a parameter or returns one. They are very common in Functional Programming.

So what happens here is that we:

1. Define the type of the data we want to bind
2. Take a function that defines how to apply that data in the context of a `TagConsumer`
3. And return a function that creates said binding

Now this piece of code might look a bit overwhelming so let's clean it up.

```kotlin
typealias TemplateBuilder<T> = TagConsumer<*>.(data: T) -> Unit

fun <T> template(
    builder: TemplateBuilder<T>
): TemplateBuilder<T> = { data ->
    builder(data)
}
```

Although we don't use classes here we can still create `typealias`es. They are very useful when we want to convey the meaning of an otherwise complicated structure.

In order to make this work we also need a function that sets up the appender for us:

```kotlin
typealias TemplateRenderer = TagConsumer<*>.() -> Unit

fun buildHtml(renderer: TemplateRenderer) = StringBuilder().apply {
    appendHTML().renderer()
}.toString()
```

All this does is that it takes a function that renders HTML in context of a `TagConsumer<*>` and calls it.

Now let's define the actual template and see how it works:

```kotlin
val heading = template<String> { title ->
    head {
        title(title)
    }
}

val homepage = template<String> { title ->
    html {
        heading(title)
        body {
            +"hello"
        }
    }
}

fun main() {
    val html = buildHtml {
        homepage("title")
    }
    println(html)
}
```

This will output:

```html
<html>
  <head>
    <title>title</title>
  </head>
  <body>hello</body>
</html>
```

So why does this work? Since everything we do is defined in the context of a `TagConsumer<*>` that can be used to stream HTML tags into an `Appendable` all we did is to augment it with the option to bind data to said tags in a concise manner.

Now what's left is the layouts. Let's see how we can make them work.

### Layouts Revisited

First of all, let's define how a layout differs from a template. It is similar to a *template* because it also binds its own parameters to a HTML structure, but it has an additional parameter: a *template* that should be added to the resulting HTML at a certain point.

So a *layout* is a *template* that serves an additional purpose: it **binds templates to templates**. Let's see how we can express this in code:

```kotlin
typealias LayoutBuilder<T> = TagConsumer<*>.(data: T, renderer: TemplateRenderer) -> Unit

fun <T> layout(
    builder: LayoutBuilder<T>
): LayoutBuilder<T> = { data, renderer ->
    builder(data, renderer)
}
```

So how does this work? First of all, we define that a `LayoutBuilder` is a function that *binds* some *data* and a *template* with itself (also a template) in the context of a `TagConsumer`.

Then we return a function that when executed creates this binding.

This looks pretty similar to the `template` function we wrote before, but why does it take a `TemplateRenderer` instead of a `TemplateBuilder`? The answer is simple: it has no business of binding the template's parameters. What it takes is a template that already has its parameters bound. This is how a *layout* looks like in practice:

```kotlin
val defaultLayout = layout<String> { title, template ->
    html {
        head(title)
        body {
            div {
                template()
            }
        }
    }
}
```

Within the `LayoutBuilder` we have access to our data (the title) and we're free to bind it any way we see fit. The *template* is just a function that will render itself whenever we call it, that's why we don't need a placeholder at all.

This is how we use the *layout* itself:

```kotlin
val items = listOf(
    Item("one"), Item("two"), Item("three")
)

defaultLayout("title") {
    homepage(items)
}
```

> This is also how [React](/tags/react) components work.    

This *layout* takes a parameter, and a funcion that can be called to render its *template*. The binding of the template's parameters happens within that function (`homepage(items)`). By reversing the ownership of the layout we ended up with a very simple mechanism that is easy to extend and use. No hacks or modifications were necessary in order to implement layouts because it all builds on a common element, the `TagConsumer`. As it seems the old adage still holds true:

> "It is better to have 100 functions operate on one data structure than 10 functions on 10 data structures."
> 
> -- Alan Perlis

## Conclusion

What we've learned is that thinking outside of the box often helps with getting better results. It is also dangerous to assume things based on prior experience. We might not need classes or a design that is often used elsewhere.

Designing a solution bottom-up from first principles using a functional approach also leads to very simple and easy to understand code.

In the next article on *functional templating* we'll learn how to incorporate the temporal evolution of our templates into our design: how to re-render them when our *data* changes. Until then...

Let's go forth and **kode on**!



