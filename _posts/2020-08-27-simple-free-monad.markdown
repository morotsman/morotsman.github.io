---
layout: post
date:   2020-08-27 09:09:00 +0200
categories: Scala Cats Free Monad
title: Free Monad - Echo Echo Echo 
subtitle:  Testing a free monad
background: '/img/hamburg2.jpg'  
---

## About the post

Let me tell a little secret of mine, I'm sort of addicted to tests. Or well, at least when writing code that will be used in production. Code that is more of an experiment I'm more relaxed with, which is evident if you care to look at [my github account]. 

What has that to do with the Free Monad you may wonder?

In this post we will write a very simple program that takes some input and then echo's it back to the user. We will use the Free Monad to separate the core program from the messy IO-stuff. We will also write tests to prove that our program works which I guess is the main focus in this post.  

## What is a Free Monad?

This is the explanation in the [cats] documentation: 

>A free monad is a construction which allows you to build a monad from any Functor. Like other monads, it is a pure way to represent and manipulate computations.

>In particular, free monads provide a practical way to:
> * represent stateful computations as data, and run them
> * run recursive computations in a stack-safe way
> * build an embedded DSL (domain-specific language)
> * retarget a computation to another interpreter using natural transformations

## The Echo program

The output when running the program should look something like this:

{% highlight scala %}
The great echo program!
'q' to quit
> hello
You wrote: 'hello'

> echo?
You wrote: 'echo?'

> q
You wrote: 'q'

Hope to see you again soon, goodbye!
{% endhighlight %}

As we can see the program is mostly about reading stuff from the prompt and then writing it back.

## Defining the Algebra

So how should the [algebra] look like for our Echo program? Well, we know that it must support reading and writing to the prompt so I defined it like this:

{% highlight scala %}
sealed trait EchoA[A]

case class Read[A]() extends EchoA[A]

case class PrintLn[A](a: A) extends EchoA[Unit]

case class Print[A](a: A) extends EchoA[Unit]
}
{% endhighlight %}

The algebra gives the program the ability to `read` things from some kind of source and also the ability to write something to somewhere using `print` or `printLn` (print and newline). 

To keep it really simple the algebra does not support error handling, but I have an example [here] where that is included. 

## Lifting the algebra

Now that we have defined our algebra we can create a type for the free monad. I also create some constructors that wraps (lifts) our algebra into the context of the free monad.

{% highlight scala %}
object Echo {
  type Echo[A] = Free[EchoA, A]

  def read[A](): Echo[A] =
    Free.liftF[EchoA, A](Read[A]())

  def printLn[A](a: A): Echo[Unit] =
    Free.liftF[EchoA, Unit](PrintLn[A](a))

  def print[A](a: A): Echo[Unit] =
    Free.liftF[EchoA, Unit](Print[A](a))
}
{% endhighlight %}

## Writing the echo program

And now we can express our program through the algebra: 

{% highlight scala %}
object EchoProgram {
  def program: Echo[Unit] = for {
    _ <- greet
    s <- loop
  } yield (s)

  private def greet = for {
    _ <- printLn("The great echo program!")
    r <- printLn("'q' to quit")
  } yield r

  def loop: Echo[Unit] = for {
    in <- readFromPrompt
    _ <- echo(in)
    s <- if (quit(in)) goodbye else loop
  } yield s

  private def readFromPrompt: Echo[String] = for {
    _ <- print("> ")
    s <- read[String]()
  } yield s

  private def echo[A](in: A): Echo[Unit] = for {
    _ <- printLn(s"You wrote: '${in}'")
    r <- printLn("")
  } yield r

  private def quit(st: String): Boolean =
    st == "q"

  private def goodbye: Echo[Unit]  = for {
    r <- printLn("Hope to see you again soon, goodbye!")
  } yield (r)
}
{% endhighlight %}

In the code above we rely on for comprehensions to give us nice syntax for the program. This is possible since we have lifted our algebra into the Free monad, very nice.

## Running the program

If we run the program like this:

{% highlight scala %}
def main(args: Array[String]): Unit = {
    program
}
{% endhighlight %}

Nothing is shown, it looks like the program doesn't do anything, how very disappointing.

