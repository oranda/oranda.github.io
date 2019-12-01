---
published: true
title: Akka in Your Pocket
---
Although Akka is usually thought of in connection with massively scalable applications on the server side, it is a generally useful model for concurrency that can be deployed in an ordinary PC application or even a mobile app.

My Nexus 4 phone has four cores. If I have some CPU-intensive processing in an Android app, I observe that I can speed it up nearly 300% by spreading load across four actors. This can mean the difference between one second response time and quarter-second response time, quite noticeable to the user. Therefore, along with regular Scala futures, Akka is a useful tool that Scala provides to Android developers to make the most of their users' computing resources.

One tricky bit is getting the initial configuration correct with SBT. After some experimentation, I found that these Proguard settings, adapted from an early version of [this project](https://github.com/scottaj/android-scala-example), work. (ProGuard shrinks and optimizes an application, but when you include a library like Akka, it needs to be reminded to keep certain classes using "keep options.")

```conf
      proguardOption in Android := 
      """
        |-keep class com.typesafe.**
        |-keep class akka.**
        |-keep class scala.collection.immutable.StringLike {*;}
        |-keepclasseswithmembers class * {public <init>(java.lang.String, akka.actor.ActorSystem$Settings, akka.event.EventStream, akka.actor.Scheduler, akka.actor.DynamicAccess);}
        |-keepclasseswithmembers class * {public <init>(akka.actor.ExtendedActorSystem);}
        |-keep class scala.collection.SeqLike {public protected *;}
        |
        |-keep public class * extends android.app.Application
        |-keep public class * extends android.app.Service
        |-keep public class * extends android.content.BroadcastReceiver
        |-keep public class * extends android.content.ContentProvider
        |-keep public class * extends android.view.View {public <init>(android.content.Context);
        | public <init>(android.content.Context, android.util.AttributeSet); public <init>
        | (android.content.Context, android.util.AttributeSet, int); public void set*(...);}
        |-keepclasseswithmembers class * {public <init>(android.content.Context, android.util.AttributeSet);}
        |-keepclasseswithmembers class * {public <init>(android.content.Context, android.util.AttributeSet, int);}
        |-keepclassmembers class * extends android.content.Context {public void *(android.view.View); public void *(android.view.MenuItem);}
        |-keepclassmembers class * implements android.os.Parcelable {static android.os.Parcelable$Creator CREATOR;}
        |-keepclassmembers class **.R$* {public static <fields>;}
      """.stripMargin
```

Note: this has only been tested to work with Akka 2.1.0 and Scala 2.10.1. If you use other versions, you may have do some more tinkering/googling.
