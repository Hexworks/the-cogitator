---
excerpt: "Many programmers have lots of types and functions in a single file. I tend to have only one file for each top-level component, and here's why."
title: "One Type, One File"
tags: [productivity, vscode, typescript]
author: addamsson
short_title: "One Type, One File"
comments: true
updated_at: 2022-03-24
draft: true
---

> This post is about why I usually only have one type exported from a file, and why it is a good idea to do this.

I've noticed that in the *node* world it is common to have a single file that contains all the types and functions for a module bundled together. As someone who comes from
the Java world I always found this a bit strange, but I understand its merits such as [Locality of Behavior](https://htmx.org/essays/locality-of-behaviour/). I still think
though, that the cons outweigh the pros of doing this. Let's explore why.


## The Pros

The first thing that comes into mind is that if you have all related code in the same file, you can easily see what is going on. You don't have to jump around to different
files or use the search functionality of your IDE to find what you are looking for.

Bundling all related code into a module file also means that the code is self-contained and if you want to move it to another project you can just copy the file and you are
pretty much done.

Finding the module you're looking for is also relatively simple as you probably have a descriptive name for it in your project. 

There are limits to what you can achieve this without it becoming a mess though.


## The Cons

Once you reach a threshold you'll probably start to notice that the file is getting a bit too big. Once you have to scroll through pages of code or having to use the
"outline" functionality of your IDE to find what you are looking for, you have probably reached that threshold.

Another problem is that you might have to import the whole module even though you only need a small part of it. This can lead to a lot of unused code being imported into
another part of your application which will also increase the size of your bundle.

Also, if you get into the habit of dumping all related code together you might end up violating the [Single-responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle).
First you just have a data type. Then you put a few utility functions in there that work with that type. Then you add an *entity*, later a database *repository* and so on.
It is very easy to end up with a bloated mess that doesn't have the proper abstractions in place.

A corollary to this is that you might end up with code that is not only hard to maintain, but also hard to test.

One day you might wake up and realize that you need some of the functionality elsewhere, but your code is now tightly coupled to the rest of the module and you have to refactor
a lot. If you're in a hurry you'll just end up copy-pasting the relevant part. This is a sure way to end up with a lot of duplicated code that could have been avoided.


## But I like it!

I'm not saying that one should never do this. I'm just saying that we should be aware of the pitfalls and try to avoid them. If you are working on a small project that you
know will never grow beyond a certain size, then go ahead and do it. If you are working on a project that you know will grow, then you should probably think twice before
doing it.

Also, there are a few non-obvious advantages of trying to focus on having a single exported type or class per file. Let's explore them.


### Searchability

I've noticed this the first time I started using [Visual Studio Code](https://code.visualstudio.com/). I was used to searching for a specific class by its name, and in
VS Code this is a bit different. If you press `<Ctrl-P>` you can search for a file by its name, but if you're looking for a type there is no explicit way to do this.
You can of course use the text search function, but this will also return all the files where your type is used.

By having the file named after the type or class it contains this becomes trivial:

![File search in VS Code](/assets/img/file-search.png)

This alone is a huge productivity boost for me, especially if you have a good *naming convention* in place.


### Separation of Concerns

By having to think through 


### Testability




## Considerations


### Don't Take It Too Far


### Using Folders


### Putting Tests Next to the Code


### Using a Naming Convention
 
## Conclusion



Until then, let's go forth and **kode on**!