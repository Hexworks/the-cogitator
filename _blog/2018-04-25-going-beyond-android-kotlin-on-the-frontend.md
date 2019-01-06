---
excerpt: While most developers use Kotlin on Android it is also a viable option on other platforms. In this article
          we'll look at how it works in the browser.
title:  "Going Beyond Android: Kotlin on the Frontend"
short_title:  "Going Beyond Android: Kotlin on the Frontend"
tags: [kotlin]
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

> The source code for this article can be found in [this](https://github.com/AppCraft-Projects/kotlin-in-browser-example) repository. It is a simplified version of the code found [here](https://github.com/czyzby/kotlin-multiplatform-example).

We'll be using [Gradle](https://gradle.org/) for building the project. There are other build tools out there but
Gradle has the best support for Kotlin.

When creating a Kotlin project which compiles to javascript you can stick to the same project structure which you
have already got used to: `src/main/kotlin` for Kotlin files and `src/test/kotlin` for test files.

Our `build.gradle` however will need some extra configuration.

First, in the `buildscript` next to the usual `kotlin-gradle-plugin` we'll also need the `gradle-node-plugin`:

```groovy
buildscript {
    ext.kotlinVersion = '1.2.30'

    repositories {
        jcenter()
        mavenCentral()
        maven { url "https://plugins.gradle.org/m2/" }
    }
    dependencies {
        classpath 'com.moowork.gradle:gradle-node-plugin:1.2.0'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlinVersion"
    }
}
```

This is a necessity because we'll delegate a lot of work to javascript libraries like [Karma](https://karma-runner.github.io/2.0/index.html)
and [Webpack](https://github.com/webpack/webpack).

> For those of you who want to **completely** get rid of javascript, I have bad news: currently it is very hard to completely do away with it. We are either stuck with *some* js tools or we have to completely rewrite everything in Kotlin including tooling.

Next, instead of applying the `kotlin` plugin we'll use the `kotlin-platform-js` along with `kotlin-dce-js` and `com.moowork.node`:

```groovy
apply plugin: 'kotlin-platform-js'
apply plugin: 'kotlin-dce-js'
apply plugin: 'com.moowork.node'
```

`kotlin-platform-js` is used in multiplatform Kotlin projects but it also works for standalone js code like in our example.
`kotlin-dce-js` is responsible for **dead code elimination** which will help us create very small js files.
`com.moowork.node` is for interfacing with *node.js*.

With all the plugins in place we'll need some dependencies:

```groovy
dependencies {
    compile "org.jetbrains.kotlin:kotlin-stdlib-js:$kotlinVersion"

    testCompile "org.jetbrains.kotlin:kotlin-test-js:$kotlinVersion"
}
```

`kotlin-stdlib-js` as its name suggest contains the Kotlin stdlib for javascript projects and `kotlin-test-js` is
for testing our code.

The rest of the configuration is to wire together the `kotlin2js` compiler with the javascript world:

```groovy
node {
    download true
}

compileKotlin2Js {
    kotlinOptions {
        metaInfo = true
        sourceMap = true
        sourceMapEmbedSources = 'always'
        moduleKind = 'umd'
    }
}

compileTestKotlin2Js {
    kotlinOptions.moduleKind = 'umd'
}

// Downloads JS dependencies
task yarnInstall(type: YarnTask) {
    args = ['install']
}

// Creates minified, packed main.bundle.js at build/dist
task bundle(type: YarnTask, dependsOn: [runDceKotlinJs, yarnInstall]) {
    args = ["run", "bundle"]
    assemble.dependsOn bundle
}

// Copies files from src/main/resouces to build/dist. These resources will be served by dev server
task copyStaticResources(type: Copy) {
    from sourceSets.main.resources
    into "${buildDir}/dist"
    bundle.dependsOn copyStaticResources
}

// Extracts JS libs from included dependencies to node_modules in build directory:
task populateNodeModules(type: Copy, dependsOn: compileKotlin2Js) {
    from compileKotlin2Js.destinationDir

    configurations.testCompile.each {
        from zipTree(it.absolutePath).matching { include '*.js' }
    }

    into "${buildDir}/node_modules"
}

// Starts dev server that serves built application in dev mode
task run(type: YarnTask, dependsOn: [copyStaticResources, populateNodeModules, yarnInstall]) {
    args = ["run", "start"]
}

// Test runner
task runKarma(type: YarnTask, dependsOn: [populateNodeModules, yarnInstall]) {
    args = ['test']
    test.dependsOn runKarma
}
```

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

```javascript
module.exports = function (config) {
    config.set({
        frameworks: ['mocha', 'browserify'],
        reporters: ['mocha'],
        files: [
            'build/node_modules/*.js',
            'build/classes/kotlin/main/*.js',
            'build/classes/kotlin/test/*.js'
        ],
        exclude: [],
        colors: true,
        autoWatch: false,
        browsers: [
            'ChromeHeadlessNoSandbox'
        ],
        customLaunchers: {
            ChromeHeadlessNoSandbox: {
                base: 'ChromeHeadless',
                flags: ['--no-sandbox']
            }
        },
        captureTimeout: 5000,
        singleRun: true,
        reportSlowerThan: 500,
        preprocessors: {
            'build/**/*.js': ['browserify'],
        }
    })
};
```

This config will work from our source and test folders, and will use a Chrome headless browser to run our tests.

### Webpack

[Webpack](https://webpack.js.org/) is a module bundler tool which can be used to create javascript artifacts. It works in a similar way like
Shade works in Maven and Shadow in Gradle. This is not entirely accurate but this will do for now just to understand
why we need it.

We'll need two files. One for development:

```javascript
const webpack = require("webpack");
const HtmlWebpackPlugin = require('html-webpack-plugin');
const BrowserSyncPlugin = require('browser-sync-webpack-plugin');
const path = require("path");

const dist = path.resolve(__dirname, "build/dist");

module.exports = {
    entry: {
        main: "main"
    },
    output: {
        pathinfo: true,
        filename: "[name].bundle.js",
        path: dist,
        publicPath: ""
    },
    watch: true,
    module: {
        rules: [{
            test: /\.css$/,
            use: [
                'style-loader',
                'css-loader'
            ]
        }]
    },
    resolve: {
        modules: [
            path.resolve(__dirname, "build/node_modules/"),
            path.resolve(__dirname, "src/main/web/")
        ]
    },
    devtool: 'cheap-source-map',
    plugins: [
        new webpack.optimize.CommonsChunkPlugin({
            name: 'vendor',
            filename: 'vendor.bundle.js'
        }),
        new HtmlWebpackPlugin({
            chunks: ['vendor', 'main'],
            chunksSortMode: 'manual',
            minify: {
                removeAttributeQuotes: false,
                collapseWhitespace: false,
                html5: false,
                minifyCSS: false,
                minifyJS: false,
                minifyURLs: false,
                removeComments: false,
                removeEmptyAttributes: false
            },
            hash: false
        }),
        new BrowserSyncPlugin({
            host: 'localhost',
            port: 8080,
            server: {
                baseDir: ['./build/dist']
            }
        })
    ]
};
```

and another one for bundling our project:

```javascript
const webpack = require("webpack");
const HtmlWebpackPlugin = require('html-webpack-plugin');
const UglifyJSPlugin = require('uglifyjs-webpack-plugin');
const path = require("path");

const dist = path.resolve(__dirname, "build/dist");

module.exports = {
    entry: {
        main: "main"
    },
    output: {
        filename: "[name].bundle.js",
        path: dist,
        publicPath: ""
    },
    module: {
        rules: [{
            test: /\.css$/,
            use: [
                'style-loader',
                'css-loader'
            ]
        }]
    },
    resolve: {
        modules: [
            path.resolve(__dirname, "build/kotlin-js-min/main"),
            path.resolve(__dirname, "src/main/web/")
        ]
    },
    plugins: [
        new HtmlWebpackPlugin({
            title: 'Kotlin in browser example'
        }),
        new UglifyJSPlugin({
            sourceMap: false,
            minimize: true,
            output: {
                comments: false
            }
        })
    ]
};
```

### The `package.json`

The [`package.json`](https://docs.npmjs.com/files/package.json) file is essentially a way to manage locally installed [npm](https://www.npmjs.com/) packages (`npm` itself is a javascript
build tool, like Maven or Gradle).

> The example project uses [yarn](https://yarnpkg.com/lang/en/) which is an upgrade over npm, but yarn uses the npm package repository in the background.

In our project we'll need a very simple setup with only some wiring for development and testing tools:

```json
{
  "name": "kotlin-in-browser-example",
  "dependencies": {},
  "devDependencies": {
    "browser-sync": "2.23.6",
    "browser-sync-webpack-plugin": "2.0.1",
    "browserify": "16.1.0",
    "css-loader": "0.28.7",
    "html-webpack-plugin": "2.30.1",
    "karma": "2.0.0",
    "karma-browserify": "5.2.0",
    "karma-chrome-launcher": "2.2.0",
    "karma-mocha": "1.3.0",
    "karma-mocha-reporter": "2.2.5",
    "mocha": "5.0.0",
    "style-loader": "0.20.1",
    "watchify": "3.10.0",
    "webpack": "3.11.0"
  },
  "scripts": {
    "start": "webpack",
    "bundle": "webpack --config webpack.deploy.config.js",
    "test": "karma start"
  }
}
```

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

```kotlin
fun main(args: Array<String>) {
    println("Hello, browser!")
}
```

Now, if we build the project with `./gradlew assemble` we'll find an `index.html` in the `build/dist` folder.
Let's open it and check the developer console:

```
Hello, browser!
```

**Congratulations!** You have successfully compiled your first Kotlin project to javascript!

## Testing

This is all well and good but you won't get far without the means to write proper unit tests. Luckily Kotlin provides us
with a testing library with which we can write Kotlin tests and it will be run with Karma under the hood.

Let's add something to test:

```kotlin
class SomeClass(val foo: String) {

    fun getFoo() = foo
}
```

Then add a test for it:

```kotlin
import kotlin.test.Test
import kotlin.test.assertEquals

class SomeClassTest {

    @Test
    fun testFoo() {

        val expected = "bar"

        assertEquals(SomeClass(expected).getFoo(), expected)
    }
}
```

Now if we run `./gradlew test` we'll be presented with a nice output for our test:

```
  org.codetome.kotlinexample.browser
    SomeClassTest
      √ testFoo

Finished in 0.006 secs / 0.001 secs @ 00:15:40 GMT+0200 (Central Europe Daylight Time)

SUMMARY:
√ 1 test completed
Done in 6.78s.
:compileTestJava NO-SOURCE
:processTestResources NO-SOURCE
:testClasses
:test
```

While this covers unit testing we're still not out of the woods yet because we also need to test our program
in its native environment: the browser.

> Note that technically the unit tests also work in the browser, but they do not touch functionality provided by it like the `window` object in the following example.

Fortunately we can use the *DOM* and we also have the option to write asynchronous tests:

```kotlin
/**
 * Example of asynchronous code testing. Any time you work with promises, intervals, resource fetching or basically any
 * asynchronous code that would otherwise end the test too early or make asserts impossible, this test template might
 * be useful.
 */
class AsyncTest {
    @Test
    fun should_perform_asynchronous_test() = Promise<Unit> { resolve, _ ->
        window.setInterval({
            assertEquals(42, 42)
            // Make sure to call resolve to end the test:
            resolve(Unit)
        }, 10)
    }

    @Test
    @Ignore
    fun should_fail() = Promise<Unit> { _, reject ->
        window.setInterval({
            // You can also explicitly fail the test with an exception:
            reject(Exception("Expected!"))
        }, 10)
    }
}
```

Now we have everything at our disposal to start working on **real** applications which will run in the browser!

## Conclusion

While this article is far from exhaustive we touched a lot of important points and I think that
Kotlin development in the browser is definitely doable and since the `1.2` version of Kotlin frontend development with it is also production ready!
We have learned that deploying Kotlin code to the browser is not hard, and it only comes with a fixed amount of boilerplate which we only have to set up once.

If you are interested there are a lot of other examples including the *kotlin-frontend-plugin* with [extra webpack config](https://github.com/Kotlin/kotlin-frontend-plugin/tree/master/examples/custom-webpack-config)
 or a [Kotlin Full-stack example](https://github.com/Kotlin/kotlin-fullstack-sample).
 
I'd also recommend to check the [Kotlin blog](https://blog.jetbrains.com/kotlin/) and the official [JetBrains repo](https://github.com/JetBrains).

The code for this example lives in [this](https://github.com/AppCraft-Projects/kotlin-in-browser-example) repository.
Feel free to download it and fiddle around with it.

So *go forth* and Kode on!

> Keep tuned! The next article in this series will be about multiplatform development, with a full stack example!
