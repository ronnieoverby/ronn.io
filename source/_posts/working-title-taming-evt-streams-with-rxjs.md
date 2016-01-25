title: Hacking Event Sequences with Reactive Extensionsr
categories:
  - Code
tags:
  - javascript
  - rx
date: 2016-01-25 23:00:00
---
### Long Introductory Exposition

Recently I was building a feature in an application to send files to a remote computer. These files could range in size from not-too-big to who-knows-how-big, and so I had the novel and completely original idea to display a progress bar.

{% asset_img progress_candy_bar.png Progress Bar™ %}

Displaying a progress bar to a user is always going to involve an inherently concurrent architecture. On the one hand, the app needs to do some important, time-consuming work and, on the the other hand, the app needs to update the progress bar. A poorly designed implementation would combine those 2 resonsibilities, with alternating logic between the 2 tasks. A well designed one would separate them: one thread (or task, actor, etc.) does the important work and occasionally graciously emits an **event** for interested parties. The event contains information about how much work has been done and how much remains. The business of any that will receive the event is their own.

Being able to publish events to interested parties will require some infrastructure. If the publisher and subscribers share an operating system process, the solution can be an in-memory queue. That's simple: event messages go in the queue and come out in the same order. But when the boundaries between publishers and subscribers widen or the number of publishers or subscribers increase, more complex infrastructure becomes necessary. 

### Boring. Has nothing to do with Rx.

Agreed. Just a little further… 

In my case, I'm building a web app. The server is doing the hard work of sending files. The user at the web browser wants to know how much more time the work will take. The infrastructure that was selected to acheive this was a combination of [IProgress<T>](http://blogs.msdn.com/b/dotnet/archive/2012/06/06/async-in-4-5-enabling-progress-and-cancellation-in-async-apis.aspx) and [SignalR](http://www.asp.net/signalr).

With this added complexity, we tend to lose the guarantee of first-in-first-out message delivery. That's exactly what happened to me. Once, I had everything in place, I ended up with a Progress Bar™ that seemed very unsure of itself.

{% asset_img bipolar_bar.gif Insecure Progress Bar %} 

You can see that the progress bar jumps around quite a bit. And although it does basically fill from left to right, it's still not the greatest user experience.

Although the publishing process emits events in order, the infrastructure responsible for the delivery of the events, doesn't take up the cause of perfectly ordered delivery. Regardless of why, the subscribers can't depend on perfectly ordered messages, so they must handle out of order events.

In the progress bar scenario, the solution is to disregard updates that would move the bar from right to left. What do you call the opposite of progress? 

{% asset_img Congress.jpg Congress. %}

I digress. Anyway… here's one way to tackle this in Javascript:

```javascript
let maxProgress = 0;

function handleProgressEvent(nextProgress) {
  // ensure that the incoming progress update
  // is greater than the max progress received
  if(nextProgress > maxProgress) {  
    maxProgress = nextProgress;
    updateProgressBar(maxProgress);
  }
}
```

Simple, but it's pretty low-level. What if our app has progress bars for sending files all over the place? Naturally, hopefully, we start looking for ways to reuse code. It's possible to put this bit of code in a common place and make use of the `handleProgressEvent` everywhere that it was needed. Also, there's state involved: the max progress. Where should that be kept? Modifying state in a callback isn't a big deal in Javascript, but what if we're programming on a multi-threaded platform? We could create a class that encapsulates that state and syncronizes updates to it. These are solutions to this problem, but for similar problems **we're forced to build similar, additional solutions**.

