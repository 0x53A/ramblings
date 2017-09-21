# Do Everything On The UI Thread
------------

_Go [back](index)_

__THIS IS CURRENTLY WORK IN PROGRESS__


Different problems need different solutions. This post is written from my experience with smallish MVVMish C# Apps.

#### GUIs and Threads, oh my

Most UI frameworks (well at least on Windows) have a concept of Thread Affinity. Some actions are only allowed on one specific dedicated thread.
This of course causes issues. You can only update the UI on the UI-Thread, but if you do long-running work on that thread, you block any updates to the UI and it becomes unresponsive.


If you read this post, then you know all this already.

#### Solution

The solution is of course to move anything long-running off the UI thread. In the olden times you had great things like BackgroundWorkers, and Threads, and because creating a Thread has lots of overhead, there is also a shared ThreadPool. Now, moving things off the UI thread is easy. The challenge then becomes updating the UI. Because you are on a random thread in the ThreadPool. And can't interact with the UI.

The addition of async/await was probably in large parts motivated by this UI misery.


#### Applying it to different paradigms

What I wrote above was targeted at MVVM in C#. My recent experience with functional UI frameworks convinces me that my intuition above was right and that everything controllerish should be dispatched to a single thread instead of fiddling with flags and locks.
There are already great in-depth explanations of the Elm model, but they don't do it justice.

If you can read F#, I would highly recommend to look at [the samples](https://fable-elmish.github.io/#samples/TodoMVC).

In short, in Elm, any interaction doesn't update anything directly like you would in C#. Instead, an event is dispatched. This has a similar (but the execution is way better than in Mvvm!) result like my guidance above: Everything important is handled synchronously!

_Go [back](index)_
