---
excerpt: Kotlin releases are quite frequent nowadays and the last few ones were not so remarkable, but 1.2.60 is somewhat special. In this article I'll explain why.
title:  "Review: Kotlin 1.2.60"
short_title:  "Review: Kotlin 1.2.60"
tags: [kotlin]
comments: true
updated_at: 2018-08-08
---

> Kotlin releases are quite frequent nowadays and the last few ones were not so remarkable, but 1.2.60 is somewhat special.
In this article I'll explain why.

## The summary

In this release we get some useful things like:

- The option to build multiplatform projects with IDEA
- Experimental kapt mode to speed up Gradle builds
- Some new refactorings in IDEA
- Fixes for issues, and some performance enhancements

but there is a very interesting part in the release notes:

> Adds optional expected annotations to multiplatform projects

They detail it a little more after the listing:

> This update introduces the @OptionalExpectation annotation that is suitable for marking expect annotation class declarations in common code
> whose actual counterparts may be omitted in the platform implementations.

## Optional what?

So why is this useful? We only get a single `@OptionalExpectation` annotation after all which can only be put on
expected annotations. The answer is that they are [planning to put this](https://youtrack.jetbrains.com/issue/KT-24478)
on `@JvmStatic`, `@JvmOverloads`, and a bunch of other annotations.

For some reason they do not brag about this, but I think that this is a big deal. If you have been using multiplatform
projects for a while, you might have had to write common classes which need builders, or something similar in
platform projects. Now having the option to put these annotations on those classes will be very helpful in the future.
I'll explain why in the following example:

## The use case

Suppose that we are writing a library for our project which implements commonly used functionality like sending data over HTTP. 
We don't want to write it twice so we put as much code in the common module as possible:

```kotlin
data class HttpRequest(val method: String,
                       val url: String,
                       val headers: List<String>) {

    companion object {
        
        fun create(method: String = "GET",
                   url: String,
                   headers: List<String> = listOf()): HttpRequest {
            return HttpRequest(method, url, headers)
        }
    }
}
```

Now if we want to use that *factory method* from Java, accessing it is a bit awkward:

```java
HttpRequest.Companion.create("GET", "http://google.com", Arrays.asList());
```

and the only way to solve this is to have some kind of helper class in the JVM project:

```kotlin
object HttpRequests {

    @JvmStatic
    @JvmOverloads
    fun create(method: String = "GET",
               url: String,
               headers: List<String> = listOf()): HttpRequest {
        return HttpRequest.create(method, url, headers)
    }
}


// usage from Java

HttpRequests.create("http://google.com");
```

More boilerplate and less maintainability in a nutshell.

## Conclusion

Having the ability to put those annotations on common code solves this problem and makes our life much easier.

Moreover, it seems that JetBrains is taking multiplatform projects seriously and they are solving *real* problems which
are plaguing the lives of actual developers.

There is one more thing to consider: all these things which the Kotlin devs have been doing lately, like giving us
a uniform coroutine API which is usable on all platforms, adding more and more things to the standard library,
having optional expected annotations are hinting at something. It seems that JetBrains has the goal to make Kotlin 
platform independent, so at some point in the future, the JVM or even the browser might be an implementation detail
and standard Kotlin code will have the ability to run anywhere. I would really like to see that happen!

> The Hungarian version of this article can be read [here](http://appcraft.hu/posts/blog/2018/08/08/Kotlin-1.2.60-attekintes.html).
