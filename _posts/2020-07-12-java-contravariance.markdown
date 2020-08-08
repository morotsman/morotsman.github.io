---
layout: post
title: Covariance and now Contravariance, do I really need to know this?
subtitle: Contravariance and the Liskov Substitution Principle
date:   2020-07-17 09:40:00 +0200
categories: Java Contravariance The Liskov Substitution Principle
background: '/img/new_york6.jpg' 
---

## About the post
This is the third post in a series of three about the Liskov Substitution Principle. The previous posts focuses on [LSP] in combination with [exceptions] and [covariance] so I will not dwell on those topics here.  

## The Liskov Substitution Principle
The Liskov Substitution Principle ([LSP]) states that: 

> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.)

This boils down to some requirements on the type signature:

> * Contravariance of method arguments in the subtype.
> * Covariance of return types in the subtype.
> * No new exceptions should be thrown by methods of the subtype, except where those exceptions are themselves subtypes of exceptions thrown by the methods of the supertype. 

Let's learn a little more about the Liskov Substitution Principle and contravariance by investigating a design problem. 

## The setup

We run a small fruit store and have created a small class hierarchy that models different kinds of fruits.

{% highlight java %}
interface Fruit {
	String color();
};

interface Eatable extends Fruit {
	String joice();
}

class Orange implements Eatable {
	public String color() {
		return "Orange";
	}

	@Override
	public String joice() {
		return "Orange joice";
	}
}

class Apple implements Eatable {
	private final String color;

	public Apple(String color) {
		this.color = color;
	}

	@Override
	public String color() {
		return color;
	}

	@Override
	public String joice() {
		return "Apple joice";
	}

	public void removeAppleCore() {
		System.out.println("Apple core removed");
	}
}
{% endhighlight %}

## A more flexible signature?

We will use the generic type `Consumer` as our test vehicle:  

{% highlight java %}
@FunctionalInterface
public interface Consumer<T> {

    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);
}
{% endhighlight %}

The `Consumer` interface was introduced in Java 8 when Java got support for lambdas and is used like so:

{% highlight java %}
Consumer<Apple> myConsumer = new Consumer<Apple>() {
	@Override
	public void accept(Apple apple) {
		System.out.println("I like: " + apple.color() + " apples!");
	}
};
myConsumer.accept(new Apple("Yellow"));

// the same but more concise syntax
Consumer<Apple> myConsumer1 = (Apple a) -> System.out.println("I like: " + a.color() + " apples!");
myConsumer1.accept(new Apple("Red"));

// the same but even more concise syntax
Consumer<Apple> myConsumer2 = a -> System.out.println("I like: " + a.color() + " apples!");
myConsumer2.accept(new Apple("Green"));
{% endhighlight %}

Ok, so let's say that you have implemented a function that supplies a `Consumer` with an `Eatable` fruit:

{% highlight java %}
private static void getEatableFruits(Consumer<Eatable> eatableConsumer) {
	eatableConsumer.accept(new Apple("Red"));
}
{% endhighlight %}

After we have sent a `Consumer` to the getEatableFruits operation it is supplied a nice `Eatable` fruit of unspecified kind. 

Armed with the knowledge from the my previous post about [covariance] we know that this is probably not the most flexible signature that is possible. Could we use covariance?

## Covariance to the rescue?

We could change the signature to this:
{% highlight java %}
private static void getEatableFruitsCovariant(Consumer<? extends Eatable> eatableConsumer)
{% endhighlight %}

and then use the updated method like so:
{% highlight java %}
Consumer<? extends Eatable> eatableConsumerCovariant;
Consumer<Apple> appleConsumer = (Apple a) -> System.out.println("With this fruit I can: " + a.removeAppleCore());
eatableConsumerCovariant = appleConsumer;
getEatableFruitsCovariant(eatableConsumerCovariant);
{% endhighlight %}

This looks amazing and it is also much more flexible! Now we only have to implement `getEatableFruitsCovariant` and we are done! It goes like this: 

{% highlight java %}
private static void getEatableFruitsCovariant(Consumer<? extends Eatable> eatableConsumer) {
	Apple apple = new Apple("Red");
	// eatableConsumer.accept(apple); // will not compile
}
{% endhighlight %}

