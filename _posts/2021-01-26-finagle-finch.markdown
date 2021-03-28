---
layout: post
date:   2021-03-28 09:09:00 +0200
categories: Scala Finagle Finch
title: Learning about Finch
subtitle: Functional bliss with Finch
background: '/img/finch.JPG'  
--- 

## About the post

I previously wrote a [post] where I tried to figure out what Finagle HTTP services is all about, more precisely I tried to implement a small REST Api (the classical Todo app). When writing the post I discovered that most people are using [Airframe] or [Fitch] instead of vanilla Finagle when doing such a task.

This post is a continuation of the previous post where I will look at Finch and try to use it to implement the same REST Api once again.  

Or that was what I thought, however as luck would have it there already exists a nice [Todo] example in the Finch repository. Hence I decided to implement the Candy Machine (yeah, you know the one from the [Red book]) using Finch instead of using a free monad as I did in this [previous post].

The implementation is heavily inspired by the by the [Todo] example mentioned above, so feel free to go directly to the source. But be warned, you will not be finding funny things like this gif there:

![Funny!](https://media.giphy.com/media/3o7ZeAiCICH5bj1Esg/giphy.gif) 

This is not an expert post about Finagle/Finch and it's echo system, it's my attempt to get an understanding what Finagle/Finch is all about.
   
## What is Finch?

> Finch is a thin layer of purely functional basic blocks atop of Finagle for building composable HTTP APIs. Its mission is to provide the developers simple and robust HTTP primitives being as close as possible to the bare metal Finagle API.
   
> Starting with 0.25, Finch artifacts are published for both Twitter Futures (Endpoint[A]) and Cats Effects (Endpoint[F[_], A]):

In this post we will expose a REST API using Cats Effects so expect to get exposed to some Cats thingies.     
   
## The rules for the Candy Machine

Here are the rules for the candy machine:

> * Inserting a coin into a locked machine will cause it to unlock if there's any candy left.
> * Turning the knob on an unlocked machine will cause it to dispense candy and become locked.
> * Turning the knob on a locked machine or inserting a coin into a unlocked machine does nothing.
> * A machine that's out of candy ignores all inputs.

And in code this looks like:

{% highlight scala %}
case class MachineState(id: Int, locked: Boolean, candies: Int, coins: Int)

object CandyRule {
  def applyRule(input: Input)(machine: MachineState): Either[Throwable, MachineState] = input match {
    case Coin =>
      if (machine.candies <= 0) {
        Left(new IllegalStateException("No candies left"))
      } else if (machine.locked) {
        Right(machine.copy(locked = false, coins = machine.coins + 1))
      } else {
        Left(new IllegalStateException("A coin has already been disposed"))
      }
    case Turn =>
      if (machine.candies <= 0) {
        Left(new IllegalStateException("No candies left"))
      } else if (!machine.locked) {
        Right(machine.copy(locked = true, candies = machine.candies - 1))
      } else {
        Left(new IllegalStateException("No coin has been disposed"))
      }
  }
}
{% endhighlight %} 

The `MachineState` is an immutable object. Note how we use `copy` to create a a new version.

The applyRule function either return a success containing an updated machine state or a failure.

## Where to keep the state?

We will keep the state in memory. The state will be injected into the application to support our tests.

{% highlight scala %}
class App(
           idRef: Ref[IO, Int],
           storeRef: Ref[IO, Map[Int, MachineState]]
         )
{% endhighlight %} 

The idRef contains the id that will be assigned to the machine up on creation.

The storRef contains a immutable map of id, MachineState pairs, each MachineState (defined above) contains the state for a single CandyMachine.

## About the IO monad

Before we dive into the Candy Machine lets talk a bit about the IO monad

> A value of type IO[A] is a computation which, when evaluated, can perform effects before returning a value of type A.

The IO monad makes it easy to distinguish between the parts of our code that handles side effects and the parts that are not. It's only possible to access impure code from code that is also marked as IO. 

An example is in order, consider this impure code:

{% highlight scala %}
  def sayHello(): Unit = println("Hello world")

  sayHello();
{% endhighlight %} 

This code is valid scala syntax and will immediately print to the console.

Contrast this will the following:

{% highlight scala %}
  def sayHelloWithIO(): IO[Unit] = IO {
    println("Hello with IO")
  }

  val program1: IO[Unit] = sayHelloWithIO()
{% endhighlight %} 

Here we have marked the impure code with the IO monad.

If we run this code absolutely nothing will be printed to the console. What we get as a result from the call to the function is a small program that if executed will print "Hello with IO" to the console.

To get the output in the console we have to add the following line `program1.unsafeRunSync()`.

Here is a slightly larger example, a number guessing game:

{% highlight scala %}
trait Console {
  def printLn(s: String): Unit
  def readLine(): String
}

class ConsoleImpl extends Console {
  override def printLn(s: String): Unit = println(s)

  override def readLine(): String = scala.io.StdIn.readLine()
}

val rnd = new scala.util.Random()
def guessTheNumberWithIO(console: Console)(answer: Int): IO[Unit] = {

  def printLn(s: String): IO[Unit] = IO {
    console.printLn(s)
  }

  def readLn(): IO[String] = IO {
    console.readLine()
  }

  def loop(numberOfGuesses: Int): IO[Unit] = for {
    line <- printLn("Guess of a number between 0 and 1000: ") >> readLn()
    _ <- {
      val guess = line.toInt
      if (guess == answer) {
        printLn(s"You are correct, the number is: $answer. You got it in $numberOfGuesses guesses.")
      } else if (guess < answer) {
        printLn(s"Your guess is too low.") >> loop(numberOfGuesses + 1)
      } else {
        printLn(s"Your guess is too high.") >> loop(numberOfGuesses + 1)
      }
    }
  } yield ()

  loop(1)
}

val program2: IO[Unit] = guessTheNumberWithIO(new ConsoleImpl)(rnd.nextInt(1001))
program2.unsafeRunSync()
{% endhighlight %} 

In the code above we ask the user to guess a number. The number is read from the console using `readLine`. Since we have marked the `readLine` with the IO monad it is only possible to access the value from inside the context of IO.

By the way `>>` in the code example is an alias for `flatMap`.

In our Candy maching we will have a mix of impure code and pure code. The `CandyRule` above is an example of pure code. Our in memory state managament is on the other hand example of impure code. 

## About the Ref

Before we come back to the Candy service lets take another detour and do a proper investigation of the Ref.

The [Ref] is a: 

> An asynchronous, concurrent mutable reference.

Which has operators that:

> Provides safe concurrent access and modification of its content

{% highlight scala %}
abstract class Ref[F[_], A] {
  def get: F[A]
  def set(a: A): F[Unit]
  def modify[B](f: A => (A, B)): F[B]
  // ... and more
}
{% endhighlight %}

We will investigate how the Ref works by using the following small function. The core functionality of the function is to increase a number stored in the Ref by one. I havel also added a lot of traces so that we can see what's going on. 

{% highlight scala %}
  def printLn(s: String): IO[Unit] = 
    IO(println(s"${Thread.currentThread()}: $s"))

  def addOne(ref: Ref[IO, Int]): IO[Unit] = for {
    v1 <- ref.get
    _ <- printLn(s"Before modify (get): $v1")
    v2 <- ref.modify(v => {
      println(s"${Thread.currentThread()}: Inside modify: $v");
      (v + 1, v)
    })
    _ <- printLn(s"Result from modify: $v2")
    v3 <- ref.get
    _ <- printLn(s"After modify (get): $v3")
  } yield ();
{% endhighlight %}

Please observe that I'm a little bit naughty when adding that `println` inside the modify callback without wrapping it in an `IO`.

![Naughty!](https://media.giphy.com/media/nXU1FF5HS2eFG/giphy.gif) 

Here is a small program that uses the `addOne` function. The result of the program should be that we have added three to the original number.

{% highlight scala %}
  def program1(ref: Ref[IO, Int]): IO[Unit] = for {
    _ <- addOne(ref)
    _ <- addOne(ref)
    _ <- addOne(ref)
  } yield ()

  val myRef: Ref[IO, Int] = Ref.unsafe[IO,Int](42)
  program1(myRef).unsafeRunSync()
{% endhighlight %}

And here is the trace: 

{% highlight scala %}
  Thread[main,5,main]: Before modify (get): 42
  Thread[main,5,main]: Inside modify: 42
  Thread[main,5,main]: Result from modify: 42
  Thread[main,5,main]: After modify (get): 43
  Thread[main,5,main]: Before modify (get): 43
  Thread[main,5,main]: Inside modify: 43
  Thread[main,5,main]: Result from modify: 43
  Thread[main,5,main]: After modify (get): 44
  Thread[main,5,main]: Before modify (get): 44
  Thread[main,5,main]: Inside modify: 44
  Thread[main,5,main]: Result from modify: 44
  Thread[main,5,main]: After modify (get): 45
{% endhighlight %}

What can we learn from the trace? 

Well, `get` obviously returns the current value in the Ref. 

On the other hand `modify` is a little bit more complicated, here is the signature again:

`def modify[B](f: A => (A, B)): F[B]`

So the modify function will return whatever we return as the second value in the tuple, in this case the current value of Ref. The first value of the tuple is used to modify the Ref, in this case to increase a number by one. 

Ok, lets find out how 'addOne' behaves in a concurrent context.

{% highlight scala %}
  val myRef2: IO[Ref[IO, Int]] = Ref[IO].of(42)

  def program3(ref: IO[Ref[IO, Int]]): IO[Unit] = for {
    r <- ref
    _ <- List(
      addOne(r),
      addOne(r),
      addOne(r)
    ).parSequence
  } yield ();

  program3(myRef2).unsafeRunSync()
{% endhighlight %}

And here is the new trace:

{% highlight scala %}
  Thread[scala-execution-19,5,main]: Before modify (get): 42
  Thread[scala-execution-21,5,main]: Before modify (get): 42
  Thread[scala-execution-20,5,main]: Before modify (get): 42
  Thread[scala-execution-19,5,main]: Inside modify: 42
  Thread[scala-execution-21,5,main]: Inside modify: 42
  Thread[scala-execution-20,5,main]: Inside modify: 42
  Thread[scala-execution-21,5,main]: Inside modify: 43
  Thread[scala-execution-20,5,main]: Inside modify: 43
  Thread[scala-execution-20,5,main]: Inside modify: 44
  Thread[scala-execution-20,5,main]: Result from modify: 44
  Thread[scala-execution-21,5,main]: Result from modify: 43
  Thread[scala-execution-19,5,main]: Result from modify: 42
  Thread[scala-execution-19,5,main]: After modify (get): 45
  Thread[scala-execution-20,5,main]: After modify (get): 45
  Thread[scala-execution-21,5,main]: After modify (get): 45
{% endhighlight %}

What can we learn from the trace? 

Well, `get` obviously still returns the current value in the Ref. Since three threads are running concurrently we see that the value actually increased from 42 to 45, this may of course vary between different executions of the program. Here for example I was only allocated two threads when running the program:

{% highlight scala %}
  Thread[scala-execution-19,5,main]: Before modify (get): 42
  Thread[scala-execution-19,5,main]: Inside modify: 42
  Thread[scala-execution-19,5,main]: Result from modify: 42
  Thread[scala-execution-19,5,main]: After modify (get): 43
  Thread[scala-execution-19,5,main]: Before modify (get): 43
  Thread[scala-execution-19,5,main]: Inside modify: 43
  Thread[scala-execution-19,5,main]: Result from modify: 43
  Thread[scala-execution-20,5,main]: Before modify (get): 44
  Thread[scala-execution-19,5,main]: After modify (get): 44
  Thread[scala-execution-20,5,main]: Inside modify: 44
  Thread[scala-execution-20,5,main]: Result from modify: 44
  Thread[scala-execution-20,5,main]: After modify (get): 45
{% endhighlight %}

We can also see that the callback inside `modify` is called more then three times (in the first trace), this since `modify` detects that there is several threads that tries to update the same memory position at the same time and hence tries to modify again if there was a conflict.

The result from modify is still always the current value of the Ref, in this case 42, 43 and 44. 

The code [snippets] are available at github.


## The API

Alright, now that we have a proper understanding of the IO-monad and Ref lets continue with implementing the Candy Machine. 

The service provides the following API:

{% highlight scala %}

// create a new machine
POST /machine

// give back the state of all machines
GET /machine

// insert a coin
PUT /machine/{id}/coin

// turn the handle to dispose a candy
PUT /machine/{id}/turn

{% endhighlight %}

## Create a machine

Ok, lets begin by creating a machine. To do this we have to: 
* Map `POST /machine` to our code
* Decode the request body to get the initial state of the machine
* Get hold of the id that we should assign the new machine
* Add the new machine to the store

{% highlight scala %}

case class MachineState(id: Int, locked: Boolean, candies: Int, coins: Int)

final val assignId: Endpoint[IO, MachineState] =
    jsonBody[Int => MachineState]
        .mapAsync(pt => idRef.modify(id => (id + 1, pt(id))))

final val createMachine: Endpoint[IO, MachineState] = 
    post(path("machine") :: assignId) { m: MachineState =>
        storeRef.modify { store =>
            (store + (m.id -> m), Created(m))
        }
    }
{% endhighlight %}

The Rest method accepts a payload that contains the initial `MachineState` as json. We use the `idRef` to assign a unique id to the machine. We then adds the new machine to the in memory store (`storeRef`). 

I want to highlight what's going on in the `assignId` function:

{% highlight scala %}
final val assignIdVerbose: Endpoint[IO, MachineState] =
  jsonBody[Int => MachineState].mapAsync((pt: Int => MachineState) => {
    idRef.modify(id => (id + 1, pt(id)))
  })
{% endhighlight %}

Please note that the input argument of `mapAsync` is a function with the following signature `Int => MachineState`, we later use this function to produce a MachineState that contains the assigned id.  

Maybe we would like to also assign the value true to `locked`. This is an easy fix, all we have to do is the change the type of `jsonBody` to `(Int, Boolean) => MachineState`. Now we receive a function that makes it possible to assign both the id and locked. Pretty cool! 

{% highlight scala %}
final val assignId: Endpoint[IO, MachineState] =
  jsonBody[(Int, Boolean) => MachineState]
    .mapAsync(pt => idRef.modify(id => (id + 1, pt(id, true))))
{% endhighlight %}

## List all machines

Now we are ready to list the state of all machines. This time we need to:
* Map `GET /machine` to our code
* Access the in memory store to get a list of all created machines

{% highlight scala %}
final val getMachines: Endpoint[IO, List[MachineState]] = 
  get(path("machine")) {
    storeRef.get.map(m => Ok(m.values.toList.sortBy(_.id)))
  }
{% endhighlight %}

Nothing much to say about this, we simply accesses the `storeRef` with `get` and then wrap the returned list in `Ok`.

## Insert a coin

Time to insert a coin. We need to:
* Map `PUT /machine/{id}/coin` to our code
* Extract `id` from the path
* Verify that we have a machine that matches `id`
* Verify that the machine is in the correct state
* Modify the state of the machine 

{% highlight scala %}

final val insertCoin: Endpoint[IO, MachineState] = 
  put(path("machine") :: path[Int] :: path("coin")) {
    handleInput(Coin)
  }

private def handleInput(input: Input): Int => IO[Output[MachineState]] = {
  id: Int => {
    storeRef.modify { updateMachineState(input, id)}
  }
}

private def updateMachineState(input: Input, id: Int)
  : Map[Int, MachineState] => (Map[Int, MachineState], Output[MachineState]) = {
    store: Map[Int, MachineState] =>
      val result = for {
        machine <- store.get(id).toRight(new NoSuchElementException)
        newMachine <- CandyRule.applyRule(input)(machine)
      } yield (newMachine)

      result match {
        case Right(m) => (store + (id -> m), Ok(m))
        case Left(e: NoSuchElementException) => (store, Output.empty(Status.NotFound))
        case Left(e: IllegalStateException) => (store, Output.empty(Status.BadRequest))
      }
}

case class MachineState(id: Int, locked: Boolean, candies: Int, coins: Int)

object CandyRule {
  def applyRule(input: Input)(machine: MachineState): Either[Throwable, MachineState] = input match {
    case Coin =>
      if (machine.candies <= 0) {
        Left(new IllegalStateException("No candies left"))
      } else if (machine.locked) {
        Right(machine.copy(locked = false, coins = machine.coins + 1))
      } else {
        Left(new IllegalStateException("A coin has already been disposed"))
      }
    case Turn =>
      if (machine.candies <= 0) {
        Left(new IllegalStateException("No candies left"))
      } else if (!machine.locked) {
        Right(machine.copy(locked = true, candies = machine.candies - 1))
      } else {
        Left(new IllegalStateException("No coin has been disposed"))
      }
  }
}
{% endhighlight %}

The usage of `IO` makes it easy to separate the impure (`insertCoin` and `handleInput`) from the pure code (the rest).

## Start the application

Time to start the application. We need to:
* Connect the service to a port
* Create our `Ref's`
* Create the `App` that contains application code
* Start the service 

{% highlight scala %}
object Main extends TwitterServer {

  private val port: Flag[Int] = flag("port", 9080, "TCP port for HTTP server")

  def main(): Unit = {

    val server = for {
      id <- Ref[IO].of(0)
      store <- Ref[IO].of(Map.empty[Int, MachineState])
    } yield {
      val app = new App(id, store)(IO.contextShift(ExecutionContext.global))
      val srv = Http.server.withStatsReceiver(statsReceiver)

      srv.serve(s":${port()}", app.toService)
    }

    val handle = server.unsafeRunSync()
    onExit(handle.close())
    Await.ready(adminHttpServer)
  }
}
{% endhighlight %}

If you like you can now start the application. The complete code is available at [github]. If you like to do some manual testing the payloads for creating a machine should look something like this:

{% highlight scala %}
{
	"locked": true,
	"candies": 10,
	"coins": 0
}
{% endhighlight %}


## Test the application

The service is ready for usage, but lets write some tests before ending this post. I will only show the tests for when we create a machine, but the rest of the [tests] are available at github.

{% highlight scala %}
class AppTest extends AnyFlatSpec with Matchers {


  private case class AppState(id: Int, store: Map[Int, MachineState])

  private case class TestApp(
                              id: Ref[IO, Int],
                              store: Ref[IO, Map[Int, MachineState]]
                            ) extends App(id, store)(IO.contextShift(DummyExecutionContext)) {
    def state: IO[AppState] = for {
      i <- id.get
      s <- store.get
    } yield AppState(i, s)
  }

  private case class MachineWithoutId(locked: Boolean, candies: Int, coins: Int) {
    def withId(id: Int): MachineState = MachineState(id, locked, candies, coins)
  }

  private val genMachineWithoutId = for {
    locked <- Gen.oneOf(true, false)
    candies <- Gen.choose(0, 3)
    coins <- Gen.choose(0, 1000)
  } yield MachineWithoutId(locked, candies, coins)

  private def genTestApp: Gen[TestApp] =
    Gen.listOf(genMachineWithoutId).map { machines =>
      val id = machines.length
      val store = machines.zipWithIndex.map { case (m, i) => i -> m.withId(i) }

      TestApp(Ref.unsafe[IO, Int](id), Ref.unsafe[IO, Map[Int, MachineState]](store.toMap))
    }

  private implicit def arbitraryTodoWithoutId: Arbitrary[MachineWithoutId] = Arbitrary(genMachineWithoutId)

  private implicit def arbitraryApp: Arbitrary[TestApp] = Arbitrary(genTestApp)

  it should "create a machine" in {
    check { (app: TestApp, machine: MachineWithoutId) =>
      val input = Input.post("/machine").withBody[Application.Json](machine)

      val shouldBeTrue: IO[Boolean] = for {
        prev <- app.state
        newMachine <- app.createMachine(input).output.get
        next <- app.state
      } yield prev.id + 1 == next.id &&
        prev.store + (prev.id -> newMachine.value) == next.store &&
        newMachine.value == machine.withId(prev.id)

      shouldBeTrue.unsafeRunSync()
    }
  }
}
{% endhighlight %}

We are using [ScalaCheck] to verify that our code follows different properties. In the case of when we create a machine we assume that:
* The id is increased by one
* The store contains the new machine
* The the new machine has the same values that we sent in in the payload and that it got assigned the correct id.

To test with different inputs we are using ScalaCheck's mechanisms to generate the test data in a semi random way.

I have created [tests] for the rest of API as well, the are available at github.


## Conclusion

In my opinion `Finch` really delivered. It was quite cool to be able to write a web service using functional programming.

I will most definitely continue with my investigation of this framework. Things I'm interested in are for example how you save state in a database and how you call another web service etc.   

All the code are available at [github].

Happy coding!

[post]: https://morotsman.github.io/scala/finagle/http/serviec/2021/01/03/finagle-http-service.html
[Airframe]: https://wvlet.org/airframe/docs/airframe-http.html
[Fitch]: https://github.com/finagle/finch
[Todo]:https://github.com/finagle/finch/tree/master/examples/src/main/scala/io/finch/todo
[previous post]: https://morotsman.github.io/scala/cats/free/monad/cli/webservice/2020/09/07/free-monad-in-a-webservice.html
[Red book]: https://www.manning.com/books/functional-programming-in-scala
[Ref]: https://typelevel.org/cats-effect/concurrency/ref.html
[referential transparent]: https://en.wikipedia.org/wiki/Referential_transparency
[snippets]: https://github.com/morotsman/investigate_finagle/blob/main/src/main/scala/com/github/morotsman/ref/InvestigateRef.scala
[github]: https://github.com/morotsman/investigate_finagle/tree/main/src/main/scala/com/github/morotsman/investigate_finagle_service/candy_finch
[tests]: https://github.com/morotsman/investigate_finagle/blob/main/src/test/scala/com/github/morotsman/investigate_finagle_service/candy_finch/AppTest.scala
[ScalaCheck]: https://www.scalacheck.org/
