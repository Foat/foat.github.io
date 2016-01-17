---
layout: article
title: Web Crawling With Akka
modified:
categories: articles
excerpt: Scala and Akka web crawler. Supports non-blocking features and introduces many interesting Reactive techniques.
tags: [ScalaJourney, Reactive, Scala, Akka, Web, Crawler]
image:
  feature:
  teaser: akka-crawler.png
  thumb:
comments: true
date: 2016-01-17T12:53:53+03:00
---

In the [previous](/articles/akka-example/) article we have seen an [Akka] introduction. Now it is time to move to something more interesting. There are many applications, where Akka shines. Today we will take a look at a web crawler example. It contains several interesting parts where Akka shows that multithreading application can be really easy to design and develop.

{% include toc.html %}

## Project design

We want to start from the design, it will provide us a quick look of how the system should look like. Where can we get some good examples for this task?

A web crawler is usually a part of a web search engine. And the most popular search engine currently is Google. You can take a look at the [The Anatomy of a Large-Scale Hypertextual Web Search Engine] article, it provides a brief overview of how Google was designed at first. So, if you want to build a similar project, you can surely start from this article.

### Assumptions

The application we want to build is much smaller. To make it more interesting and resilient we will base our web crawler on several conditions:

* Web crawler must not spam websites, so each request to a single website should be delayed.
* All operations must be non-blocking.
* We assume that the web connection is not good, so the crawler might fail sometimes. In such a case the system must recover in the face of failure.
* Since we will be running the crawler on a regular computer, then we need to stop the system when we scrap enough links. The check itself will be regulated by a simple threshold value.

### Actors design

Our system will be divided into Actors. Each Actor represents a logical block that operates with a different data. The first thing that we need is a starting point, this can be just a simple object class, which creates a main Actor and starts the messaging. The main Actor itself will regulate some high order operations like how many links we still need to scrap or what to do in case of a scraping failure. In addition, it needs to manage other Actors. I think, the name that suits this Actor well is `Supervisor`.

It will be much easier, if we create a separate Actor for each website we need to crawl. In that case we do not need to worry of how to synchronize delays with other websites scrapers. Since this Actor operates with a single website, we will call it `SiteCrawler`. The `Supervisor` Actor will send here the information to be scraped.

The `SiteCrawler` needs to get all needed information from a web page. The problem is that the scrapping might fail, as we assumed previously. To separate the scraping functions we create a `Scraper` Actor. It will process the provided url and send the status back to the `SiteCrawler`. If the scraping fails, `SiteCrawler` will send an error message to the `Supervisor` after the timeout.

The scraped information will be sent from the `Scraper` directly to the `Indexer` Actor, which will store the content information. Once it receives the information it will not only store the necessary data, but also send urls that needs to be scraped to the `Supervisor`. To make the `Indexer` a bit more interesting, we will print all collected data before stopping it.

### Project diagram

You can assume that each Actor can represent a single machine, so to make the communication process better, we try to minimize the information that is sent from one Actor to another.

The diagram of the project classes is represented as follows:

<figure>
  <img src="/images/crawler.svg">
  <figcaption>Project diagram. Shows main parts and processes of the system.</figcaption>
</figure>

## Classes in details

We finished our preparations. Now, we want to implement the system using [Scala] and [Akka]. Let's start that from the same order as we did in the previous section.

### StartingPoint

The `App` class of the project should do several things, such as:

* creating the main Actor and initializing it with a first website we want to process,
* stopping the whole system if the process was not ended yet.

Here is the object implementation:
{% highlight scala %}
import akka.actor.{ActorSystem, PoisonPill, Props}

import scala.concurrent.Await
import scala.concurrent.duration._
import scala.language.postfixOps

object StartingPoint extends App {
  val system = ActorSystem()
  val supervisor = system.actorOf(Props(new Supervisor(system)))

  supervisor ! Start("https://foat.me")

  Await.result(system.whenTerminated, 10 minutes)

  supervisor ! PoisonPill
  system.terminate
}
{% endhighlight %}

The thing I want to highlight is that the stopping after the delay demostrated here is a last resort. It is much better to stop the system when the program has accomplished its goal.

### Supervisor

The Supervisor has four basic variables:

