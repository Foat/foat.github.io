---
layout: article
title: Reactive Fun With Scala.js
modified: 2015-08-29T15:30:07+03:00
categories: articles
excerpt: Scala reactive notes, part 2. Demo application using reactive library and Scala.js.
tags: [ScalaJourney, reactive, Scala, Java, ScalaJS, ScalaRX]
image:
  feature:
  teaser: reactive_js_teaser.png
  thumb:
comments: true
date: 2015-08-29T14:33:07+03:00
---

I want to extend the whole idea behind my notes for the [Principles of Reactive Programming] course. I will include here some interesting materials that I have found over the last week. Likely, [Scala] is a really wide field to discover something new. The idea is to tie topics from the course to the articles and to form a better understanding of the whole Scala language.

In this article we will see how to use [Reactive Programming] paradigm in your [Scala] applications.

{% include toc.html %}

## Introduction to Reactive Programming

In the imperative programming when you have, for example, a variable `a = b + c`, then the changes of `b` or `c` will not affect the `a`. The core idea of [Reactive Programming] is simple --- when you change `b` or `c`, then the `a` automatically changes too, based on new values of `b` and `c`.

## Used libraries

### Scala.Rx

I have chosen [Scala.Rx] library to show the power of [Reactive Programming]. This is an awesome library which enables an ability to auto-propagate variables in a reactive way!

### Scala.js

What is [Scala.js] actually?

Let's take a look in the past. I was influenced by the [GWT] project a lot when I just started to learn [Java]. At first, I thought that [Scala.js] is something similar. And it is mostly used to create admin consoles for websites. But this is a really different topic.

Scala.js provides a way to compile your [Scala] code to [JavaScript]. There is not much sense to just compile Scala to JavaScript without Scala and Java APIs. And guess what? [Scala.js] includes almost everything from those APIs!

Furthermore, you can reuse the same code for client and server by using [Scala.js], this is a relief for many developers who needed to write a lot of similar code on front end and back end.

## Before we start

I base my example mainly on materials from a [Hands-on Scala.js] book. If you want to learn [Scala.js], you should definitely read this book. It provides comprehensive examples and covers most of the topics on this library.

## sbt configuration file

First thing we need to do is to configure our project. I will not spend much time on this part, since it is covered well in the official [tutorial](http://www.scala-js.org/doc/tutorial.html) of the [Scala.js] library. First of all, we add the below line to the `project/plugins.sbt` file:

{% highlight scala %}
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "0.6.4")
{% endhighlight %}

Then, enable the [Scala.js] plugin in the `build.sbt` file. Eventually, we add two dependencies: [scala-js-dom] and [Scala.Rx] libraries. The scala-js-dom library adds dom manipulation functionality to the Scala.js. Here is the complete `build.sbt` file:

{% highlight scala %}
enablePlugins(ScalaJSPlugin)

name := "scala-js-reactive-mouse"
version := "1.0"
scalaVersion := "2.11.7"

libraryDependencies ++= Seq(
  "org.scala-js" %%% "scalajs-dom" % "0.8.0",
  "com.lihaoyi" %%% "scalarx" % "0.2.8"
)
{% endhighlight %}

Probably, you have notices that we use `%%%` for dependencies. This allows compiler to use proper version of a library for each platform: Scala.js and Scala-JVM.

## Scala meets JavaScript

In the current application we will build a simple page where we get a mouse position and draw a rectangle on a canvas under the cursor.

Here is a [Scala] source file for that:
{% highlight scala %}
@JSExport
object MousePosition {
  @JSExport
  def start(div: html.Div, canvas: html.Canvas): Unit = {
    val h = dom.window.innerHeight
    val w = dom.window.innerWidth
    canvas.height = h
    canvas.width = w

    // default rectangle size
    val s = 10
    val half = Point(s, s) / 2

    val ctx = canvas.getContext("2d")
      .asInstanceOf[dom.CanvasRenderingContext2D]

    val mousePos = Var(Point(0, 0))
    val center = Rx { mousePos() - half }

    Obs(center, skipInitial = true) {
      ctx.fillStyle = "gray"
      ctx.fillRect(center().x, center().y, s, s)
    }

    def clear = {
      ctx.fillStyle = "white"
      ctx.fillRect(0, 0, w, h)
    }

    div.appendChild(mousePos)
    div.appendChild(center)

    dom.window.onkeypress = { e: dom.KeyboardEvent => if (e.keyCode == 27) clear }
    dom.window.onmousemove = { e: dom.MouseEvent => mousePos() = Point(e.pageX.toInt, e.pageY.toInt) }
  }
}
{% endhighlight %}

Looks pretty much like [JavaScript]! Except this is [Scala]. The source can be divided into three main parts:

1. Initial entities.
1. Reactive variables.
1. Event handing.

The first line is the `@JSExport` annotation. It indicates that the entity (object, method etc.) will be exported to the JS file.

Next, check the function definition:
{% highlight scala %}
def start(div: html.Div, canvas: html.Canvas): Unit
{% endhighlight %}
Static typing! In a big [JavaScript] project it is a real problem not to have this.

