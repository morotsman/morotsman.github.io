---
layout: post
date:   2021-01-03 09:09:00 +0200
categories: Scala Finagle Airframe
title: Learning about Airframe
subtitle: Implementing a todo app with Finagle and Airframe
background: '/img/airframe.jpg'  
--- 

## About the post

I previously wrote a [post] where I tried to figure out what Finagle HTTP services is all about, more precisely I tried to implement a small REST Api. When writing the post I discovered that most people are using [Airframe] or [Fitch] instead of vanilla Finagle when doing such a task.

This post is a continuation of the previous post where I will look at Airframe and try to use it to implement the same REST Api once again.  

This is not an expert post about Finagle and it's echo system, it's my attempt to get an understanding what Finagle is all about.
    
![Resilient]({{ site.url }}/img/itsairframe.jpg){:height="80%" width="80%"} 

## Airframe?

> airframe-http is a library for creating REST HTTP web servers at ease. airframe-http-finagle is an extension of airframe-http to use Finagle as a backend HTTP server.

This sounds good, in my previous [post] where I used vanilla Finagle HTTP to create a REST Api for a Todo app I had to write a lot of boilerplate code to handle stuff like error handling and transforming JSON to Scala objects, it would be really nice if this was handled by the framework.

## The todo app

The Todo app should support the following API:

{% highlight scala %}
GET todo/
POST todo/

GET todo/id
PUT todo/id
DELETE todo/id
{% endhighlight %}

It should be possible to send in a request body for the POST and PUT methods:

{% highlight javascript %}
{"id": 0, "title": "A todo", "completed": false}
{% endhighlight %}

The id is optional. 

## Create a Todo

Lets start by implementing an endpoint that makes it possible to create a Todo:

{% highlight scala %}
case class Todo(id: Option[Long], @required title: String, @required completed: Boolean)

@Endpoint(method = HttpMethod.POST, path = "/todo")
def createTodo(todo: Todo): Future[Todo] =
    context.todoActor
      .ask((ref: ActorRef[CreateTodoReply]) => CreateTodo(ref, todo))
      .map(r => r.payload.get)
      .as[Future[Todo]]
{% endhighlight %}

Airframe takes care of transforming our json payload to a `Todo`. Since I wanted the title and completed fields to be mandatory I had to mark them with a `@required` annotation, otherwise Airframe gives me default values which is not what I want (read about id [here]).  

The state of the app is inside an Actor, so the endpoint is pretty trivial, just ask for the Actor to create the Todo and then return the result (all in a non blocking fashion of course). 

