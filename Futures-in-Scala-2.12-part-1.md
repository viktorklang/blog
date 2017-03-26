# Futures in Scala 2.12 (part 1)

This is the first of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.

## Missing canonical combinators: flatten

Are you one of us Future-users who have grown tired of the old `flatMap(identity)` boilerplate for un-nesting `Future`s as in:

~~~scala
val future: Future[Future[X]] = ???
val flattenedFuture /*: Future[X] */ = future.flatMap(identity)
~~~

Then I have some great news for you!
Starting with Scala 2.12 `scala.concurrent.Future` will have a `flatten`-method with the following signature:

~~~scala
def flatten[S](implicit ev: T <:< Future[S]): Future[S]
~~~

Allowing you to write:

~~~scala
val future: Future[Future[X]] = ???
val flattenedFuture /*: Future[X] */ = future.flatten
~~~

### Benefits:

1. Less to type
2. Less to read
3. Does not require any `ExecutionContext`

**Bonus**: Doesn't allocate a function instance as `flatMap(identity)` does:

~~~scala
scala> def sameInstance[T](first: T => T, second: T => T) = first eq second
sameInstance: [T](first: T => T, second: T => T)Boolean

scala> sameInstance[Int](identity, identity)
res0: Boolean = false
~~~

[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

[Click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-2.md) for the next part in this blog series.

Cheers,
√
