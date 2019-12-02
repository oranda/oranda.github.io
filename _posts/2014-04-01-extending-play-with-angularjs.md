---
published: true
title: 'Extending Play with AngularJS: Nested MVCs in a Webapp'
---
I finally got a bit of spare time to tighten up the JavaScript in [Libanius-Play](https://github.com/oranda/libanius-play), the web interface to my quiz application. For a long time, I was dissatisfied with the way data was being passed around from server to client and then back from client to server: there was too much repetition. The JavaScript library jQuery was of use to create AJAX callbacks, but still, any time there was a new parameter, it had to be added in too many places.

**AngularJS** is another JavaScript library. Like jQuery it is geared towards asynchronous communication, but it provides an MVC framework. The Play Framework is also MVC, so the result is an MVC-within-MVC architecture (see diagram). It seems to work quite well.


![libanius-nestedMVC-small.jpg]({{site.baseurl}}/assets/libanius-nestedMVC-small.jpg)


The AngularJS controller has a services layer to fetch data from the Scala back-end. Then it updates the page. The nice thing about it is that I don't have to update variables in the web page one by one: I can just **bind** variables in the web page to attributes in the model. This is achieved in the HTML using AngularJS tags like ng-model and the use of curly braces like this: 

```raw
<span id="prompt-word">{{ quizData.prompt }}</span>
```

There are a number of variables like this scattered through the HTML, but they can all be updated at once in a single line of JavaScript if the Play backend returns a JSON structure. (The data structure can be defined as a case class in Scala on the server, and will be dynamically mirrored as a JavaScript structure on the client-side without the need to map attributes manually.) Here is a simplified version of what happens in the JavaScript controller after the server has processed a user response: basically it returns a new quiz item. 

```javascript
services.processUserResponse($scope.quizData, response).then(function(freshQuizData) {
   $scope.quizData = freshQuizData.data // updates values in the web page automagically
   labels.setColors($scope.quizData)    // extra UI work using a custom JavaScript module called labels
}
```
                                                             

In the first line, services.processUserResponse is a function in the services.js file which sends user input to the server, and immediately returns a **promise** of new data. The promise has a handy **then** function which takes a callback argument, describing what happens when the data actually comes through, i.e. updating the view.

All in all, a good experience with AngularJS.

