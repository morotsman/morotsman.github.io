---
layout: post
subtitle:  "Creating our own Applicative using Scalaz"
date:   2020-08-04 09:09:00 +0200
categories: Scala Scalaz Applicative Example
title: What on earth is an Applicative and how can I get one of my own?  
background: '/img/new_york9.jpg'  
---

## About the post

This post will cover the basic usage of an Applicative. We will also create our own data type and turn it into an Applicative.

First a caveat: I'm not a seasoned Scalaz developer with years and years of experience. This post represents my current understanding of the topic. Feel free to drop me a mail if you have suggestions for improvements so I can learn more!

The code examples are available at [github].

## About Applicative's

So, what's an `Applicative`? This is the definition on wikipedia ([ref1]):

> In functional programming, an applicative functor is a structure intermediate between functors and monads, in that they allow sequencing of functorial computations (unlike plain functors) but without deciding on which computation to perform on the basis of the result of a previous computation (unlike monads).

The definition of `Functor` on wikipedia is ([ref2]): 
> In functional programming, a functor is a design pattern inspired by the definition from category theory, that allows for a generic type to apply a function inside without changing the structure of the generic type.

## Functor

Ok, so what does that mean? Let's be concrete and look at an example of a `Functor`:

{% highlight scala %}
def addOne(a: Int) = a + 1

val of = Functor[Option]

val result: Option[Int] = of.map(Some(3))(addOne)
    
assert(result == Some(4))
{% endhighlight %} 

The generic type in the case is `Option`, the function to apply inside is `addOne` and the structure after we have invoked `map` is the same as before we called the function. We can say that `map` lifts `addOne` (that has no knowledge of `Option`) so that we can execute it inside the generic type `Option`.

Scalaz provides a `lift` function that is implemented like this:
{% highlight scala %}
def lift[A, B](f: A => B): F[A] => F[B] = 
    map(_)(f)
{% endhighlight %} 

And is used like so:

{% highlight scala %}
val addOneLifted: Option[Int] => Option[Int] = of.lift(addOne)
    
val result2: Option[Int] = addOneLifted(Some(3))
    
assert(result2 == Some(4))
{% endhighlight %}


This is all very nice and we use `Functor` everyday to map List's, Option's, Set's etc, maybe without even thinking about that we are using it.

{% highlight scala %}
List(1, 2, 3).map(addOne)
    
Set(1, 2, 3).map(addOne)
    
Some(1).map(addOne)
{% endhighlight %} 

## The problem

Useful as it is the `Functor` get's into trouble if we would like to lift a function that takes two arguments:

{% highlight scala %}
def addTwo(a: Int, b: Int): Int = a + b

val someOne = 1.some
val someTwo = 2.some

val result = someOne.map(a => someTwo.map(b => addTwo(a, b)))
    
assert(result == Some(Some(3)))
{% endhighlight %} 

The attempt above doesn't produce the result that we would like: `Some(3)` but instead gives back a nested structure of `Options`'s. The `Functor` is not powerful enough to help us here!   

One way to fix this is to use `Option` as a `Monad` which gives us access to `flatMap`:

{% highlight scala %}
val result = for {
    a <- someOne
    b <- someTwo
} yield addTwo(a, b)
    
assert(result == Some(3))
{% endhighlight %} 

But as we will find out later in the post, it is not always preferable to go the whole way to a `Monad`. Enter `Applicative`.

## Applicative

From the definition above we know that an `Applicative` is more then a `Functor` but less then a `Monad`. Will it give us enough power to solve the problem above?   

If we look at Scalaz implementation of Applicative we can find a signature that looks promising:

{% highlight scala %}
override def apply2[A, B, C](fa: => F[A], fb: => F[B])(f: (A, B) => C): F[C]
{% endhighlight %} 

We can use the function like so:

{% highlight scala %}
val ao = Applicative[Option]
import ao._
    
val result = apply2(someOne, someTwo)(addTwo)

assert(result == Some(3))
{% endhighlight %} 

Scalaz provides us with an `Applicative[Option]`, on it there is a function called `apply2` that gives us just enough power to solve our problem. 

And that's  :

