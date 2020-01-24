---
layout: post
title: 'Keywords as Clutter: Doing More with Less'
date: '2012-09-30 09:55:00 +0100'
categories: jekyll update
published: true
background: '/assets/mountain-690122_1920.jpg'
---
When I first started writing Scala, I found it very easy to code in a Java-like way. Perhaps a little too easy in fact. The problem is that if you persist in writing code in an imperative style, you miss a lot of the FP (functional) advantages that Scala provides.

For example, consider a simple **for loop** in Java, the unenhanced version where you need the index:

```java
for (int i = 1; i <= 10; i++) {
    System.out.println("I like the number " + i);
}
```

It is easy to transliterate this into Scala like this:

```scala
for (i <- 1 to 10)
  println("I like the number " + i)
```

This is OK. However, a more idiomatic way is to use a **foreach** function:

```scala
Range(1, 10).foreach(println("I like the number " + i))
```

This is functional, rather than procedural. People will have various reasons for preferring this style. But the interesting thing to me is that foreach is not a Scala keyword like for. It is just another function. Code is easier to compose when you get rid of the magic of keywords. You can do more with less. Another magic keyword is **if**. Again, Scala allows you to use it, and sometimes you need it. However, if you think about it a little, very often you find that using a Scala idiom is better. For example, it is tempting to translate a Java null check to something like this:

```scala
if (functionResult.isDefined) // dubious style
  doMyProcessing(functionResult.get)
```

However this check on the functionResult Option can be a bit tedious. The same thing can be written like this:

```scala
functionResult.foreach { doMyProcessing(_) }
```

This may look a bit odd if functionResult is only a single value. However, by writing it in this way you have succeeded in moving the boring check down into the functionResult Option implementation. This is especially useful when you have a lot of such "null checks" and wish to compose them. You don't want the code to become spaghetti-like with if statements. [Here is a classic cheat sheet](https://blog.tmorris.net/posts/scalaoption-cheat-sheet/) about dealing with Option's in a functional way. A different way of getting rid of if is to use **pattern matching**. Here is some Scala written in a Java-like way:

```scala
if (actorMessage == "toast_warm")
  pop()
else if (actorMessage == "toast_cold")
  toastSomeMore()
else
  println("Who moved my toast?")
```

Better is the Scala idiom:

```scala
actorMessage match {
  case "toast_warm" => pop()
  case "toast_cold" => toastSomeMore()
  case _ => println("Who moved my toast?")
}
```

Admittedly, match and case are still keywords. But in this case, it's probably worth it. Unlike the Java case label, Scala's case clause can take, not just an int or a String, but anything, including nested structures. That is the power of pattern matching. I am collecting Scala idioms, so if you can think of other good examples, especially ones that do away with keywords, add a comment.
