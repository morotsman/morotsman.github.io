---
layout: post
title: Kleisli you say? That sounds bad! Is it contagious?
date:   2020-07-28 09:40:00 +0200
categories: Scala Scalaz Kleisli Example
subtitle: "Composing functions with Scalaz Kleisli"  
background: '/img/new_york8.jpg' 
---

## About the post

This is the first post in a series of two(?) about Scalaz's Kleisli construction. 

This post will cover the basic usage of Kleisli. A yet to be written post will cover different operators that are defined on Kleisli. 

First a caveat: I'm not a seasoned Scalaz developer with years and years of experience. This post represents my current understanding of the topic. Feel free to drop me a mail if you have suggestions for improvements so I can learn more!

The code examples are available at [github].

## Kleisli

Kleisli is used for functional functional composition. In this post we will investigate Kleisli by looking a couple of basic examples. We will also try to understand the implementation of Keisli in Scalaz.  

So in the word's of Ramones:

![Hey ho](https://media.giphy.com/media/26hitEne8c3secRPi/giphy.gif)


## Compose a function

Let's say that we have two different functions:

{% highlight scala %}
def addOne(i: Int): Int = i + 1

def double(i: Int): Int = i * 2
{% endhighlight %}

We could combine the two functions like so:

{% highlight scala %}
def addOneAndDouble(i: Int): Int = double(addOne(20))

assert(42 == addOneAndDouble(20))
{% endhighlight %}

I.e. we first invoke the addOne function and then invokes the double function with the result from the first computation.

This is another way to do the same thing:
  
{% highlight scala %}
val composedFunction: Int => Int = addOne _ andThen double

assert(42 == composedFunction(20))
{% endhighlight %}

This is called functional composition, i.e. we compose a new function `composedFunction` from the existing `addOne` and `double` functions.

Here's another example:

{% highlight scala %}
def addOne(i: Int): Int = i + 1

def toAs(number: Int): String = Array.fill(number)("a").mkString

val composedFunction2: Int => String = addOne _ andThen toAs
    
assert("aaaaaa" == composedFunction2(5))
{% endhighlight %}


For this to work the types must lineup, i.e. the type of the output from the first function must match the type on the input of the next function. In the second example above `addOne` produces an `Int`, `toAs` expects an `Int`   as input and hence the types lines up!

This is an example where functional composition is not working:

{% highlight scala %}
case class Person(name: String, city: String, books: Set[String], salary: Int)

def nameValidator(p: Person): Either[String, Person] =
    if (p.name.length > 0 && p.name.length < 21) Right(p) else Left("Name failed the validation")

def cityValidator(p: Person): Either[String, Person] =
    if (p.city.length > 0 && p.city.length < 21) Right(p) else Left("City failed the validation")
    
def salaryValidator(p: Person): Either[String, Person] =
    if (p.salary > 0 && p.salary < 100000) Right(p) else Left("Salary failed the validation")    
    
val composedFunction3 = (nameValidator _).andThen(salaryValidator) // will not compile, the types don't line up    
{% endhighlight %}

The code above contains three functions that is used to validate that content of `Person` follows some rules. It would be nice if we could compose a single `validator` function from the three different lesser validation functions.  

But alas, the types doesn't line up since nameValidator produces an `Either[String, Person]` and the `cityValidator` expects a `Person` as input.

![Alas](https://media.giphy.com/media/3ohs7Ys9J8XyFVheg0/giphy.gif)


This is where [Kleisli] comes in. The Kleisli gives support to compose functions when the results of the function invocation is [Monadic]. Examples of Monads are plentiful: `List`, `Option`, `Either` to name a few. Both the `nameValidator` and `cityValidator` in the example above are examples where we can use Kleisli.

## Compose functions with the help of Kleisli

So let's compose `nameValidation`, `cityValidator` and `salaryValidator` into a single validation function:

{% highlight scala %}
val validator = Kleisli(nameValidator) andThen Kleisli(salaryValidator) andThen Kleisli(cityValidator)
{% endhighlight %}

The composed function is used like this:

{% highlight scala %}
val books = Set("Programing in Scala")
val validPerson = Person("Niklas", "Malmö", books, 50000)
val invalidPerson = Person("NiklasNiklasNiklasNiklasNiklasNiklasNiklas", "Malmö", books, 500000)

assert(Right(validPerson) == validator(validPerson))
assert(Left("Name failed the validation") == validator(invalidPerson))
{% endhighlight %}

Alternative syntaxes for the functional composition above are:

{% highlight scala %}
val validator2 = Kleisli(nameValidator) >=> Kleisli(salaryValidator) >=> Kleisli(cityValidator)

val validator3 = Kleisli(nameValidator) >==> salaryValidator >==> cityValidator
{% endhighlight %}

## What about flatMap?

Yes, flatMap can be used to the same effect like this:

{% highlight scala %}
def personValidatorWithFlatMap(p: Person): Either[String, Person] =
    nameValidator(p).flatMap(salaryValidator).flatMap(cityValidator)
{% endhighlight %}

Or like this:
{% highlight scala %}
def personValidatorWithFlatMap2(p: Person): Either[String, Person] =
    for {
        _ <- nameValidator(p)
        _ <- salaryValidator(p)
        _ <- cityValidator(p)
    } yield (p)
{% endhighlight %}

However this is a post about Kleisli, so let's get back to that.

## The signature of Kleisli

Now that we know what Kleisli can be used for, let's take a look at some code from the Scalaz [library]:

{% highlight scala %}
final case class Kleisli[M[_], A, B](run: A => M[B]) 
{% endhighlight %}

We start by looking at the constructor of the Kleisli class. We can see that it accepts functions of the following kind: `run: A => M[B]`. We can also see that it only accepts a M that has a type parameter `M[_]`. This mean that the following code will not compile:

{% highlight scala %}
def personToPerson(p: Person): Person = p
Kleisli(personToPerson) // will not compile
{% endhighlight %}

This since a `Person` don't accepts any type parameters. This however will work perfectly fine:

{% highlight scala %}
def personToPersons(p: Person): List[Person] = List(p)
Kleisli(personToPersons)
{% endhighlight %}

This since that List accepts a type parameter. The conclusion is that Kleisli will only work for functions that results in a type constructor. 

The next thing from the Kleisli class I would like to highlight is `>=>` and it's alias `andThen` :

{% highlight scala %}
def andThen[C](k: Kleisli[M, B, C])(implicit b: Bind[M]): Kleisli[M, A, C] = 
    this >=> k

def >=>[C](k: Kleisli[M, B, C])(implicit b: Bind[M]): Kleisli[M, A, C] =  
    kleisli((a: A) => b.bind(this(a))(k.run))
{% endhighlight %} 

Well, on second thought to understand the code above we probably should look at the signature for `bind` first of all:

{% highlight scala %}
trait Bind[F[_]] extends Apply[F]

def bind[A, B](fa: F[A])(f: A => F[B]): F[B]
{% endhighlight %} 

So it looks like the signature for `bind` corresponds to the signature of `flatMap`. Let's do an experiment with `bind` and `Either`:

{% highlight scala %}
val b = implicitly[Bind[({type f[x] = Either[String, x]})#f]]

assert(Right(2) == b.bind(Right(1))(i => Right(i + 1)))
assert(Left("failure") == b.bind(Left("failure"): Either[String, Int])(i => Right(i + 1)))
{% endhighlight %}

Looks like `flatMap` to me! 

By the way, since `Bind` only accepts constructs with one type parameter and `Either[+A, +B]` has two, we have to partially apply one of the type parameters like this: `[({type f[x] = Either[String, x]})#f]`. The syntax for this in Scala could be better :-).

![Syntax error](https://media.giphy.com/media/ifdPjn6m4WyNlnXMTj/giphy.gif)

Ok, so now that we knows what `bind` does let's take a look at `>=>` again:

{% highlight scala %}
def >=>[C](k: Kleisli[M, B, C])(implicit b: Bind[M]): Kleisli[M, A, C] =  
    kleisli((a: A) => b.bind(this(a))(k.run))
{% endhighlight %} 

We can see that it creates a new Kleisli with a function that will do a flatMap when we eventually chooses to invoke the Kleisli. 

Maybe a concrete example is in place:

{% highlight scala %}
Kleisli(nameValidator) >=> Kleisli(salaryValidator)

//can be simplified to
Kleisli((a: Person) => b.bind(Kleisli(nameValidator)(a))(Kleisli(salaryValidator).run))

//can be simplified to
Kleisli((a: Person) => b.bind(nameValidator(a))(salaryValidator))

{% endhighlight %}  

If we further substitutes `a` (i.e. invokes the Kleisli) with validPerson we get:

{% highlight scala %}
b.bind(nameValidator(validPerson))(salaryValidator)

//can be simplified to
b.bind(Right(validPerson))(salaryValidator)

//can be simplified to
Right(validPerson)
{% endhighlight %}  

In other words we are essentially building a chain of flatMap's.

Finally i would like to highligt: 

{% highlight scala %}
def >==>[C](k: B => M[C])(implicit b: Bind[M]): Kleisli[M, A, C] = this >=> kleisli(k)
{% endhighlight %} 

As we can see the only thing it does is to wrap the in argument in a Kleisli and the call `>=>`.  

This gives us the ability (as already shown above) to compose a function using this simple syntax:

{% highlight scala %}
Kleisli(nameValidator) >==> salaryValidator >==> cityValidator
{% endhighlight %} 

And that's it, this is the basics of Kleisli, it is nothing but a construction to get the ability to compose functions!

## Conclusion

We have shown that Kleisli can be used to compose functions when the result from the function invokation is Monadic. We have also investigated the Scalaz code that makes this work. 

We have concluded that we can get the same effect using `flatMap` but with a less concise syntax:

{% highlight scala %}
val personValidator = Kleisli(nameValidator) >==> salaryValidator >==> cityValidator

// is the same as: 

def personValidatorWithFlatMap(p: Person): Either[String, Person] =
    nameValidator(p).flatMap(salaryValidator).flatMap(cityValidator)
{% endhighlight %} 

In the next post which is yet to be written we will investigate different operators that are defined on Kleisli.

## Github
The code examples are available at [github].

[Monadic]: https://en.wikipedia.org/wiki/Monad_(functional_programming) 
[Kleisli]: https://github.com/scalaz/scalaz/blob/series/7.1.x/core/src/main/scala/scalaz/Kleisli.scala
[library]: https://github.com/scalaz/scalaz/blob/series/7.1.x/core/src/main/scala/scalaz/Kleisli.scala
[github]: https://github.com/morotsman/about_scala/tree/master/src/main/scala/scalaz_experiments/kleisli_usage
