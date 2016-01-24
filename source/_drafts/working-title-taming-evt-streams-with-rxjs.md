title: Taming Event Streams with Reactive Extensions
categories:
  - Code
tags:
  - javascript
  - rx
date: 2016-01-23 17:55:00
---
## Long Introductory Exposition

Recently I was building a feature in an application that needs EDIT ME to generate and send a file to a remote computer. These files could range in size from not-too-big to who-knows-how-big, and so I had the novel and completely original idea to display a progress bar.

{% asset_img progress_candy_bar.png Progress Bar™ %}

Displaying a progress bar to a user is always going to involve an inherently concurrent architecture. On the one hand, the app needs to do some important, time-consuming work and, on the the other hand, the app needs to update the progress bar. A poorly designed implementation would combine those 2 resonsibilities, with alternating logic between the 2 tasks. A goodly designed one would separate them: one thread (or task, actor, etc.) does the important work and occasionally graciously emits an **event** for interested parties. The event contains information about how much work has been done and how much remains. The business of any that will receive the event is their own.

### Boring. Has nothing to do with rxjs.

Agreed. Just a little further… Being able to publish events to interested parties will require some infrastructure. If the publisher and subscribers share an operating system process, the solution can be an in-memory queue. That's simple: event messages go in the queue and come out in the same order. But when the boundaries between publishers and subscribers widen or the number of publishers or subscribers increase, more complex infrastructure becomes necessary. 

In my case, I'm building a web app. The server is doing the hard work of sending a file. The user at the web browser wants to know how much more time the work will take. The infrastructure that was selected to acheive this was a combination of [IProgress<T>](http://blogs.msdn.com/b/dotnet/archive/2012/06/06/async-in-4-5-enabling-progress-and-cancellation-in-async-apis.aspx) and [SignalR](http://www.asp.net/signalr).

With this added complexity, we tend to lose the guarantee of first-in-first-out message delivery. That's exactly what happened to me. Once, I had everything in place, I ended up with a Progress Bar™ that was very unsure of itself.

{% asset_img bipolar_bar.gif Insecure Progress Bar %} 

You can see that the progress bar jumps around quite a bit. And although it does *mostly* fill from left to right, it's still not the greatest user experience.

Although the process that is publishing the progress events does so in order, the infrastructure responsible for the delivery of the events, doesn't take up the cause of perfectly ordered delivery. Regardless of why, the subscribers can't depend on perfectly ordered messages, so they must handle out of order events.

In the progress bar scenario, the solution is to disregard updates that would move the bar from right to left. By the way, what's the opposite of progress? 

{% asset_img Congress.jpg Congress. %}

Anyway… here's one way to tackle this in Javascript:

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

Simple, but it's pretty low-level. What if our app has many places with progress bars where this would be necessary to do. Naturally, we start looking for ways to reuse code. We could put this bit of code in a common place and make use of the `handleProgressEvent` everywhere that it was needed. Also, there's state involved: the max progress. Where should that be kept? Modifying state in a callback isn't a big deal in Javascript, but what if we're programming on a multi-threaded platform? We could create a class that encapsulates that state and syncronizes updates to it. These are solutions to this problem, but for similar problems **we're forced to build similar, additional solutions**.

{% asset_img better_way.gif There's got to be a better way! %}

## Finally. rxjs

What we're dealing with here, and in many other areas of most applications, are sequences of events. Most of us have been using higher-order functions to deal with sequences in an elegant and expressive way. For example, [LINQ in .NET](https://msdn.microsoft.com/en-us/library/bb397926.aspx) provides interfaces, extension methods, and even language constructs for processing sequences of data. [Lodash](https://lodash.com/) has been my go-to for the same functionality in Javascript. Many languages are providing these conveniences, but they tend to be centered around sequences of data that are synchronous in nature: we *pull* the next item from the sequence and if it's not available yet, then we wait for it. We even have language constructs for producing such sequences: `yield` in C# and Javascript.

What about processing event sequences, asynchronously? How can we process sequences in a way that ***doesn't require waiting for the next item***? This wasn't impossible before. Callbacks aren't new. And more recently, promise libraries and language constructs like `async/await` have helped us to reduce the perceived complexity of asynchronous programming. Those tools are nice, but don't offer the composition benefits of the higher order functions found in LINQ, Lodash, or similar. We need something more.

## Reactive Extensions (rx for short)

Rx, gives developers a way to use composable, higher-order functions to process sequences asynchronously. Our progress bar scenario is a perfect fit for this tool. Progress update events are being **pushed** to the client, and we have a need to discard some of the events in the sequence.

Rx has a ton of "operators". These are the functions that can be chained together to modify, combine, or aggregate sequences. The two in particular that I've found to be the most help for discarding irrelevant progress events are called [Scan](http://reactivex.io/documentation/operators/scan.html) and [a variant of Distinct called DistinctUntilChanged](http://reactivex.io/documentation/operators/distinct.html). But, to use the operators, we need a sequences to operate on. Rx calls  such a sequence an observable.

Observables can be created from arrays, iterables, DOM events, or callback APIs.

Why treat sequences of events any differently than the finite data sequences we have in memor finite sequences of data structures, or 

_talk about issue with out of order progrs events
 - out of order unexpected
 - signalr (diff story)
 - watned to find clean way to fix progress bars jumping around

 - what's rxjs
 - why it felt like a solution (nature of progress events (theyre streams of events))
 - progress display is inherently concurrent and how rxjs fits in with that
 - operators that I used (scan is key to this)
 - the simulation: interval, do, from(array), 
 
 
 <p data-height="268" data-theme-id="0" data-slug-hash="yevPee" data-default-tab="result" data-user="ronnieoverby" data-preview="true" class='codepen'>See the Pen <a href='http://codepen.io/ronnieoverby/pen/yevPee/'>yevPee</a> by Ronnie Overby (<a href='http://codepen.io/ronnieoverby'>@ronnieoverby</a>) on <a href='http://codepen.io'>CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>