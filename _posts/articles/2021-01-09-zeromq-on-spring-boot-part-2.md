---
layout: post
subheadline: Spring Boot Series
title: ZeroMQ with Spring Boot
teaser: Get more done with less setup using ZeroMQ with Spring Boot.
categories:
  - articles
tags:
  - spring boot
  - messaging
  - development
related:
  - /blog/articles/stateless-sessions-with-spring-boot.html
  - /blog/articles/zeromq-on-spring-boot.html
image:
    title: stock/adobe/lone-window.jpg
    thumb: stock/adobe/lone-window-t.jpg
    homepage: stock/adobe/lone-window.jpg
---

## Introduction

In the [previous article](/blog/articles/zeromq-on-spring-boot.html), we discussed using a heavy-weight broker for communication in a microservices architecture versus a more zen/zero/minimal approach. In this article, we'll explore some examples and practical applications and look at how well it might actually work.

## Use Case - Distributed Logging

Let's say we want to build a distributed logging solution for our distributed architecture.
Further, let's make it more difficult and say that our organization has placed limitations on us that we can either use our cloud platform's fire-hose as input and get this for free, or build a solution that requires no additional infrastructure to stand up.

Now, the only readily available option our ops team has given us is to use the fire-hose.
Effectively, this means using the STDOUT stream from each of our JVMs, Node.js servers, what have you, to go directly to the logging server, for example a Splunk cluster.
However, this solution is ignorant to the nature of our logs.
On the JVM, errors in the form of a stack trace are structured as multiple lines written to STDOUT.
There would be no clean way to view all lines of the stack trace as a single record in Splunk.
True story.

Let's build a solution

### Logging Adapter

Let's say you're using slf4j to log to a logback appender, which logs to STDOUT.
This is the default in Spring Boot.
Most teams won't ever go beyond this unless forced to.
Here's an intentionally over-simplified logback appender to log to STDOUT:

```java
public class ConsoleAppender implements Appender<ILoggingEvent> {
    ...

    public void doAppend(ILoggingEvent event) {
        System.out.println(event.getMessage());
    }
}
```

Let's replace this with Spring ZeroMQ. To do that, we will need a `ZmqTemplate`. For simplicity, let's assume we can take it as a constructor parameter.

```java
public class ZmqAppender implements Appender<ILoggingEvent> {
    private final ZmqTemplate zmqTemplate;

    public ZmqAppender(ZmqTemplate zmqTemplate) {
        this.zmqTemplate = zmqTemplate;
    }

    public void doAppend(ILoggingEvent event) {
        zmqTemplate.send(event.getMessage());
    }
}
```

Now at this point, we aren't doing much of anything impressive.
We're probably dropping the stack trace (if any) on the floor, and we throw away most of the useful data of the event as well, in favor of capturing only the log message.
We'll address this in a future post if there's interest.

From here, we must ask ourselves: "How do we get this data into Splunk?" Remember that `ZmqTemplate`? Configure it this way:

```java
@Configuration
@EnableZmqPublisher
public class PublisherConfiguration {
    @Bean
    public ZmqTemplate zmqTemplate() {
        ZmqTemplate zmqTemplate = new ZmqTemplate();
        zmqTemplate.setTopic("logs");
        zmqTemplate.setRoutingKey("LoggingEvent");
        zmqTemplate.setMessageConverter(new LoggingEventMessageConverter());

        return zmqTemplate;
    }
}
```

And configure the publisher thread in your `application.yml`:

```yaml
zmq:
  publisher:
    host: log-service.apps.mydomain.com
    port: 5678
    topics:
      - logs
```

So we probably need some kind of highly-available broker to slurp in those logs, right?
We could do that, but our ops team has said that's a non-starter.
Curse them!
Ok, what else can we do?

I'm guessing you already have a microservice somewhere in your architecture that isn't doing anything most of the time.
In a recent project, we had a remote deposit capture service sitting there that occasionally processed deposits from a mobile phone.
The rest of the time, it was bored.
Let's just package up a library that turns that service into a logging service.