{% asset_img better_way.gif There's got to be a better way! %}

What we're dealing with here, and in many other areas of most applications, are sequences of events. By now we're used to using higher-order functions to deal with sequences in an elegant and expressive way. For example, [LINQ in .NET](https://msdn.microsoft.com/en-us/library/bb397926.aspx) provides interfaces, extension methods, and even language constructs for processing sequences of data. [Lodash](https://lodash.com/) has been my go-to for the same functionality in Javascript. Many languages are providing these conveniences, but they tend to be centered around sequences of data that are synchronous in nature: we *pull* the next item from the sequence and if it's not available yet, then we wait for it. We even have language constructs for producing such sequences: `yield` in C# and Javascript.

What about processing event sequences, asynchronously? How can we process sequences in a way that ***doesn't require waiting for the next item***? This wasn't impossible before. Callbacks aren't new. And more recently, promise libraries and language constructs like `async/await` have helped us to reduce the perceived complexity of asynchronous programming. Those tools are nice, but don't offer the composition benefits of the higher order functions found in LINQ, Lodash, or similar. We need something more.

### Finally. Reactive Extensions (rx for short)

Rx, gives developers a way to use composable, higher-order functions to process sequences asynchronously. Our progress bar scenario is a perfect fit for this tool. Progress update events are being **pushed** to the client, and we have a need to discard some of the events in the sequence.

Rx has a ton of "operators". These are the functions that can be chained together to modify, combine, or aggregate sequences. The two in particular that I've found to be the most help for discarding irrelevant progress events are called [Scan](http://reactivex.io/documentation/operators/scan.html) and [a variant of Distinct called DistinctUntilChanged](http://reactivex.io/documentation/operators/distinct.html). But to use the operators, we need a sequence to operate on. Rx calls  such a sequence an [observable](http://reactivex.io/documentation/observable.html).

In Javascript, observables can be created from arrays, iterables, DOM events, or callback APIs. I've been talking about this in the context of my specific problem, and I want to stay on that track, but converting SignalR events to an observable isn't as interesting as what you can do with an observable. I'm going to blog-cheat and skip showing the creating of the observable. However, I have written some code to simulate the problem without needing to put SignalR in place. You can see that in the demo towards the bottom of the post.

Once, you've got an instance of an observable you can subscribe to it to begin handling it's events. That looks like this:

```Javascript
sourceObservable.subscribe (
  pct => updateProgressBar(pct),
  e   => handleError(e),
  ()  => fileTransferComplete()
);
```

The subscribe method let's you hook into the 3 fundamental things that are going to happen:
 - An event is received
 - An error occurred
 - A signal is received that there are no more events.
 
The problem has yet to be solved, though. Events are still coming in out of order. We need to mould the source observable into a new one before subscribing to it. I mentioned the `scan` operator above. Scan can be used to replace the `handleProgressEvent` function from above.

```Javascript
sourceObservable
  .scan((max, pct) => Math.max(max, pct))
  .subscribe (
    pct => updateProgressBar(pct),
    // same subscription hooks as above
  );
```

The scan operator takes a function that takes the result of prior scan results and the currently emitted event from the source stream. Confusing? Reading about these operators can be. Most API documentation reads pretty terrible if you can pay attention long enough. Fortunately, the [documentation for scan](http://reactivex.io/documentation/operators/scan.html) has a nice interactive demo of how the operator can transform the event sequence.

In my example, the scan function is simply returning the maximum of 2 numbers. Scan will be called for each event in the source stream having the max and current event passed to it. Simple.

Where is the max stored during all of this? Who cares?! Rx operators handle state tracking for the duration of subscriptions on the observables.

This is actually enough to solve the problem. The progress bar no longer jumps around and the user should have a better perception of how long the file transfer will take. However, there's one minor improvement to be made. Let's say the source progress events come in the order:

| Source Event    |   Running Max (Scan)   |  Subscription Logic |
|-------|----|----------|
| 47%   | 47% |  updateProgressBar(47%)
| 50%  | 50% |  updateProgressBar(50%)
| 48% | 50% |   updateProgressBar(50%)
| 49%  |  50%  |   updateProgressBar(50%)

`updateProgressBar(pct)` is needlessly being called many times for an unchanged percentage complete. Simply chaining in the `DistinctUntilChanged` operator will take care of this.

```Javascript
sourceObservable
  .scan((max, pct) => Math.max(max, pct))
  .distinctUntilChanged()
  .subscribe (
    pct => updateProgressBar(pct),
    // same subscription hooks as above
  );
```

Now we will have reduced the # of progress bar updates to the minimum:

| Source Event   |     Running Max (Scan)     | DistinctUntilChanged     |  Subscription Logic |
|-------|-------|-------|------|
| 47%  | 47% | 47% |  updateProgressBar(47%)
| 50%   | 50% | 50% |  updateProgressBar(50%)
| 48%  | 50% |  | | 
| 49% |  50%  |    | |  
| 52%  |  52% |  52% | updateProgressBar(52%)

You can see that `DistinctUntilChanged` discards events that have equality with the most recently published event.

### Demo

Rx feels intuitive to me now, but I admit that my early attempts to understand it weren't successful. The easier path to understanding these concepts are by visualizations and reading and writing working code. I've whipped up a demo to demonstrate my exact problem and how I solved it with reactive extensions. There are some additional Rx features being used and some DOM manipulation, but nothing too daunting or distracting. Check it out:
 
 <p data-height="400" data-theme-id="0" data-slug-hash="yevPee" data-default-tab="result" data-user="ronnieoverby" data-preview="true" class='codepen'>See the Pen <a href='http://codepen.io/ronnieoverby/pen/yevPee/'>yevPee</a> by Ronnie Overby (<a href='http://codepen.io/ronnieoverby'>@ronnieoverby</a>) on <a href='http://codepen.io'>CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

### Some last thoughts

I've been talking about Rx in the context of front-end Javascript and server-side .NET programming. Rx was born at Microsoft as a .NET library. Next, a Javascript implementation came along. Times have changed and Rx has been ported to [many languages](http://reactivex.io/languages.html). More than a library for a specific language, Rx is in this strange place of being a comprehensive implementation of a philosphy for dealing with sequences in an async way. And with little to no competition.

The forthcoming Angular 2.0 is taking a dependency on the Javascript implementation of Rx. That's telling.

### tl;dr

Reactive Extensions are cool, dude.

### Some Resources

Here are some awesome resources for learning about reactive extensions:
- http://reactivex.io - primary documentation
- http://rxmarbles.com - interactive visualizations of many of the operators in action.
- http://www.introtorx.com - A kind of technical web based book explaining the fundamentals and guiding the reader through many of the operators.
- https://github.com/Reactive-Extensions - Use the source, Luke
