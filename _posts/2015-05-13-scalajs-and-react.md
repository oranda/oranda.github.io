---
published: true
title: 'Scala.js and React: Building an Application for the Web'
---
[Scala.js](http://scala.js/) compiles Scala code to JavaScript. I noticed on the [Reactive Programming course at Coursera](https://www.coursera.org/course/reactive) that Scala.js has been integrated into it to implement a basic spreadsheet on a Web page, suggesting good support from the Scala establishment. The principal developer of Scala.js is a collaborator of Martin Odersky at EPFL, Sébastien Doeraene, and [he will be speaking about it next month (June 2015) at Scala Days](http://event.scaladays.org/scaladays-amsterdam-2015#!#schedulePopupExtras-6936) in Amsterdam.

Before getting into the sample application, let's talk about the motivation for Scala.js. Basically, the Web continues to be a powerful platform for application development. Despite its many problems, it has features that the desktop and mobile phone cannot match. For example, the install process for a web application is negligible from a user's perspective: just load the web page for the first time and it is already there. Also, web applications can connect with each other via hyperlinks and REST calls.

Unfortunately on the Web, JVM languages have traditionally been limited to the server-side. Functionality on the client-side is dominated by JavaScript. The trouble is, from a developer's perspective, two languages mean a lot of extra complexity. You can't share code between them, and to pass objects at runtime between client and server, you end up writing serialization and validation logic in both languages. It would be preferable to implement a web application in a single language.

How is this possible, given that browsers only understand JavaScript? One possibility is to run JavaScript on the server-side, which is the approach that Node.js takes. However, JavaScript meets with many complaints: it is essentially untyped, and you can't take advantage of the solidity and scalability that the JVM has been providing on the server-side for so long. It would be better to use a language on both sides that can take advantage of the performance of JVM, the safety of typing, and also the FP principles crossing over into the mainstream during the past ten years. This is made possible though the use of **transpilers** which convert one source language (Scala in the case of Scala.js) to JavaScript.

One of the big challenges in Web programming is coordinating events so that elements in the view (client-side) are updated when the model (server-side) changes. For desktop apps, the [Observer design pattern](https://sourcemaking.com/design_patterns/observer) is often used, but on the Web, it takes a bit more work, and we often employ the help of some MVC (Model-View-Controller) Web framework. The most general term for getting changes to propagate automatically like this (as opposed to manually making calls from the view all the time) is "Reactive Programming". A particular form of Reactive Programming is **Functional Reactive Programming (FRP)** which is about capturing relationships in a composable way between values that change over time ("signals"). A related approach is to use a message passing system like Akka that keeps components loosely coupled. In both cases the key goals are to avoid the inefficiency of blocking operations and the hazards of mutable data, so making the overall system [scalable and resilient](http://www.reactivemanifesto.org/).

I would propose that the term **FWP** (Functional Web Programming) be used to cover systems that bring FP and Reactive Programming to the Web: including [Elm](http://elm-lang.org/), [ClojureScript/Om](https://github.com/omcljs/om), and Play Iteratees. The FWP implementation I have chosen is a combination of Scala.js and React: this article describes setting up a simple application without going into depth about FRP or being a full tutorial.

<img style="float:right;" src="{{site.baseurl}}/assets/libanius-scalajs-react-v0.2-screenshot.png">

While Scala.js was being developed, back in the JavaScript world frameworks were being developed that tackled the issues of Reactive Programming. One of the most popular has been [React](https://facebook.github.io/react/), developed by Facebook and Instagram. When it was introduced in May 2013, it surprised seasoned developers as it seemed to violate establish best practices. In JavaScript updating the browser DOM is slow, so it was common to only update the necessary parts when the backing model changed. However, in React when any component's state is changed, a complete re-render is done from the application developer's perspective. It's almost like serving a whole new page, guaranteeing that every place data is displayed it will be up-to-date. This avoids the dangers of mutating state but it seems like it would be very slow: still, it actually outperforms other frameworks like AngularJS thanks to some a clever diffing algorithm involving a "virtual DOM" that is maintained independently of the browser's actual DOM.

Another advantage of React is that developers concentrate on re-using components rather than templates where messy logic tends to accumulate. Components are typically written in JSX, a JavaScript extension language, and then translated to actual JavaScript using a preprocessor. For instance, consider the score text at the top left in the Libanius app (see screenshot). In JSX this would be written:


```scala
var ScoreText = React.createClass({
  render: function() {
    return (
      <span className="score-text">
        Score: {this.props.scoreText}
      </span>
    );
  }
});
React.render(
  <ScoreText />,
  document.getElementById('content')
);<b>
</b>
```

This is JavaScript plus some syntactic sugar. Notice how the score variable is passed in using the props for the component.

The code above is converted to normal JavaScript by running the preprocessor. On the command-line you would typically run something like this to watch your source directory and translate code to raw JavaScript in the build directory whenever anything changes:

```conf
> jsx --watch src/ build/
```

This is using standard React so far. However this is not the way we do it with Scala.js.

Firstly, there exists a Scala library from Haoyi Li called [Scalatags](http://lihaoyi.github.io/scalatags/), that includes Scala equivalents for HTML tags and attributes. Let's assume we have a file `QuizScreen.scala` in which we are writing the view. The core code may start off looking a bit like this:

```scala
@JSExport
object QuizScreen {
  @JSExport
  def main(target: html.Div) = {
    val quizData = // … Ajax call
    target.appendChild(
      span(`class` := "alignleft", "Score: " + quizData.score)
      // ... more view code here
    )
  }
} 
```

Notice that `span` is a Scala method (the Scalatags library has to be imported). Assuming you've configured SBT to use Scala.js (see below), this is converted to JavaScript in SBT by calling:

```conf
> fastOptJS 
```

A good thing about the Scala.js compiler is that it keeps the target JavaScript small by eliminating any code from included libraries that is not used. To stop this from happening on the entry point itself in `QuizScreen`, it is necessary to use the `@JSExport` annotation both on the object and the `main` method. This guarantees that `main()` will be callable from JavaScript. 

So now we have seen the React way and the Scala.js way. How do we combine them? A good option is to use the [scalajs-react](https://github.com/japgolly/scalajs-react%3Escalajs-react) library from David Barri. Now the ScoreText component looks like this:

```scala
val ScoreText = ReactComponentB[String]("ScoreText")
    .render(scoreText => <.span(^.className := "alignleft", "Score: " + scoreText))
    .build
```

Compare this with the JSX version. The Scala code is more concise. Notice that the `render()` method is present. It's also possible to use other React lifecycle methods if necessary, like `componentDidMount()` for initializing the component.

Notice also that it uses a Scala `span`. A specialized version of Scalatags is used here. At first the extra symbols look intimidating, but just remember that `<` is used for tags and `^` is used for attributes, and they are imported like this:

```scala
import japgolly.scalajs.react.vdom.prefix_<^._
```

In classic JavaScript React, a component can hold state, as seen in references to `this.state` and the `getInitialState()` method, which might look like this.

```scala
getInitialState: function() {
  return {data: []};
}
```

The Scala version lets us define the state more clearly because it is a strongly typed language. For example, the state for the `QuizScreen` looks like this:

```scala
case class State(userToken: String, currentQuizItem: Option[QuizItemReact] = None,
    prevQuizItem: Option[QuizItemReact] = None, scoreText: String = "",
    chosen: Option[String] = None, status: String = "")
```

It is best to centralize the state for the screen like this and pass it down to components, rather than having each sub-component have its own separate state object. That could get out of hand!

By the way, you can compose components just as you can in classic React. The central component of the `QuizScreen` object is the `QuizScreen` component, and it contains the `ScoreText` component along with the various other bits and pieces. The code below shows how this is all put together.

```scala
val QuizScreen = ReactComponentB[Unit]("QuizScreen")
  .initialState(State(generateUserToken))
  .backend(new Backend(_))
  .render((_, state, backend) => state.currentQuizItem match {
    // Only show the page if there is a quiz item
    case Some(currentQuizItem: QuizItemReact) =>
      <.div(
        <.span(^.id := "header-wrapper", ScoreText(state.scoreText),
          <.span(^.className := "alignright",
            <.button(^.id := "delete-button",
              ^.onClick --> backend.removeCurrentWordAndShowNextItem(currentQuizItem),
                  "DELETE WORD"))
        ),
        QuestionArea(Question(currentQuizItem.prompt,
            currentQuizItem.responseType,
            currentQuizItem.numCorrectResponsesInARow)),
        <.span(currentQuizItem.allChoices.map { choice =>
          <.div(
            <.p(<.button(
              ^.className := "response-choice-button",
              ^.className := cssClassForChosen(choice, state.chosen,
                  currentQuizItem.correctResponse),
              ^.onClick --> backend.submitResponse(choice, currentQuizItem), choice))
          )}),
        PreviousQuizItemArea(state.prevQuizItem),
        StatusText(state.status))
    case None =>
      if (!state.quizEnded)
        <.div("Loading...")
      else
        <.div("Congratulations! Quiz complete. Score: " + state.scoreText)
  })
  .buildU
```

The central component (`QuizScreen`) contains the other components (implementations not shown) and also has access to a `State` and a `Backend`. The backend contains logic that is a bit more extended. For example, in the code above, observe that `submitResponse` is called above on the `backend` when a button is clicked by the user. The code invoked is:

```scala
class Backend(scope: BackendScope[Unit, State]) {
    def submitResponse(choice: String, curQuizItem: QuizItemReact) {
      scope.modState(_.copy(chosen = Some(choice)))
      val url = "/processUserResponse"
      val response = QuizItemAnswer.construct(scope.state.userToken, curQuizItem, choice)
      val data = upickle.write(response)
 
      val sleepMillis: Double = if (response.isCorrect) 200 else 1000
      Ajax.post(url, data).foreach { xhr =>
        setTimeout(sleepMillis) { updateStateFromAjaxCall(xhr.responseText, scope) }
      }
    }
 
    def updateStateFromAjaxCall(responseText: String, scope: BackendScope[Unit, State]): Unit = {
      val curQuizItem = scope.state.currentQuizItem
      upickle.read[DataToClient](responseText) match {
        case quizItemData: DataToClient =>
          val newQuizItem = quizItemData.quizItemReact
          // Set new quiz item and switch curQuizItem into the prevQuizItem position
          scope.setState(State(scope.state.userToken, newQuizItem, curQuizItem,
              quizItemData.scoreText))
      }
    }
    // more backend methods...
  }
```

`submitResponse` makes an Ajax POST call to the server, collects the results and updates the `State` object. The React framework will take care of the rest, i.e. updating the DOM to reflect the changes to `State`.

In making the Ajax call, the upickle library (again from Haoyi Li) is used for serialization/deserialization. This is also used on the server side of our Scala.js application. The core of the server side is a Spray server. A simple router is defined which recognizes the call to `processUserResponse` made above:
 
```scala
object Server extends SimpleRoutingApp {
  def main(args: Array[String]): Unit = {
    implicit val system = ActorSystem()
    lazy val config = ConfigFactory.load()
    val port = config.getInt("libanius.port")
 
    startServer("0.0.0.0", port = port) {
      // .. get route not shown here
      post {
        path("processUserResponse") {
          extract(_.request.entity.asString) { e =>
            complete {
              val quizItemAnswer = upickle.read[QuizItemAnswer](e)
              upickle.write(QuizService.processUserResponse(quizItemAnswer))
            }
          }
        }
      }
    }
  }
}
```

The "processUserResponse" path extracts the post data using upickle then passes the call on to the `QuizService` singleton which contains the mid-tier business logic, and relies on the core Libanius library to run the main back-end functionality on the Quiz. I won't go into detail about this logic, but note that both for historical reasons and future portability it relies on simple files to hold quiz data rather than a database system.

Back to the Spray server: when the `QuizScreen` page is initially loaded, this route is used:

```scala
get {
  pathSingleSlash {
    complete{
      HttpEntity(
        MediaTypes.`text/html`,         
        QuizScreen.skeleton.render
      )
    }
  } 
}
```

The `QuizScreen` mentioned here is not the `QuizScreen` on the client-side that is described above. In fact, it is a _server-side_ `QuizScreen` that makes a call to the client-side `QuizScreen`. Like this:

```scala
object QuizScreen {
 
  val skeleton =
    html(
      head(
        link(
          rel:="stylesheet",
          href:="quiz.css"
        ),
        script(src:="/app-jsdeps.js")
      ),
      body(
        script(src:="/app-fastopt.js"),       
        div(cls:="center", id:="container"),
        script("com.oranda.libanius.scalajs.QuizScreen().main()")
      )
    )
} 
```

Again the tags are from Scalatags. The main call is in the last `script` tag. Recall that on the client-side we use `@JSExport` to make the `QuizScreen().main()` available:

```scala
@JSExport
def main(): Unit = {
  QuizScreen() render document.getElementById("container") 
}
```

Also notice in the skeleton above, there are two included JavaScript libraries:

- `app-fastopt.js`: In a Scala.js application, the `*-fastopt.js` file is the final output of the `fastOptJS` task, containing the JavaScript code that has been generated from your Scala code.
- `app-jsdeps.js`: In a Scala.js application, the `*-jsdeps.js`, contains all additional JavaScript libraries: in our case, the only thing it incorporates is `react-with-addons.min.js`.

Here are the essentials of the SBT configuration, which can be used as a starting point for other Scala.js projects, as it just uses the most basic dependencies, including Scalatags, upickle for serialization, and utest for testing.

```scala
import sbt.Keys._

name := "Libanius Scala.js front-end"

// Set the JavaScript environment to Node.js, assuming that it is installed, rather than the default Rhino

scalaJSStage in Global := FastOptStage 

// Causes a *-jsdeps.js file to be generated, including (here) React
skip in packageJSDependencies := false
 
val app = crossProject.settings(
  unmanagedSourceDirectories in Compile +=
    baseDirectory.value  / "shared" / "main" / "scala",
 
  libraryDependencies ++= Seq(
    "com.lihaoyi" %%% "scalatags" % "0.5.1",
    "com.lihaoyi" %%% "utest" % "0.3.0",
    "com.lihaoyi" %%% "upickle" % "0.2.8"
  ),
  scalaVersion := "2.11.6",
  testFrameworks += new TestFramework("utest.runner.Framework")
).jsSettings(
  libraryDependencies ++= Seq(
    "org.scala-js" %%% "scalajs-dom" % "0.8.0",
    "com.github.japgolly.scalajs-react" %%% "core" % "0.8.3",
    "com.github.japgolly.scalajs-react" %%% "extra" % "0.8.3",
    "com.lihaoyi" %%% "scalarx" % "0.2.8"
  ),
  // React itself (react-with-addons.js can be react.js, react.min.js, react-with-addons.min.js)
  jsDependencies += "org.webjars" % "react" % "0.13.1" / "react-with-addons.js" commonJSName "React"
  
).jvmSettings(
  libraryDependencies ++= Seq(
    "io.spray" %% "spray-can" % "1.3.2",
    "io.spray" %% "spray-routing" % "1.3.2",
    "com.typesafe.akka" %% "akka-actor" % "2.3.6",
    "org.scalaz" %% "scalaz-core" % "7.1.2"
  )
)
 
lazy val appJS = app.js.settings(
  // make the libanius core JAR available
  // ...
  unmanagedBase <<= baseDirectory(_ / "../shared/lib")
)

lazy val appJVM = app.jvm.settings(
  // make sure app-fastopt.js, app-jsdeps.js, quiz.css, the libanius core JAR, application.conf
  // and shared source code is copied to the server
  // ...
)
```

As you can see, the special thing about a Scala.js client-server SBT configuration is that it is divided into three parts: `js`, `jvm`, and `shared`. The js folder contains code to be compiled by ScalaJS, the `jvm` folder contains regular Scala code used on the server-side, and the shared folder contains code and configuration that should be accessible to both `js` and `jvm`. This is achieved by using the `crossProject` builder from Scala.js, which constructs two separate projects, the `js` one and the `jvm` one.

So far we've been assuming that any generated JavaScript will run in a browser. However, Scala.js also works with "headless runtimes" like Node.js or PhantomJS to ensure you can run it from the command-line on the server-side too: this is important in testing. Notice the `scalaJSStage in Global := FastOptStage` line above.

Now for a grand overview of the web application, let's look at the directory structure. You can see how slim the application really is: there are only a few key source files.

```conf
libanius-scalajs-react/  
  **build.sbt**
  app/
    js/
      src/
        main/
          scala/
            com.oranda.libanius.scalajs/
              **QuizScreen.scala**
      target/
    jvm/
      src/
        main/
          resources/
            **application.conf**
          scala/
            com.oranda.libanius.sprayserver/
              **QuizScreen.scala**
              **QuizService.scala**
              **Server.scala**
      target/
    shared/
      lib/
        **libanius-0.982.jar**
      src/
        main/
          resources/
            **quiz.css**
          scala/
            com.oranda.libanius.scalajs/
              **ClientServerObjects.scala**
                QuizItemReact
```

Again notice there is a `QuizScreen` on both the server-side and client-side: the former calls the latter.

One thing that I didn't mention yet is the `quiz.css` file that is used in the client-side `QuizScreen`. This is just an old-fashioned CSS file, but of course it also possible to use LESS files. Furthermore, if you don't anticipate having a graphic designer want to change your styles, you could even go the whole way in making your application _type safe_, and write your styles in Scala with [ScalaCSS](https://github.com/japgolly/scalacss).

The full code for this front-end to Libanius is [on Github](https://github.com/oranda/libanius-scalajs-react). As of writing there is a [deployment on Heroku](https://thawing-stream-3905.herokuapp.com/) (may not be supported indefinitely). For a full tutorial on Scala.js, see Hands-on Scala.js from Haoyi Li. There is also a [small official tutorial](http://www.scala-js.org/doc/tutorial.html).
