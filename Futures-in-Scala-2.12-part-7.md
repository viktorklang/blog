#Futures in Scala 2.12 (part 7)

This is the seventh of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.
For the previous post, [click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-6.md).

##Prepare for Deprecation

`ExecutionContext.prepare` has been deprecated without replacement (at this time)—it was ill-specced and it was too easy to forget to call it, or even know when to call it, or call it more times than needed.

(If you have ideas for how to propagate context across asynchronous boundaries, or want to participate in coming up with a replacement, I'd like to hear from you! :smile:)

##Missing BlockContext.defaultBlockContext

I'd like to think that [`scala.concurrent.BlockContext`](http://www.scala-lang.org/api/2.12.0-M2/index.html#scala.concurrent.BlockContext) is well-known, but I know for a fact that it isn't. `BlockContext` is the mechanics which allows to hook in `blocking {}`-blocks into the `ExecutionContext` which executes the code, and allows it to take actions to prevent deadlocking or starvation.

With `ForkJoinPool`-based `ExecutionContext`s it would for instance hook into the [`ManagedBlocker`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.ManagedBlocker.html) functionality, which spawns additional threads to take care of existing work to prevent stalls and unbounded starvation.

If no `BlockContext` is installed, a default one is used, and previously it was impossible to get to that instance from outside of the Scala Standard Library, and so `BlockContext.defaultBlockContext` has been added.

This is almost exclusively needed if you write your own `ExecutionContext` implementation, or you want to override the behavior of the currently installed `BlockContext` to use the default behavior, as in:

~~~scala
BlockContext.withBlockContext(BlockContext.defaultBlockContext) {
  someMethodWhichUsesBlocking() // Will use the default BlockContext
}

//vs.

someMethodWhichUsesBlocking() // Will use the currently installed BlockContext

//For reference
def someMethodWhichUsesBlocking(): Unit = blocking {
  println("foo")
}
~~~

Perhaps not the «coolest» of features, but when you need it, it is now available!

##Hardening ExecutionContext.global
A common issue with `ExecutionContext(.Implicits).global` when used with `blocking{}` was that the number of extra threads was virtually unbounded, and this in combination with the potential of having nested `blocking{}`-calls triggering the spawning of multiple additional `ForkJoinWorkerThread`s meant that things could go horribly wrong—think `OutOfMemoryError`-wrong.

For that reason 3 things have been put in place:

1. `ExecutionContext(.Implicits).global` now has a property to control the maximum number of threads concurrently existing as a result of managed blocking. The default is set to 256 Threads but can be changed by configuring the following System Property: `scala.concurrent.context.maxExtraThreads`
This means that `global` will have at most `scala.concurrent.context.maxThreads` + `scala.concurrent.context.maxExtraThreads` concurrent Threads.

2. We fixed so that nested `blocking{}`-blocks would not trigger subsequent extra thread creation.

3. Thanks to [Jessica Kerr](https://twitter.com/jessitron) we also improved the thread names for `global`. New format is: `scala-execution-context-global-${Thread.getId}`

##Bonus: Refactors

I apologize in advance if this is «boring», but I feel like it is an important thing! Having [`transform`](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-3.md) and [`transformWith`](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-4.md) in `Future` (at last!). It meant that I was able to encode most other combinators directly on top of them.

That means that the `scala.concurrent.Future` trait does not create *any* `Promise` directly, which means that the implementor of `Future` is in full control over which implementation of `Promise` will be used.

##Bonus: Self Control

Nobody would try to complete a Promise with its own Future, right?
Right?! :disappointed:

Soooo, self-checks were added in `completeWith` and `tryCompleteWith` to guard against cycles like:

~~~scala
val p = Promise[Foo]
p.completeWith(p.future) // OHNOES
//or
p.tryCompleteWith(p.future) // OOOOPS
~~~

So, now doing that is a no-op rather than waiting for a miracle to happen.

[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

To comment on the blog post itself, [click here](https://github.com/viktorklang/blog/pull/10/files).

Stay tuned for many more updates about Futures in Scala 2.12!

Cheers,
√