* How many pages we visited (sent for scraping).
* Which pages we still need to scrap.
* How many times we tried to visit a particular page. This is needed, since we think that scraping might fail and in that case we need to visit a page several times.
* Store for `SiteCrawler` Actors for each host we need to deal with.

Here is one way you can represent those variables:
{% highlight scala %}
var numVisited = 0
var toScrap = Set.empty[URL]
var scrapCounts = Map.empty[URL, Int]
var host2Actor = Map.empty[String, ActorRef]
{% endhighlight %}

When we scrap a url, we need to send it to an Actor which processes urls for one particular host.

{% highlight scala %}
def scrap(url: URL) = {
  val host = url.getHost
  println(s"host = $host")
  if (!host.isEmpty) {
    val actor = host2Actor.getOrElse(host, {
      val buff = system.actorOf(Props(new SiteCrawler(self, indexer)))
      host2Actor += (host -> buff)
      buff
    })

    numVisited += 1
    toScrap += url
    countVisits(url)
    actor ! Scrap(url)
  }
}
{% endhighlight %}

Here, we check if an Actor for the provided host is presented, if not, we create it and add to our `host2Actor` container. Next, we add the url to the collection of pages we want to scrap. After all, we notify the `SiteCrawler` Actor that we want to scrap the url.

The `receive` function body of the `Supervisor` class contains a handler for each received message. Each message is a case class. To see all messages, check the [`Messages`] class in the project repository.

{% highlight scala %}
def receive: Receive = {
  case Start(url) =>
    println(s"starting $url")
    scrap(url)
  case ScrapFinished(url) =>
    println(s"scraping finished $url")
  case IndexFinished(url, urls) =>
    if (numVisited < maxPages)
      urls.toSet.filter(l => !scrapCounts.contains(l)).foreach(scrap)
    checkAndShutdown(url)
  case ScrapFailure(url, reason) =>
    val retries: Int = scrapCounts(url)
    println(s"scraping failed $url, $retries, reason = $reason")
    if (retries < maxRetries) {
      countVisits(url)
      host2Actor(url.getHost) ! Scrap(url)
    } else
      checkAndShutdown(url)
}
{% endhighlight %}

Let's take a closer look at `IndexFinished` and `ScrapFailure` handlers.

When the `Indexer` finishes its processing, it sends the url information to the `Supervisor` using `IndexFinished` message. We check if we want to scrap received urls or not based on the number of pages we already visited. If yes, we proceed with each url that we did not try to scrap before. The `checkAndShutdown` function removes url from `toScrap` set and, if there is no urls to visit anymore --- shutdowns the whole system.

The `ScrapFailure` message is received when the scraping fails. In that case we need to decide if we want to go on scraping a url or not, we do that by counting the number of visits for the url.

