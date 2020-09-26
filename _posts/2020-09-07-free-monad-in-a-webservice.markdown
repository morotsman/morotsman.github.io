---
layout: post
date:   2020-09-07 09:09:00 +0200
categories: Scala Cats Free Monad Cli WebService
title: The candy machine
subtitle: Implementing a web service and cli tool using the free monad
background: '/img/hamburg3.jpg'  
---

## About the post

In this post we will expand on the exercise 6.11 in [Functional programming in Scala]. The exercise invites the reader to use the State monad to implement a finite state automaton. The automaton simulates a candy machine. 

Here we will use the Free monad instead. We will also try to use our program in different contexts, first in a CLI-tool and then in a web service.

In the post I will skip some of the boilerplate code but the fully working example (including boilerplate and some tests) are available at [github].

## What is a Free Monad?

This post assumes that you already have some basic familiarity with the Free monad, if not, read all about it [here].

## The rules for the Candy Machine

Here are the rules for the candy machine:

> * Inserting a coin into a locked machine will cause it to unlock if there's any candy left.
> * Turning the knob on an unlocked machine will cause it to dispense candy and become locked.
> * Turning the knob on a locked machine or inserting a coin into a unlocked machine does nothing.
> * A machine that's out of candy ignores all inputs.

## A CLI version of the candy machine

Ok, so here we go, let's try to implement a version of the candy machine that provides a CLI interface for the user.

The output from the program should look something like this:

{% highlight scala %}
Welcome to the candy machine
Available commands
s - get current state of machine
c - insert a coin
t - turn
h - help
q - quit
s
MachineState(Some(0),true,20,0)
c
Coin disposed, turn to get your candy!
t
Here is your candy!
s
MachineState(Some(0),true,19,1)
q
{% endhighlight %} 

## MachineState

We need something that represent the current state of the candy machine:

{% highlight scala %}
case class MachineState(
    id: Option[Long] = None, 
    locked: Boolean, 
    candies: Int, 
    coins: Int
) 
{% endhighlight %} 

When we do the web service implementation we need an id so that it's possible to create several candy machines, in the cli version we will always set the id to 0.

## Defining the Algebras

We need two algebras, one for the IO:

{% highlight scala %}
sealed trait IOA[A]

case class Write[A](message: A) extends IOA[Either[Throwable, Unit]]

case class Read[A]() extends IOA[Either[Throwable, A]]
{% endhighlight %} 

and one for the state changes:

{% highlight scala %}
sealed trait MachineOp[A]

case class InitialState(m: MachineState) extends MachineOp[Either[Throwable, MachineState]]

case class UpdateState(id: Long, f: MachineState => Either[Throwable,MachineState]) extends MachineOp[Either[Throwable, MachineState]]

case class CurrentState(id: Long) extends MachineOp[Either[Throwable, MachineState]]
{% endhighlight %} 

As you can see the algebras takes into account that things can go wrong and handles this by using the `Either` monad.

The `IO` algebra gives our program the ability to read/write things from some external source, in the context of the CLI program this will be from the prompt.

The `MachineOp` algebra on the other hand gives the ability to get the current state, update the state and finally set the initial state for the candy machine. 

## Lifting the algebras

Now we have to lift the algebras into the Free monad, first the `IO`:

{% highlight scala %}
class IO[F[_]](implicit I: InjectK[IOA, F]) {
  def write[A](message: A): Free[F, Either[Throwable, Unit]] = 
    Free.inject[IOA, F](Write(message))

  def read[A](): Free[F, Either[Throwable, A]] = 
    Free.inject[IOA, F](Read())
}
{% endhighlight %} 

and then the `MachineOp`:

{% highlight scala %}
class Machine[F[_]](implicit I: InjectK[MachineOp, F]) {
  def updateState(id: Long, f: MachineState => Either[Throwable, MachineState]): Free[F, Either[Throwable, MachineState]] = 
    Free.inject[MachineOp, F](UpdateState(id, f))

  def currentState(id: Long): Free[F, Either[Throwable, MachineState]] = 
    Free.inject[MachineOp, F](CurrentState(id))

  def initialState(m: MachineState): Free[F, Either[Throwable, MachineState]] = 
    Free.inject[MachineOp, F](InitialState(m))
}
{% endhighlight %} 

If you are curious about that inject thingy I suggest you read this [blog post] since it awesome.

