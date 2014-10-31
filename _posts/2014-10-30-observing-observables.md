---
layout: post
title: Observing Observables in Mobile with RxJava for Android
---

Functional Reactive Programming has been adopted in many programming communities, and for good reason. Trying to manage multiple asynchronous calls usually results in a mess of code that is not only tricky to debug, but difficult to maintain and build upon — not to mention the many “gotchas” surrounding the use of AsyncTasks on various different versions of the Android SDK. This kind of code usually ends up being the bane of an Android developer’s existence.

Enter the Android module for RxJava.

By using a souped up version of the Observer pattern, we gain the ability to create incredibly powerful chains of logic that can be specifically run against various threads (UI, background, etc) and that can be further modified by users of these observables to tune them exactly for the UI they are trying to populate.

<!-- more -->

RxJava lets you create Observables, which are streams of events that are emitted to subscribers. Subscribers can implement the following methods: onNext, onComplete, and onError.

When an observer emits something, it is delivered to the onNext callback. onComplete is called when the observer is done emitting and tells the subscriber that there won’t be anything else coming down the pipe, and onError is called when something goes wrong like an exception being thrown or invalid data being found. Similar to onComplete, once an error is encountered, no more items will be emitted.

If you dig into the RxJava documentation, you’ll find lots of diagrams that help explain what is going on visually. The notation guide for which is shown below. I’ll include appropriate marble diagrams for each code example that follows:

<p align="center"><img src="/assets/legend.png" width="650" alt="RxJava Legend" /></p>

A simple example would be to make an observable that wraps a Volley call that goes and gets a list of tweets for a particular user. The support for futures in Volley makes this pretty straightforward:

{% highlight java %}
public Observable<List<Tweet>> getUserTweets(...) {
   RequestFuture<List<Tweet>> future = RequestFuture.newFuture();

   // Create a Volley request with the future as both 
   // success and error callbacks

   // Add request to requestQueue instance

   return Observable.from(future, Schedulers.io());
}
{% endhighlight %}

<p align="center"><img src="/assets/from.Future.s.png" width="650" alt="From Future" /></p>

The initial response returned from Volley will be a list of tweets which the observer will emit as a single emission to the subscribers’ onNext implementations. This is perfect for some places in our UI, so let’s keep this method.

But what if we only wanted one tweet at a time to be emitted in another area of the application? Should we write a whole new method to handle this case? Using the composability of observables, we can simply call the flatMap operator on the observable returned by our existing method to transform the item emitted by it into a new observable of many items:

{% highlight java %}
getUserTweets(...)
   .flatMap(new Func1<String, Observable<List<Tweet>>>() {
      @Override
      public Observable<String> call(List<Tweet>> tweetList) {
         return Observable.from(tweetList);
      }
   })
   .subscribeOn(Schedulers.newThread())
   .observeOn(AndroidSchedulers.mainThread());
{% endhighlight %}

<p align="center"><img src="/assets/flatMap.png" width="650" alt="Flat Map" /></p>

Subscribers to this version of the observable will now get each individual tweet emitted via the onNext callback. Also note that we tell the framework to do the network call associated with this observable on a background thread, but call our onNext callback on the UI thread so we can directly manipulate our UI.

Now what if a fragment that uses this observer wants to rate-limit the emissions of this observable so it can let the user click a button to control what they see displayed? You can create a subject, which is like a proxy that acts as an observable and as a subscriber, and then have the subject emit an event every time the user clicks a button:

{% highlight java %}
final PublishSubject<Object> clickStream = PublishSubject.create();

findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
   @Override
   public void onClick(View view) {
      // Emit a single object to subscribers to say a click occurred
      clickStream.onNext(new Object());
   }
});
{% endhighlight %}

<p align="center"><img src="/assets/S.PublishSubject.png" width="650" alt="Publish Subject" /></p>

That’s all fine and good, but the Android module of RxJava comes with a really handy helper called ViewObservable that makes this even easier:

{% highlight java %}
Observable buttonClickObservable = 
         ViewObservable.clicks(findViewById(R.id.button1), false);
{% endhighlight %}

This does essentially the same thing as the PublishSubject and saves you from having to create your own OnClickListener. The boolean parameter lets you choose if the resulting Observable will emit it’s current state when it is subscribed to.

