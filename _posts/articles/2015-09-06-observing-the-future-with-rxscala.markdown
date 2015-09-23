---
layout: article
title: Observing the Future With RxScala
modified:
categories: articles
excerpt: Scala reactive notes, part 3. Reactive paradigm which leads to a better code.
tags: [ScalaJourney, reactive, Scala, Java, RxScala, RxJava]
image:
  feature:
  teaser: observing_futures_teaser.png
  thumb:
comments: true
date: 2015-09-07T21:11:05+03:00
---

In the [previous](/articles/reactive-fun-with-scala-js/) article, I have talked about [Scala.Rx] library. It is clear that this library is useful for many applications and it can replace such libraries like [AngularJS] in the [Scala] world (see [Scala.js]). Here, I want to outline another reactive library. Its platform, [ReactiveX], is very popular now in the Java world.

We will talk about [RxScala] library. Basically, it is a wrapper for [RxJava]. It provides a new abstraction layer to write your code and solve problems like callback hell.

{% include toc.html %}

## Quick example

The first look at the [RxScala] API:

{% highlight scala %}
val intervals: Observable[Long] = Observable.interval(100 millis).take(10)

intervals subscribe {
  v => println(s"value = $v")
}
Await.result(Future { Thread.sleep(1500) }, 2 seconds)
{% endhighlight %}

We create an `Observable` which emits values starting from `0` each 100 milliseconds. Then, we take only first ten emitted elements. Next, we subscribe a method to the elements the `Observable` emits.

Run the code and see the output:

    value = 0
    value = 1
    value = 2
    ...

What if we remove the `Await` from the code? In that case you will not see anything, since the main thread will stop its execution first, while the `intervals` computation will be performed in a different thread.

## When it is too simple

There are many simple examples that can be created with that library. For example, using `just` function:

{% highlight scala %}
Observable.just(5, 4, 2).subscribe(print(_))
{% endhighlight %}

Really? I know that this is a different paradigm for developing things, but I do not think that this is a good example for showing the good sides of the library, when you can simply write this using `foreach` function:

{% highlight scala %}
List(5, 4, 2).foreach(print(_))
{% endhighlight %}

Because of that some examples in the web are not that catchy and do not show interesting and useful ways of how to use that library.

## Asynchronous

One of the problems that reactive libraries tend to solve is to provide a better way for asynchronous calls. Let's see what [RxScala] can do with asynchronous computation. We will emulate it with futures. Think of it like calling from a client to a server.

{% highlight scala %}
val asyncEmulation = Observable
    .just(1, 2, 3)
    .map(e => e + 1)
    .flatMap(e => Observable.from(Future { Thread.sleep(400 - 100 * e); e }))

val cd = new CountDownLatch(3)
asyncEmulation subscribe {
  v => println(s"received = $v"); cd.countDown()
}
cd.await()
{% endhighlight %}

As you can see, we are not creating a callback function here. Instead, we use such functions like `map`, `flatMap` and [others](http://reactivex.io/rxscala/scaladoc/#rx.lang.scala.Observable). This prevents us from using a callback in a callback in a ... Simply, callback hell. In addition to reactive extensions, there are many techniques which can solve that problem, starting from naming functions to using `async`/`await` combination, e.g. [asyncawait].

Next, in the code above, you can see a different approach of waiting for the end of a computation --- [CountDownLatch](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html). This class is designed for multithreading, you just set the initial count of the latch, and when it reaches zero by calling `countDown` method, all waiting threads are released. Besides, you can set the maximum time for `await` function, this will be useful, if you do not want the main thread to wait too long.

Finally, [RxScala] is an excellent tool which you can use in your applications to get much cleaner and understandable code. In some cases, when you do not need to deal with a lot of callbacks or do any heavy computation, you may prefer to use pure callback related approaches. But when it comes to do something more difficult, then the [ReactiveX] platform is the thing that will improve your code.

## Source code
The source code is available on [github](https://github.com/Foat/articles/tree/master/observing-futures) under MIT License.

[ReactiveX]: http://reactivex.io
[RxJava]: https://github.com/ReactiveX/RxJava
[RxScala]: https://github.com/ReactiveX/RxScala
[Scala]: http://www.scala-lang.org
[Scala.Rx]: https://github.com/lihaoyi/scala.rx
[Scala.js]: http://www.scala-js.org
[asyncawait]: https://github.com/yortus/asyncawait
[AngularJS]: https://angularjs.org
