# Futures in Scala 2.12 (part 3)

This is the third of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.
For the previous post, [click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-2.md).

## Missing canonical combinators: transform

Now, if you're thinking "Hey Viktor, there's already a `transform`-method on `Future`!" then you're most definitely right. And I'll explain myself.

When the old `transform`<sup>[1](#transformNote)</sup> was added, many moons ago, it was a means of mapping over successes and failures. However, the astute reader might notice that it is asymmetric: an exception thrown when mapping over the successful case will result in a failed `Future`, but there is no way to turn a failed case into a successful one.

One way of looking at a `scala.concurrent.Future` is that it's an "`Eventually[Try[_]]`", in other words, a container which will eventually contain a `scala.util.Try` parameterized by some type.

What if we had a way to *transform* `Future` more generically, and symmetrically?

Perhaps its signature ought to look something like this:

~~~scala
def transform[S](f: Try[T] => Try[S])(implicit executor: ExecutionContext): Future[S]
~~~

So for instance, that would mean that `Future.map` could be implemented as follows (and is!):

~~~scala
  def map[S](f: T => S)(implicit executor: ExecutionContext): Future[S] = transform(_.map(f))
~~~

And `Future.recover` is possible to implement as this (and is!):

~~~scala
def recover[U >: T](pf: PartialFunction[Throwable, U])(implicit executor: ExecutionContext): Future[U] =
    transform { _ recover pf }
~~~

It also makes it *dead simple* to *lift* a `Future[T]` to a `Future[Try[T]]`:

~~~scala
val someFuture: Future[String] = …
val lifted/*: Future[Try[String]]*/ = someFuture.transform(Try(_))
~~~
 
Nice, right?


### Benefits:

1. Strictly more powerful / generic than the previous `transform`-method
2. Straightforward to consume/transform a `Future`s `Try[_]` without having to use `onComplete` or unboxing `Future.value`
3. For implementors of the `Future` trait, less methods to implement

[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

[Click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-4.md) for the next part in this blog series.

Cheers,
√

<a name="transformNote">1</a>:
  ~~~scala
  def transform[S](s: T => S, f: Throwable => Throwable)(implicit executor: ExecutionContext): Future[S]
  ~~~
