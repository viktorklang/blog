#Futures in Scala 2.12 (part 6)

This is the sixth of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.
For the previous post, [click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-5.md).

##Missing utilities: unit & never

Something I've always felt was missing is having a «zero» for `Future`—or more frequently called a «default instance» of a `Future`.

What's *nice* about having this is that, technically, `Future.apply[T](logic: => T)(implicit ec: ExecutionContext)` could be viewed as, and implemented like:

~~~scala
def apply[T](logic: => T)(implicit ec: ExecutionContext): Future[T] =
  unit.map(_ => logic)
~~~

Another important scenario, which wasn't really readily solvable unless you were comfortable with implementing it yourself was a `Future` which would never complete.

Now, the naïve implemention of said `Future` would look something like:

~~~scala
val neverCompletingFuture: Future[Nothing] = Promise[Nothing].future
~~~

Take a few seconds to think about the following question: In what ways would that solution be undesirable?

…

…

…

…

Hint: What happens with logic which gets added to `neverCompletingFuture`?
Example:

~~~scala
val fooFuture = neverCompletingFuture.map(_ => "foo")

//or

neverCompletingFuture.onComplete(println)
~~~

Well, what happens is that the logic is *packaged* and added to the `neverCompletingFuture` instance in order to be executed when `neverCompletingFuture` completes (which is never), and now we have a hard to spot invisible memory leak!

So, in order to support the case when you want to be able to represent a `Future` which never completes, but also doesn't leak memory when used as a plain `Future`, use `Future.never`.

###Benefits:

1. Future.unit is a «zero» instance which is cached
2. Future.never removes the risk of memory leaks when used as a `Future` instance which never completes

[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

To comment on the blog post itself, [click here](https://github.com/viktorklang/blog/pull/8/files).

Stay tuned for many more updates about Futures in Scala 2.12!

Cheers,
√
