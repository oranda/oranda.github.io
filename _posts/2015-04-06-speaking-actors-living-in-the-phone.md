---
published: true
title: Speaking Actors Living in the Phone
background: '/assets/mountain-690122_1920.jpg'
---
A Hollywood actor who couldn't speak wouldn't get far. How soon before we say the same of mobile apps?

<img style="float:right" src="{{site.baseurl}}/assets/photoQuizScreen.jpg" width="215" height="240">

This article is a bit off-topic for this blog, but it's a bit of fun and the subject deserves more attention. Over the past 18 months the [Google Text-to-Speech API](http://en.wikipedia.org/wiki/Google_Text-to-Speech) has come a long way on the Android platform. I decided to take advantage of it in the [Libanius quiz app](https://play.google.com/store/apps/details?id=com.oranda.libanius&hl=en). For example, when the app presents a quiz item, it should speak the prompt ("Clairvoyance" in the picture).
     
Libanius uses the actor model from Akka to coordinate its subsystems. To see how Android can be set up to use Akka actors, see [this](http://typesafe.com/activator/template/macroid-akka-pingpong) from Typesafe, or [this](http://scala-bility.blogspot.de/2013/08/akka-in-your-pocket.html) from me. Assuming the Akka infrastructure is in place, a speaking actor can be created like this:

```scala
class Voice(implicit ctx: Context) extends Actor with TextToSpeech.OnInitListener {

  val tts = new TextToSpeech(ctx, this)

  override def receive = {
    case Speak(text: String, quizGroupKeyType: String) => // see next code snippet
    case _ => logError("Voice received an unknown command")
  }
}
```

The Context of the Android activity is passed to the actor because the `TextToSpeech` class needs it for initialization. Speak is just a case class for the message that is sent to the actor. We have to implement what happens when that message is received.

The prompt could be in various different languages. The Google API supplies a different voice for each language, so we need to take advantage of this.

```scala
case Speak(text: String, quizGroupKeyType: String) =>
  KnownQuizGroups.getLocale(quizGroupKeyType).foreach(<b>speak</b>(text, _))
```

`KnownQuizGroups` is a simple map of Libanius quiz group types (e.g. "Spanish") to locales
recognized by `TextToSpeech`.

The speak method finally uses the `TextToSpeech` API to set the voice according to the given locale, and actually speak the text. It works fine whether the text is a word or a full sentence.

```scala
def speak(text: String, locale: Locale) {
  setSpeechLanguage(locale)
  tts.speak(text, TextToSpeech.QUEUE_FLUSH, null)
}
 
private[this] def setSpeechLanguage(locale: Locale): Unit = {
  val result: Int = tts.setLanguage(locale)
  if (result == TextToSpeech.LANG_MISSING_DATA || result == TextToSpeech.LANG_NOT_SUPPORTED)
    logError("Language is not available.")
}
```

Also we must remember in the `Speak` code to reset the language after the message is spoken.

```scala
setSpeechLanguage(DEFAULT_LOCALE)
```

While we're adding sound to the Libanius quiz, how about a ping sound for a correct answer, and a buzz sound for a wrong answer? This is a job for another actor: SoundPlayer. The guts of it look like this:

```scala
class SoundPlayer(implicit ctx: Context) extends Actor {
 
  val audioManager: AudioManager =
    ctx.getApplicationContext.getSystemService(Context.AUDIO_SERVICE).asInstanceOf[AudioManager]
  val soundPool: SoundPool = new SoundPool(10, AudioManager.STREAM_MUSIC, 0)
  var soundPoolMap = Map[SoundSample, Int]()
 
  override def receive = {
    case Load() =>
      loadSounds()
    case Play(soundSample: SoundSample) =>
      val curVolume = audioManager.getStreamVolume(AudioManager.STREAM_MUSIC)
      soundPool.play(soundPoolMap(soundSample), curVolume, curVolume, 1,  0, 1f)
    case _ =>
      logError("SoundPlayer received an unknown command")
  }
}
```

There is [more complete code](https://github.com/oranda/libanius-android/tree/master/src/main/scala/com/oranda/libanius/mobile/actors) on Github. This is all for the Android platform, but of course you can also use Akka actors for sound on other platforms like the Mac: see Alvin Alexander's [Wikipedia Reader](http://alvinalexander.com/scala/wikipedia-page-reader-scala-akka) page for some sample code. Have fun!