Then, there is a simple block where we set `canvas` size and get its context to use it later. It should be familiar to you, if you know [JavaScript].

### Reactive way

We are going reactive!

To enable [Scala.Rx] features do not forget to add `import rx._` line in a [Scala] source file.

The first thing we define is a mouse position. The `Point` class is wrapped in a class named `Var`. When the `Var` value is updated, it forces to recalculate all connected entities such as `Rx`.

{% highlight scala %}
val mousePos = Var(Point(0, 0))
val center = Rx { mousePos() - half }
{% endhighlight %}

You can get the current value of both `Var` and `Rx` by calling `mousePos()` or `center()`. This is equivalent to `entity.apply()` method call.

Next, the `Obs` class observes `Rx` or `Var` entities and does some post-processing (side-effects) when the observed value is changed. `skipInitial` means that the body of `Obs` will not run after the declaration and it will be evaluated only after a change of an observed value.

{% highlight scala %}
Obs(center, skipInitial = true) {
  ctx.fillStyle = "gray"
  ctx.fillRect(center().x, center().y, s, s)
}
{% endhighlight %}

Next, we append our created reactive entities to the provided `<div>` element. So the content of that `<div>` changes when the mouse position is changed.

{% highlight scala %}
div.appendChild(mousePos)
div.appendChild(center)
{% endhighlight %}

Finally, we define two events. First, for the `ESC` button to clear the screen. Second, to set the mouse position.

See how mouse position update is declared?

{% highlight scala %}
mousePos() = Point(e.pageX.toInt, e.pageY.toInt)
{% endhighlight %}

This can be rewritten as

{% highlight scala %}
mousePos.update(Point(e.pageX.toInt, e.pageY.toInt))
{% endhighlight %}

Yes, this is another syntactic sugar in [Scala].

## Wait.

Wait. WAIT! If you look closer at the `appendChild` function, then you will see that it needs a `Node` type there, not `Rx`! This is actually the magic of [Scala], it is called `implicit`. I have already mentioned that a bit in the [previous article](/articles/the-beginning-of-scala-journey-2/#scalacheck-and-yield).

Let's add an additional code for the implicit conversion from `Rx` to `Node`. I have modified the original conversion from [Hands-on Scala.js], to be applicable for [scala-js-dom] library:

{% highlight scala %}
import scala.language.implicitConversions

implicit def rxNode[T](r: Rx[T]): Node = {
  def rSafe: dom.Node = {
    val node = dom.document.createElement("div")
    node.textContent = r().toString
    node
  }
  var last = rSafe
  Obs(r, skipInitial = true) {
    val newLast = rSafe
    js.Dynamic.global.last = last
    last.parentNode.replaceChild(newLast, last)
    last = newLast
  }
  last
}
{% endhighlight %}

The main idea is that when the observer finds out that the value of `Rx` entity is changed, it replaces the previous `<div>` element with a new one.

## HTML file

Here is a part of an HTML file, which is used for production:
{% highlight html %}
<div id="position">
    Mouse position:
</div>
<div>
    <canvas id="canvas"></canvas>
</div>

<script type="text/javascript" src="../scala-js-reactive-mouse-opt.js"></script>
<script>
    article.MousePosition().start(document.getElementById('position'), document.getElementById('canvas'));
</script>
{% endhighlight %}
First, we place two `<div>` elements for displaying the position of a mouse cursor and a canvas, where rectangles are drawn.

Next, we add generated JavaScript file here, its name is based on a project name and a suffix `opt` (means optimized), there is a version which have `fastopt` suffix, it is used for developing and contains additional information for debugging.

The project can be built with `sbt fastOptJS` and `sbt fullOptJS` commands for develop and release versions respectively. By default, the output HTML file is placed in a `<project path>/target/scala-<scala version>/classes/` folder.

We run our `start` function by calling `article.MousePosition().start` method and providing necessary parameters.

## Live demo and source code

We have successfully completed the example. There are several things that I did not mention here, since they are not so important. You can see the complete source code below.

I really like both [Scala.js] and [Scala.Rx] libraries. They can help you to write a better code in their own way. Scala.js --- by providing static typing and the power of [Scala] libraries. Scala.Rx --- by implementing a new programming paradigm which can help you to write a cleaner code.

Now we can see the result. Yay!

* [Live demo](/examples/scala-js-reactive-mouse/)
* The source code is available on [github](https://github.com/Foat/articles/tree/master/scala-js-reactive-mouse) under MIT License.

[Scala]: http://www.scala-lang.org
[Java]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Principles of Reactive Programming]: https://www.coursera.org/course/reactive
[Reactive Programming]: https://en.wikipedia.org/wiki/Reactive_programming
[Scala.Rx]: https://github.com/lihaoyi/scala.rx
[Scala.js]: http://www.scala-js.org
[Hands-on Scala.js]: http://lihaoyi.github.io/hands-on-scala-js/
[GWT]: http://www.gwtproject.org
[JavaScript]: https://developer.mozilla.org/en-US/docs/Web/JavaScript
[scala-js-dom]: http://scala-js.github.io/scala-js-dom/
