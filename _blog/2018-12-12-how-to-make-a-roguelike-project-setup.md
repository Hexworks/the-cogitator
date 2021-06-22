---
excerpt: This article explains how to set up the tutorial project on your computer and get started with it.
title: "How To Make a Roguelike: #1 Project Setup"
tags: [zircon, caves-of-zircon, roguelikes, kotlin]
author: addamsson
short_title: "How To Make a Roguelike: #1 Project Setup"
series: coz
comments: true
guest_post: true
guest_site: https://hexworks.org
guest_name: Hexworks
canonical_url: https://hexworks.org/posts/tutorials/2018/12/12/how-to-make-a-roguelike-project-setup.html
updated_at: 2020-12-12
---

> Now that you have decided to write your own roguelike, the first thing to do is to set up the project on your computer!
  
## What you'll need

To get started with the project you'll need to install:

- [Git](https://git-scm.com/)
- [A JDK](https://docs.aws.amazon.com/corretto/latest/corretto-11-ug/downloads-list.html) and
- [Intellij IDEA Community Edition](https://www.jetbrains.com/idea/download/)

The latter is not mandatory, but it is highly recommended.

## What we're gonna write

In this tutorial we'll write [Caves of Zircon](https://github.com/Hexworks/caves-of-zircon-tutorial).
The game is about an adventurer, who descends into the **Caves of Zircon** to gather zircon crystals
and fight zombies, kobolds and other mythical creatures along the way.

This game is a classic roguelike. This means that you only have one life, the creatures are
ruthless, and you'll even have to look out for food, since you can starve to death!

The game will look like this when we're done:

![Caves of Zircon gameplay](/assets/img/coz_gameplay.gif)

## Checking the project out

To get started, let's check out the code from Git. Navigate to an empty folder and perform
this command:

```bash
git clone https://github.com/Hexworks/caves-of-zircon-tutorial.git
```

then navigate to the newly created project:

```bash
cd caves-of-zircon-tutorial
```

This project contains the code from start to finish and for each article there is a Git *commit*.
So let's reset our project to see the initial project setup by running this command:

```bash
git reset --hard 00047e1
```

### Using Gradle

We're not going to go into details about how to use Gradle, so the only thing you have to know
is the command for building the project:

> If the CLI complains about execution permissions run this command: `chmod +x gradlew`

```bash
./gradlew clean build
```

You can run this command any time you want to build a `.jar` file from your project.
To run it you just have to call Java with your `.jar` file as a parameter:

```bash
java -jar build/libs/caves-of-zircon.jar
```

With little luck you'll see this screen:

![Hello, from Caves of Zircon](/assets/img/hello_coz.png)

> If you are curious the command above (`./gradlew clean build`) consists of 3 parts:
 - `./gradlew`: this will call the `gradlew` script that will use the embedded build tool to compile our project
 - `clean` will just delete the `build` folder. The results of a compilation go to this folder so deleting it will indeed clean up any previous builds
 - `build` as its name suggests will compile our program and create a runnable `.jar` file

## Importing the project to IDEA

Now let's import the project we just downloaded into IDEA, so we can start coding.
For this we're gonna use the *New > Project from Existing Sources* menu:

![New project from existing sources](/assets/img/project_import_00.png)

Then we select Gradle:

![Gradle select](/assets/img/project_import_01.png)

and finally we'll need to select "Use auto import" and a JVM:

![import + jvm](/assets/img/project_import_02.png)

If you have trouble setting up a JDK, then navigate to the *File > Project Structure* menu:

![project structure](/assets/img/sdk_00.png)

and add a new JDK. Use the one which you installed previously.

## Conclusion

We're all set now! We cloned the skeleton project from GitHub which contains all the setup
which is necessary to get started with our project and also imported it into our IDE.

In the next article we'll take a look at how our tools work and how we can start writing
our *actual* game.

> The code of this article can be found in commit #1.
