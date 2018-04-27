---
layout: post
title:  "Going Beyond Android: Kotlin on the Frontend"
excerpt: While most developers use Kotlin on Android it is also a viable option on other platforms. In this article
          we'll look at how it works in the browser.
comments: true
---
<div id="series">
This article is part of a <a href="/2017/12/21/beyond-android-exploring-kotlin-areas-of-application.html">series</a>.
Check out the previous part <a href="/2018/01/06/going-beyond-android-kotlin-on-the-backend.html">here</a>.
</div>
<div id="tldr">
While most developers use Kotlin on Android it is also a viable option on other platforms. In this article we'll look at how it works in the browser. We'll walk through an example which is rather minimal but it comes with batteries included: testing, minification, and deployment.
</div>

While the *kotlin2js* plugin have been around for quite some time it was not considered production ready because
the javascript code it generated was measured in megabytes and it was also hard to set up testing and external javascript
dependencies. How this changed lately? Enter [Kotlin Multiplatform Projects](https://kotlinlang.org/docs/reference/multiplatform.html).


With the [advent](https://blog.jetbrains.com/kotlin/2017/11/kotlin-1-2-released/) of this new feature now it is possible
to share code between multiple components in your program and using the javascript compiler has also become much easier.
I'll talk about *Multiplatform Projects* in another article so now let's focus on getting it to work in the browser.

## Project Setup

> The source code for this article can be found in [this](https://github.com/AppCraft-Projects/kotlin-in-browser-example) repository.

We'll be using [Gradle](https://gradle.org/) for building the project. There are other build tools out there but
Gradle has the best support for Kotlin.

When creating a Kotlin project which compiles to javascript you can stick to the same project structure which you
have already got used to: `src/main/kotlin` for Kotlin files and `src/test/kotlin` for test files.

Our `build.gradle` however will need some extra configuration.

First, in the `buildscript` next to the usual `kotlin-gradle-plugin` we'll also need the `gradle-node-plugin`:

{% gist ca94c9e258326506ece63cc3dfd9a3af %}

This is a necessity because we'll delegate a lot of work to javascript libraries like [Karma](https://karma-runner.github.io/2.0/index.html)
and [Webpack](https://github.com/webpack/webpack).

> For those of you who want to **completely** get rid of javascript, I have bad news: Currently there is no way
to completely do away with it. We are still stuck with *some* js tools.

Next, instead of applying the `kotlin` plugin we'll use the `kotlin-platform-js` along with `kotlin-dce-js` and `com.moowork.node`:

{% gist 2c8fd685aad67fc1b316844296d126de %}

`kotlin-platform-js` is used in multiplatform Kotlin projects but it also works for standalone js code like in our example.
`kotlin-dce-js` is responsible for **dead code elimination** which will help us create very small js files.
`com.moowork.node` is for interfacing with *node.js*.

With all the plugins in place we'll need some dependencies:

{% gist 17c0772914cedf7dd66b3a9d73f8c98d %}

`kotlin-stdlib-js` as its name suggest contains the Kotlin stdlib for javascript projects and `kotlin-test-js` is
for testing our code.

The rest of the configuration is to wire together the `kotlin2js` compiler with the javascript world:

{% gist 7cb37f06fafabc249658b78a15ddfb0e %}

## A quick javascript tooling crash course

We're done with Gradle but we still need to set up Karma, Webpack and a `package.json` is also necessary.
"Why are these necessary?" you might ask. The answer is that if we don't want to rewrite everything in Kotlin
which is not part of our business domain (like testing tools, package management and such), wel'll need to
use those which are present within the javascript ecosystem.

Fortunately the plugins we applied above help out with these.

> Note that discussing these tools in depth is out of the scope of this article. There are links below
so you can read their documentation.

### Karma

[Karma](https://karma-runner.github.io/2.0/index.html) is a test runner tool for javascript. You can think of it as something like JUnit in the Java world.

It will pick up the `karma.conf.js` automatically when we test our project:

{% gist 069483ca6f458b78bcb44054a6925536 %}

This config will work from our source and test folders, and will use a Chrome headless browser to run our tests.

### Webpack

[Webpack](https://webpack.js.org/) is a module bundler tool which can be used to create javascript artifacts. It works in a similar way like
Shade works in Maven and Shadow in Gradle. This is not entirely accurate but this will do for now just to understand
why we need it.

We'll need two files. One for development:

{% gist 3ff4eaaca9bc8deb22d2b876f481492b %}

and another one for bundling our project:

{% gist 62db253891d51a5f78792969bf697f50 %}

### The `package.json`

The [`package.json`](https://docs.npmjs.com/files/package.json) file is essentially a way to manage locally installed [npm](https://www.npmjs.com/) packages (`npm` itself is a javascript
build tool, like Maven or Gradle).

In our project we'll need a very simple setup with only some wiring for development and testing tools:

{% gist 4beb4a887c6eaf1d4a5c038a29a74128 %}

## Are we there yet?

Yep, we pretty much wired together all the tools and plugins. I have to note here that there are many ways to set up
Kotlin frontend projects like using the [kotlin-frontend-plugin](https://github.com/Kotlin/kotlin-frontend-plugin)
or using the [gradle-js-plugin](https://github.com/eriwen/gradle-js-plugin).

The reason why I chose this setup is that this way we can exploit all the functionality these tools give us like:

- hot code replace
- browser sync
- using javascript libraries from npm
- creating bundles with Webpack
- Unit **and** functional testing including async tests

## Let's write some code

Now that we have set up the project we can finally start writing some code! Let's create a `Main.kt` with a `main`
function in our source folder:

{% gist 3a0a7a7df45cdab325fed29327dee0b4 %}

Now, if we build the project with `./gradlew assemble` we'll find an `index.html` in the `build/dist` folder.
Let's open it and check the developer console:

{% gist 58acb851ca971a23cf51d32d8ee17136 %}

**Congratulations!** You have successfully compiled your first Kotlin project to javascript!

## Testing

This is all well and good but you won't get far without the means to write proper unit tests. Luckily Kotlin provides us
with a testing library with which we can write Kotlin tests and it will be run with Karma under the hood.

Let's add something to test:

{% gist 04a0ef9f87ccf98f5129a5a940add282 %}

Then add a test for it:

{% gist 070c1e5c0f2ec66c8c310528f164b783 %}

Now if we run `./gradlew test` we'll be presented with a nice output for our test:

{% gist b3324cd54a287712b0a7977c48f995a2 %}

While this covers unit testing we're still not out of the woods yet because we also need to test our program
in its native environment: the browser.

Fortunately we can use the *DOM* and we also have the option to write asynchronous tests:

{% gist 31d7e78c72aa3c2bd3e488f684b2be51 %}

Now we have everything at our disposal to start working on **real** applications which will run in the browser!

## Conclusion

While this article is far from exhaustive we touched a lot of important points and I think that
Kotlin development in the browser is definitely doable and since the `1.20` version of Kotlin it is also production ready!
We have learned that deploying Kotlin code to the browser is not hard, and it only comes with a fixed amount of boilerplate which we only have to set up once.

If you are interested there are a lot of other examples including the *kotlin-frontend-plugin* with [extra webpack config](https://github.com/Kotlin/kotlin-frontend-plugin/tree/master/examples/custom-webpack-config)
 or a [Kotlin Full-stack example](https://github.com/Kotlin/kotlin-fullstack-sample).
 
I'd also recommend to check the [Kotlin blog](https://blog.jetbrains.com/kotlin/) and the official [JetBrains repo](https://github.com/JetBrains).

The code for this example lives in [this](https://github.com/AppCraft-Projects/kotlin-in-browser-example) repository.
Feel free to download it and fiddle around with it.

So *go forth* and Kode on!

> Keep tuned! The next article in this series will be about multiplatform development, with a full stack example!
