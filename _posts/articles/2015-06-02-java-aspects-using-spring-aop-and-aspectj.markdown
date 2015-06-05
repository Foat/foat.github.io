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

[Aspect-oriented paradigm] is a really cool feature that you can use in your projects. With it you can easily apply additional functionality to your code without modifying the main body of a function.

{% include toc.html %}

Here, we will see how to configure a simple [Gradle] project using [Java] with such popular frameworks like [Spring AOP] and [AspectJ].

## Gradle configuration
We start from a gradle configuration file:

{% highlight groovy linenos=table %}
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

As you can see here, we use [dynamic versions] for gradle, e.g. `4.+`. This specifies the placeholder for the latest major version of `4`. In addition, we apply an `application` plugin, it is really useful if you do not use any IDE. You can just run in command line `gradle run` to start the application and that's it!

## Example method
As a starting point we will use [Spring Boot]. It is an excellent framework with the help of which you can quickly build a test [Spring Framework] application.

First, we create a simple method which takes number as a string and adds multiplication symbol to the end of the number.

{% highlight java linenos=table %}
@RequestMapping(value = "/multiply", method = RequestMethod.GET)
@AroundMethod
public String multiply(HttpServletRequest request, @RequestParam @ChangeParam String number) {
    log.info("Processing multiply method with a number parameter = {}", number);
    return number + " * ";
}
{% endhighlight %}

Probably, you have already mentioned `@AroundMethod` annotation. This annotation is used to add new functionality around our method. In addition, there is `@ChangeParam` annotation, we will talk about it a bit later.

## Aspects review
Here comes the `@Aspect` time!

{% highlight java linenos=table %}
@Aspect
public class ExampleAspect {
    @Around("execution(@me.foat.articles.aspects.annotations.AroundMethod * *.*(..)) && @annotation(change)")
    public Object process(ProceedingJoinPoint joinPoint, AroundMethod change) throws Throwable {}
}
{% endhighlight %}

This is a common aspect which is applied around a method, there are also many different types of aspect, such as
`@Before` and `@After` which are used before and after method respectively (what a surprise!). And do not forget to add `@Aspect` as a class annotation.

This aspect is applied only to those methods which have `@AroundMethod` annotation. Plus, we add this annotation as an argument, so we can address its fields.

We have finished the preparation part. Now lets see what can we do with all of that.

We can get method arguments. Great!
{% highlight java %}
Object[] args = joinPoint.getArgs();
{% endhighlight %}

Or even method information itself. Much better!
{% highlight java %}
Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
{% endhighlight %}

Or even more better - extract all annotations of method arguments. Excellent!
{% highlight java %}
Annotation[][] annotations = method.getParameterAnnotations();
{% endhighlight %}

### Java 8 features
Lets get an argument index. We use java 8, in that case let me show you some cool features that java provides in a functional way.

`annotations` is an array of arrays, because for each method argument we may have several annotations.
First, we create an index range:
{% highlight java %}
IntStream
    .range(0, annotations.length)
{% endhighlight %}

we need to filter it somehow to get an appropriate index. We apply a filter with a predicate for that:
{% highlight java linenos=table %}
.filter(i ->
        Arrays
        	// convert array to stream to be able to use java 8 features
                .stream(annotations[i])
                // check if any of argument annotations meet this condition
                .filter(a -> a instanceof ChangeParam)
                .findAny()
                .isPresent()) 
{% endhighlight %}

And as a final point we add `.findFirst();` to get the first argument index which meet the above conditions.

This is more natural style to write the code, write it more and you will get used to it. Just try!

We have got the index. We can change that argument and do whatever we want with it before the function starts. Oh, and how do we actually execute a function?

{% highlight java %}
Object result = joinPoint.proceed(args);
{% endhighlight %}

This runs the function and returns its result. Now, we can override it. If you want, you can do that by placing a new result as a return value.

## Example running
Lets run an example. [http://127.0.0.1/multiply?number=42](http://127.0.0.1/multiply?number=42) if you follow this url you will see the result `(42 * 100)`, where `42` was applied before the method (in the aspect), `*` added by the method, `* 100` after the method (in the aspect), where `100` is the value of `@ChangeParam` annotation.

## Internal method calls
Lest move on. Suppose we want to add another function and call it from `multiply` method.

{% highlight java %}
@AroundMethod(value = 200)
private String internalMethodAdd(@ChangeParam String number) { return number + " + "; }
{% endhighlight %}

We want the result to be `((42 + 200) * 100)` in that case. But what we see is `(42 + * 100)`. That is not correct! What could go wrong?

Actually, nothing. That is how [Spring AOP] works. It is uses proxies to call methods which use aspects. In that case if you call an internal method you call the exact method, not a proxy you need to. What can we do with that? We need native [AspectJ].

### AspectJ config
To include native [AspectJ] functionality uncomment this in `gradle.build` file:
{% highlight groovy linenos=table %}
configurations {
    aspectjweaver
}

aspectjweaver "org.aspectj:aspectjweaver:1.+"
runtime configurations.aspectjweaver.dependencies

applicationDefaultJvmArgs = [
        "-javaagent:${configurations.aspectjweaver.asPath}"
]
{% endhighlight %}

Here we get `aspectjweaver` library from gradle cache and add it as a `javaagent`. Now, try to open the link again.
`((42 + 200) * 100)`. Bingo! We did it!

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