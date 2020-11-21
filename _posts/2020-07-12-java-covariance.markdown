---
layout: post
title:  Covariance? Is that useful?
date:   2020-07-12 10:44:33 +0200
categories: Java, Covariance, The Liskov Substitution Principle
subtitle: "Covariance and the Liskov Substitution Principle"
background: '/img/new_york7.jpg' 
---

## About the post
This is the second post in a series of three about the Liskov Substitution Principle. The other posts focuses on [LSP] in combination with [exceptions] and [contravariance] so I will not dwell on those topics here.  


## The Liskov Substitution Principle

The Liskov Substitution Principle [LSP] states that: 

> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.)

This boils down to some requirements on the type signature:

> * Contravariance of method arguments in the subtype.
> * Covariance of return types in the subtype.
> * No new exceptions should be thrown by methods of the subtype, except where those exceptions are themselves subtypes of exceptions thrown by the methods of the supertype. 

Let's learn a little more about the Liskov Substitution Principle and covariance by investigating a design problem.

## The setup

We run a small fruit store and have created a small class hierarchy that models different kinds of fruits.

{% highlight java %}

interface Fruit {
	String color();
};

class Orange implements Fruit {
	public String color() {
		return "Orange";
	}
}

class Apple implements Fruit {
	private final String color;

	public Apple(String color) {
		this.color = color;
	}

	public String removeAppleCores() {
		return "Apple cores removed";
	}

	@Override
	public String color() {
		return color;
	}
}

{% endhighlight %}

## The Problem

We would like to group our fruits based on color. This is not a made up thing by the way, people actually do things like that.  

![Fruit grouped by color]({{ site.url }}/assets/covariance/fruits_grouped_by_color.jpeg)

In Java code this looks like:

{% highlight java %}
private static Map<String, List<Fruit>> groupFruitByColor(List<Fruit> fruits) {
	return fruits.stream().collect(groupingBy(Fruit::color));
}
{% endhighlight %}

And the client would use it like so:
{% highlight java %}
List<Fruit> listOfFruit = new ArrayList<>(asList(new Apple("Red"), new Apple("Green"), new Orange(), new Orange()));
groupFruitByColor(listOfFruit)
{% endhighlight %}

Ok, so now group this list of apples instead.

{% highlight java %}
List<Apple> listOfApple = new ArrayList<>(asList(new Apple("Red"), new Apple("Green")));
groupFruitByColor(listOfApple) // will not compile
{% endhighlight %}

But oh, here comes trouble, the compiler tells us that it expects a `List<Fruit>`, not a `List<Apple>`. 

Or in the words of Gandalf:

