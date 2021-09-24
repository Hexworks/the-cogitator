---
excerpt: "This article series is about my quest for building a content aggregation service."
title: "Building a Content Gateway"
tags: [typescript]
author: addamsson
short_title: "Building a Content Gateway"
series: content-gateway
comments: true
updated_at: 2021-09-19
---

> This is the first article in a series where I'm going to share my thought process as I'm working my way through implementing a Content Gateway.

I've recenetly joined [Bankless DAO](https://www.bankless.community/) and as I was sifting through all of the dev-related things on the Discord channel I quickly bumped into a project proposal. It was called "Content Gateway".

I became intrigued as it sounded something I worked on in the past. As it turns out it is indeed a very similar idea!


## Rationale

An average company uses at least 3-4 different services for their different needs. They might have a website, managed by a CMS, a Jira installation that deals with their projects and maybe a Confluence page for internal documentation. Everything is more or less integrated, Jira can talk to Confluence, Confluence might get synchronized to the site, so data is more or less easy to access.

A DAO on the other hand is inherently *decentralized* and *autonomous*. There are no managers telling you what to do and not many company guidelines telling you what to use. This usually leads to a bazillion of different tools used by different parts of the organization. So how can you figure out answers to your many questions like "Who should I talk to about matter `X`?

In our case we have some [Notion pages](https://www.notion.so/Bounty-Board-ca221c5feab847a6b2793ce5dcf93289) where you might be able to find this information. Then you manually have to look at every document that might be interesting, slowly finding bits and pieces of information that might be useful. Then you have *Substack*, *Snapshot*, *Coordinape* and probably many other utilities that you're using as a DAO. Not very efficient, right?

Enter the Content Gateway.


## Integrated Data Infrastructure

A *Content Gateway* is a service that aggregates all this information from all *downstream systems*; sytems that participate in creating a place where data can be efficiently accessed. In short *Content Gateway* is *Integrated Data Infrastructure*.

This might sound a bit intimidating at first since many of you probably heard about things like Big Data, ETL (Extract, Transform, Load), Stream Processing and so on.

Don't worry, we're going to start with a very simple architecture that only contains abstractions and we'll defer most decisions to the last possible moment, when we have the most information about the task at hand.

The method we're going to use will also enable us to mix and match *concrete implementations* behind the abstractions enabling us to gradually make our system more scalable and feature-rich. This approach to architecure is called [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html).

Another guideline we're going to follow is [SOLID](https://en.wikipedia.org/wiki/SOLID) that will help us create easy-to-maintain code and [Occam's Razor](https://en.wikipedia.org/wiki/Occam%27s_razor) that will enable us to keep a *pragmatic* approach to problem solving.


## Conclusion

In the following articles we'll work our way through all the problems that come up with designing a *Content Gateway* and we'll also explore the solutions to them.


Until then, go forth and **kode on**!