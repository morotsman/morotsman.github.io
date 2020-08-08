---
layout: post
title:  Being naughty when subclassing
subtitle: Breaking the Law - Violating the Liskov Substitution Principle by throwing a new Exception in a Derived class
date:   2020-07-05 10:44:33 +0200
categories: The Liskov Substitution Principle
background: '/img/new_york10.jpg' 
---

## About the post
This is the first post in a series of three about the Liskov Substitution Principle. The other posts focuses on [LSP] in combination with [covariance] and [contravariance] so I will not dwell on those topics here.  

## The Liskov Substitution Principle


The Liskov Substitution Principle [LSP] states that: 

> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.)

This boils down to some requirements on the type signature:

> * Contravariance of method arguments in the subtype.
> * Covariance of return types in the subtype.
> * No new exceptions should be thrown by methods of the subtype, except where those exceptions are themselves subtypes of exceptions thrown by the methods of the supertype. 

Let's learn a little more about the Liskov Substitution Principle by investigating a design problem and during that process break the law...

![Alt Text](https://media.giphy.com/media/23nzO2bkwXykU/giphy.gif)


## Fruit store

Let's say that you run a fruit store and you have created a general interface for fruits:

{% highlight kotlin %}
interface Fruit {
    val color: String
    val weightInGrams: Int
    fun calories(): Float
    //price and so on
}
{% endhighlight %}

A concrete implementation of the `Fruit` interface could look like so:

{% highlight Ruby %}
data class Apple(override val color: String, override val weightInGrams: Int) : Fruit {
    override fun calories(): Float = (weightInGrams.toFloat() / 100) * 52
}

data class Orange(override val color: String, override val weightInGrams: Int) : Fruit {
    override fun calories(): Float = (weightInGrams.toFloat() / 100) * 47
}
{% endhighlight %}

You decide to implement a calculator so that the customer can calculate the number of fruits and calories consumed during the day. The `Person` object functions as an aggregator.

{% highlight Ruby %}
data class Person(
    val name: String,
    val caloriesConsumed: Float = 0.0f,
    val weightOfConsumedFuits: Int = 0
) {
    fun eat(fruit: Fruit): Person =
        this.copy(
            caloriesConsumed = (caloriesConsumed + fruit.calories()),
            weightOfConsumedFuits = weightOfConsumedFuits + fruit.weightInGrams
        )
    }
    
private fun caloriesAndWeightCalculator(person: Person, fruits: List<Fruit>): Person =
    fruits.fold(person){ acc, fruit -> acc.eat(fruit) }
{% endhighlight %}

The calculator can be used as follows:

{% highlight Ruby %}
val consumedFruits = listOf(
    Apple("Red", 50),
    Apple("Red", 100),
    Orange("Orange", 100)
)
    
println(caloriesAndWeightCalculator(Person("Niklas"), consumedFruits))
#=> prints Person(name=Niklas, caloriesConsumed=125.0, weightOfConsumedFuits=250)
{% endhighlight %}

## So let's Break the Law. 

![Alt Text](https://media.giphy.com/media/ARz1MgbdjyH4s/giphy.gif)

Ackee is a fruit, however it is poisonous if it is not prepared in the correct way. Since it's used for medical purposes it could still be that you sell it in the store. 

So it could be that we implement the `Fruit` api like this for `Ackee` (no need to calculate the calories since it should not be eaten):

{% highlight Ruby %}
data class Ackee(override val color: String, override val weightInGrams: Int) : Fruit {
    override fun calories(): Float = throw RuntimeException("You can't eat this!")
}
{% endhighlight %}

If we by mistake consumes an `Ackee` fruit and then try to calculate our daily intake of calories we get the following result:

{% highlight Ruby %}
val consumedFruits = listOf(
    Apple("Red", 50),
    Apple("Red", 100),
    Orange("Orange", 100),
    Ackee("Orange", 100)
)
        
println(caloriesAndWeightCalculator(Person("Niklas"), consumedFruits))
        
#=> prints Exception in thread "main" java.lang.RuntimeException: You can't eat this!
{% endhighlight %}

One could argue that getting an Exception is the least of our problems at the moment, but there is no argue in that we broke the law (the Liskov Substitution Principle that is).

## The Liskov Substitution Principle revisited

As mentioned above the Liskov Substitution Principle [LSP] states that: 
> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.)

In this case we introduced an `Exception` in one of the subtypes (`Ackee`) which causes our calculator to crash, i.e. breaking the law.

## Alternative design

So how could we avoid the problem? Lets introduce a new interface called `EatableFruit` and move the calories function to it:

{% highlight kotlin %}
interface Fruit {
    val color: String
    val weightInGrams: Int
    //and so on
}

interface EatableFruit: Fruit {
    fun calories(): Float
}
{% endhighlight %}

We now implement the subtypes like this:

{% highlight kotlin %}
data class Apple(override val color: String, override val weightInGrams: Int) : EatableFruit {
    override fun calories(): Float = (weightInGrams.toFloat() / 100) * 52
}

data class Orange(override val color: String, override val weightInGrams: Int) : EatableFruit {
    override fun calories(): Float = (weightInGrams.toFloat() / 100) * 47
}

data class Ackee(override val color: String, override val weightInGrams: Int) : Fruit
{% endhighlight %}

We have to update the calculator:

{% highlight kotlin %}
data class Person(
    val name: String,
    val caloriesConsumed: Float = 0.0f,
    val weightOfConsumedFuits: Int = 0
) {
    fun eat(fruit: EatableFruit): Person =
        this.copy(
            caloriesConsumed = (caloriesConsumed + fruit.calories()),
            weightOfConsumedFuits = weightOfConsumedFuits + fruit.weightInGrams
        )
}

private fun caloriesAndWeightCalculator(person: Person, fruits: List<EatableFruit>): Person =
    fruits.fold(person) { acc, fruit -> acc.eat(fruit) }
{% endhighlight %}

And we use the calculator like this:

{% highlight kotlin %}
val consumedFruits = listOf(
        Apple("Red", 50),
        Apple("Red", 100),
        Orange("Orange", 100)
     )

println(caloriesAndWeightCalculator(Person("Niklas"), consumedFruits))
{% endhighlight %}

It no longer possible to send `Ackee` to the calculator since it is not an eatable fruit. 

The code examples implemented in Kotlin are available at: [github]

[github]: https://github.com/morotsman/about_kotlin/blob/master/src/main/kotlin/org/example/liskov/LiskovBreakingTheLaw.kt
[LSP]: https://en.wikipedia.org/wiki/Liskov_substitution_principle
[covariance]: https://morotsman.github.io/java,/covariance,/the/liskov/substitution/principle/2020/07/12/java-covariance.html
[contravariance]: https://morotsman.github.io/java/contravariance/the/liskov/substitution/principle/2020/07/17/java-contravariance.html

