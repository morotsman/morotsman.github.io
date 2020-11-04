---
layout: post
date:   2020-10-25 09:09:00 +0200
categories: Agile Tests Automatic
title: Good code going bad
subtitle: Why I love automatic tests and you should do to
background: '/img/hamburg5.jpg'  
---

## About the post

If you have been working in the software industry for a while then maybe, just maybe you have encountered systems where you have dreaded to make even a small change? I most certainly have. These are the systems that we call legacy. 

![Fear!](https://media.giphy.com/media/14ut8PhnIwzros/giphy.gif)

When working in a legacy system we always add a lot more time on the estimates than we ever thought we will need. But even so we somehow ends up spending more time than it was estimated and on top of that cause some sort of problem in production with a functionality that we didn't even knew was supported by the system.   

The managers are not happy and remembers the good old times when the system first was written by a stellar team that got functionality out of the door fast as lightning. It's a pity that those guys doesn't work here anymore :-(.

But what has gone wrong? Are the new developers you hired really that bad or has something else happened? 

## The death of a system

Here's what I think:

Once upon a time there was a team tasked to develop a service. They where very focused on doing a quick delivery and actually managed to achive their goals, yeah, these where the good old times. The team didn't bother writing automatic tests or documenting the functionality of the system, be that of lack of time or lack of finding it important. 

As time goes by new requirements are received and implemented in the system. Not all requirements totally fit with the architecture and sometimes the teams does the occasional "hack" to implement them but mainly everything is good.

More time goes by and people starts to leave the team and are replaced by new developers. 

New requirements still arrives but now the code is not that clear anymore, the team has a bit of trouble to understand the original intent of the architecture. 

New technologies emerges on the scene and the newer developers doesn't know so much about the old tech stack used. On top of that the system has grown pretty large, in fact so large that no one knows exactly about all the functionality that it supports. 

Fast forward a couple of years more: The system is now in pretty bad shape and even small changes takes a long time to implement, on top of that bugs are more often introduced. Since no one has a clear understanding of how things was meant to be implemented it is hard to know if a change is a hack or follows the intention of the original architecture. Eventually the teams notifies the management that the system can't be maintained anymore, it needs to be rewritten. 

The system is now dead, long live the system (provided that the rewrite project is a success). 

And now the cycle repeats?

## Daddy, does all systems have to die?

Well, in life nothing last forever, and I belive this to be true in the world of computer systems as well. But I also believe, that a system should be able to live on until there is no need for it anymore, until it is utterly and absolutely obsolete.

Wow, that sounds like the lyrics for a song, please feel free to use it:

{% highlight scala %}
In life nothing last forever
And I belive this to be true 
In the world of computer systems as well
But I also believe
That a system should be able to live on

Until there is no need for it anymore, 
Until it is utterly and absolutely obsolete.

Until there is no need for it anymore, 
Until it is utterly and absolutely obsolete.
{% endhighlight %}

![Sing a song!](https://media.giphy.com/media/h1RM9LfnL6PZAdKieL/giphy.gif)

By obsolete I mean that there will eventually come a time when the environment in that the system exists has change so much so that we have no use for the system anymore. Maybe the organisation that developed the system cease to exist, maybe the business case that the system supported no longer gives value.

So how can we avoid a rewrite? Well the first thing we have to do is recognize the forces we are working against so that they can be countered! 

## Information loss 
    
In the beginning of a green field project we have information in abundant. Someone has done an investigation of what the responsibility of the system should be about, most people in the team are involved, we live and breath the system. But even here in the beginning the downfall start.

Have you ever noticed that code you wrote just a week ago looks a little bit more unclear then when you wrote it? In reality it's the same code but since then you have implemented three other functionalities and hence your memory is a little (just a little) bit vague and the code is a little (just a little) harder to read. 

In just that week we have lost a little bit information. At this stage it doesn't matter that much since we still have access to the source of the information. 

Forward a couple of years.

One by one people starts to leave the team, and now the [whispering game] starts. The people that are leaving are supposed to transfer their knowledge to the other team members but each time knowledge is transferred some information are lost or misunderstood. 

New people are hired that never was part of the green field project and their knowledge of the domain and the system are incomplete.

Reorganisations in the company twists, turns and sometimes even breaks the team, and each time some information are lost. At worst responsibility boundaries are moved, maybe the team is split in a frontend/backend team or or part of it is outsourced, teams that used to be close are suddenly remote, no respect for [Conway's law], yeah, you know the drill. 

Eventually so much information has been lost that nobody knows the full responsibility of the system or for that matter the intent of the original architecture of the system.
    
## Taking short cuts 

Sooner or later there will come a time when a requirement just don't fit in. For some reason the architecture of the system doesn't flex in the right way. You know that you have to refactor the code in that sensitive part of the system and then you have to perform regression tests that will last for weeks. However the marketing team has promised the users that this functionality will be available next week.

A tricky situation, and then someone realise: Oh, but we can do this quick fix. For sure it violates the "this and that" principle of the architecture but not by that much. We can also register an issue in the tech backlog so that we do a cleanup when we have more time.

Strange enough this mythical era "`when we have more time so that we can do cleanups`" never seem to materialise. Instead the pattern described above is repeated over and over again.

Given enough time all those small violations of the architecture will lead to incomprehensible code.

A funny thing to do when finding incomprehensible code, well at least if you are a nerd like me, is to take a historical journey (using git as the time machine) and look how it come to be in this state. Often you find nicely written code that over time was bent and twisted to something far less appealing.  

![Nerd!](https://media.giphy.com/media/phGElmSM4P0sg/giphy.gif)

## Not updating third party dependencies

It's a sad thing, opening up a dependency management file and finding that it has not been updated for several years. The funny thing is that third party libraries will become outdated even if you don't do anything (or rather because of it), it happens by itself, just give it time.

Outdated libraries becomes a problem when you try to use that new library and  get conflicts with your transitive dependencies. Another obvious problem is security. Also it is not that funny as a developer to use Spring 4 when you know that Spring 5 is out there waiting for you.

So why just not update the dependencies? Well once again there is that dreadful regression thingy to think about, who knows what bugs you introduce when upgrading the libraries and they can occur in any place of the system. 

## Not updating the tech stack

Do you remember Angular 1? I do since I have been working with it, and a lot of other people have too. Do you remember how you felt when they announced Angular 2 and you realised that all that Angular 1 code now was [outdated]. That you would have to rewrite a lot of stuff if you wanted to upgrade? Suddenly your application didn't feel that fresh anymore, am I right?   
   
This is an example where the tech stack gets outdated and it will happen to every system sooner or later. There are many other examples of technologies that doesn't feel so fresh anymore.

An outdated tech stack makes it harder to recruit people to your team, partly because people lack knowledge, partly because developers prefer to work with things that looks good on their CV.

The "not updating the tech stack" is similar to "not updating third party dependencies" but not exactly the same. With updating the tech stack I'm thinking about bigger changes such as switching from JSP to ReactJS or from Angular 1 to Angular 2. With updating dependecies I'm more thinking about upgrading a library that is already in use.  

## Counter measures

So now we have gone through the things that makes good code go bad: Loss of information, Taking shortcuts, not updating third party dependencies and finaly not updating the tech stack. 

Let's see if we can find some counter measures!

## Counter measure for information loss

So one obvious counter measure to prevent information loss is writing documentation about the code and architecture. However for this to work the team must be rigorous and always update the documentation when things change. This is something I have yet to experience, often I find a lot of documentation that was written when the project first was started but then hasn't been updated in a long time.

Wouldn't it be great if we instead automatically detected if the documentation is stale and even fail the build if that is the case?

So how can we achieve this? By using a [BDD] framework!

If you use a framework like [cucumber] you specify your requirements in the [gherkin] syntax like this

{% highlight scala %}
Feature: Dispense a coin
  The customer must insert a dollar to unlock the candy machine

  Scenario: Unlock the candy machine
    Given the candy machine is locked
    When a dollar is inserted
    Then the candy machin should be unlocked
    
  Scenario: Get a candy
    Given the candy machine is unlocked
    When the customer turns the knob
    Then a candy should be dispensed    
{% endhighlight %}

Each statement in the specification is connected to code that tests that the system behaves as expected. For example, if the system is a web service the `When a dollar is inserted` is connected to code that makes a call to a Rest API. `Then the candy machin should be unlocked` on the other hand would be connected to test code that calls the Rest API to get the current status of the system.

If the price for the candy is increased to two dollars, the test will fail the build and someone has to update the documentation.

I have found it very useful to document the system in this way when writing functional tests, i.e. tests that test the whole system. In this way we both have a automatic test regression suite and a living documentation of how the system is supposed to work.

As an example when we had great usage of test cases like this was when one of our micro services got into trouble with load in production. We decided to rewrite the whole service. The requirement on the new service was the same as the old (apart from better scalability). Here we had great usage for our automatic tests, they told us the requirements of the system. In fact the test code was in the end the only thing that remained from the first version of the system (slightly modified) 

By the way did we kill the system when rewriting it? It's a philosophical question. We didn't think of it that way, the service had to be rewritten since the user base had grown. We always knew that we someday had to do this but we choose to implement a simple and cheep solution and then evolve it when it was needed. Since we had our runnable specifications we were in total control.

An example of when I wished that we had this kind of documentation in a legacy system was when my product owner asked me why we didn't allow a dot (.) as a valid character in the street address and why it was limited to 32 characters when the limit in the db was 40. 

Alas I didn't have this information since the code was written several years before I started at the company, and I had to tell him that, well I don't know. Could it be a bug? Does one of the systems that we call have a limit on 32 characters on the street address? The system that we call, calls yet other systems, maybe one of them have a limit on 32 characters. Could it be something that is printed that looks ugly if we have more the 32 characters?

What I wished for was that someone had written a test case like this:

{% highlight scala %}
Feature: Only allow 32 characters on the street address field
  We can not allow more then 32 characters on the street address field

  Scenario: Enter the street address
    Given a webshop order
    When providing the street address 
    Then the address must be limited to 32 characters since...
{% endhighlight %}

## Counter measure for taking short cuts

Why do we take short cuts? There can be several reason, maybe we want to avoid a long regression test, maybe we want to avoid changing code because of fear. Whatever the reason for the short cut or a "hack" the end result is the same, a code base that has been compromised and by that, harder read and to modify in the future.

The counter measure for this is to refactor the code and the architecture so that the new requirement fits in. So why don't we always do that?

Well, refactoring is enabled by having automatic tests. If you don't have automatic tests and are refactoring code that you are not familiar with, you take a huge risk. Maybe you misunderstand something and by that introduce a bug in production, not a funny feeling.

![Bug!](https://media.giphy.com/media/3o6ZtaRlFW0zPiTb0c/giphy.gif)

If the tests on the other hand do exist, you can do your refactoring without risk! Here we have use for both unit tests and functional tests. The unit tests for a smaller refactoring that only affects a single class, functional tests for larger refactorings that affect more of the system. 

## Counter measure for not updating third party dependencies

Once again our automatic test suites come to rescue! Since we have tests that documents how the system is supposed to behave, we can do our dependency update and then run the tests. If the tests works we are in the clear, if not we can see exactly what breaks and then fix the problem.

## Counter measure for not updating the tech stack

Do we have usage for our automatic test here as well? You bet!

Replace the database? Why not? Your tests that calls your Rest API will tell you if everything has worked out. 

Switch from Java/Spring to Scala/Play? Just run the tests. As long as they have been written in a technology agnostic way there is no problem.  

A good idé is to also adopt a micro service architecture. You can then choose the DB that is a perfect fit for a certain use case or maybe choose to use the Play framework to get better scalability for that heavily used "this and that" function.  

It'a also easier to replace components of the tech stack in a smaller code base.

## Treat you test code as a first class citizen

Since the automatic test suites are a key component to successfully maintain a system it's important to treat the test code as a first class citizen. This means that you have to follow good practices when writing the code and make sure that the code is maintainable. 

Yes, you will have to do refactoring in the test code as well, and yes the code review for this code is as important as for the production code and if it's hard or takes a long time to write the tests, something is wrong and you have to do something about it! 

And who knows, maybe the test code will be the only thing that survives the next time you do a rewrite of your micro service?

## Finding the right balance

I love automatic tests, in fact I sometimes have been accused to overdo it. As with everything it's important to find the right balance and that is something I have been working on.

For example, I don't write tests for things that I will throw away or is write for experimental usage. Take the code on my github account as an example. Most of the code here is not meant be used in production, it's have been written as poc's or to demonstrate things for my team, hence no need for tests. 

For code that should be used in production, decide in the team if something should be tested in a unit test, functional test or an integration test. Some very important function should perhaps be tested by all three types of tests, others only by an unit test. Let the context decide.

Remember to read up about how to write automatic in a way so that they are maintainable. A messy test code base is hard to work with and will lower the enthusiasm in the team. 

You will spend a lot of time writing tests, but on the other side you will spend a lot less time doing regression tests and hopefully avoid to do that total rewrite of the system.

## Conclusion

So this is why I love automatic tests and I hope that you do to.

Write those tests and prosper!

![Bug!](https://media.giphy.com/media/3o7TKUcreLvhQNwCFG/giphy.gif)

Happy coding!


[Conway's law]: https://en.wikipedia.org/wiki/Conway%27s_law
[whispering game]: https://en.wikipedia.org/wiki/Chinese_whispers
[outdated]: https://blog.usejournal.com/comparison-between-angular-1-vs-angular-2-vs-angular-4-62fe79c379e3
[BDD]: https://en.wikipedia.org/wiki/Behavior-driven_development
[cucumber]: https://cucumber.io/
[gherkin]: https://cucumber.io/docs/gherkin/
