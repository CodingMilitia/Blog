---
author: johnny
date: 2018-09-12 19:45:00+01:00
layout: post
title: 'First look at Kotlin - my first pains'
summary: "A quick walk-through of a couple of problems I ran into when first trying to build a web api with Kotlin"
image: '/assets/2018/09/12/first-look-at-kotlin-post-image.jpg'
categories:
- kotlin
tags:
- kotlin
- gradle
- learning
---

[![Kotlin](/assets/2018/09/12/first-look-at-kotlin-post-image.jpg)](/assets/2018/09/12/first-look-at-kotlin-post-image.jpg)

## Intro
I've been wanting to take a look at Kotlin for a while and have been delaying, but this past weekend I had a couple of hours I had no good idea how to use them so... I finally wrote some Kotlin code (granted the result is about 30 lines long and most of them I've seen on tutorials, but still, I typed it and saw it running on my computer!).

As I didn't really do anything of much interest, this post is basically about a couple of problems I ran into while following some tutorials.

## Getting started
I already had IntelliJ installed (for along time I might add) in hopes of playing around with Kotlin, so that step was out of the way. What I did need to install was the Java JDK.

With the required software setup done, I headed on to the Kotlin website to see how to start doing something. I ended up on the docs reference [page](http://kotlinlang.org/docs/reference/) and started reading through. After the basics I lost my patience and having noticed [Ktor](https://github.com/ktorio/ktor), for building web applications, in the docs, I decided to start building a web api.

Then following the instructions on [Ktor docs for IntelliJ and Gradle](https://ktor.io/quickstart/quickstart/intellij-idea/gradle.html) I had a simple web api ready in no time, but with a couple of problems on my hands.

## The (problematic) code
So my initial code was as follows - don't use this, it has problems! (and are all on the `build.gradle` file).

{% gist 9972e7742fb05692e69ff4de060f185c %}

I had no problems when I ran the code with a right click on the `Server.kt` file, like so:
[![right click run](/assets/2018/09/12/right-click-run.jpg)](/assets/2018/09/12/right-click-run.jpg)

But as soon as I ran it using Gradle, it failed with strange errors.

### The first error
The first error was the following:
```
19:12:24: Executing task 'run'...

> Task :compileKotlin
> Task :compileJava NO-SOURCE
> Task :processResources NO-SOURCE
> Task :classes UP-TO-DATE

> Task :run FAILED
Error: Could not find or load main class ktkindergarten.Server
Caused by: java.lang.ClassNotFoundException: ktkindergarten.Server

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':run'.
> Process 'command 'C:\Program Files\Java\jdk-10.0.2\bin\java.exe'' finished with non-zero exit value 1

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 34s
2 actionable tasks: 2 executed
Process 'command 'C:\Program Files\Java\jdk-10.0.2\bin\java.exe'' finished with non-zero exit value 1
19:12:59: Task execution finished 'run'.
```

It says it can't find the entry point, which is on the `Server.kt` file, in the `ktkindergarten` package. It all seemed right to me, as I had seen in a couple of Kotlin examples, what was the problem?! 

It took me hoooooooours but finally figured it out. As you can see, I'm not really declaring a class, only the `main` function on the `Server.kt` file. What Kotlin does is generate a class with the name of the file with a `Kt` suffix. So the fix for this issue that was driving me crazy was just to go to the `build.gradle` file and change the line `mainClassName = 'ktkindergarten.Server'` to `mainClassName = 'ktkindergarten.ServerKt'`. 

(now you see why I'm posting this... so I can avoid future stupid problems of the sort 🤬)

### The second error
```
19:25:12: Executing task 'run'...

> Task :compileKotlin UP-TO-DATE
> Task :compileJava NO-SOURCE
> Task :processResources NO-SOURCE
> Task :classes UP-TO-DATE

> Task :run FAILED
Starting server...
... (removed logs for reduced noise)
Exception in thread "main" java.lang.VerifyError: Bad type on operand stack
Exception Details:
  Location:
    ktkindergarten/ServerKt$main$1$2$1.doResume(Ljava/lang/Object;Ljava/lang/Throwable;)Ljava/lang/Object; @182: getfield
  Reason:
    Type 'java/lang/Object' (current frame, stack[0]) is not assignable to 'ktkindergarten/ServerKt$main$1$2$1$doResume$$inlined$respond$1'
  Current Frame:
    bci: @182
    flags: { }
    locals: { 'ktkindergarten/ServerKt$main$1$2$1', 'java/lang/Object', 'java/lang/Throwable', 'io/ktor/util/pipeline/PipelineContext', 'kotlin/Unit', integer, 'io/ktor/application/ApplicationCall', 'ktkindergarten/SomeResponse', top, 'java/lang/Object', top, top, top, top, top, 'java/lang/Object' }
    stack: { 'java/lang/Object' }
  Bytecode:
    0000000: b800 1f3a 0f2a b400 23aa 0000 0000 0147
    0000010: 0000 0000 0000 0000 0000 0013 2c59 c600
    0000020: 04bf 572a b400 254e 2ab4 0027 3a04 2ab4
    0000030: 002b b400 30b4 0036 5959 b400 3b04 60b5
    0000040: 003b b400 3b36 052d 3a06 1906 b600 3ec0
    0000050: 0040 3a06 bb00 4259 1505 1505 04a0 0008
    0000060: 1244 a700 1cbb 0046 59b7 004a 124c b600
    0000070: 5015 05b6 0053 1255 b600 50b6 0059 b700
    0000080: 5c3a 072a c100 5e99 0023 2ac0 005e 3a09
    0000090: 1909 b600 6212 637e 9900 1219 0959 b600
    00000a0: 6212 6364 b600 67a7 000d bb00 6959 2ab7
    00000b0: 006c 3a09 1909 b400 703a 0a19 09b4 0074
    00000c0: 3a0b b800 1f3a 0c19 09b6 0075 aa00 0000
    00000d0: 0000 0076 0000 0000 0000 0001 0000 0018
    00000e0: 0000 0057 190b 59c6 0004 bf57 1906 b900
    00000f0: 7901 00b9 007f 0100 1906 1907 1909 1909
    0000100: 1906 b500 8219 0919 07b5 0085 1909 04b6
    0000110: 0086 b600 8c59 190c a600 2619 0c3a 0d57
    0000120: a700 2c19 09b4 0085 3a07 1909 b400 82c0
    0000130: 0040 3a06 190b 59c6 0004 bf57 190a 57a7
    0000140: 000d bb00 8e59 1290 b700 93bf b200 96b0
    0000150: bb00 8e59 1290 b700 93bf               
  Stackmap Table:
    full_frame(@28,{Object[#2],Object[#161],Object[#163],Top,Top,Top,Top,Top,Top,Top,Top,Top,Top,Top,Top,Object[#161]},{})
    same_locals_1_stack_item_frame(@34,Object[#163])
    full_frame(@101,{Object[#2],Object[#161],Object[#163],Object[#11],Object[#13],Integer,Object[#64],Top,Top,Top,Top,Top,Top,Top,Top,Object[#161]},{Uninitialized[#84],Uninitialized[#84],Integer})
    full_frame(@126,{Object[#2],Object[#161],Object[#163],Object[#11],Object[#13],Integer,Object[#64],Top,Top,Top,Top,Top,Top,Top,Top,Object[#161]},{Uninitialized[#84],Uninitialized[#84],Integer,Object[#165]})
    full_frame(@170,{Object[#2],Object[#161],Object[#163],Object[#11],Object[#13],Integer,Object[#64],Object[#66],Top,Top,Top,Top,Top,Top,Top,Object[#161]},{})
    full_frame(@180,{Object[#2],Object[#161],Object[#163],Object[#11],Object[#13],Integer,Object[#64],Object[#66],Top,Object[#161],Top,Top,Top,Top,Top,Object[#161]},{})
    full_frame(@228,{Object[#2],Object[#161],Object[#163],Object[#11],Object[#13],Integer,Object[#64],Object[#66],Top,Object[#161],Object[#161],Object[#163],Object[#161],Top,Top,Object[#161]},{})
    same_locals_1_stack_item_frame(@235,Object[#163])
    same_frame(@291)
    full_frame(@315,{Object[#2],Object[#161],Object[#163],Object[#11],Object[#13],Integer,Object[#64],Object[#161],Top,Object[#161],Object[#161],Object[#163],Object[#161],Top,Top,Object[#161]},{Object[#163]})
    same_locals_1_stack_item_frame(@318,Object[#161])
    full_frame(@322,{Object[#2],Object[#161],Object[#163],Object[#11],Object[#13],Integer,Object[#64],Object[#66],Top,Object[#161],Object[#161],Object[#163],Object[#161],Top,Top,Object[#161]},{})
    full_frame(@332,{Object[#2],Object[#161],Object[#163],Object[#11],Object[#13],Integer,Object[#64],Object[#161],Top,Object[#161],Object[#161],Object[#163],Object[#161],Top,Top,Object[#161]},{})
    full_frame(@336,{Object[#2],Object[#161],Object[#163],Top,Top,Top,Top,Top,Top,Top,Top,Top,Top,Top,Top,Object[#161]},{})

	at ktkindergarten.ServerKt$main$1$2.invoke(Server.kt:19)
	at ktkindergarten.ServerKt$main$1$2.invoke(Server.kt)
	at io.ktor.routing.Routing$Feature.install(Routing.kt:65)
	at io.ktor.routing.Routing$Feature.install(Routing.kt:51)
	at io.ktor.application.ApplicationFeatureKt.install(ApplicationFeature.kt:59)
	at io.ktor.routing.RoutingKt.routing(Routing.kt:93)
	at ktkindergarten.ServerKt$main$1.invoke(Server.kt:18)
	at ktkindergarten.ServerKt$main$1.invoke(Server.kt)
	at io.ktor.server.engine.ApplicationEngineEnvironmentReloading.instantiateAndConfigureApplication(ApplicationEngineEnvironmentReloading.kt:266)
	at io.ktor.server.engine.ApplicationEngineEnvironmentReloading.createApplication(ApplicationEngineEnvironmentReloading.kt:119)
	at io.ktor.server.engine.ApplicationEngineEnvironmentReloading.start(ApplicationEngineEnvironmentReloading.kt:234)
	at io.ktor.server.netty.NettyApplicationEngine.start(NettyApplicationEngine.kt:90)
	at ktkindergarten.ServerKt.main(Server.kt:26)

FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':run'.
> Process 'command 'C:\Program Files\Java\jdk-10.0.2\bin\java.exe'' finished with non-zero exit value 1

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 2s
2 actionable tasks: 1 executed, 1 up-to-date
Process 'command 'C:\Program Files\Java\jdk-10.0.2\bin\java.exe'' finished with non-zero exit value 1
19:25:15: Task execution finished 'run'.

```

The first error was bad, but I think one understands better that a class cannot be found (even for an obscure reason) than this. `java.lang.VerifyError: Bad type on operand stack` - what?!

Ok, the problem here was a version mismatch between the installed Kotlin version and the one I declared on the `build.gradle` file. So, changing the line `ext.kotlin_version = '1.2.41'` to `ext.kotlin_version = '1.2.61'` fixed it (at the time I was testing this).

## Outro
If someone gets into the same problems I did, I hope Google helps them find this post (or any other search engine for that matter), it would have saved me some hours 😛

Thanks for reading, cyaz!