![Very nice](https://media.giphy.com/media/l0HTYUmU67pLWv1a8/giphy.gif)

Scalaz provides `Applicative`'s for many common data types, like `List` and `Option`. Later in the post we will create an `Applicative` for a data type the we have created our self. 

But first we will investigate different functions that are defined on `Applicative`.

## Applicative basics

First let's look on alternative syntax for `apply2`:

{% highlight scala %}
val result = apply2(someOne, someTwo)(addTwo)
assert(result == Some(3))

// is the same as

val result = ^(someOne, someTwo)(addTwo)
assert(result == Some(3))

// is the same as

val result = (someOne |@| someTwo) (addTwo)
assert(result == Some(3))
{% endhighlight %} 

This syntax scales nicely:

{% highlight scala %}
def addFour(a: Int, b: Int, c: Int, d: Int): Int = a + b + c + d

val result = ^^^(someOne, someTwo, someOne, someTwo)(addFour)
assert(result == Some(6))

val result = (someOne |@| someTwo |@| someOne |@| someTwo) (addFour)
assert(result == Some(6))
{% endhighlight %} 

Another useful operator is `sequence`:

{% highlight scala %}
def sequence[A, G[_]: Traverse](as: G[F[A]]): F[G[A]]
{% endhighlight %} 

As we can see from the signature it takes as input a traversable data type that contains an `Applicative` and turn it inside out.

Example of usage where the traversable is a `List` and the `Applicative` is an `Option`:

{% highlight scala %}
assert(sequence(List(someOne, someTwo, someThree)) == Some(List(1, 2, 3)))
assert(sequence(List(None, someTwo, someThree)) == None)
{% endhighlight %} 

Here's another example where we use `sequence` to wait until all `Futures` has completed:

{% highlight scala %}
def successFuture = Future {
    Thread.sleep(900)
    1
}
      
val failedFuture = Future.failed(new RuntimeException("oops"))

val af = Applicative[Future]
import af._
      
val listOfFutures: List[Future[Int]] = List(successFuture, successFuture, successFuture)
val futureOfList : Future[List[Int]] = sequence(listOfFutures)
val result1 = Await.result(futureOfList, 1000 millis)
assert(result1 == List(1, 1, 1))

val recoveredFuture = failedFuture.recoverWith{ case _ => Future.successful(2)}
val futureResult2 : Future[List[Int]] = sequence(List(successFuture, successFuture, recoveredFuture))
val result2 = Await.result(futureResult2, 1000 millis)
assert(result2 == List(1, 1, 2))
{% endhighlight %} 

Let's look at a similar operator called `traverse`:
{% highlight scala %}
def traverse[A, G[_], B](value: G[A])(f: A => F[B])(implicit G: Traverse[G]): F[G[B]]
{% endhighlight %} 

Example of usage:
{% highlight scala %}
def all(i: Int): Option[Int] = Some(i)

assert(traverse(List(1, 2, 3))(all) == Some(List(1, 2, 3)))

def onlyEven(i: Int): Option[Int] = if (i % 2 == 0) Some(i) else None

assert(traverse(List(1, 2, 3))(onlyEven) == None)
assert(traverse(List(2, 4))(onlyEven) == Some(List(2, 4)))
{% endhighlight %} 

We can compose `Applicatives` like this:

{% highlight scala %}
val ao = Applicative[Option]
val la = Applicative[List]
val lacao = la.compose(ao)

val listOfOptions = List(someOne, someTwo, None)
val result = lacao.map(listOfOptions)(a => a + 10)
assert(result == List(Some(11), Some(12), None))
{% endhighlight %} 

We can also get the product of two `Applicatives`:

{% highlight scala %}
val ao = Applicative[Option]
val la = Applicative[List]
val lapao = la.product(ao)

val result = lapao.map((List(1, 2), Some(2)))(a => a + 10)
assert(result == (List(11, 12), Some(12)))
{% endhighlight %} 

![Boring](https://media.giphy.com/media/LTYT5GTIiAMBa/giphy.gif)

I think I finish here because this is starting to get a a bit boring but I have added some more examples of different operators at [github].


## Creating our own Applicative

But what if we creates our own data type, can I get an `Applicative` for that one as well? 


![Yes we can!](https://media.giphy.com/media/l0ErOholJjSmFlMFG/giphy.gif)

So let's do it!

I will use the classic validation example since it highlights why it is sometimes better to use an `Applicative` then a `Monad` in a nice way.

Ok, so let's say that we have created or own `Validator`
{% highlight scala %}
sealed trait Validator[+E, +T]

object Validator {
  type Validated[T] = Validator[String, T]
}

case class Valid[+T](t: T) extends Validator[Nothing, T]

case class Invalid[+E](es: Vector[E]) extends Validator[E, Nothing]

object Invalid {
  def apply[E](e: E): Invalid[E] = Invalid(Vector(e))
}
{% endhighlight %}

If we would use the `Validator` as a `Monad` the client would use it like this:

{% highlight scala %}
def validateName(name: String): Validated[String] =
    if (name.length > 0 && name.length <= 21) Valid(name)
    else Invalid("Name validation failed")

def validateAge(age: Int): Validated[Int] =
    if (age >= 0) Valid(age)
    else Invalid("Age failed the validation")
    
val validator = Monad[Validated]

val person5 = for {
    name <- validateName("Niklas")
    age <- validateAge(35)
} yield (Person(name, age))

assert(person5 == Valid(Person("Niklas",35)))

val person6 = for {
    name <- validateName("NiklasNiklasNiklasNiklasNiklasNiklasNiklas")
    age <- validateAge(-2)
} yield (Person(name, age))

assert(person6 == Invalid(Vector("Name validation failed")))
{% endhighlight %}

It get's the work done but it's kind of half baked. If we look at the the result from the validation we see that we only reported that the name was invalid, in this case both the name and the age was wrong.

Why is that? Let's look at how we implemented the `Monad` for `Validator`

{% highlight scala %}
implicit val validationMonad: Monad[Validated] = new Monad[Validated] {
    override def point[A](a: => A): Validated[A] = Valid(a)

    override def bind[A, B](fa: Validated[A])(f: A => Validated[B]): Validated[B] = fa match {
      case Valid(a) => f(a)
      case Invalid(i) => Invalid(i)
    }
}
{% endhighlight %}

As you can see we have no option but to follow `bind`'s (also known as `flatMap`) short circuit behaviour when a failure occur (i.e. when something is Invalid). If the result from the previous computation was invalid we have no way to somehow merge failures or to continue the computation. 

We can also see that the happy path is a chain of calculations, we first checks if the first validation was "Valid" we then proceed with the next validation and so on. 

In this case we really don't need all the power that a `Monad` is giving us (even if it's tempting).

![Unlimited power!](https://media.giphy.com/media/zCv1NuGumldXa/giphy.gif)

And why don't we need the power? Well if we look at our validation functions:

{% highlight scala %}
def validateName(name: String): Validated[String] =
    if (name.length > 0 && name.length <= 21) Valid(name)
    else Invalid("Name validation failed")

def validateAge(age: Int): Validated[Int] =
    if (age >= 0) Valid(age)
    else Invalid("Age failed the validation")
    
val person = validateName("Niklas").flatMap(name => validateAge(30).map(age => Person(name, age)))    
{% endhighlight %}

We can see that neither of the functions depends on any computations that we have done before it was invoked. Instead all our computations are isolated i.e. they don't need any input from the previous calculation step. We could in fact run them in parallel if we wanted.  

So the very thing that makes the `Monad` powerful (the chaining of computations and short circuit behaviour) is to our disadvantage in this use case.

So like when Galadriel rejected the one ring, let's settle for less power and use an `Applicative` instead (but with less drama).

>'And now at last it comes. You will give me the Ring freely! In place of the Dark Lord you will set up a Queen. And I shall not be dark, but beautiful and terrible as the Morning and the Night! Fair as the Sea and the Sun and the Snow upon the Mountain! Dreadful as the Storm and the Lightning! Stronger than the foundations of the earth. All shall love me and despair!'
    
> She lifted up her hand and from the ring that she wore there issued a great light that illuminated her alone and left all else dark. She stood before Frodo seeming now tall beyond measurement, and beautiful beyond enduring, terrible and worshipful. Then she let her hand fall, and the light faded, and suddenly she laughed again, and lo! she was shrunken: a slender elf-woman, clad in simple white, whose gentle voice was soft and sad.
    
> I pass the test,' she said. 'I will diminish, and go into the West and remain Galadriel.'

First we look at how the client would use it:

{% highlight scala %}
val person1: Validated[Person] = ^(validateName("Niklas"), validateAge(27)) { (name, age) =>
    Person(name, age)
}

assert(person1 == Valid(Person("Niklas", 27)))

val person2: Validated[Person] = ^(validateName("NiklasNiklasNiklasNiklasNiklasNiklas"), validateAge(-1)) { (name, age) =>
    Person(name, age)
}

assert(person2 == Invalid(Vector("Age failed the validation", "Name validation failed")))
{% endhighlight %}

Success! Or yeah, well maybe failure, but still! Success! Both errors are reported!

![Booya!](https://media.giphy.com/media/t6K7XFKkKPpkhddIWa/giphy.gif)

Here is the code for the `Applicative[Validator]`

{% highlight scala %}
implicit val validationApplicative: Applicative[Validated] = new Applicative[Validated] {
    override def point[A](a: => A): Validated[A] = Valid(a)

    override def ap[A, B](fa: => Validated[A])(f: => Validated[A => B]): Validated[B] = (fa, f) match {
      case (Valid(a), Valid(fab)) => Valid(fab(a))
      case (Valid(a), Invalid(es)) => Invalid(es)
      case (Invalid(es), Valid(a)) => Invalid(es)
      case (Invalid(es1), Invalid(es2)) => Invalid(es1 ++ es2)
    }
}
{% endhighlight %}

As we can see, we now have the possibility to merge the errors so that all our shortcomings can be exposed. Also we don't have the short circuit behaviour, all the validation functions get's a chance to run. All is good!

## Conclusion

So to sum up, sometimes it's better to use an `Applicative` then a `Monad` even if it's less powerful. 

If we want all our calculations to run even if some fails then we should use an `Applicative` instead of a `Monad`.

If we use the Scalaz library it's pretty easy to create our own `Monads` and `Applicatives`, as a by-product we also get access to all those nifty little operators that the people behinde Scalaz has implemented over the years!


## Github
The code examples are available at [github].


[github]: https://github.com/morotsman/about_scala/tree/master/src/main/scala/scalaz_experiments/validation
[ref1]: https://en.wikipedia.org/wiki/Applicative_functor
[ref2]: https://en.wikipedia.org/wiki/Functor_(functional_programming)
