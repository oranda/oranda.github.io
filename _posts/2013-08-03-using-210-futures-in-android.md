---
published: true
title: Using 2.10 Futures in Android
---
In an Android app, it is usually necessary to run some tasks in the background so they don't block the UI. The standard way to do this in Java code is with an AsyncTask. Basically you fill out the body of an AsyncTask with two parts: what needs to happen in the background, and what needs to happen in the foreground once that is complete. The return value of the background part is passed as a parameter to the foreground part:

Here is an example expressed in Scala, as always more concise than Java:

```scala
    new AsyncTask[Object, Object, String] {
      override def doInBackground(args: Object*): String = calculateScore
      override def onPostExecute(score: Int) { showScore(score) }
    }.execute()
```

From a Scala perspective, it might occur to you that this is a bit similar to what a Future does. Futures are more general and concise, so in my opinion they are to be preferred. Indeed, following the example of Terry Lin, I tried to get the identical functionality as the above working using the Future trait in Scala 2.10, and it worked fine. The new code has this structure:

```scala
    future {
      calculateScore
    } map {
      runOnUiThread(new Runnable { override def run() { showScore(score) } }))
    }
```

This code is a bit shorter. The one thing to remember is that after the score is returned, we have to remind Android to return control to the main thread -- that's what the runOnUiThread() is for.
