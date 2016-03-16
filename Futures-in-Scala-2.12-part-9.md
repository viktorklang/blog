#Futures in Scala 2.12 (part 9)

This is the ninth, and last, of several posts describing the evolution of `scala.concurrent.Future` in Scala `2.12.x`.
For the previous post, [click here](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-8.md).

##WARNING: ADVANCED TOPICS AHEAD


##Feature: Improved Promise Linking (I promise!)

###ACPS (no, not an abbreviation of ALLCAPS)
A really useful technique when working with `Future` is what I call `Async Continuation Passing Style` (ACPS), which was introduced to me by Marius Eriksen ([@marius](https://twitter.com/marius)) years ago, and while we're on the topic—Marius is awesome.

My *casual* definition of `ACPS` is the use of Future.flatMap/recoverWith to «Divide and Conquer»™ problems into what looks like ordinary recursive invocations. For Scala 2.12 and onwards, we also have [`transformWith`](https://github.com/viktorklang/blog/blob/master/Futures-in-Scala-2.12-part-4.md).

Let's say we want to create a method which will take a `List` of `Future` of `Int`s, and returns a `Future` of `List` of `Try` of `Int`. (Now that's quite a mouthful isn't it?!)

*(The astute reader will react in horror at such a specific method signature, and will immediately yell at me for not calling it by its widely accepted name of [`sequence`](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future$@sequence[A,M[X]<:TraversableOnce[X]](in:M[scala.concurrent.Future[A]])(implicitcbf:scala.collection.generic.CanBuildFrom[M[scala.concurrent.Future[A]],A,M[A]],implicitexecutor:scala.concurrent.ExecutionContext):scala.concurrent.Future[M[A]]). BUT THIS IS AN EXAMPLE, PEOPLE! ^^)*

~~~scala
import scala.concurrent._
import scala.util.{Try, Success}

//Define type aliases to cut down on the bracket-madness
//for this blog post format
type ListOfFutures = List[Future[Int]]
type FutureListOfTrys = Future[List[Try[Int]]]

def results(l: ListOfFutures)(implicit ec: ExecutionContext): FutureListOfTrys = {
  // An inner method which allows us to recursively decompose
  // the problem and build up the solution
  // `f` is our current result, and `remaining` is what we have left to do
  def acps(f: FutureListOfTrys, remaining: ListOfFutures): FutureListOfTrys =
    remaining match {
      case Nil => f // When we hit the end of the list, we're done
      case r :: tail =>
        f.flatMap(list =>
          // as `f` completes, use the current result `list`
          // and when `r` completes, add its result to `list`
          // and carry forward `tail` as the `remaining` for the next `acps`
          acps(r.transform(result => Success(result :: list)), tail))
        )
    }
  // Our initial result is `Nil`, and initially `l` is what is `remaining`
  // and we need to `reverse` the result once it is done since
  // we are building up the answer in reverse order (List)
  acps(Future.successful(Nil), l).map(_.reverse)
}
~~~

Alright, I don't know about you, but for me the first time I saw something like this it was some pretty mind-blowing stuff so if this is your first experience with it, take a couple of minutes to let it sink in, then have a look at the example below.

~~~scala
scala> val example = {
     |   import ExecutionContext.Implicits._
     |   List(Future(5),
     |        Future[Int](throw new Exception("foo")),
     |        Future("pigdog".hashCode))
     | }
example: List[scala.concurrent.Future[Int]] = List(Future(Success(5)), Future(Failure(java.lang.Exception: foo)), Future(Success(-988364562)))

scala> results(example)(ExecutionContext.global)
res0: FutureListOfTrys = Future(<not completed>)

scala> res0 // Or do `res0 foreach println`
res1: FutureListOfTrys = Future(Success(List(Success(5), Failure(java.lang.Exception: foo), Success(-988364562))))
~~~

*Hopefully* I have been able to illustrate how one can use ACPS to decompose problems recursively, without building up a callstack, and being more in line with [trampolining](https://en.wikipedia.org/wiki/Trampoline_(computing)).


###Promise Linking 

Now, if an implementation of `Future` is not careful, it is easy to create a space-leak with ACPS code.

The following example is an adaptation of the [regression test for SI-7336](https://github.com/scala/scala/blob/2.12.x/test/files/run/t7336.scala)

~~~scala
import scala.concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits._

val arraySize = 1000000 // Pick a large number
val tooManyArrays = (Runtime.getRuntime().totalMemory() / arraySize).toInt * 100 // Make sure that we will hit OOME if Promise Linking doesn't work

// An example of possible space leak using ACPS by virtue of `recoverWith`:
def loopRW(i: Int, arraySize: Int): Future[Unit] = {
  val array = new Array[Byte](arraySize)
  Future(throw new Exception).recoverWith { case _ =>
    if (i == 0) {
      Future(())
    } else {
      array.size // Force closure to refer to array
      loopRW(i - 1, arraySize)
    }
  }
}
~~~

The code above *may* warrant some explaining.

The `array.size`-line is an example of using `array` in the closure. This forces it to be retained by the generated closure which is passed to `flatMap`/`recoverWith`. 

The shape of the solutions which use ACPS is a «Chain of Futures» where the returned `Future` is only completed once the whole chain is finished, which creates the risk of «space leaks». (No, this is not about interstellar plumbing)

These two interact to create a space leak that grows over the Chain. This is even more problematic for chains of unbounded length.

`Promise Linking` is a solution to this problem—
since we know that a Future returned from `flatMap`/`recoverWith`/`transformWith` can *only* be completed by the `Future` returned by the function passed to it, we can implement something very similar to the concept of [Tail Call Optimization](http://c2.com/cgi/wiki?TailCallOptimization) but for ACPS.

*This elides the intermediate `Promise`s in this `Chain of Future`s and instead directly completes the `Promise` associated with the `Future`.* This involves migrating registered callbacks to the «root» Promise—the one which started the chain.

~~~scala
loopRW(tooManyArrays, arraySize) onComplete println
~~~

Outcome:

  * Fails (OutOfMemoryError) on 2.10.6 and 2.11.8
  * Succeeds on 2.12.0-M3
  
The reason for failing on 2.10.6 and 2.11.8 is that `Promise Linking` is for those versions only supported for `flatMap`, not `recoverWith`. And `transformWith` does not exist in those versions. In 2.12.x Promise Linking is implemented for `transformWith` which both `flatMap` and `recoverWith` uses, so they get the feature for free.

###Benefits

1. ACPS is a very useful way of encoding solutions
2. Promise Linking addresses a certain class of space leaks introduced by ACPS
3. 2.12 has Promise Linking properly implemented across the board

PS. `scala.collection.immutable.List` is a [Stack](https://twitter.com/viktorklang/status/709460948130598912)

[Here's the RSS feed of this blog](https://github.com/viktorklang/blog/commits/master.atom) and—as I love feedback—please [share your thoughts](https://github.com/viktorklang/blog/issues/3).

To comment on the blog post itself, [click here](https://github.com/viktorklang/blog/pull/12/files).

I hope you've enjoyed this series of blog posts on Scala Futures for 2.12!

Cheers,
√
