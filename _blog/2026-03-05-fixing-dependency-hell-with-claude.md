---
excerpt: "When was the last time you had to tackle the dependency hell? What was your solution? With LLMs this just got much simpler"
title: "Fixing Dependency Hell with Claude"
tags: [java, kotlin, claude, llm]
author: addamsson
short_title: "Fixing Dependency Hell with Claude"
comments: true
updated_at: 2026-03-05
---

> When was the last time you had to tackle the dependency hell? What was your solution? With LLMs this just got much simpler
> In this article we'll look at a particular problem and a solution that is just unfeasible if you are a human.
> LLMs however will happily execute hundreds of commands without making a single mistake.

## The Problem

I was working on a project the other day that failed on startup because of a missing class. As it turns out the project had 2 dependencies:

- `splunk-library-javalogging:1.11.8` that transitively depends on `okhttp:4.12.0` at runtime scope and
- `opentelemetry-exporter-sender-okhttp:1.55.0` that depends on `okhttp-jvm:5.2.1`

Setting an explicit version can be a solution (`5.2.1` in our case) but will it be compatible with Splunk's `4.12.0`?

Traditionally you'd sift through dozens of pages of release notes and check for breaking changes, or maybe you'd just update the version,
start up the application and wait for a crash (if you're adventurous). Alternatively you can check the source code of both versions.

Regardless of what you choose to do it is going to be tedious and error prone.


## Enter Claude

I've been using LLMs for a while so my first thought was "What would Claude do?" and the answer was eye-opening:

```md
3. Check actual compatibility — we can quickly verify whether
Splunk works with OkHttp 5.x by checking if the API
surface it uses has changed:

jar tf ~/.m2/repository/com/splunk/logging/splunk-library-javalogging/1.11.8/splunk-library-javalogging-1.11.8.jar |
grep -i http

This would show us which OkHttp classes Splunk actually references,
and we can verify they exist and are compatible in 5.x.

Want me to check the compatibility, or would you prefer to
go with option 2 and align everything on 4.x?
```

I looked at this and I was wondering for a moment what it is trying to do. What's `jar tf` anyway?

```bash
❯ What does jar tf do?

● jar tf lists the contents (table of files) of a JAR archive.

  - t — "table", meaning list the entries
  - f — the next argument is the filename of the JAR

  Example:

  jar tf myapp.jar

  This prints every file path inside the JAR, e.g.:

  META-INF/MANIFEST.MF
  com/example/Main.class
  com/example/Utils.class

  It's essentially the JAR equivalent of tar tf for tar archives.
```

Oh. I've never used this before, in fact I didn't even know this was possible even though I've been using Java for 10+ years! Let's see:

```bash
❯ jar tf ~/.m2/repository/com/splunk/logging/splunk-library-javalogging/1.11.8/splunk-library-javalogging-1.11.8.jar | grep -i "HttpEvent\|okhttp\|Http"
com/splunk/logging/HttpEventCollectorLog4jAppender.class12 
com/splunk/logging/HttpEventCollectorMiddleware$IHttpSenderCallback.class
com/splunk/logging/HttpEventCollectorErrorHandler.class
com/splunk/logging/HttpEventCollectorEventInfo.class
com/splunk/logging/HttpEventCollectorSender$1.class
com/splunk/logging/HttpEventCollectorSender$4.class
com/splunk/logging/HttpEventCollectorResendMiddleware$Callback.class
com/splunk/logging/HttpEventCollectorMiddleware$IHttpSender.class
com/splunk/logging/HttpEventCollectorSender.class
com/splunk/logging/HttpEventCollectorMiddleware$HttpSenderMiddleware.class
com/splunk/logging/HttpEventCollectorLogbackAppender.class
com/splunk/logging/HttpEventCollectorErrorHandler$ErrorCallback.class
com/splunk/logging/HttpEventCollectorMiddleware.class
com/splunk/logging/HttpEventCollectorErrorHandler$ServerErrorException.class
com/splunk/logging/HttpEventCollectorSender$SendMode.class
com/splunk/logging/HttpEventCollectorResendMiddleware.class
com/splunk/logging/HttpEventCollectorLoggingHandler.class
com/splunk/logging/HttpEventCollectorSender$TimeoutSettings.class
com/splunk/logging/HttpEventCollectorSender$3.class
com/splunk/logging/HttpEventCollectorSender$2.class
```

So now we know where to look. Then it went on to create a temp dir, and extracted the class for further analysis.

```bash
jar xf ~/.m2/repository/com/splunk/logging/splunk-library-javalogging/1.11.8/splunk-library-javalogging-1.11.8.jar com/splunk/logging/HttpEventCollectorSender.class
```

After that it disassembled the class:

```bash
javap -c -p com/splunk/logging/HttpEventCollectorSender.class 2>&1 | grep -E "okhttp3|OkHttp|okhttp"
```

> 📘 Are you wondering what `javap` is? I was too:
> 
> javap is the Java class file disassembler that ships with the JDK. It takes compiled .class 
> files and prints out their structure in a human-readable format.

This shows us something like this:

```bash
private okhttp3.HttpUrl url;
private static final okhttp3.OkHttpClient httpSharedClient;
private okhttp3.OkHttpClient httpClient;
      33: putfield      #9                  // Field httpClient:Lokhttp3/OkHttpClient;
     141: invokestatic  #32                 // Method okhttp3/HttpUrl.parse:(Ljava/lang/String;)Lokhttp3/HttpUrl;
     192: invokevirtual #39                 // Method okhttp3/HttpUrl.newBuilder:()Lokhttp3/HttpUrl$Builder;
     198: invokevirtual #41                 // Method okhttp3/HttpUrl$Builder.addQueryParameter:(Ljava/lang/String;Ljava/lang/String;)Lokhttp3/HttpUrl$Builder;
     212: invokedynamic #43,  0             // InvokeDynamic #0:accept:(Lokhttp3/HttpUrl$Builder;)Ljava/util/function/BiConsumer;
     225: invokevirtual #45                 // Method okhttp3/HttpUrl$Builder.build:()Lokhttp3/HttpUrl;
     228: putfield      #46                 // Field url:Lokhttp3/HttpUrl;
     254: invokestatic  #32                 // Method okhttp3/HttpUrl.parse:(Ljava/lang/String;)Lokhttp3/HttpUrl;
     257: putfield      #46                 // Field url:Lokhttp3/HttpUrl;
       1: getfield      #9                  // Field httpClient:Lokhttp3/OkHttpClient;
      14: getfield      #9                  // Field httpClient:Lokhttp3/OkHttpClient;
      17: invokevirtual #90                 // Method okhttp3/OkHttpClient.dispatcher:()Lokhttp3/Dispatcher;
      27: invokevirtual #91                 // Method okhttp3/Dispatcher.queuedCallsCount:()I
```

Not too easy to read for humans, but for an LLM it is perfect.

After having disassembled all components it just compared all the invocations and confirmed that `4.12` was indeed compatible with `5.2`. This whole process took around `10` seconds.
Not only it knew __exactly__ what to do, it was able to execute incomprehensibly faster than any human could.


## What Did We Learn?

LLMs are opening up possiblities that were previously unfeasible because of the tedium of having to execute hundreds of operations manually or simply so error prone that no human
tried it before.

Also just by paying attention wo that Claude was _actually_ doing I learned a lot about Java, a language that I've been using for most of my programming career.
