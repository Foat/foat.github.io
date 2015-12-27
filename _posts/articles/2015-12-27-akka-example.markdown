---
layout: article
title: Simple Akka Example
modified:
categories: articles
excerpt: Akka Introduction. Simple Sender-Store Actor model with a PoisonPill usage.
tags: [ScalaJourney, reactive, Scala, Akka]
image:
  feature:
  teaser: akka-example.png
  thumb:
comments: true
date: 2015-12-27T15:45:04+03:00
---

[Typesafe] stack brings many interesting features in software development and design. One of those is [Akka] --- an open-source toolkit. It is a powerful library that takes multitreading to a whole new level. Mainly, it simplifies the building of concurrent applications. Here, we will create a simple Akka application to show its main concepts. We will see that it is really easy to build a non-blocking multitreading application using this library.

{% include toc.html %}

## Introduction to Actors

There are many resources which can introduce you to [Akka]. I just want to point to two of them. First of all, the [Akka documentation]. Next, an excellent coursera course --- [Principles of Reactive Programming]. Those two sources of information provide enough material to start building Akka applications.

[Akka] is based on actor-based concurrency model. The naive Actor definition can be represented as: Actor is a synchronous object which processes all received messages successively. The multithreading model come out from the concept that everything that is outside an Actor is asynchronous and processed in different threads. It is much better to use an Actor as an object with a state, since there is no concurrency within an Actor. All communication with an Actor must be done by sending messages to it.

## Actors Example

To use the full potential of [Akka] Actors, we will use Actors with a state. Each Actor in [Scala] must be extended from an `Actor` trait. It contains a method which needs to be defined:

{% highlight scala %}
def receive: Receive
{% endhighlight %}

The method must implement the partial function.

**Actor or Future?** If there is no state in an `Actor`, then it is much better to use a simple `Future` or something related which is focused on a stateless behavior.
{: .notice-inverse}

We will create two types of Actors which will create a communication layer --- `Sender` which will send all messages to the second Actor --- `Store`, the Actor that will store the received messages.

## Sender

Take a look at the `Sender` implementation.

{% highlight scala %}
import akka.actor.{Actor, ActorRef, PoisonPill}

class Sender(store: ActorRef, id: Int) extends Actor {
  var count = 0

  def receive: Receive = {
    case "start" =>
      println(s"$id sends $count to the store")
      store ! s"$id => $count"
      count += 1
      if (count < 5)
        self ! "start"
      else self ! PoisonPill
  }
}
{% endhighlight %}

It contains a `count` variable which stores the number of sent messages.

Actors in Scala use `!` symbol to send a message. As you can see here, the `Sender` sends a message with its `id` and `count` in a string to the `Store`. The Actors can send not only a string, but an object too.

We do not want to send messages forever, so we add a check for `count` variable here. If it is smaller than the threshold, we send a start message to the Actor itself to continue the cycle. Otherwise, we send a `PoisonPill` message. It is used to stop an Actor without breaking the message queue. So all messages which were received before the pill will be processed as usual.

## Store

{% highlight scala %}
import akka.actor.Actor

class Store extends Actor {
  var list = List.empty[String]

  def receive: Receive = {
    case msg: String =>
      list = msg :: list
  }

  @throws[Exception](classOf[Exception])
  override def postStop(): Unit = {
    super.postStop()
    println(list.reverse)
  }
}
{% endhighlight %}

The Actor is simple, when we receive a string message, we store it in a list. The interesting part here is a `postStop` function. It is processed after the Actor was stopped, e.g. after processing the `PoisonPill` message. In the method we just print all received messages in a correct order.

## Running the Program

To run the whole process we put Actor creation in an `App` object.

{% highlight scala %}
import akka.actor.{ActorSystem, PoisonPill, Props}

import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.language.postfixOps

object Main extends App {
  val system = ActorSystem()
  val store = system.actorOf(Props(new Store))

  val sender1 = system.actorOf(Props(new Sender(store, 1)))
  val sender2 = system.actorOf(Props(new Sender(store, 2)))
  val sender3 = system.actorOf(Props(new Sender(store, 3)))

  sender1 ! "start"
  sender2 ! "start"
  sender3 ! "start"

  system.scheduler.scheduleOnce(2 seconds)({store ! PoisonPill; system.terminate})
}
{% endhighlight %}

First, we create 4 Actors: 1 for storing and 3 for sending messages. Next, we send a `"start"` message to all `Sender` Actors.

Finally, to stop the `Store` and the whole system, we create a scheduler, which will be called once after 2 seconds.

Let's check the output:

    3 sends 0 to the store
    2 sends 0 to the store
    1 sends 0 to the store
    3 sends 1 to the store
    2 sends 1 to the store
    3 sends 2 to the store
    1 sends 1 to the store
    2 sends 2 to the store
    3 sends 3 to the store
    1 sends 2 to the store
    2 sends 3 to the store
    3 sends 4 to the store
    2 sends 4 to the store
    1 sends 3 to the store
    1 sends 4 to the store
    List(3 => 0, 2 => 0, 1 => 0, 3 => 1, 2 => 1, 3 => 2, 1 => 1, 2 => 2, 3 => 3, 1 => 2, 2 => 3, 3 => 4, 2 => 4, 1 => 3, 1 => 4)

It may differ on your computer, since all messages are send in different threads. As you can see, all processes are logged properly and finally `Store` prints the output.

## Summary
[Akka] is a really powerful tool to create multithreading projects. It gives the most benefits when you use stateful objects. In the next articles, we will be creating a simple web crawler and will be digging in Akka a bit more.

## Source code
The source code is available on [github](https://github.com/Foat/articles/tree/master/akka-example) under MIT License.

[Typesafe]: http://www.typesafe.com
[Akka]: http://akka.io
[Akka documentation]: http://akka.io/docs/
[Scala]: http://www.scala-lang.org
[Principles of Reactive Programming]: https://www.coursera.org/course/reactive
