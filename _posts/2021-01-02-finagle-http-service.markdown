---
layout: post
date:   2021-01-03 09:09:00 +0200
categories: Scala Finagle Http Serviec
title: Learning about Finagle HTTP
subtitle: Implementing a todo app with a Finagle HTTP Service
background: '/img/finagle_http.JPG'  
---

## About the post

In this post I will try to use vanilla Finagle to implement the classic a Todo rest service. 

When I was writing this I discovered that one often use [Finch] or [Airframe] (which are addons to Finagle) when createing a Rest api, however I will try to only use Finagle.

I will look at Finch in a future post.

This is not an expert post about Finagle and it's echo system, it's my attempt to get an understanding what Finagle is all about.

If you find this interesting I have written another [post] where I investigate the Finagle client.

## Creating a Hello world Finagle HTTP service 

Ok, to get started, lets implement a small Hello world service:

{% highlight scala %}
import com.twitter.finagle.{Http, Service, http}
import com.twitter.util.{Await, Future}

object SimpleHttpService extends App {

  val service = new Service[http.Request, http.Response] {
    def apply(req: http.Request): Future[http.Response] = {
      println(s"Received request: $req")
      Future.value{
        val response = http.Response(req.version, http.Status.Ok)
        response.contentString = "Hello world!"
        response
      }
    }
  }

  val server = Http.serve(":8080", service)
  Await.ready(server)
}
{% endhighlight %} 

This is fairly straightforward, we define that the server should listen on port 8080 and we also create a service that handles the incomming requests.

The most important thing to notice is that the result from the service is a `Future[Response]`. This is nice since we can write in a non blocking fashion.

