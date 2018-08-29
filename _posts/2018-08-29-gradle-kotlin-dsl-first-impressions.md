---
layout: post
title:  "Gradle Kotlin DSL - First impressions"
excerpt: The Kotlin DSL for writing Gradle build scripts have been around for some time. In this article we'll take a look at it and see how useful it is.
comments: true
---
<div id="tldr">
The Kotlin DSL for writing Gradle build scripts have been around for some time. In this article we'll take a look at it and see how useful it is.
</div>

## So why do we need this?

The first question you might ask when someone adds a new supported language to their tool is why they introduced it
in the first place. I think this is a perfectly valid question in this case as well, so let's take a look at some of
the pain points with [Groovy](http://groovy-lang.org/).

If you are a Java developer and you worked with [Maven](https://maven.apache.org/) before it is probable that Gradle
will be a little weird. The first problem is that we don't just write some `xml`, but we write actual code. This
in itself can be a huge burden for someone who is used to the declarative configuration *Maven* gives us.

On top of this the default language to use with *Gradle* is *Groovy* which is a dynamic language. So Gradle not only
takes away the declarative nature of xml, but the static nature as well.

The IDE support for editing Groovy files is also a bit lacking so we don't get features like refactoring. This is
especially bad when we mistype a name somewhere and it is not visible in the ide.

Groovy never took off so finding developers who know the language is very hard, so most of us need to learn at
least a bit of Groovy along the way which leads to a lot of half-versed developers who mostly fumble when some
language issue arises.

Enter [kotlin-dsl](https://github.com/gradle/kotlin-dsl) which adds Kotlin support for Gradle. The rationale behind
this project is that we already have superb IDE support for Kotlin so let's just use it instead of Groovy and we'll
get auto-completion, refactoring, and a statically-typed language which is suited for building *DSL*s. Let's take
a look at how it works.

## A quick glance

If you check the [samples](https://github.com/gradle/kotlin-dsl/tree/master/samples) in the *kotlin-dsl* repository
you'll see some very good, but rather simplistic examples:

{% gist b6269825d3a0151ba69d6b09f0ebd820 %}

Looks familiar? On the surface *kotlin-dsl* is rather similar to the old Groovy config but there are some
minor differences. Let's take a look at some of them.

### Applying built-in plugins

Anything which is in the official Gradle plugin repository can be applied with the `plugins` block:

{% gist 63831edea1297fbaf67e84eeb2229c3b %}

The same with *kotlin-dsl* looks like this:

{% gist 70e9fceb5bec8d96057d157339ccfe62 %}

Wait, what happens here? Where does `java` and `application` come from? If we take a look at the source it becomes
clear:

{% gist 312889137af584446bf89ede080984ee %}

This clever trick lets us configure built-in plugins in a very simple way and it is also compile-checked. Mistyped
strings won't be a problem anymore.

What if I want to use a non-built-in plugin? Well, there is still the old way:

{% gist 11e44e37d1c4111606fdd6dc3acac881 %}

and in Kotlin:

{% gist c92805acf88c28ed5fa715b72753e007 %}

### Dependencies

Declaring dependencies is a very important part of a Gradle script so let's take a look at the differences:

{% gist 33a6e2e217b74dca9fc5cdd707d54a76 %}

and with Kotlin:

{% gist dfa5433bcf51e6b2ec21c4c1edcdf1ed %}

Again, very similar, but a little more readable.

### Tasks

After plugins and dependencies tasks might be the next in the list of important things in a Gradle config file.
This is simple enough in Groovy:

{% gist b024695e21e0d3f9a12afc6db7fa9a2e %}

but with Kotlin there is a little more magic involved:

{% gist be3bf8dade6e5370c8290369b76fdf1f %}

What happens here is that *kotlin-dsl* provides some delegates for us:

{% gist 260230b7cafe7ce7e845e28142bc6ad2 %}

> If you are interested in more examples, check [here](https://github.com/jnizet/gradle-kotlin-dsl-migration-guide).

So it seems that *kotlin-dsl* is the next big thing, right? We can finally refactor the configuration, we have
code completion, everything works, so we can ditch Groovy. Well, not quite.

## The downsides

While the samples in the *kotlin-dsl* project are very good and most of the articles which we can read on the internet
praise *kotlin-dsl* there are some very real downsides to it *in practice*.

### Slow IDE support

The first one I bumped into when I started using *kotlin-dsl* is the relative slow speed with which the IDE picks up
the configuration changes. I added a `buildSrc` to my multiplatform project and some `object`s to hold the versions
and the library dependencies. For some reason it took IDEA 3 minutes to recognize the changes. I tried it on 3 different
computers on 2 different platforms, but I had the same experience on all of them.

Plugin configurations are equally slow. If we add a plugin using the `plugins` block, the plugin
configuration is available in the local scope so we can configure it like in Groovy:

{% gist d15f5283e5582820c1af46ce7dc3d1cd %}

In practice the IDE support is a bit wonky. Sometimes it needs to be restarted in order for the configuration options
to be visible, but it always takes minutes.

### Lack of code examples

Gradle has a ton of Groovy samples which we can just copy-paste and customize to get things done. With *kotlin-dsl*
there are not a lot of them around and this is worse with custom plugins. Most projects have Groovy-only examples
of how to use them. I checked dozens of project pages and I am yet to find a single one which has *kotlin-dsl* examples.
This leads to the next problem:

### Custom task type problems

Sometimes when you try to use a plugin which has custom tasks you need to take a look at the source code of
the plugin to figure out the actual types it uses. For example to convert this Groovy snippet to Kotlin:

{% gist 54ac59ffdf40e88515c2b24252733fb2 %}

we need to specify the class of the task:

{% gist 4c95f0fb15f096f3bf16a3677760e8df %}

and this means digging into the source code of it.

This might not look like a big problem, but it takes time and adds up quickly.

## Is it worth it?

We've seen that *kotlin-dsl* brings some powerful tools to the table, but they are not inherently Kotlin advantages.
For example we could have refactor support for Groovy as well, but it is just not there.

On the flip side there are some big disadvantages which in practice will lead to a lot of head scratching when trying
to configure custom plugins.

Overall *kotlin-dsl* is a *very-useful* tool but it is just not mature enough. I would not recommend using it until
it gets adopted by more people. Most of its downsides come from this, and there is no reason to think that
it won't get better in time. We are just not there yet.

Of course if you are a pioneer type, or just hate Groovy then by all means go forth and use *kotlin-dsl* with impunity!






