![Obviously!](https://media.giphy.com/media/IbI9JesSiQ7ay5ZXLL/giphy.gif)

## All the endpoints

The rest of the endpoint are implemented in the same way:

{% highlight scala %}
class TodoApi(
               val context: SystemContext,
               implicit val ec: ExecutionContextExecutor,
               implicit val system: ActorSystem[Setup]) {

  private implicit val timeout: Timeout = 3.seconds

  @Endpoint(method = HttpMethod.GET, path = "/todo")
  def getTodos: Future[List[Todo]] =
    context.todoActor
      .ask((ref: ActorRef[GetTodosReply]) => ListTodos(ref))
      .map(r => r.payload.get)
      .as[Future[List[Todo]]]

  @Endpoint(method = HttpMethod.POST, path = "/todo")
  def createTodo(todo: Todo): Future[Todo] =
    context.todoActor
      .ask((ref: ActorRef[CreateTodoReply]) => CreateTodo(ref, todo))
      .map(r => r.payload.get)
      .as[Future[Todo]]

  @Endpoint(method = HttpMethod.PUT, path = "/todo/:id")
  def modifyTodo(id: Long, todo: Todo): Future[Todo] =
    context.todoActor
      .ask((ref: ActorRef[ModifyTodoReply]) => ModifyTodo(ref, id, todo))
      .map(r => r.payload.get)
      .as[Future[Todo]]

  @Endpoint(method = HttpMethod.DELETE, path = "/todo/:id")
  def deleteTodo(id: Long): Future[Todo] =
    context.todoActor
      .ask((ref: ActorRef[DeleteTodoReply]) => DeleteTodo(ref, id))
      .map(r => r.payload.get)
      .as[Future[Todo]]

  @Endpoint(method = HttpMethod.GET, path = "/todo/:id")
  def getTodo(id: Long): Future[Todo] =
    context.todoActor
      .ask((ref: ActorRef[GetTodoReply]) => GetTodo(ref, id))
      .map(r => r.payload.get)
      .as[Future[Todo]]

}
{% endhighlight %}


## Staring the application

And this is how you start the application:

{% highlight scala %}
object AirframeHttpService extends App {
  implicit val system: ActorSystem[Setup] =
    ActorSystem(ActorSystemInitializer.setup, "todo")

  implicit val timeout: Timeout = 3.seconds
  implicit val ec: ExecutionContextExecutor = system.executionContext

  val setupActorSystem = system.ask((ref: ActorRef[SystemContext]) => Setup(ref))
  val systemContext = scala.concurrent.Await.result(setupActorSystem, 3.seconds)

  val router = Router.add(ErrorHandler).andThen[TodoApi]

  val design = Finagle.server
    .withPort(8080)
    .withRouter(router)
    .design

  design
    .bind[SystemContext].toInstance(systemContext)
    .bind[ExecutionContextExecutor].toInstance(ec)
    .bind[ActorSystem[Setup]].toInstance(system)
    .build[FinagleServer] { server =>
      server.waitServerTermination
    }

}
{% endhighlight %}

The first part is all about setting up the Actor system. 

We then create a `router` where we say that the `TodoApi` will resolve the requests. We also configure the Finagle server with the router and the port to use. Last but not least we do some dependency injection which we will cover in the next section.

## Dependency Injection

Consider the following rows in the service setup:

{% highlight scala %}
    .bind[SystemContext].toInstance(systemContext)
    .bind[ExecutionContextExecutor].toInstance(ec)
    .bind[ActorSystem[Setup]].toInstance(system)
{% endhighlight %}

Here we bind concrete instances of the SystemContext, ExecutionContextExecutor and the ActorSystem to the types. These are later injected when the `TodoApi` class is instantiated.

{% highlight scala %}
class TodoApi(
               val context: SystemContext,
               implicit val ec: ExecutionContextExecutor,
               implicit val system: ActorSystem[Setup])
{% endhighlight %}

Here we get references to the context, an Executor and the ActorSystem. 

## Error handling

Most of the error handling is taken care of by Airframe (for example validation of the payload), but I added a Filter to take care if the cases when a clients tries to do things with a non existing Todo.

{% highlight scala %}
object ErrorHandler extends FinagleFilter {
  def apply(request: Request, context: Context): Future[Response] = {
    context(request).handle {
      case e: NoSuchElementException =>
        Response(Status.NotFound)
    }
  }
}
{% endhighlight %}

## The TodoActor

You can find the implementation at [github] together with the rest of the code.

## Conclusion

Airframe takes care of a lot of boilerplate code which is very nice. If I compare with my previous [post] where I used vanilla Finagle I could skip a lot of code, and the code that was left was clearer to read.

Also, if you come from the Spring Boot world you feel right at home with the Dependency injection and annotation thingies. This might be a good way to introduce Scala on a Java/Spring Boot centric company. 

All code is available at [github].

Happy coding!

[post]: https://morotsman.github.io/scala/finagle/http/serviec/2021/01/03/finagle-http-service.html
[Airframe]: https://wvlet.org/airframe/docs/airframe-http.html
[Fitch]: https://github.com/finagle/finch
[here]: https://wvlet.org/airframe/docs/airframe-codec
[github]: https://github.com/morotsman/investigate_finagle/tree/main/src/main/scala/com/github/morotsman/investigate_finagle_service/todo_airframe