![Awsome!](https://media.giphy.com/media/14sy6VGAd4BdKM/giphy.gif)

## The CLI Candy Machine

Ok, so now we almost are ready to write our Cli candy program, but first let's define some helper types so that the function signatures becomes a little less verbose:

{% highlight scala %}
trait Program {
    type CandyMachine[A]

    type FreeProgram[A] = Free[CandyMachine, A]

    type Program[A] = EitherT[FreeProgram, Throwable, A] 
}
{% endhighlight %}  

I leave out the actual type of the CandyMachine so that the implementor (i.e. me, myself and I) have a little more freedom when later assembling our application, more about that later. 

I have divided the CLI program in two parts, one that handles IO from the user and one that takes care of maintaining the rules for the machine and the state. 

We start by looking at the IO-part:

{% highlight scala %}
trait CliCandyProgram extends Program {
  def cliProgram(eventHandler: Request => Program[MachineState])(implicit I: IO[CandyMachine]): Program[Unit] = {

    def main(): Program[Unit] = for {
      _ <- welcome
      _ <- showCommands
      _ <- doWhileM(processInput)(input => input != QuitRequest())
      _ <- goodbye
    } yield ()

    def welcome: Program[Unit] =
      write("Welcome to the candy machine")

    def showCommands: Program[Unit] = for {
      _ <- write("Available commands")
      _ <- write("s - get current state of machine")
      _ <- write("c - insert a coin")
      _ <- write("t - turn")
      _ <- write("h - help")
      _ <- write("q - quit")
    } yield ()

    def doWhileM[A](p: Program[A])(expr: => A => Boolean): Program[Unit] = for {
      a <- p
      _ <- if (expr(a)) doWhileM(p)(expr) else noop
    } yield ()

    def processInput: Program[Request] = for {
      request <- getRequest
      _ <- handleRequest(request)
    } yield request

    def getRequest: Program[Request] = (for {
      input <- read[String]()
      request <- toRequest(input)
    } yield request).recoverWith(e => handleInvalidRequest(e))

    def toRequest(s: String): Program[Request] = {
      val result = if (s == "c")
        Right(InsertCoin(0L))
      else if (s == "t")
        Right(Turn(0L))
      else if (s == "q")
        Right(QuitRequest())
      else if (s == "s")
        Right(GetMachineState(0))
      else if (s == "h")
        Right(HelpRequest())
      else
        Left(new IllegalArgumentException(s"Invalid request: $s"))

      pure(result)
    }

    def handleInvalidRequest(e: Throwable): Program[Request] = for {
      _ <- write(e.getMessage)
      r <- getRequest
    } yield r

    def handleRequest(request: Request): Program[Unit] = (request match {
      case QuitRequest() => noop
      case HelpRequest() => showCommands
      case CreateMachine(_) => for {
        _ <- eventHandler(request)
      } yield ()
      case GetMachineState(_) => for {
        m <- eventHandler(request)
        _ <- write(m.toString)
      } yield ()
      case InsertCoin(_) => for {
        _ <- eventHandler(request)
        _ <- write("Coin disposed, turn to get your candy!")
      } yield ()
      case Turn(_) => for {
        _ <- eventHandler(request)
        _ <- write("Here is your candy!")
      } yield ()
    }).recoverWith(e => write(s"Error when handling request: ${e.getMessage}"))

    def noop: Program[Unit] = pure(Right(()))

    def pure[A](v: Either[Throwable, A]): Program[A] = {
      def pureFreeProgram[A](v: A): FreeProgram[A] = Free.pure[CandyMachine, A](v)
      EitherT(pureFreeProgram(v))
    }

    def read[A](): Program[A] = EitherT(I.read[A]())

    def write[A](s: A): Program[Unit] = EitherT(I.write(s))

    def goodbye: Program[Unit] =
      write("Goodbye, hope to see you again soon!")

    main()
  }
}
{% endhighlight %}

The essense of the program is to get input from the user, translate it into a `Request` and then provide the request to an `eventHandler` that handles the request and then finally give feedback to the user about the outcome.

Error handling is also included since it could be that the user gives the wrong input or breaks one of the rules of the candy machine.

Let's move on:

{% highlight scala %}
trait RequestHandlerProgram extends Program {

  def requestHandler(request: Request)(implicit D: Machine[CandyMachine]): Program[MachineState] = {
    import D._

    sealed trait Input

    case object Coin extends Input

    case object Turn extends Input

    def applyRule(input: Input)(machine: MachineState): Either[Throwable, MachineState] = input match {
      case Coin =>
        if (machine.candies == 0) {
          Left(new IllegalStateException("No candies left"))
        } else if (machine.locked) {
          Right(machine.copy(locked = false, coins = machine.coins + 1))
        } else {
          Left(new IllegalStateException("A coin has already been disposed"))
        }
      case Turn =>
        if (machine.candies == 0) {
          Left(new IllegalStateException("No candies left"))
        } else if (!machine.locked) {
          Right(machine.copy(locked = true, candies = machine.candies - 1))
        } else {
          Left(new IllegalStateException("No coin has been disposed"))
        }
    }

    val result = for {
      response <- request match {
        case CreateMachine(m) => initialState(m)
        case GetMachineState(id) => currentState(id)
        case InsertCoin(id) => updateState(id, applyRule(Coin))
        case Request.Turn(id) => updateState(id, applyRule(Turn))
      }
    } yield response

    EitherT(result)
  }
}
{% endhighlight %}

This part of the program is all about making sure that we are following the rules of the machine and move our machine from one state to another. Please observe that we are still in the pure functional world at this stage, the actual state handling will be done in the interpreter of the algebra.

Let's assemble our programs to a complete CLI-program:
 
{% highlight scala %}
object CliProgram extends CliCandyProgram with RequestHandlerProgram {
  type CandyMachine[A] = EitherK[MachineOp, IOA, A]

  def cliProgram(implicit I: IO[CandyMachine], D: Machine[CandyMachine]): Program[Unit] = cliProgram(requestHandler)
}
{% endhighlight %}

Here I commit myself to a concrete type for the `CandyMachine`: `EitherK[MachineOp, IOA, A]`, `EitherK` gives us the possibility to mix our two algebras.

I also injects the `requestHandler` into the `CliCandyProgram`.

## What about tests for the CliProgram?

Yes, you can find them at [github]. 

![Awsome!](https://media.giphy.com/media/l3q2YLHpmkiq4kBmU/giphy.gif)

## The compilers

Let's move away from our pure functional world and travel to the realm of side effects.

The compiler for IO is pretty simple, `Read` tries to take input from the user and then wrap it in a `Future`. The same goes for `Write` but here we try to give feedback to the user instead.

{% highlight scala %}
type ProgramResult[A] = Future[A]

object PromptAsyncIOInterpreter extends (IOA ~> ProgramResult) {

  override def apply[A](i: IOA[A]): ProgramResult[A] = i match {
    case Write(message) => Future {
      Try(System.out.println(message)).toEither
    }
    case Read() => Future {
      Try(StdIn.readLine()).toEither
    }
  }

}
{% endhighlight %}

Please note that we also commit ourselves to that the Free monad should be mapped to `ProgramResult` (just an alias for `Future`).

Here is the compiler for the `RequestHandlerProgram`:

{% highlight scala %}
object ActorMachineInterpreter {
  def actorMachineInterpreter(ref: ActorRef[MachineRequest])(implicit timeout: Timeout, scheduler: Scheduler, ec: ExecutionContextExecutor): (MachineOp ~> ProgramResult) = new (MachineOp ~> ProgramResult) {
    override def apply[A](fa: MachineOp[A]): ProgramResult[A] = fa match {
      case UpdateState(id, f) => ref
        .ask((ref: ActorRef[UpdateStateReply]) => UpdateStateRequest(id, f, ref))
        .map(r => r.result)
      case CurrentState(id) => ref
        .ask((ref: ActorRef[CurrentStateReply]) => CurrentStateRequest(id, ref))
        .map(r => r.result)
      case InitialState(machine) => ref
        .ask((ref: ActorRef[InitialStateReply]) => InitialStateRequest(machine, ref))
        .map(r => r.result)
    }
  }
}
{% endhighlight %}

The interpreter forwards the request to an `Actor` that maintain the state of the application. The behave function of the actor looks like this:

{% highlight scala %}
def behave(currentId: Long, machines: Map[Long, MachineState]): Behavior[MachineRequest] = Behaviors.receive { (context, message) =>
    message match {
      case UpdateStateRequest(id, f, replyTo) => {
        val newMachine = for {
          oldM <- machines.get(id).toRight(new NoSuchElementException(s"Could not find a machine with id: $id"))
          newM <- f(oldM)
        } yield newM

        replyTo ! UpdateStateReply(newMachine)
        newMachine match {
          case Right(m) =>
            behave(currentId, machines + (id -> m))
          case Left(_) =>
            Behaviors.same
        }
      }
      case CurrentStateRequest(id, replyTo) => {
        replyTo ! CurrentStateReply(machines.get(id).toRight(new NoSuchElementException(s"Could not find a machine with id: $id")))
        Behaviors.same
      }
      case InitialStateRequest(m, replyTo) => {
        val newMachine = m.copy(id = Some(currentId))
        replyTo ! InitialStateReply(Right(newMachine))
        behave(currentId + 1, machines + (currentId -> newMachine))
      }
    }
  }
{% endhighlight %}

The state is updated by changing the behaviour of the actor. Since the Actor is single threaded (illusion) this is all very thread safe. For the full implementation of the Actor I refer to [github].

## Running the program

Time to run the program:

{% highlight scala %}

val asyncProgram: Future[Unit] = for {
    i <- setupActorSystem().map(createInterpreter)
     _ <- runProgram(i, initialMachine)
} yield ()

asyncProgram.onComplete(_ => system.terminate())

def createInterpreter(context: SystemContext) =
    ActorMachineInterpreter.actorMachineInterpreter(context.machineActor) or PromptAsyncIOInterpreter
    
def runProgram[A](interpreter: CandyMachine ~> ProgramResult, initialMachine: MachineState) = {
    val program = for {
      _ <- createMachine(initialMachine)
      _ <- CliProgram.cliProgram
    } yield ()

    program.value.foldMap(interpreter)
} 
{% endhighlight %}

Once again I have left out some code (mostly concerning the setup of the actor system), but this is the essence: assemble our interpreter and then use it to run our program.

## A REST API for the Candy Machine

Let's try to use our program in a different context, let's implement a REST API. We will use [Akka Http] to create the API. I plan to reuse the RequestHandlerProgram and the ActorMachineInterpreter that we developed for our CLI program.

{% highlight scala %}
object RequestHandler extends RequestHandlerProgram {
  type CandyMachine[A] = MachineOp[A]
}
{% endhighlight %}

We don't need the IO-part of our program since this will be handled by Akka Http. Here's the route for our Rest service:

{% highlight scala %}
class CandyServer(val interpreter: MachineOp ~> ProgramResult) extends Directives with JsonSupport {
  val route =
    concat(
      post {
        path("candy") {
          entity(as[MachineState]) { machine =>
            onComplete(handler(CreateMachine(machine))) {
              case Success(value) => toResponse(value)
              case Failure(ex) => internalServerError(ex)
            }
          }
        }
      }, get {
        path("candy" / LongNumber) { (id) => {
          onComplete(handler(GetMachineState(id))) {
            case Success(value) => toResponse(value)
            case Failure(ex) => internalServerError(ex)
          }
        }
        }
      }, put {
        path("candy" / LongNumber / "coin") { (id) => {
          onComplete(handler(InsertCoin(id))) {
            case Success(value) => toResponse(value)
            case Failure(ex) => internalServerError(ex)
          }
        }
        }
      }, put {
        path("candy" / LongNumber / "turn") { (id) => {
          onComplete(handler(Turn(id))) {
            case Success(value) => toResponse(value)
            case Failure(ex) => internalServerError(ex)
          }
        }
        }
      }
    )
    
   private def handler(r: Request): Future[Either[Throwable, MachineState]] =
       RequestHandler.requestHandler(r).value.foldMap(interpreter) 
{% endhighlight %}

We use our pure program and the interpreter to handle the requests as they arrive from the client.

The API gives the client the possibility to create a new machine, get the status of the machine and to insert a coin or try to get a candy.

We create the interpreter like this:

{% highlight scala %}
 ActorMachineInterpreter.actorMachineInterpreter(context.machineActor)
{% endhighlight %}

Once again I have left out things that is not that important for this post (mostly to setup the actor system and the startup of the web service), check [github] for the full implementation. 

## Conclusion

We have tried and succeeded to implement a CLI program for a Candy Machine, we later reused part of the program to provide a REST Api. 

## Github

All the code is available at [github].

[Functional programming in Scala]: https://www.manning.com/books/functional-programming-in-scala
[here]: https://morotsman.github.io/scala/cats/free/monad/2020/08/27/simple-free-monad.html
[github]: https://github.com/morotsman/about_scala/tree/master/src/main/scala/scalaz_experiments/free_monad/candy3
[Akka Http]: https://doc.akka.io/docs/akka-http/current/index.html
[blog post]: https://underscore.io/blog/posts/2017/03/29/free-inject.html