![disappointing!](https://media.giphy.com/media/KHPiexunFp1XktTNXY/giphy.gif)

But it's also understandable considering that the program is an instruction of what we want to accomplish. To actually run the program we need something that interpret it! 

By the way if your interested in the inner works of the Free monad, pick up the book: [Functional programming in Scala], it has a nice section about it.

## The compiler

Let's write a compiler so that we can actually use our program for something.

{% highlight scala %}
def compilerWithSideEffects: EchoA ~> Id =
    new (EchoA ~> Id) {
    def apply[A](fa: EchoA[A]): Id[A] = fa match {
        case Read() =>
            StdIn.readLine.asInstanceOf[A]
        case PrintLn(s) =>
            System.out.println(s)
        case Print(s) =>
            System.out.print(s)
    }
}
{% endhighlight %}

The compiler translates our algebra to concrete IO instructions, for example `Println(s) -> System.out.println`. 

We also selects a concrete monad `Id` to represents our free monad in the "real world". The type of the `Id` monad is `type Id[A] = A` so I think of it as the equivalent of the id function `def identity[A](a: A): A = a` but for types.

We could of course have selected another monad if we wanted to (like Future, Try etc) which is a really cool feature. Later in the post we will in fact do so when writing tests for our program.

## Running the program again

Let's apply the compiler to the program.

{% highlight scala %}
def main(args: Array[String]): Unit = {
    program.foldMap(compilerWithSideEffects)
}
{% endhighlight %}

The output is as follows:
{% highlight scala %}
The great echo program!
'q' to quit
> echo!
You wrote: 'echo!'

> q
You wrote: 'q'

Hope to see you again soon, goodbye!
{% endhighlight %}

![It's a wrap!](https://media.giphy.com/media/lrylNue4ExZM5GLoDQ/giphy.gif)

## Wait, what about the tests?

Oh yeah, now I seem to recall that I promised tests for our Free monad. Ok, let's do it.

## A compiler for the tests

When testing we don't want to use our previous compiler since it maps our algebra to concrete IO operations. That's good when we want to let the user do something with the program, but not so much if we want to test the program. 

Here's a compiler that is good for testing:

{% highlight scala %}
case class Buffers[A](in: List[A], out: List[A])

type EchoState[A] = State[Buffers[Any], A]
val pureCompilerNiceOutput: EchoA ~> EchoState = new (EchoA ~> EchoState) { 
    override def apply[A](fa: EchoA[A]): EchoState[A] = fa match {
        case Read() => for {
            old <- State.get
            _ <- State.modify(addToOutput(old.in.head))
            _ <- State.modify(addToInput(old.in.tail))
            } yield old.in.head.asInstanceOf[A]
        case PrintLn(output) => for {
            _ <- State.modify(addToOutput(output))
            } yield ()
        case Print(output) => for {
            _ <- State.modify(addToOutput(output))
            } yield ()
    }
}

def addToOutput[A](i: A)(s: Buffers[Any]): Buffers[Any] =
    s.copy(out = i :: s.out)

def addToInput[A](i: List[A])(s: Buffers[Any]): Buffers[Any] =
    s.copy(in = i)
{% endhighlight %}

Here we have replaced the `Id` monad with the `State` monad which lets us handle state in a functional and pure way. The state contains two lists, one called `in` that keeps track on the input to the program, and one called `out` that keeps track on, well the output of the program. 

## Usage of the new compiler

Here is an example of how to run the program now that we are using the `State` monad. The main differens is that we now have to provide the initial state to the program:

{% highlight scala %}
val initialState = Buffers(List[Any]("echo", "q"), List())

val result: (Buffers[Any], Unit) = program.foldMap(pureCompilerNiceOutput).run(initialState).value

val expectedResult = List(
    "The great echo program!", 
    "'q' to quit", 
    "> ", 
    "echo", 
    "You wrote: 'echo'", 
    "", 
    "> " , 
    "q", 
    "You wrote: 'q'", 
    "", 
    "Hope to see you again soon, goodbye!"
)
    
expectedResult == result._1.out.reverse
{% endhighlight %}

The initialState contains the user input that we want to simulate. After the program has finished it's execution we get back a representation of the state that contains the corresponding output. All very pure and immutable:

![It's a wrap!](https://media.giphy.com/media/ZEIVY62QI6Hrc59GX1/giphy.gif)

## The test

Let's use the ScalaCheck library to generate input for us: 

{% highlight scala %}
property("echo") = forAll { (input: List[String]) =>
    val myInput = input.filter(_ != "q").appended("q")

    val result: (Buffers[Any], Unit) = program.foldMap(pureCompilerNiceOutput).run(Buffers(myInput, List())).value

    val hello = List(
      "The great echo program!",
      "'q' to quit"
    )

    val goodbye = List(
      "Hope to see you again soon, goodbye!"
    )

    val output = myInput.flatMap(s => List("> ", s, s"You wrote: '$s'", ""))

    val expectedResult = hello ++ output ++ goodbye

    expectedResult == result._1.out.reverse
}
{% endhighlight %} 

And that's it, we have written a test for our program! 

## Conclusion

We have implemented a simple program using an algebra and the Free monad. A test has been written which leaves a good feeling in the stomach.

We concluded that's it's pretty cool that the free monad let us defer our choice of the actual monad that will be used when running the program.  

If you are not tired yet I have written another [post] with a larger example where we use the Free monad in the context of a CLI tool and then in a web service.


## Github
The code examples are available at [github].

[my github account]: https://github.com/morotsman 
[cats]: https://typelevel.org/cats/datatypes/freemonad.html
[here]: https://github.com/morotsman/about_scala/blob/master/src/main/scala/scalaz_experiments/free_monad/echo/echo_with_error_handling/EchoWithErrorHandling.scala
[algebra]: https://en.wikipedia.org/wiki/Algebraic_data_type
[Functional programming in Scala]: https://www.manning.com/books/functional-programming-in-scala 
[github]: https://github.com/morotsman/about_scala/blob/master/src/main/scala/scalaz_experiments/free_monad/echo/simple_echo/Echo.scala
[post]: https://morotsman.github.io/scala/cats/free/monad/cli/webservice/2020/09/07/free-monad-in-a-webservice.html
