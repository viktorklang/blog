#Futures in Scala 2.12 (part 5)

This is the fifth of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.
For the previous post, [click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-4.md).

##Deprecations: onSuccess and onFailure

Since its inception and subsequent inclusion in the Scala Standard Library, `Future` has had 3 distinctly identifiable callback methods. `onComplete`, `onSuccess`, and `onFailure`.

Now, you're perhaps asking yourself why it is that `onSuccess` and `onFailure` are special-cases of `onComplete`, operating on eiter side of success/failure? Well, at some point it was conceived as useful.

"But…", I hear you say, "isn't `foreach` *just* a total version of `onSuccess`?" Well, yes it is. So let's use that instead, or `onComplete`!


So, if you call `someFuture.onSuccess(somePartialFunction)` in Scala 2.12.x you'll get the following deprecation message:

~~~txt
use `foreach` or `onComplete` instead (keep in mind that they take total rather than partial functions)
~~~

And then I hear you thinking "But, if `onSuccess` is the partial version of `foreach`, doesn't that mean that `onFailure` is the partial version of `failed.foreach`?" YES—exactly that!

This is why `someFuture.onFailure(somePartialFunction)´ in Scala 2.12.x yields this deprecation message:

~~~txt
use `onComplete` or `failed.foreach` instead (keep in mind that they take total rather than partial functions)
~~~

Worth keeping in mind, as with all the callbacks & `foreach`—there is no guaranteed order of execution of callbacks attached to the same `Future`.
To illustrate:

~~~scala
val someFuture: Future[Missile] = …
someFuture.foreach(_.neutralize(PERMANENTLY))
someFuture.foreach(_.launch(target))
//Could be executed in any order and not unlikely in parallel
~~~

###Benefits:

1. Promotion of for-comprehension compatible API instead of callbacks
2. In the end there'll be fewer methods on Future, being less confusing as to what to use and when
3. `onComplete` then remains as a performance-optimization of (`transform`)[https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-3.md], not having to create `Future`s to return.


[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

To comment on the blog post itself, [click here](https://github.com/viktorklang/blog/pull/7/files) and comment on the PR.

[Click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-6.md) for the next part in this blog series.

Cheers,
√
