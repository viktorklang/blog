#Futures in Scala 2.12 (part 2)

This is the second of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.
For the previous post, [click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-1.md).

##Missing canonical combinators: zipWith

Being able to *join* two `Future`s together to produce a new `Future`
which contains a `Tuple` of the results of both has been available for a while with the `zip` operation, and this looks something like this:

~~~scala
val future1: Future[String] = …
val future2: Future[Int] = …
val zippedFuture: Future[(String, Int)] = future1 zip future2
~~~

But what if we don't want a `Tuple`? What if we want to *combine* the results
of the two `Future`s to create something else?

This typically means having to combine `zip` and `map`:

~~~scala
/* The reason why we use a «PartialFunction literal» (yes, I made that up)
 * here is because otherwise the code has to use the arguably uglier _._1 and
 * _._2 syntax for `Tuple` value extraction.
 */
val combinedFuture: Future[String] =
    future1 zip future2 map { case (string, int) => s"$string & $int" }
~~~

Now, if you're like me and you dislike allocations, tuples and lack of genericity, you'll see that `zip` is a specialization of `x.zipWith(y)(Tuple2.apply)`! Well, perhaps it helps if we share the signature of said `zipWith`:

~~~scala
def zipWith[U, R](that: Future[U])(f: (T, U) => R)(implicit executor: ExecutionContext): Future[R]
~~~

This allows us to express the example above as:

~~~scala
val combinedFuture: Future[String] =
    future1.zipWith(future2)((string, int) => s"$string & $int")
~~~

###Benefits:

1. Less to type
2. Less to read
3. Fewer allocations
4. Fewer asynchronous steps
5. More general than `zip`

[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

Stay tuned for many more updates about Futures in Scala 2.12!

Cheers,
√