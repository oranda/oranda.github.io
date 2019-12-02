---
published: true
title: 'A Sample Application Using Akka Cluster: Libanius, the Quiz'
---

Let's describe a sample Akka application. The Libanius Scala library holds an immutable `Quiz` class with associated functionality. A Web UI accesses it on behalf of users via Akka.

The quiz is originally read from a file, but then each user has his own version of it. Each user is assigned an actor which holds the quiz for him, updates it according to his performance, and serves quiz items according to that performance. The actor class is descriptively called `QuizForUserActor`. It starts like this:

```scala
class QuizForUserActor(quiz: Quiz) extends PersistentActor {

  private var state = QuizState(quiz)
   ...
}
```

Notice that, whereas a regular Akka actor would just extend `Actor`, this one extends `Persistent Actor`. That's because the quiz state needs to be saved to disk. Akka Persistence provides support for this. However, it is not a relational database. The persistence paradigm is called **Event Sourcing** and it means that changes to state are stored in a log and then replayed whenever the latest state needs to be "recovered". This sounds like recovery could take a long time if a million events have occurred, but it's ok to cheat and take **snapshots** of the state from time to time so that recovery only needs to proceed from those snapshots.

Back to the code. A typical Akka actor will have a method called `receive` where it gets messages and acts on them. For persistent actors, the method is called `receiveCommand`. The core of it looks like this:

```scala
  override def receiveCommand: Receive = {
    case ScoreSoFar(_) =>
      sender() !  state.quiz.scoreSoFar
    case ProduceQuizItem(_) =>
      sender() ! produceQuizItem(state.quiz)  
    ...  
  }
```  

This code is within `QuizForUserActor` and is pretty self-explanatory. If the UI sends a `ScoreSoFar` message to `QuizForUserActor`, then it calls the `scoreSoFar` function on the `quiz` object and sends back the result. If the UI needs a new quiz item, it will send a `ProduceQuizItem` message to `QuizForUserActor`, and again the actor will pass on the call to a function to get a result. 

There are also messages that change quiz state, and these are converted to events which are stored via calls to the Akka Persistence API, so that they can be replayed as necessary.

Akka Persistence is just one of the many extensions to Akka that exist. Another is called **Akka Cluster Sharding** and this pertains to the topic of this blog: scalability. For a single-user quiz you might not bother with Akka at all, but if you have millions of users and actors for each one, those actors will likely be stored on multiple servers -- probably in "the cloud". Keeping track of all that distributed functionality is a hard problem, so you are going to end up relying on some piece of third-party clustering software. Of course I recommend Akka Cluster Sharding, so let's see how to use it.

The core of the code looks like this:

```scala
import akka.cluster.sharding.{ClusterSharding, ClusterShardingSettings, ShardRegion}

object QuizForUserSharding {

  def startQuizForUserSharding(system: ActorSystem, quiz: Quiz): ActorRef = {
    ClusterSharding(system).start(
      typeName = "QuizForUser",
      entityProps = Props(new QuizForUserActor(quiz)),
      settings = ClusterShardingSettings(system),
      extractEntityId = extractEntityId,
      extractShardId = extractShardId
    )
  }
}
```

The first thing you will notice in this code is the call to instantiate a `QuizForUserActor`. That's the entity that the clustering system is taking command of. Then there's a reference to some settings, and then two callbacks: `extractEntityId` and `extractShardId`.

```scala
  val extractEntityId: ShardRegion.ExtractEntityId = {
    case qc: QuizMessage => (qc.userId.toString, qc)
  }
```

Remember `ScoreSoFar(_)` and `ProduceQuizItem(_)`? Those are types of `QuizMessage`. If you wondered what the _ was, it's a placeholder for the `userId` which every message must carry. The `userId` is not actually used within `QuizForUserActor`, but it's picked up by the clustering framework to identify actors. What happens is that when a user first appears in the system, the server (**Akka Http**) assigns a unique `userId` to him, and stores it in a cookie on the front-end. When the user makes a request, this is modelled as a `QuizMessage` including the `userId`. Let's assume Akka Http has already started up cluster sharding using this call:

```scala
val quizForUserShardRegion = QuizForUserSharding.startQuizForUserSharding(system, quiz)
```
`quizForUserShardRegion` is an actor. Now on a user request, Akka Http will send on the message to it like this:

```scala
quizForUserShardRegion ? ProduceQuizItem(userId)
```

The `?` stands for `ask`, meaning a result is expected. That result needs to be extracted, so the full call is actually:

```scala
 (quizForUserShardRegion ? ProduceQuizItem(userId)).mapTo[Option[QuizItemViewWithChoices]]
```

When the `quizForUserShardRegion` actor gets the `ProduceQuizItem` message, it looks for an actor with the given `userId`: if it doesn't exist, it creates it and manages it from then on. 

Don't forget there was a second callback in `startQuizForUserSharding` called `extractShardId`. A **shard** is a subgroup of entities (where `QuizForUserActor` is a typical entity), and multiple shards are called a **shard region**. Each shard region has a **shard region actor**, which is a local proxy on a particular node, and all the shard regions are managed by a singleton actor called the **shard coordinator**. How many entities should be in a shard? You may need to test performance to determine the optimal number. Since several entities may share the same shard ID, this kind of logic goes in the `extractShardId` method.

Anyway, you can see there is a lot of work going on in the background to locate actors on various nodes, and balance load, and keep track of node health, and that is what Akka Clustering is doing for you. To take a closer look at the code, start with the [`QuizForUserActor` class](https://github.com/oranda/libanius-akka/blob/master/src/main/scala/com/oranda/libanius/actor/QuizForUserActor.scala) in the [`libanius-akka` project](https://github.com/oranda/libanius-akka), and the [`QuizForUserSharding` object](https://github.com/oranda/libanius-scalajs-react-akka/blob/master/app/jvm/src/main/scala/com/oranda/libanius/server/actor/QuizForUserSharding.scala) in the [`libanius-scalajs-react-akka` project](https://github.com/oranda/libanius-scalajs-react-akka).
