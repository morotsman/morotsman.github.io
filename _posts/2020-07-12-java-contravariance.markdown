---
layout: post
title:  "Contravariance and the Liskov Substitution Principle"
date:   2020-07-17 09:40:00 +0200
categories: Java Contravariance The Liskov Substitution Principle
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

This is a work in progress! 

## The setup

## The Problem

## Covariance to the rescue?

## Contravariance to the rescue

## Another example

## The Liskov Substitution Principle revisited

The Liskov Substitution Principle [LSP] states that: 

> Substitutability is a principle in object-oriented programming stating that, in a computer program, if S is a subtype of T, then objects of type T may be replaced with objects of type S (i.e. an object of type T may be substituted with any object of a subtype S) without altering any of the desirable properties of the program (correctness, task performed, etc.)

This boils down to some requirements on the type signature:

> * Contravariance of method arguments in the subtype.
> * Covariance of return types in the subtype.
> * No new exceptions should be thrown by methods of the subtype, except where those exceptions are themselves subtypes of exceptions thrown by the methods of the supertype. 

So why should the return types be covariant? Because we get a more flexible API (no copy/paste) and it is safe for the client to use.
    
## A final example



[covariance]: https://morotsman.github.io/java,/covariance,/the/liskov/substitution/principle/2020/07/12/java-covariance.html
[LSP]: https://en.wikipedia.org/wiki/Liskov_substitution_principle
[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
[generic types]: https://docs.oracle.com/javase/tutorial/java/generics/types.html
[github]: https://github.com/morotsman/about_scala/tree/master/src/main/scala/generics/java/covariance
[exceptions]: https://morotsman.github.io/the/liskov/substitution/principle/2020/07/05/breaking-the-law.html
