---
layout: post
date:   2020-09-26 09:09:00 +0200
categories: Scala Cats Reader Monad
title: Why don't you just ask for it?
subtitle: Using the Reader monad to write maintainable code 
background: '/img/hamburg4.jpg'  
---

## About the post

I recently read a nice [blog post] about the Reader monad written by Kyle Corbelli. The post uses Haskell for it's example so I decided to translate it to Scala using the Cats library and put my own words to it. 

## The problem

Have you ever noticed a function that takes input parameters that it's not using it self, the function only forwards them to other function that it calls.

This post is all about that and how to use the Reader monad to get well like, more readable code.

![Pun intended!](https://media.giphy.com/media/3oEduG8nljrAsExGg0/giphy.gif)

## The Core

In the [example] provided by Kyle we are developing a web page using our own small little library. The core of the library looks like this:

{% highlight scala %}

type HTML = String

private def div(children: List[HTML]): HTML =
    "<div>" + children.mkString("") + "</div>"

private def h1(children: List[HTML]): HTML =
    "<h1>" + children.mkString("") + "</h1>"

private def p(children: List[HTML]): HTML =
    "<p>" + children.mkString("") + "</p>"
    
{% endhighlight %}    

Three simple small functions that we will use to build the more complex structure that forms our web page.

## The page

The page will have the following component hierarchy:

* view
    * page
        * topnav
        * content (needs email)
            * left
            * right
                * article
                    * widget (needs email)

As we can see it's a pretty standard page. 

Notice that some of the components are marked with "needs email"? Since this is a dynamic page we will need to provide the email address each time it's rendered (the address varies each time).

## First attempt
So let's try and implement it:

{% highlight scala %}
def view(email: String) = div(List(page(email)))

private def page(email: String) =
    div(List(
      topNav(),
      content(email)
    ))

private def topNav(): HTML =
    h1(
      List("OurSite.com")
    )

private def content(email: String): HTML =
    div(List(
      h1(List("Custom content for " + email)),
      left(),
      right(email)
    ))

private def left(): HTML =
    p(List("This is the left side"))

private def right(email: String) =
    div(List(article(email)))

private def article(email: String) =
    div(List(
      p(List("This is an article")),
      widget(email)
    ))

private def widget(email: String) =
    div(List(p(List("Hey " + email + ", we've got a great offer for you!"))))
{% endhighlight %}   


The result if we invoke the view with an email address is:

{% highlight html %}
<div>
    <div>
        <h1>OurSite.com</h1>
        <div>
            <h1>Custom content for leopold.niklas@gmail.com</h1>
            <p>This is the left side</p>
            <div>
                <div>
                    <p>This is an article</p>
                    <div>
                        <p>Hey leopold.niklas@gmail.com, we've got a great offer for you!</p>
                    </div>
                </div>
            </div>
        </div>
    </div>
 </div>
{% endhighlight %} 

The code will do the job, but don't you find it irritating that we have to pass the email address all the way down the hierarchy? Take for example the `article` function, it doesn't use the mail address for anything, it just passes it on to `widget`. The code feels a little bit dirty don't you think?

In fact wouldn't it be nicer if `widget` somehow just could ask for the email address and let `artice` go about it's own business?

Something like this:
{% highlight html %}
private def widget() = {
    val email = ask
    return div(
        List(
            p(List("Hey " + email + ", we've got a great offer for you!"))
        )
    )
}
{% endhighlight %} 

However we still want the function to be pure, i.e. only depend on the input it is given, can we still do it? Can we use some magic? 

![Magic!](https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif)

Turn's out that we can!


## So why don't you just ask for it?

Let's refactor `widget` so that it's using the Reader monad:

{% highlight scala %} 
private def widget(): Reader[Context, HTML] = for (
        context <- ask[Context]
    ) yield div(
        List(
            p(List("Hey " + context.email + ", we've got a great offer for you!"))
        )
    ) 
{% endhighlight %}

If we squint our eyes just like so, is this not almost the same as:

{% highlight scala %}
private def widget() = {
    val email = ask
    return div(
        List(
            p(List("Hey " + email + ", we've got a great offer for you!"))
        )
    )
}
{% endhighlight %} 


The magic is provided by the `Reader` monad and the `ask` helper function.

{% highlight scala %}
private def ask[R]: Reader[R, R] = Reader(r => r)
{% endhighlight %} 

As you can see we are no longer returning HTML, instead we wrap HTML in the Reader monad together with a Context that contains the email address.

{% highlight scala %}
case class Context(email: String)
{% endhighlight %} 

To get the HTML we need to provide the context to the Reader by invoking it's run method.

{% highlight scala %}
val widgetReader: Reader[Context, HTML] = widget();
val widgetResult = widgetReader.run(Context("leopold.niklas@gmail.com"))
assert(widgetResult == "<div><p>Hey leopold.niklas@gmail.com, we've got a great offer for you!</p></div>")
{% endhighlight %}

Have we succeeded? Can we just ask for it and still be pure? I say

![Hell yeah!](https://media.giphy.com/media/dkGhBWE3SyzXW/giphy.gif)

## Let's apply it to the rest


So let's apply the same pattern to the rest of the code:

{% highlight scala %}
def view: Reader[Context, HTML] = for (
    p <- page
  ) yield div(List(p))

private def page = for (
    c <- content
  ) yield div(List(topNav, c))

private def topNav: HTML =
    h1(List("OurSite.com"))

private def content = for (
    context <- ask[Context];
    r <- right
  ) yield div(List(h1(List("Custom content for " + context.email)), left, r))

private def ask[R]: Reader[R, R] = Reader(r => r)

private def left: HTML =
    p(List("This is the left side"))

private def right = for (
    a <- article
  ) yield div(List(a))

private def article = for (
    w <- widget
  ) yield div(List(p(List("This is an article")), w))

private def widget = for (
    context <- ask[Context]
  ) yield div(List(p(List("Hey " + context.email + ", we've got a great offer for you!"))))
{% endhighlight %}

And that's it.

## Conclusion

If you find you're self providing input arguments to functions that are only forwarded, the `Reader` monad could be something for you.

Once again, here is the original [blog post] that I translated to Scala and put some other words to.

Happy coding!

## Github

All the code is available at [github].

[blog post]: https://engineering.dollarshaveclub.com/the-reader-monad-example-motivation-542c54ccfaa8
[example]: https://engineering.dollarshaveclub.com/the-reader-monad-example-motivation-542c54ccfaa8
[github]: https://github.com/morotsman/about_scala/blob/master/src/main/scala/scalaz_experiments/reader_monad/ReaderMonadUsage.scala