![Nice!](https://media.giphy.com/media/Od0QRnzwRBYmDU3eEO/giphy.gif)

After the  server is starting and I go to `localhost:8080` in my browser I get the pretty unsurprising result:

![Hello world]({{ site.url }}/assets/finagle_http/hello_world.png){:height="80%" width="80%"} 

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

## Extracting the request body

To be able to create a Todo I need to extract the information from the request body. You can get request body as a String from the `http.request`.

I feel that working with a String is pretty low level, I would prefer to get the request body as an object instead:

{% highlight scala %}
case class Todo(id: Option[Long], title: String, completed: Boolean)
{% endhighlight %} 

I could not find any built in support in Finagle for transforming the json payload to Scala object so I decided to use the `bijection-json` library.  

The library give you the ability to define how you transform the payload to an object (`Todo` in this case) and back again (i.e. transform a `Todo` to a String).

{% highlight scala %}
implicit val todoBuilder: Injection[Todo, Map[String, JsonNode]] =
    Injection.build[Todo, Map[String, JsonNode]](todoToMap)(mapToTodo)

private def mapToTodo(m: Map[String, JsonNode]): Try[Todo] =
    Try(Todo(
      m.get("id").map(id => fromJsonNode[Long](id).get),
      fromJsonNode[String](m("title")).get,
      fromJsonNode[Boolean](m("completed")).get
    ))

private def todoToMap(t: Todo): Map[String, JsonNode] =
    if (t.id.isDefined) {
      Map("id" -> toJsonNode(t.id.get), "title" -> toJsonNode(t.title), "completed" -> toJsonNode(t.completed))
    } else {
      Map("title" -> toJsonNode(t.title), "completed" -> toJsonNode(t.completed))
    }

implicit val mapToString: Injection[Map[String, JsonNode], String] =
    JsonInjection.toString[Map[String, JsonNode]]

implicit val todoToJsonString: Injection[Todo, String] =
    Injection.connect[Todo, Map[String, JsonNode], String](todoBuilder, mapToString)
{% endhighlight %}

To be able to handle a `Bad request` the payload is transformed to a `Try[Todo]`.

I also wrote a small convenience function to make things easier:

{% highlight scala %}
def withBody[A](req: Request)(handler: A => ScalaFuture[Try[Body]])(implicit ev: Conversion[String, Try[A]]) = {
    req.contentString.as[Try[A]] match {
      case Success(a) =>
        handler(a)
      case Failure(e) =>
        ScalaFuture(Failure(new IllegalArgumentException(e)))
    }
}
{% endhighlight %}

We are using the `bijection` library here: `req.contentString.as[Try[A]]`. The result from the call will either be a valid object of type `A` or a failure.

## Create and get a Todo

Below is a code snippet from the service that handles the API methods for listing all Todos and to create a new Todo:

{% highlight scala %}
case Root / "todo" =>
    req.method match {
        case Method.Get =>
          context.todoActor
            .ask((ref: ActorRef[GetTodosReply]) => ListTodos(ref))
            .map(asBody(_))
        case Method.Post =>
          withBody[Todo](req) { todo =>
            context.todoActor
                .ask((ref: ActorRef[CreateTodoReply]) => CreateTodo(ref, todo))
                .map(asBody(_))
          }
        case _ =>
          ScalaFuture(Failure(new NoSuchMethodError(s"Unknown resource: ${req.path}")))
    }
{% endhighlight %}    

First we match on the path and on the method to resolve which AP method that is called. 

The state for the Todo app is inside an `Actor` so to handle the requests we send messages to the Actor. Finally we us `asBody` another convenience function that get us the response body as a String.

{% highlight scala %}
def asBody[A](ta: Reply[A])(implicit ev: Conversion[A, String]): Try[Body] =
    ta.payload.map(t => t.as[Body])
{% endhighlight %}

## The service

So here is the whole service:

{% highlight scala %}
def service(context: SystemContext): Service[Request, Response] = (req: Request) => (Path(req.path) match {
    case Root / "todo" =>
      req.method match {
        case Method.Get =>
          context.todoActor
            .ask((ref: ActorRef[GetTodosReply]) => ListTodos(ref))
            .map(asBody(_))
        case Method.Post =>
          withBody[Todo](req) { todo =>
            context.todoActor
                .ask((ref: ActorRef[CreateTodoReply]) => CreateTodo(ref, todo))
                .map(asBody(_))
          }
        case _ =>
          ScalaFuture(Failure(new NoSuchMethodError(s"Unknown resource: ${req.path}")))
      }
    case Root / "todo" / Long(id) =>
      req.method match {
        case Method.Get =>
          context.todoActor
            .ask((ref: ActorRef[GetTodoReply]) => GetTodo(ref, id))
            .map(asBody(_))
        case Method.Put =>
          withBody[Todo](req) { todo =>
            context.todoActor
                .ask((ref: ActorRef[ModifyTodoReply]) => ModifyTodo(ref, id, todo))
                .map(asBody(_))
          }
        case Method.Delete =>
          context.todoActor
            .ask((ref: ActorRef[DeleteTodoReply]) => DeleteTodo(ref, id))
            .map(asBody(_))
        case _ =>
          ScalaFuture(Failure(new NoSuchMethodError(s"Unknown resource: ${req.path}")))
      }
    case _ =>
      ScalaFuture(Failure(new NoSuchMethodError(s"Unknown resource: ${req.path}")))
}).map(toResponse).as[Future[http.Response]]
{% endhighlight %}

For me it's kind of annoying that I had to include the error handling for when calling unknown resources in the service (on three different places):

{% highlight scala %}
case _ =>
    ScalaFuture(Failure(new NoSuchMethodError(s"Unknown resource: ${req.path}")))
{% endhighlight %}

In fact I briefly considered to implement yet another convenience function so that it's possible to compose the service to hide this, but then I have almost implemented my own small REST wrapper for the Finagle http service and that's not what I'm after.

Lets investigate what help we can get from `Finch` in another post instead!

## Error handling

The different errors are handled in the `toResponse` function:

{% highlight scala %}
def toResponse(body: Try[Body]): http.Response = body match {
    case Success(b) =>
      val response = http.Response(Status.Ok)
      response.setContentTypeJson()
      response.contentString = b
      response
    case Failure(e: NoSuchMethodError) =>
      http.Response(Status.NotFound)
    case Failure(e: IllegalArgumentException) =>
      http.Response(Status.BadRequest)
    case Failure(e: NoSuchElementException) =>
      http.Response(Status.NotFound)
    case    _ =>
      http.Response(Status.InternalServerError)
}

{% endhighlight %}

## The Actor

For completeness, here is the implementation of the `TodoActor`: 

{% highlight scala %}
def behave(currentId: Long, todos: Map[Long, Todo]): Behavior[Request] = Behaviors.receive { (context, message) =>
    message match {
      case ListTodos(replyTo) =>
        replyTo ! GetTodosReply(Try(todos.values.toList.sortBy(t => t.id.get)))
        Behaviors.same
      case CreateTodo(replyTo, todo) =>
        val newTodo = todo.copy(id = Some(currentId))
        replyTo ! CreateTodoReply(Success(newTodo))
        behave(currentId + 1, todos + (currentId -> newTodo))
      case GetTodo(replyTo, id) =>
        val todo = Try(todos(id))
        replyTo ! GetTodoReply(todo)
        Behaviors.same
      case ModifyTodo(replyTo, id, todo) =>
        val newTodo = Try(todos(id)).map(_.copy(title = todo.title, completed = todo.completed))
        println(newTodo)
        replyTo ! ModifyTodoReply(newTodo)
        newTodo match {
          case Success(t) =>
            behave(currentId, todos + (id -> t))
          case _ =>
            Behaviors.same
        }
      case DeleteTodo(replyTo, id) =>
        val todo = Try(todos(id))
        replyTo ! DeleteTodoReply(todo)
        todo match {
          case Success(_) =>
            behave(currentId, todos - id)
          case _ =>
            Behaviors.same
        }
      case _ => ???
    }
}
{% endhighlight %}

Since this is not a post about Actor I will not dwell on the code, I will only point out that I choose to always wrap the payloads in the Replies in a `Try` to get an uniform API. As an example consider `ListTodos`, this will not fail, however I still wrapp it in a `Try`. This to simplify the logic in the service.   

![Uniform!](https://media.giphy.com/media/1fkd2Pjgt2YHvt72Rz/giphy.gif)


## Conclusion

When implementing the service I could not find that many good examples of how to develop a Finagle http service. I also found that I had to write more boiler plate code that I would have expected. It could be that I have missed something.

I got the impression that most people use Finch or Airframe. In the [next post] I will investigate what kind of help I get when introducing `Finch` or `Airframe`.

All code examples are available at [github].

Happy coding!

[Finch]: https://github.com/finagle/finch
[Airframe]: https://wvlet.org/airframe/docs/airframe-http.html
[post]: https://morotsman.github.io/scala/finagle/client/2020/12/30/finagle-client.html
[github]: https://github.com/morotsman/investigate_finagle/tree/main/src/main/scala/com/github/morotsman/investigate_finagle_service
[next post]: https://morotsman.github.io/scala/finagle/airframe/2021/01/03/finagle-airframe.html
