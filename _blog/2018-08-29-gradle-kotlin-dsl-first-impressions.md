---
excerpt: The Kotlin DSL for writing Gradle build scripts have been around for some time. In this article we'll take a look at it and see how useful it is.
title:  "Gradle Kotlin DSL - First impressions"
short_title:  "Gradle Kotlin DSL - First impressions"
tags: [kotlin, gradle]
comments: true
updated_at: 2018-08-29
---

> The Kotlin DSL for writing Gradle build scripts have been around for some time. In this article we'll take a look at it and see how useful it is.

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

```groovy
plugins {
    application
    kotlin("jvm") version "1.2.60"
}

application {
    mainClassName = "samples.HelloWorldKt"
}

dependencies {
    compile(kotlin("stdlib"))
}

repositories {
    jcenter()
}
```

Looks familiar? On the surface *kotlin-dsl* is rather similar to the old Groovy config but there are some
minor differences. Let's take a look at some of them.

### Applying built-in plugins

Anything which is in the official Gradle plugin repository can be applied with the `plugins` block:

```groovy
plugins {
    id 'java'
    id 'application'
}
```

The same with *kotlin-dsl* looks like this:

```groovy
plugins {
    java
    application
}
```

Wait, what happens here? Where does `java` and `application` come from? If we take a look at the source it becomes
clear:

```kotlin
/**
 * The builtin Gradle plugin implemented by [org.gradle.api.plugins.JavaPlugin].
 *
 * Visit the [plugin user guide](https://docs.gradle.org/current/userguide/java_plugin.html) for additional information.
 *
 * @see org.gradle.api.plugins.JavaPlugin
 */
inline val org.gradle.plugin.use.PluginDependenciesSpec.`java`: org.gradle.plugin.use.PluginDependencySpec
    get() = id("org.gradle.java")
```

This clever trick lets us configure built-in plugins in a very simple way and it is also compile-checked. Mistyped
strings won't be a problem anymore.

What if I want to use a non-built-in plugin? Well, there is still the old way:

```groovy
apply plugin: 'checkstyle'
```

and in Kotlin:

```kotlin
apply(plugin = "checkstyle")
```

### Dependencies

Declaring dependencies is a very important part of a Gradle script so let's take a look at the differences:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'io.jsonwebtoken:jjwt:0.9.0'
    runtimeOnly 'org.postgresql:postgresql'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude(module: 'junit')
    }
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine'
}
```

and with Kotlin:

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("io.jsonwebtoken:jjwt:0.9.0")
    runtimeOnly("org.postgresql:postgresql")
    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(module = "junit")
    }
    testRuntimeOnly("org.junit.jupiter:junit-jupiter-engine")
}
```

Again, very similar, but a little more readable.

### Tasks

After plugins and dependencies tasks might be the next in the list of important things in a Gradle config file.
This is simple enough in Groovy:

```groovy
task docZip(type: Zip) {
    archiveName = 'doc.zip'
    from 'doc'
}
```

but with Kotlin there is a little more magic involved:

```kotlin
val docZip by tasks.creating(Zip::class) {
    archiveName = "doc.zip"
    from("doc")
}
```

What happens here is that *kotlin-dsl* provides some delegates for us:

```kotlin
val docZip by tasks.creating(Zip::class) {
    archiveName = "doc.zip"
    from("doc")
}
```

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

```kotlin
plugins {
    `maven-publish`
}

publishing {
    publications {
        register("mavenJava", MavenPublication::class) {
            from(components["java"])
            artifact(sourcesJar.get())
        }
    }
}
```

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

```groovy
bootJar {
    archiveName = 'app.jar'
    mainClassName = 'com.ninja_squad.demo.Demo'
}

bootRun {
    main = 'com.ninja_squad.demo.Demo'
    args '--spring.profiles.active=demo'
}
```

we need to specify the class of the task:

```kotlin
tasks {
    "bootJar"(BootJar::class) {
        archiveName = "app.jar"
        mainClassName = "com.ninja_squad.demo.Demo"
    }

    "bootRun"(BootRun::class) {
        main = "com.ninja_squad.demo.Demo"
        args("--spring.profiles.active=demo")
    }
}
```

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

