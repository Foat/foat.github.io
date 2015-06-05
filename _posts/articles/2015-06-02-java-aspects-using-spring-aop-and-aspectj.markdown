---
layout: article
title: Java Aspects Using Spring AOP and AspectJ
modified:
categories: articles
excerpt:
tags: []
image:
  feature:
  teaser:
  thumb:
date: 2015-06-02T15:52:42+03:00
---

[Aspect-oriented paradigm] is a really cool feature that you can use in your projects. With it you can easily add additional functionality to your code without modifying the main body of a function.

Here, we will see how to configure a simple [Gradle] application using [Java] with such popular frameworks like [Spring AOP] and [AspectJ].

We start from a gradle configuration file

{% highlight groovy %}
group 'me.foat.articles.aspects'
version '1.0'

apply plugin: 'java'

apply plugin: 'application'
mainClassName = "me.foat.articles.aspects.Application"

sourceCompatibility = 1.8

repositories {
    mavenCentral()
}

dependencies {
    compile "org.springframework:spring-webmvc:4.+"
    compile "org.springframework:spring-aop:4.+"
    compile "org.springframework.boot:spring-boot-starter-web:1.+"
    compile "org.aspectj:aspectjrt:1.+"
    compile "org.slf4j:slf4j-api:1.+"
}
{% endhighlight %}

As you can see here, we use [dynamic versions] for gradle: ```4.+```. This specifies the placeholder for the latest major version of 4. In addition, we apply an ```application``` plugin, it is really useful if you do not use any IDE. You can just run in command line ```gradle run``` to start the application and that's it!

As a starting point we will use [Spring Boot]. It is an excellent framework with the help of which you can quickly build a test [Spring Framework] application.

[Aspect-oriented paradigm]: http://en.wikipedia.org/wiki/Aspect-oriented_programming
[Gradle]: https://gradle.org
[Java]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
[Spring AOP]: http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html
[AspectJ]: https://eclipse.org/aspectj/
[Spring Boot]: http://projects.spring.io/spring-boot/
[Spring MVC]: https://spring.io/guides/gs/serving-web-content/
[Spring Framework]: http://projects.spring.io/spring-framework/

[dynamic versions]: https://docs.gradle.org/1.8-rc-1/userguide/dependency_management.html#sub:dynamic_versions_and_changing_modules
[Application plugin]: https://docs.gradle.org/current/userguide/application_plugin.html