Either of these click emitting options can then be tied to the original observer using the zip operator, which can then trigger a UI update when the button is clicked, delivering the next data element as required. Assuming we stored the flat-mapped observable into an Observable named tweetObservable:

{% highlight java %}
Observable.zip(
   tweetObservable,
   buttonClickObservable,
   new Func2<Tweet, Object, Tweet>() {
      @Override
      public Tweet call(Tweet tweet, Object object) {
         return tweet;
      }
   })
   .subscribe(new Action1<Tweet>()) {
      @Override
      public void call(Tweet tweet) {
         // Update UI to display current tweet
      }
   });
{% endhighlight %}

<p align="center"><img src="/assets/zip.png" width="650" alt="Zip" /></p>

The above is a pretty simple example of Observables, and their true power isn’t really exposed here. Observables become invaluable when you’re trying to construct a set of data from multiple endpoints and you don’t want to call them each synchronously. If you have observables already for each of the endpoints, you can quickly compose just such a reply with very little effort.

One such example would be trying to fetch Facebook events for a particular user. We used to do a single FQL call, but the Graph API requires us to make multiple requests to get all event types. To do this with Volley you would have to add all four requests to the queue and wait for each callback to get called. You would then have to combine the results into a common list with some extra smarts in each request’s onSuccess and onError callback pair to determine if all the requests have finished and, if so, return the final result. The thought of having to jump back into this code a few months down the road to modify it or fix some strange edge case should be enough to convince anyone that there’s got to be a better way.

Observe (sorry!) how Observables can make this complexity trivial to manage. As per our previous example, the getEventOfType() method simply creates a new Graph API call and adds it to the Volley request queue and returns an Observable tied to the request’s future:

{% highlight java %}
public Observable<List<FacebookEvent>> getEvents(String id, String facebookAuth) {
      return Observable.mergeDelayError(
             getEventOfType(PATH_EVENT_ATTENDING, id, facebookAuth),
             getEventOfType(PATH_EVENT_MAYBE, id, facebookAuth),
             getEventOfType(PATH_EVENT_DECLINED, id, facebookAuth),
             getEventOfType(PATH_EVENT_NOT_REPLIES, id, facebookAuth)
      ).collect(new ArrayList<FacebookEvent>(), 
             new Action2<List<FacebookEvent>, List<FacebookEvent>>() {
                @Override
                public void call(List<FacebookEvent> output, List<FacebookEvent> input) {
                   output.addAll(input);
                }
             });
   }
{% endhighlight %}

<p align="center"><img src="/assets/mergeDelayError.png" width="650" alt="Merge Delay Error" /></p>

Here we use the mergeDelayError operator to compose a new Observable. The standard merge operator will just take all emitted items of all provided Observables into a new single stream. The issue with a normal merge is that if a provided Observable emits an error, it will trigger the onError callback for the subscriber and then not emit anything else that comes in afterwards. mergeDelayError will delay the emission of any errors until all other Observables complete or error out themselves. This is another huge help for making easy to maintain code that handles a wide number of edge cases.

<p align="center"><img src="/assets/collect.png" width="650" alt="Collect" /></p>

We also are using the operator collect to combine the results of each of the separate network calls. Without doing so, we would get a separate emission of a list for each successful response. If we used flatMap, it would be a separate emission of each event. Collect allows us to build up a single collection and emit that as a single response.

Unfortunately, the syntax for anonymous methods is pretty gnarly as we don’t have lambda support yet in Android. Adventurous developers could try using the Retrolambda library, which backports lambdas all the way back to Java5, but we’ve decided to hold off on using Retrolambda in production code until we can test it more thoroughly.

In summary, using observables in conjunction with networking calls lets you write very broad network requests that can easily be re-used in a wide number of ways throughout your app, with very little code modification required. They also allow you to write extremely complex asynchronous actions in a very readable and maintainable way.

<h3>Sources/Further Learning:</h3>

[RxJava for Android](https://github.com/ReactiveX/RxJava/wiki/The-RxJava-Android-Module)<br>
[Netflix talk re: Functional Reactive Programing](http://www.youtube.com/watch?v=_t06LRX0DV0)<br>
[Introduction to Reactive Programming](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)<br>
[Rx and Retrofit Presentation by Jake Wharton](https://www.youtube.com/watch?v=aEuNBk1b5OE#t=2480)<br>
