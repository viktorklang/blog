#Futures in Scala 2.12 (part 8)

This is the eighth of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.
For the previous post, [click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-7.md).

##Goodbye, sun.misc.Unsafe

A `Future` can be seen as a tri-state<sup>[1](#tristate)</sup> state machine with the following distinct states:

1. Uncompleted
2. Completed with a successful result
3. Completed with a failed result

In order to have an «asynchronous Future» it is vital to be able to register logic to be executed once the `Future` becomes Completed.

This means that we can encode the state `1` as a «sequence of callbacks & their `ExecutionContext`», state `2` as a `scala.util.Success`, and state `3` as a `scala.util.Failure`.

Given that, we only need to have a single `var` in our `Future`-implementation, which will either be a `Seq`, a `Success` or a `Failure`.

Since `Future` can both be completed and have new callbacks added *concurrently* we need to be able to access this `var` atomically, so simply making it `@volatile` won't be enough: we need *Compare-And-Set* semantics.

The first `Akka Future` which served as the main implementation inspiration for [SIP-14](http://docs.scala-lang.org/sips/completed/futures-promises.html), it used an inner field of type [`AtomicReference`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html).

In the initial `scala.concurrent.Future` design, instead of taking the cost of having to allocate an extra object for the `AtomicReference` and take the cost for that indirection, we used what's known as an [`AtomicReferenceFieldUpdater`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReferenceFieldUpdater.html) (ARFU).

Now, from a Scala perspective there are 2 problems with ARFUs: 1. They require the use of static fields—which Scala does not support, and, 2. They are about 10-20% slower than `sun.misc.Unsafe` access.

Since performance is always important in the Standard Library, we changed to use `sun.misc.Unsafe` for the implementation of `scala.concurrent.Future` and be OK with it requiring us to have a base-class in Java with a static field to hold the memory address offset of the field.

But! For 2.12.x we decided to take a «better» way, which eliminates the need for static fields and `sun.misc.Unsafe`:

We have now *completely replaced* the use of `sun.misc.Unsafe` for `DefaultPromise` (the internal implementation of the Scala Standard Library Promises) with *extending* [`AtomicReference`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html) internally, this means that there is no need for a base-class in Java and no need for ARFUs or `sun.misc.Unsafe`.

###Benefits

1. The same, excellent, performance as previously
2. With much better platform compatibility and security

<a name="tristate">1</a>: In practice it turns out to be a four-state state machine due to the feature of «Promise Linking», we'll cover that topic in the next post.

[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

To comment on the blog post itself, [click here](https://github.com/viktorklang/blog/pull/11/files).

[Click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-9.md) for the next part in this blog series.

Cheers,
√