### Logging Service

In the config above, notice we referenced a hypothetical service called `log-service`.
Using your cloud platform of choice, go ahead and bind that URL to your bored microservice.
In our case, `log-service.apps.mydomain.com` &mdash;> `remote-deposit-service.apps.mydomain.com`.

Now, we create a project that utilizes Spring Boot dependencies (the build file looks pretty much just like a spring boot app minus the spring boot plugin), but isn't itself a microservice.
Call it a micro library.

Add a `src/main/resources/META-INF/spring.factories` with the following:

```properties
org.springboot.boot.autoconfigure.EnableAutoConfiguration=\
com.mydomain.logging.autoconfigure.LoggingAutoConfiguration
```

Here's the `LoggingAutoConfiguration` class:

```java
@Configuration
@EnableZmqSubscriber
public class LoggingAutoConfiguration {
    @Bean
    public LoggingSubscriber loggingSubscriber() {
        return new LoggingSubscriber();
    }

    @Bean
    public MessageConverter messageConverter() {
        return new LoggingEventMessageConverter();
    }
}
```

<div class="alert alert-info" role="alert">
  Note: The <code>@EnableZmqSubscriber</code> above doesn't actually work right now. You'll need to add <code>zmq.subscriber.enabled: true</code> in your <code>application.yml</code> for now. This will be fixed in a future release.
</div>

Implementing the `LoggingEventMessageConverter` is slightly beyond the scope of this article.
However, refer to [this project](https://github.com/sjohnr/gsl-java-codecs) for a way to generate code that works with Spring ZeroMQ.

We just need a `LoggingSubscriber` to finish it out:

```java
@ZmqSubscriber({
    @QueueBinding(topic = "logs", key = "LoggingEvent", queue = "splunk-logs")
})
public class LoggingSubscriber {
    @ZmqHandler
    public void onLoggingEvent(LoggingEvent event) {
        // Send log data to Splunk...
    }
}
```

Sadly, I will not be covering how to actually put the log data into Splunk.
It is just a convenient use-case and example.
You will have to do that work yourself.

Now, just drop this jar into the microservice that needs a second job (think sand-bucket holder).
If you find that you can't afford to tax this service with the job any longer, it takes less than 5 minutes to move it to another.
If you can't find anything available that has spare compute to do the job, then just stand up a brand new microservice to do it.
Takes another 5 minutes.
In this way, you are embracing the ZeroMQ philosophy (and just a darn good philosophy in general) that if you start simple, and do the next right thing, you will always find a simple path forward to the next one, and so on.

But will it be worth it?

## Does it work?

To answer whether or not this solution will work well in practice, first consider the alternative.
In order to do this in a "traditional way," we would either need a solution handed to us by our ops team (please don't count on this, though thank them if they provide one to you), or we would need to convince our organization to buy one for us.
Buying a logging solution (they've already paid for Splunk in this example), is likely not an option.
Why do they hire engineers to solve problems for them, if the engineers say "It's not something I can solve. Just buy something."
That's ridiculous. Yet all too common.

Another alternative would be to demand a message broker. That's a fine option.
But how much would it save you? Answer: Nothing.
It actually costs you something extra. And it's the same amount of work.
Yet you don't get near the same flexibility. You aren't in complete control of your architecture.
You're now reliant on the ops team (remember the one who refused to provide a solution in the first place?) to maintain the broker.
It's OK... but not a great place to be if you don't need to be there.
I prefer to be in complete control over the entire architecture, and let the ops team handle non-solution-oriented infrastructure only.
Trust me, they prefer it that way too.

## Conclusion

In this article, we've learned how to solve a common and annoying problem with distributed logging by using spare compute, a bit of creativity, and Spring Boot without the use of a message broker.
We were able to leverage [in-spring-zeromq](https://github.com/InSourceSoftware/in-spring-zeromq) to do a job that seemed daunting, and spend less than hours on the solution.
Next time, we'll examine another interesting use case involving user activity, to find out what our users are up to.