![Computer says no](https://media.giphy.com/media/kQbMO5X7UA1C8/giphy.gif)

**Wait! What! Why?**

Well, it turns out that it is not safe to declare a `Consumer` to be covariant on `Eatable`. Why is that?

Let's pretend for a moment that we did not get a compile error in the example above! If a `Consumer` is declared to be covariant it would be ok to call it like this:

{% highlight java %}
private static void getEatableFruitsCovariant(Consumer<? extends Eatable> eatableConsumer) {
	Orange orange = new Orange();
	eatableConsumer.accept(orange); // will not compile in reality

	Apple apple = new Apple("Red");
	eatableConsumer.accept(apple); // will not compile in reality
}
{% endhighlight %}

And the client would use the method like this: 
{% highlight java %}
Consumer<? extends Eatable> eatableConsumerCovariant;
Consumer<Apple> appleConsumer = (Apple a) -> System.out.println("With this fruit I can: " + a.removeAppleCore());
eatableConsumerCovariant = appleConsumer;
getEatableFruitsCovariant(eatableConsumerCovariant);
{% endhighlight %}  

Do you see the problem? Since we have declared that the `Consumer` is covariant on `Eatable` we have said that `Consumer<Apple>` is a subtype of `Consumer<Eatable>`. If that is the case we can (according to [LSP]) replace a `Consumer<Eatable>` with a `Consumer<Apple>`. 

Since an `Orange` also is a subtype of `Eatable` it is totally ok to feed the `Consumer<? extends Eatable>` with an `Orange`, which is the core of the problem. 

So, the client is free to use a `Consumer<Apple>` when calling `getEatableFruitsCovariant` but at the same time `getEatableFruitsCovariant` is free to call the `Consumer<? exends Eatable>` with an `Orange`. But an `Orange` doesn't have the `removeAppleCore` that we later calls in `Consumer<Apple>`!  

We fought the law and the law won ([LSP] that is).

![Alt Text](https://media.giphy.com/media/ARz1MgbdjyH4s/giphy.gif)

It is clear that we can't declare a `Consumer<Apple>` to be a subtype of `Consumer<Eatable>`.


## Contravariance to the rescue

Ok, so what can we do? We can use [contravariance]! When using contravariance we goes in the other direction when subtyping. So if an `Eatable` is a subtype if `Fruit` then a `Consumer<Fruit>` is a subtype of `Consumer<Eatable>`.

It's defined like this on [wikipedia]:

> Within the type system of a programming language, a typing rule or a type constructor is:
> * covariant if it preserves the ordering of types (≤), which orders types from more specific to more generic;
> * contravariant if it reverses this ordering

In Java that looks like this:

{% highlight java %}
Fruit fruit;
Eatable eatable = new Apple("Red");
fruit = eatable; //works since eatable is a subtype of fruit
		
Consumer<? super Eatable> eatableConsumerContravariant;
Consumer<Fruit> fruitConsumer = (Fruit f) -> System.out.println("The color of the fruit is: " + f.color());
eatableConsumerContravariant = fruitConsumer; // works since fruitConsumer is a subtype of eatableConsumerContravariant
{% endhighlight %}  

Ok, so let's try it out, we implement `getEatableFruits` like this instead: 

{% highlight java %}
private static void getEatableFruitsContravariant(Consumer<? super Eatable> eatableConsumer) {
	Orange orange = new Orange();
	eatableConsumer.accept(orange); 

	Apple apple = new Apple("Red");
	eatableConsumer.accept(apple); 
	
	Fruit fruit = new Fruit() {
    	@Override
    	public String color() {
    		return "Red";
    	}
    };
    // eatableConsumer.accept(fruit); // will not compile
}
{% endhighlight %}

No complaints from the compiler this time, so far so good! The client of getEatableFruitsContravariant will use the method like this:

{% highlight java %}
Consumer<? super Eatable> eatableConsumerContravariant = (Eatable e) -> System.out.println("With this fruit I can produce: " + e.joice());
getEatableFruitsContravariant(eatableConsumerContravariant);

Consumer<Fruit> fruitConsumer = (Fruit f) -> System.out.println("The color of the fruit is: " + f.color());
getEatableFruitsContravariant(fruitConsumer);
{% endhighlight %}

Still no complaints from the compiler! We have found the flexible signature for our method, and it's always safe to use for the client! Why is that?
  
Well, we have to look at what is given to the `Consumer` and what the consumer expects. We have stated that a `Consumer<Fruit>` is a subtype of `Consumer<Eatable>`. 

We know that the type of what we give to the `Consumer` in this case always will be an `Eatable` or a subtype of `Eatable` (for example an `Apple`).

So the `Consumer<Eatable>` in this case will be given at least an `Eatable` which is no problem since it assumes that this is the case, it is not possible for it to use methods on for example `Apple`.

What about `Consumer<Fruit>`? It will also be given at least an `Eatable` but it assumes even less than `Consumer<Eatable>`, it assumes that it is given a `Fruit`. Since it only uses methods on `Fruit` it is quite safe to send an `Eatable` (or a subtype of `Eatable`) to it since `Eatable` is a subtype of `Fruit` and hence must support all operations on `Fruit`. 

All is well!    


## Conclusions

The Liskov Substitution Principle [LSP] states that: 

> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.)

This boils down to some requirements on the type signature:

> * Contravariance of method arguments in the subtype.
> * Covariance of return types in the subtype.
> * No new exceptions should be thrown by methods of the subtype, except where those exceptions are themselves subtypes of exceptions thrown by the methods of the supertype. 

In our example we declared `Consumer<Eatable>` to be contravariant on `Eatable` (the in argument) which lead to that `Consumer<Fruit>` is a subtype of `Consumer<Eatable>`.
    
We observed that this is a safe thing to do and that we got a more flexible API. 
    
The code examples are available at [github].

[covariance]: https://morotsman.github.io/java,/covariance,/the/liskov/substitution/principle/2020/07/12/java-covariance.html
[LSP]: https://en.wikipedia.org/wiki/Liskov_substitution_principle
[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
[generic types]: https://docs.oracle.com/javase/tutorial/java/generics/types.html
[github]: https://github.com/morotsman/about_scala/tree/master/src/main/scala/generics/java/contravariance
[exceptions]: https://morotsman.github.io/the/liskov/substitution/principle/2020/07/05/breaking-the-law.html
[contravariance]: https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)
[wikipedia]: https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)
