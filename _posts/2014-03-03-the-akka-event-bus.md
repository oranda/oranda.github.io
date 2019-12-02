---
published: true
title: The Akka Event Bus
---
Event buses are useful for keeping components in a system loosely coupled, and we generally see them in two areas: distributed systems, and UIs. This post is about the use of an event bus in a UI, specifically Android.

There exist event bus libraries for Android like [Otto](https://github.com/square/otto) and [greenrobot](https://github.com/greenrobot/EventBus). There also exists an [Event Bus API for Akka](http://doc.akka.io/docs/akka/snapshot/java/event-bus.html). What I haven't seen yet is the Akka Event Bus set up on Android. So, here it is!

First a word on the motivation. In the [Libanius Android](https://github.com/oranda/libanius-android) application, I recently separated part of a screen (in Android's parlance, an Activity) into its own separate class. A problem was that, in response to certain events, the new class needed to add data back to the data structure in the main class for the Activity. My initial approach was to define a closure that could do the work of adding the data, and pass the closure as an argument to the new class (the sub-Activity). However, this felt a bit ad hoc, so I decided it was time to introduce a general event-handling mechanism into Libanius Android. Having helped create an event bus for the GWT UI framework before, I thought it would be interesting to see if the same idea could be applied in Android.

The Libanius Event Bus just extends Akka's `ActorEventBus`. It is nearly as simple as this:

```scala
class LibaniusActorEventBus extends ActorEventBus with LookupClassification
    with AppDependencyAccess {

  type Event = EventBusMessageEvent

  type Classifier = String

  protected def classify(event: Event): Classifier = event.channel
  
  protected def publish(event: Event, subscriber: Subscriber) {
    subscriber ! event
  }
}
````

The diagram below shows what use we are actually making of the event bus:

![eventbus.jpg]({{site.baseurl}}/assets/eventbus.jpg)



The Android parts are `OptionsScreen` and `DictionarySearch`. In the UI, `DictionarySearch` is actually part of the `OptionsScreen` activity, but they communicate through the event bus. When the user clicks on a button from `DictionarySearch` to add a new word to the quiz, a `NewQuizItemMessage` is constructed. This event is just a wrapper for the quiz item and an identifying header:

```scala
case class NewQuizItemMessage(header: QuizGroupHeader, quizItem: QuizItem)
  extends EventBusMessage
```

The `NewQuizItemMessage` is published to the QUIZ_ITEM_CHANNEL by calling `sendQuizItem()` in `LibaniusActorSystem`. `LibaniusActorSystem` is a facade to `LibaniusActorEventBus` and other actors in the system not discussed here.

```scala
object LibaniusActorSystem extends AppDependencyAccess {
  val system: ActorSystem = ActorSystem("LibaniusActorSystem")

  val appActorEventBus = new LibaniusActorEventBus
  val QUIZ_ITEM_CHANNEL = "/data_objects/quiz_item"

  def sendQuizItem(qgh: QuizGroupHeader, quizItem: QuizItem) {   
    val newQuizItemMessage = NewQuizItemMessage(qgh, quizItem)
    appActorEventBus.publish(EventBusMessageEvent(QUIZ_ITEM_CHANNEL, newQuizItemMessage))
  }
}
```

That's the guts of it. There is some more code in `OptionsScreen` to create an actor to subscribe to the QUIZ_ITEM_CHANNEL, and actually add the `QuizItem` when it sees the event. For more detail, see the code in the Libanius Android project.
