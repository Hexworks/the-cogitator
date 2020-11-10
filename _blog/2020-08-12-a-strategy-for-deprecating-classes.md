---
excerpt: "If you have worked on a library before then chances are that you've tried to deprecate and remove classes form your API. In this article we'll explore how this can be done in a civilized way."
title: "A Strategy for Deprecating Classes"
tags: [kotlin]
author: addamsson
short_title: "A Strategy for Deprecating Classes"
comments: true
---

> If you have worked on a library before then chances are that you've tried to deprecate and remove classes form your API. In this article we'll explore how this can be done in a civilized way.

## Introducing a New Interface

I have been working on a library lately that has some mixins for handling common functionality. `TextHolder` is one of those interfaces:

```kotlin
interface TextHolder {

    /**
     * The (mutable) [text].
     */
    override var text: String

    /**
     * A [Property] that wraps the [text] and offers data binding and
     * observability features.
     *
     * @see Property
     */
    override val textProperty: Property<String>

}
```

After working with *mixins* for a while I realized that I need two kinds of those: one that's only exposes an accessor:

```kotlin
interface HasTheme {
    val theme: ColorTheme
}
```

and one that makes it mutable:

```kotlin
interface ThemeOverride : HasTheme {

    /**
     * The (mutable) [ColorTheme].
     */
    override var theme: ColorTheme

    /**
     * A [Property] that wraps the [theme] and offers data binding and
     * observability features.
     *
     * @see Property
     */
    override val themeProperty: Property<ColorTheme>
}
```

## The Problem

These interfaces work fine but the problem is that the users of my APIs now have to figure out all the `HasX`, `XOverride` and `XHolder` classes on their own since the nomenclature is no longer consistent.

What makes this even more confusing is that `TextHolder` holds mutable state despite not stating it clearly with its name.

> In fact this was the reason I created the `HasX` and `XOverride` convention.

Now I can go on and rename all these classes to harmonize the different mixins I have but it will break the API for the downstream users.

How can we solve this problem?

## The Solution

Let's see what would break the API. If I just renamed `TextHolder` to `TextOverride` all imports for `TextHolder` would break.

Introducing `HasText` has no effect here since it is just another interface `TextOverride` should implement. So let's create this split and see what else we need to add to keep the API intact. `HasText` will have an accessor for `text`:

```kotlin
interface HasText {

    val text: String
}
```

and `TextOverride` will hold the mutable state:

```kotlin
interface TextOverride : HasText {

    /**
     * The (mutable) [text].
     */
    override var text: String

    /**
     * A [Property] that wraps the [text] and offers data binding and
     * observability features.
     *
     * @see Property
     */
    val textProperty: Property<String>

}
```

Now the solution for the API problem becomes really simple: we should just keep `TextHolder` as-is and add a depreciation warning to it:

```kotlin
@Deprecated("TextHolder was renamed to TextOverride, TextHolder will be removed in the next release")
interface TextHolder : HasText {

    override var text: String
    val textProperty: Property<String>

}
```

and make `TextOverride` implement it:

```kotlin
interface TextOverride : TextHolder {

    /**
     * The (mutable) [text].
     */
    override var text: String

    /**
     * A [Property] that wraps the [text] and offers data binding and
     * observability features.
     *
     * @see Property
     */
    override val textProperty: Property<String>

}
```

## Conclusion

We've learned that with this strategy we can clean up our API in two steps:

1. First we deprecate an old api and leave a warning in place while retaining the type
2. Then in the next release we can delete the deprecated type

Our users will receive a warning in advance and they can also see what they should do from the deprecation message so we can safely delete these classes in the next release.

Now let's go forth and **kode on**!