![You shall not pass](https://media.giphy.com/media/njYrp176NQsHS/giphy.gif)

But why you may ask? After all it looks totally safe to send in a `List<Apple>` to method that expect a `List<Fruit>`, this since an `Apple` is a subtype of `Fruit` and hence should support all the operations of `Fruit`. 

The operation used by `groupFruitByColor` for example is color and we know that all subtypes of `Fruit` (i.e. `Apple` and `Orange`) must implement this method. 

One way of solving the problem would be to use that old copy/paste trick:

{% highlight java %}
private static Map<String, List<Fruit>> groupFruitByColor(List<Fruit> fruits) {
	return fruits.stream().collect(groupingBy(Fruit::color));
}

private static Map<String, List<Apple>> groupAppleByColor(List<Apple> fruits) {
	return fruits.stream().collect(groupingBy(Apple::color));
}
{% endhighlight %}

But this violates the [DRY] principle, so what to do, what to do?

![Quandary](https://media.giphy.com/media/LNlvkpX17qp2Q3ARN0/giphy.gif)


## Covariance to the rescue

So how can we convince the compiler that it's safe to call `groupFruitByColor` with a `List<Apple>`? Well, we could use [covariance]!

Covariance is declared like this in Java:

{% highlight java %}
private static Map<String, List<Fruit>> groupFruitByColorCovariant(List<? extends Fruit> fruits) {
	return fruits.stream().collect(groupingBy(Fruit::color));
}
{% endhighlight %}

And now the compiler is happy!

{% highlight java %}
List<Fruit> listOfFruit = new ArrayList<>(asList(new Apple("Red"), new Apple("Green"), new Orange(), new Orange()));
List<Apple> listOfApple = new ArrayList<>(asList(new Apple("Red"), new Apple("Green")));
groupFruitByColorCovariant(listOfFruit);
groupFruitByColorCovariant(listOfApple);
{% endhighlight %}

Better check with Gandalf...

![Gandalf happy](https://media.giphy.com/media/N30oja8TjKdos/giphy.gif)

Oh yeah, Gandalf is happy!

But what exactly did we do? Consider the following code:

{% highlight java %}
Fruit fruit;
Apple apple = new Apple("Red");
fruit = apple;
{% endhighlight %}

It's safe to assign an `Apple` to a `Fruit` variable since all operations of the supertype (`Fruit`) must be supported by the subtype (`Apple`).

But if we do the same to a `List<Fruit>` and a `List<Apple>`, the compiler complains.

{% highlight java %}
List<Fruit> fruits;
List<Apple> apples = new ArrayList<>(asList(new Apple("Red"), new Apple("Green")));
fruits = apples; // will not compile 
{% endhighlight %}

The reason for this is that [generic types]: like `List`, `Optional` or `Set` is by the default invariant in Java. By that we mean that `List<Apple>` is not a subtype of `List<Fruit>`. So if we want `List<Apple>` to be a subtype of `List<Fruit>` (i.e. to be covariant on `Fruit`) we must override the default behaviour like this:

{% highlight java %}
List<? extends Fruit> fruitsCovariant;
fruitsCovariant = apples;
Fruit someFruit = fruitsCovariant.get(0);
{% endhighlight %} 

Even if `fruitsCovariant` in the example above in reality is a `List<Apple>` it is safe to use for the client to use since `Apple` supports all operations of `Fruit`. We have made `fruitsCovariant` covariant on `Fruit`. 

So what is [covariance]? Well, it's defined like this on [wikipedia]:

> Within the type system of a programming language, a typing rule or a type constructor is:
> * covariant if it preserves the ordering of types (≤), which orders types from more specific to more generic
> * contravariant if it reverses this ordering

In our case we preserve the ordering of types since: `Fruit` is the supertype of `Apple` and we have declared the `List<Fruit>` (`fruitsCovariant`) to be the supertype of all lists that contains `Fruit` or any subtype of `Fruit` (for example `List<Apple>`). This means that we have fulfilled the definition of something that is covariant. 

So why is not a `List` covariant by default in Java, after all this looks like the desired behaviour?

## The perils of mutability

Well, the thing is that Java relies on mutability to get the work done. Let's investigate how a generic type would behave if it was covariant by default by looking on an array.

An array in Java is covariant by default:

{% highlight java %}
Fruit[] arrayOfFruit;
Apple[] arrayOfApple = new Apple[]{new Apple("Red"), new Apple("Green")};
{% endhighlight %} 
  
Better check with Gandalf...

![Gandalf happy](https://media.giphy.com/media/N30oja8TjKdos/giphy.gif)

Oh yeah, Gandalf is still happy! But wait! Should he?

What happens if we do the following:

{% highlight java %}
arrayOfFruit[0] = new Orange();
{% endhighlight %} 

Well, the compiler don't complain, it's ok to insert an `Orange` in an array of `Fruit`, this since an `Orange` is a subtype of `Fruit`. But if we do the following: 

{% highlight java %}
arrayOfApple[0] = new Orange();
{% endhighlight %} 

We get an error by the compiler since an `Orange` is not a subtype of `Apple`, how does that add up? Well it don't, the array in Java is broken and it's very possible to make a fool of the compiler. The enemy is past the gate and we ends up with a run time exception: `ArrayStoreException`.

![Gandalf sad](https://media.giphy.com/media/oM5xTkZM5N1ZK/giphy.gif)

But why did we end up in trouble? Well the reason was that the array is mutable, i.e. it can be modified after it is created. In this case the fact that `arrayOfFruit` and `arrayOfApple` references the same data structure, coupled with the fact that an array is mutable leads to disaster. 

So have we introduced the same problem when making our List of `Fruit` covariant? Well yes and no, we would have if the compiler let us add or update an element in the List, but does it?

{% highlight java %}
List<? extends Fruit> listOfFruitCovariant = new ArrayList<>(asList(new Apple("Red"), new Apple("Green")));
listOfFruitCovariant = listOfApple;
listOfApple.add(new Apple("Green"));
Fruit myFruit = listOfFruit.get(0); // it's safe to read from a covariant List
Apple myApple = listOfApple.get(0);

// The compiler will however complain about this:
listOfFruitCovariant.add(new Apple("Red")); // dangerous operation, not allowed by the compiler
listOfFruitCovariant.add(new Orange()); // dangerous operation, not allowed by the compiler
{% endhighlight %} 

No, we have not! The downside(?) is that we can only read from the covariant array which of course is a problem in a language like Java that relies on mutability to get the work done. 


## Another example

Let us investigate how another another generic type behaves in regard of covariance: the `Supplier`. Also since it looks like I have captured your interest I will skip the GIF's in the rest of this post, on the other hand feel free to skip the rest of the post if the GIF's was what captured your interest. 

{% highlight java %}
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
{% endhighlight %}

The `Supplier` interface was introduced in Java 8 when Java got support for lambdas and is used like so:

{% highlight java %}
Supplier<Apple> supplierOfApple = () -> new Apple("Green");
Apple apple = supplierOfApple.get();
{% endhighlight %}

Since generic types are invariant the following is not possibe:

{% highlight java %}
Supplier<Fruit> supplierOfFruit;
Supplier<Apple> supplierOfApple = () -> new Apple("Green");
supplierOfFruit = supplierOfApple; // will not compile
{% endhighlight %}

But is that a problem? Consider the following code:

{% highlight java %}
private static void printColorOfFruit(Supplier<Fruit> supplier) {
	printColorOfFruit(supplier.get());
}

private static void printColorOfFruit(Fruit fruit) {
	System.out.printf("The color is: %s%n", fruit.color());
}
{% endhighlight %}

`printColorOfFruit` can be invoked like this:

{% highlight java %}
Supplier<Fruit> supplierOfFruit = new Supplier<Fruit>() {
	@Override
	public Fruit get() {
		return new Apple("Red");
	}
}; 

// or in a more concise syntax
printColorOfFruit(() -> new Apple("Red"));
{% endhighlight %}

But what if we would like to call `printColorOfFruit` with a `Supplier<Apple>`? After all this should be safe since an `Apple` is a subtype of `Fruit` and must support all operations of Fruit.

{% highlight java %}
Supplier<Apple> supplierOfApple = () -> new Apple("Green");
printColorOfFruit(supplierOfApple); // will not compile
{% endhighlight %}

This is not possible. One way to fix the problem is to use copy/paste but we have already concluded that this is not optimal.
{% highlight java %}
private static void printColorOfFruit(Supplier<Fruit> supplier) {
	printColorOfFruit(supplier.get());
}

private static void printColorOfApple(Supplier<Apple> supplier) {
	printColorOfFruit(supplier.get());
}
{% endhighlight %}

Could we use covariance again? Turn's out that we can:

{% highlight java %}
private static void printColorOfFruitCovariant(Supplier<? extends Fruit> supplier) {
	printColorOfFruit(supplier.get());
}

printColorOfFruitCovariant(supplierOfFruit);
printColorOfFruitCovariant(supplierOfApple);
{% endhighlight %}

This works since by declaring the `Supplier` as covariant in `Fruit` we have turned the `Supplier<Apple>` to a subtype of a `Supplier<Fruit>`.

{% highlight java %}
Supplier<? extends Fruit> supplierOfFruitCovariant;
supplierOfFruitCovariant = supplierOfApple; // supplierOfApple is now a subtype
{% endhighlight %}

## The Liskov Substitution Principle revisited

The Liskov Substitution Principle [LSP] states that: 

> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.)

This boils down to some requirements on the type signature:

> * Contravariance of method arguments in the subtype.
> * Covariance of return types in the subtype.
> * No new exceptions should be thrown by methods of the subtype, except where those exceptions are themselves subtypes of exceptions thrown by the methods of the supertype. 

So why should the return types be covariant? Because we get a more flexible API (no copy/paste) and it is safe for the client to use.
    
{% highlight java %}
Supplier<? extends Fruit> supplierOfFruitCovariant;
supplierOfFruitCovariant = supplierOfApple; // supplierOfApple is now a subtype
Fruit fruit = supplierOfFruitCovariant.get();
{% endhighlight %}

In the code above it is always safe for the client to call `supplierOfFruitCovariant`. The return type in this case is `Fruit` which we have declared covariant as instructed by Liskov.

And why is it safe? It is safe since the client expects that a `Fruit` is returned by supplierOfFruitCovariant. In the reality the client gets an `Apple`, but that doesn't matter since an `Apple` is a subtype of `Fruit` and hence safe to use for the client.

The same goes for our covariant list of fruit:

{% highlight java %}
List<Apple> apples = new ArrayList<>(asList(new Apple("Red"), new Apple("Green")));
List<? extends Fruit> fruitsCovariant;
fruitsCovariant = apples;
Fruit someFruit = fruitsCovariant.get(0);
{% endhighlight %} 

The client expects a `List<Fruit>`, it could not care less if in reality it gets a `List<Apple>` or a `List<Orange>` (or a mix) since it only uses operations supported by `Fruit` which in turn must be supported by the subtypes of `Fruit` (`Apple` and `Orange`).  

## A final example

{% highlight java %}
interface FruitCovariant {
	Joice getJoice();
};

class OrangeCovariant implements FruitCovariant {
	/* will not compile since Object is not a subtype of Joice, it is a supertype
	@Override
	public Object getJoice() {
		return new OrangeJoice();
	}
	 */

    /* works like a charm since OrangeJoice is a subtype of Joice */
	@Override
	public OrangeJoice getJoice() {
		return new OrangeJoice();
	}
}

class AppleCovariant implements FruitCovariant {
    
    /* this will also work */ 
	@Override
	public Joice getJoice() {
		return new AppleJoice();
	}
}

interface Joice {
}

class AppleJoice implements Joice {
}

class OrangeJoice implements Joice {
}
{% endhighlight %} 

In the code above we are covariant in the return type of the `getJoice` method in `OrangeCovariant` which is a subtype of `FruitCovariant`, i.e. a direct translation of "Covariance of return types in the subtype". Once again it is quite safe for the for a client of `FruitCovariant` to use `getJoice` since it will get a `Fruit` or a subtype of `Fruit` in return. 

The code examples are available at [github].

If you enjoyed this post you may be interested in a related post about [contravariance] where we investigate how it can be used to get a more flexible API.


[covariance]: https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)
[LSP]: https://en.wikipedia.org/wiki/Liskov_substitution_principle
[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
[generic types]: https://docs.oracle.com/javase/tutorial/java/generics/types.html
[github]: https://github.com/morotsman/about_scala/tree/master/src/main/scala/generics/java/covariance
[contravariance]: https://morotsman.github.io/java/contravariance/the/liskov/substitution/principle/2020/07/17/java-contravariance.html
[exceptions]: https://morotsman.github.io/the/liskov/substitution/principle/2020/07/05/breaking-the-law.html
[wikipedia]: https://morotsman.github.io/java/contravariance/the/liskov/substitution/principle/2020/07/17/java-contravariance.html
