#Futures in Scala 2.12 (part 4)

This is the third of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.
For the previous post, [click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-3.md).

##Missing canonical combinators: transformWith

As we saw in the previous post, `transform` provides a nice unification of both `map` and `recover` on `Future`, and I know what you're thinking now: "What about `flatMap` and `recoverWith`?"

Say no more! `transformWith` is the answer, and has the following lovely signature:

~~~scala
def transformWith[S](f: Try[T] => Future[S])(implicit executor: ExecutionContext): Future[S]
~~~

Remember to use the solution with the least power which will suffice for the task at hand, so prefer to use the `flatMap`s and the `recoverWith`s primarily, opting for the `transformWith` as required.

And, before you say anything, yes, `flatMap` and `recoverWith` are implemented in terms of `transformWith` in Scala 2.12!

###Benefits:

1. Ultimate power, for when you require it

[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

To comment on the blog post itself, [click here](https://github.com/viktorklang/blog/pull/4) and comment on the PR.

Stay tuned for many more updates about Futures in Scala 2.12!

Cheers,
√
