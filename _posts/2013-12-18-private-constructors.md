---
published: true
title: Private Constructors
background: '/assets/mountain-690122_1920.jpg'
---
Private constructors: mostly you won't need them. Commonly in Scala we just define a class and let the constructor and parameters be exposed to the world:

```scala
case class SocialFeedCache(tweets: List[Tweet]) { ... }
```

But suppose the tweets need to be organized for efficiency, and we don't want the cache to be constructed in an invalid state? Then we might have a factory method that performs the necessary initialization. In Java this would be a static method. In Scala we put it in a companion object:

```scala
object SocialFeedCache {
  def apply(tweets: List[Tweet]): SocialFeedCache = {
    // ... organize the tweets ...
    SocialFeedCache(organizedTweets) // call the primary constructor
  }
}
```

So now we must always call the factory method rather than the primary constructor. How can we enforce this? How can we block anyone from calling the constructor directly? A first attempt might look like this:

```scala
private case class SocialFeedCache(tweets: List[Tweet]) { ... }
```

But this doesn't just hide the constructor; it hides the whole class! In fact there will be a compile-time error, because the factory method in object SocialFeedCache is trying to expose a class that is now hidden.

The correct syntax is actually:

```scala
case class SocialFeedCache private(tweets: List[Tweet]) { ... }
```

And that's all there is to it. This is a useful technique whenever you have special initialization taking place in factory methods.
