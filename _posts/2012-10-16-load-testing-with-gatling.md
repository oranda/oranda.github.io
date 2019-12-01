---
published: true
---
## Load Testing with Gatling

Recently, I've been working with an organization that had a huge increase in traffic due to mobile phones, and needed to determine how its legacy application would handle the millions of users flooding into its servers. They were accustomed to using JMeter, as it's a full-featured mature Java application for load testing and also useful for functional testing.

However, it turns out that when the numbers get really big, you have to figure out how to deal with the load of JMeter itself! You [end up distributing the JMeter application across several servers](http://jmeter.apache.org/usermanual/remote-test.html). Surely this shouldn't be necessary? Also, because it's so UI-oriented, the "scripts" you use for testing are big clunky XML files. You might prefer writing them in a modern programming language, giving you the option to integrate them into a general network management system.

Enter [Gatling](https://gatling.io/). This tool is based on the modern Akka framework, so it has much less overhead than JMeter. In fact, I was able to run it on a single machine all the time. Furthermore, since the scripts you write are in Scala, you can make it as easy or complex as the needs of the organization warrant. A simple script (or "simulation") looks like this: 

```scala
val myScenario = scenario("My Sequence Of Actions")
  .feed(csv("feed_of_users.csv"))
  .exec(http("Confirm Newspaper Subscription")
  .get("/myapp/hasSubscription?userId=" + "${userId}")
  .check(status.is(200))
  .check(regex("""<response status='ok' """).exists))
```
  
Run gatling.sh, and choose the script you want to run. After it finishes Gatling gives you a HTML page filled with graphs. Here is an example. This graph is actually showing a problem with the application under the test. The bumps in the red line mean that connections are being dropped from time to time.

![gatlingGraph.png]({{site.baseurl}}/_posts/gatlingGraph.png)


In summary, Gatling is not as fancy as JMeter yet, but I think this is the future of load testing.
