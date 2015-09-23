---
layout: article
title: The Beginning of Scala Journey 2
modified: 2015-08-20T23:45:12+03:00
categories: articles
excerpt: Scala reactive notes, part 1. Implicit, yield and ScalaCheck quick start.
tags: [ScalaJourney, reactive, Scala, Java, Coursera, ScalaCheck, yield]
image:
  feature:
  teaser: beginning_teaser.png
  thumb:
comments: true
date: 2015-08-20T23:10:23+03:00
---

These are my notes for the [Principles of Reactive Programming] course. The first part of this article includes my back-story of learning [Scala]. The second --- some interesting ideas that I have learned from the first week of the course.

{% include toc.html %}

## The Beginning

Why 2 in the article title? And where was the first? Good questions.

It was a long time ago since I was developing something using [Scala]. This is a really amazing programming language which provides you flexibility to write complex programs for almost anything. I really love [Scala], for me it was like moving to the next level after [Java]. This reminds me of a [xkcd] comics, probably, the closest example that I can think about.

It is time for me to refresh my memories and learn something cool about that awesome language. But first, let me give you some background.

It was March of 2013. And my first course on [Coursera]: [Functional Programming Principles in Scala]. The same year I participated in [Principles of Reactive Programming] course. Worth noting that I got the highest score in both. The courses are really interesting, even if you do not know neither [Java] nor [Scala], you can apply in [Coursera] to learn some principles of functional and reactive programming. I began to develop more on [Scala], but I did not focused on it much. So eventually, I moved to another field --- computer vision. But that is a story for another time.

Several weeks ago I was thinking, why not to participate in those courses again? Actually, this is a good point, since new sessions of the courses provide new materials. Especially, there are new programming assignments.
And because of that, learning the courses the second time will not be boring. The bad thing is, I was a bit late for the second session of the [Principles of Reactive Programming] course. The good thing, I applied to the course some time ago, I just did not have time to do the course tasks. Besides, I have access to the course materials and programming assignments. I can even send the assignments via an automatic checking system (I have already tested that). Eventually, things are not bad after all. Yes, I will not receive a grade, but who cares? I just need to refresh my memories.

I have a plan now of how to keep all things that I have learned in one place --- blog. If you want to understand something, try to explain that to someone else. This is a modified version of a brilliant quote from [Dirk Gently's Holistic Detective Agency] by [Douglas Adams]: "If you really want to understand something, the best way is to try and explain it to someone else. That forces you to sort it out in your mind."

## ScalaCheck and yield

The one thing that you need to always keep in mind when you write on [Scala] is there are many *implicit* things. And by implicit I mean not just a language construction. You need to know language perfectly to produce not only an efficient code, but error free too.

The first week of the [Principles of Reactive Programming] course presented an awesome library --- [ScalaCheck]. Its author provides an exhaustive documentation on it: [ScalaCheck User Guide].

I want to highlight a part for generating test data in [ScalaCheck].

But first, I want to start from [yield]. From the documentation, we can see that this is just a syntactic sugar for `flatMap` and `map`. For example,

{% highlight scala %}
for(x <- c1; y <- c2) yield {...}
{% endhighlight %}

produces

{% highlight scala %}
c1.flatMap(x => c2.map(y => {...}))
{% endhighlight %}

Funny moment is that you can write a class as below:

{% highlight scala %}
class MapExample[T](val base: List[T]) {
  def map[V](f: T => V): MapExample[V] = new MapExample(base.map(f))
}
{% endhighlight %}

and call it using for-expression:

{% highlight scala %}
val me = new MapExample(List(1, 2, 3))
for {
  v <- me
} yield v + 1
{% endhighlight %}

even if it does not make much sense, because `for { v <- me }` construction means *for each `v` from `me`*. You can remove `base` from `MapExample` class, write `def map[V](f: T => V) = 1` and still successfully compile and run the code.

Lets go back to the [ScalaCheck]. I want to write a simple test case for it. It will check `base` equivalence after `yield` and `map`.

{% highlight scala %}
class MapExample[T](val base: List[T]) {
  def map[V](f: T => V): MapExample[V] = MapExample(base.map(f))

  override def toString: String = base.toString()
}

object MapExample {
  def apply[T](base: List[T]) = new MapExample[T](base)
}

object MapExampleSpecification extends Properties("MapExample") {
  type T = MapExample[Int]

  val generator: Gen[T] = for {
    l <- arbitrary[Int]
    v <- oneOf(const(MapExample(Nil)), generator)
  } yield MapExample(l :: v.base)

  implicit lazy val arbT: Arbitrary[T] = Arbitrary(generator)

  property("map example") = forAll { a: T =>
    a.map(v => v * 2).base == (for {v <- a} yield v * 2).base
  }
}
{% endhighlight %}

We need to provide a generator for `MapExample` and add an `Arbitrary`implicit value for this type. The generator above is based on `Int`. We can rewrite it using `List`:

{% highlight scala %}
val generator: Gen[T] = for {
  l <- arbitrary[List[Int]]
} yield MapExample(l)
{% endhighlight %}

This will also include empty lists.

If you check the source code of the `Gen` class, you will see that it implements a `map` function too. Good luck trying to understand of how the code works without knowing the result of the `yield` production.

Sometimes [Scala] seems like a very difficult programming language. However, there is no reason not to get big advantages:

* Obtaining the same performance as in Java (or even better in some cases).
* Writing 2-3 times less code compared to Java.

## Source code
The source code is available on [github](https://github.com/Foat/articles/tree/master/scalacheck-generators) under MIT License.

[Scala]: http://www.scala-lang.org
[Java]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
[xkcd]: https://xkcd.com/353/
[Coursera]: https://www.coursera.org/
[Functional Programming Principles in Scala]: https://www.coursera.org/course/progfun
[Principles of Reactive Programming]: https://www.coursera.org/course/reactive
[Dirk Gently's Holistic Detective Agency]: https://en.wikipedia.org/wiki/Dirk_Gently%27s_Holistic_Detective_Agency
[Douglas Adams]: https://en.wikipedia.org/wiki/Douglas_Adams
[ScalaCheck]: https://www.scalacheck.org
[yield]: http://docs.scala-lang.org/tutorials/FAQ/yield.html
[ScalaCheck User Guide]: https://github.com/rickynils/scalacheck/wiki/User-Guide