Following [this link](https://github.com/Foat/articles/blob/master/akka-web-crawler/src/main/scala/me/foat/crawler/Supervisor.scala) you can check the whole source code for the `Supervisor` class.

### SiteCrawler

One of the most interesting things in the system is how we handle the delays for each website in a non-blocking way. To achieve that we placed this process in a separate class. In addition to this, we need to recover after the scraping failure somehow.

Want to see how it works? See the `SiteCrawler` class implementation below.

{% highlight scala %}
class SiteCrawler(supervisor: ActorRef, indexer: ActorRef) extends Actor {
  val process = "Process next url"

  val scraper = context actorOf Props(new Scraper(indexer))
  implicit val timeout = Timeout(3 seconds)
  val tick =
    context.system.scheduler.schedule(0 millis, 1000 millis, self, process)
  var toProcess = List.empty[URL]

  def receive: Receive = {
    case Scrap(url) =>
      // wait some time, so we will not spam a website
      println(s"waiting... $url")
      toProcess = url :: toProcess
    case `process` =>
      toProcess match {
        case Nil =>
        case url :: list =>
          println(s"site scraping... $url")
          toProcess = list
          (scraper ? Scrap(url)).mapTo[ScrapFinished]
            .recoverWith { case e => Future {ScrapFailure(url, e)} }
            .pipeTo(supervisor)
      }
  }
}
{% endhighlight %}

First of all, we create an instance of `SiteCrawler` for each website (host) in the `Supervisor` Actor as we saw previously. In that case we need to deal with only one website and do not worry about synchronizations with the others.

We create a scheduler that sends a `process` message to the Actor each second. We cannot call the internal process directly, since there is no synchronization between it and the `SiteCrawler` Actor. Without the scheduler it might be possible that we modify `toProcess` variable in two places at the same time: when we add a new url to that variable (`Scrap` message) and when we remove the last added element from the list (`process` message).

When we need to get a response from an Actor, we can use the [ask pattern]. It uses a `?` symbol. The `timeout` variable implicitly defines how much time we wait before the `ask` fails automatically. And if it does, the process recovers with the `ScrapFailure` message. Finally, we send the status back to the `Supervisor`.

After all we get a non-blocking Actor, which processes a website without spamming it.

### Scraper

The `Scraper` Actor is simple. It does not contain any state (so we could use a simple `Future` instead of an `Actor`) and its purpose is to scrap the provided url and send the success flag to the sender and the scraped information to the `Indexer`. Here is the `receive` method of the Actor:

{% highlight scala %}
def receive: Receive = {
  case Scrap(url) =>
    println(s"scraping $url")
    val content = parse(url)
    sender() ! ScrapFinished(url)
    indexer ! Index(url, content)
}
{% endhighlight %}

The most interesting part of this Actor is a `parse` function:

{% highlight scala %}
def parse(url: URL): Content = {
  val link: String = url.toString
  val response = Jsoup.connect(link).ignoreContentType(true)
    .userAgent("Mozilla/5.0 (Windows NT 6.1; WOW64; rv:40.0) Gecko/20100101 Firefox/40.1").execute()

  val contentType: String = response.contentType
  if (contentType.startsWith("text/html")) {
    val doc = response.parse()
    val title: String = doc.getElementsByTag("title").asScala.map(e => e.text()).head
    val descriptionTag = doc.getElementsByTag("meta").asScala.filter(e => e.attr("name") == "description")
    val description = if (descriptionTag.isEmpty) "" else descriptionTag.map(e => e.attr("content")).head
    val links: List[URL] = doc.getElementsByTag("a").asScala.map(e => e.attr("href")).filter(s =>
      urlValidator.isValid(s)).map(link => new URL(link)).toList
    Content(title, description, links)
  } else {
    // e.g. if this is an image
    Content(link, contentType, List())
  }
}
{% endhighlight %}

We used a popular library for web scraping --- [Jsoup] to get links and other information from a web page. Since it is a Java library, we needed to convert the received information to the Scala format. It is actually possible to do that with a `scala.collection.JavaConverters` object. We also do a simple check if the received url is a valid content, since we want to parse only html pages.

### Indexer

The two main goals of the `Indexer` Actor is to store the sent information and to send all scraped urls to the `Supervisor`. To make things more fun, we print all received data before the Actor stops working.

{% highlight scala %}
class Indexer(supervisor: ActorRef) extends Actor {
  var store = Map.empty[URL, Content]

  def receive: Receive = {
    case Index(url, content) =>
      println(s"saving page $url with $content")
      store += (url -> content)
      supervisor ! IndexFinished(url, content.urls)
  }

  @throws[Exception](classOf[Exception])
  override def postStop(): Unit = {
    super.postStop()
    store.foreach(println)
    println(store.size)
  }
}
{% endhighlight %}

## Summary

If you run a program, you will see a bunch of messages that it produces. They just show the stage of a processing for a url.

We have built a simple web crawler that can successfully crawl several websites. Moreover, we achieved that using [Akka], we did not use standard multithreading techniques, just messages and Actor instances.

### Source code
The source code is available on [github](https://github.com/Foat/articles/tree/master/akka-web-crawler) under MIT License.

[The Anatomy of a Large-Scale Hypertextual Web Search Engine]: http://ilpubs.stanford.edu:8090/361/1/1998-8.pdf
[Akka]: http://akka.io
[Scala]: http://www.scala-lang.org
[`Messages`]: https://github.com/Foat/articles/blob/master/akka-web-crawler/src/main/scala/me/foat/crawler/Messages.scala
[ask pattern]: http://doc.akka.io/docs/akka/current/scala/actors.html#Ask__Send-And-Receive-Future
[Jsoup]: http://jsoup.org


