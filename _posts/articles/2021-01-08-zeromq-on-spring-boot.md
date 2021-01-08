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
  - /blog/articles/custom-authorization-with-spring-boot.html
  - /blog/articles/custom-authentication-with-spring-boot.html
image:
    title: stock/adobe/spring-rabbit.jpg
    thumb: stock/adobe/spring-rabbit-t.jpg
    homepage: stock/adobe/spring-rabbit.jpg
---

## Introduction

In the [previous article](/blog/articles/stateless-sessions-with-spring-boot.html), we discussed how to use Zuul's reverse-proxy functionality to propagate session information in a stateless way. In this article, we'll discuss something entirely different, namely a library for getting more mileage for your messaging buck.

## New Solutions to Old Problems

In the last year, I've worked heavily on some side projects that required thinking outside the box a bit in terms of minimizing dependencies and setup.
When wanting to deploy an efficient and elegant solution to a messy problem, the last thing I wanted to tell a client or potential customer was that they had to spend more or learn additional tools just to use my solution.
One thing that comes up a lot when connecting distributed architectures together is a communication layer.

There are a lot of options for this (though the obsession with messaging tools has died down a bit as microservices have become more mainstream, and AI and ML and other data-centric tech took center stage in the last few years).
What I've seen working for a large private bank here in the midwestern U.S. is a fairly 1-size-fits-all approach.
Just pick a single, decently good messaging platform, and bake it into your operational service offerings for all the developers in your organization to pull into their projects and use in any way they see fit.
Normally, I'd be fine with this, as an enterprise knows what they want, and if it's a good tool, then widespread adoption just makes it cheaper, easier to reuse, and lowers the onboarding and maintenance burden considerably.

(It's worth mentioning at this point that not all is sunshine and roses in the messaging garden. More on this later.)

What if you want to develop a more generalized tool that doesn't cost a fortune, yet lowers development costs, saves cycle time, hardware resources, and all the other niceties you want in a fancy sales pitch?
When it comes to messaging tools, most in the (JVM) microservices circuit will probably go with Kafka or RabbitMQ, with RabbitMQ being possibly the more common choice (depending on the use case and the organization).
After all, it's free and open source of course.
It seems well accepted and well understood.
Better yet, it has awesome support in most modern frameworks and cloud ecosystems.
So why not use it?

I'm not going to sell you on that point. Let's just look at an alternative, and you be the judge.

## ZeroMQ for connectivity

I won't say much here for why or why not choose ZeroMQ as a technology for your connectivity needs.
If you're curious, you can check out the [ZeroMQ Guide](http://zguide.zeromq.org/) for an in-depth look at this technology.
Just know, it's not necessarily going to be the new hotness or fastest growing open source library in 2021.
But it's been around a long time, and is steadily growing in maturity, and solidly proves itself time and time again as an approachable tool for connecting distributed architectures.

0MQ also has an amazing and vibrant community which is incredibly accepting, open and approachable.
I love it because I don't have time to compete with a lot of hot-shot committers who are all over things because they don't have anything else to do.
Good for them, but I need a community that wants whatever I have time to put out there.
It's so cool!

## Introducing Spring ZeroMQ

With that being said, I'd like to introduce a project that I released yesterday evening on Maven Central, [in-spring-zeromq](https://github.com/InSourceSoftware/in-spring-zeromq).
Check out the [README](https://github.com/InSourceSoftware/in-spring-zeromq/blob/master/README.md) for some getting started info on the library.

A quick background is in order. I mentioned [RabbitMQ](https://www.rabbitmq.com/). I like RabbitMQ actually. It's nice.
It has worked well in the simple use cases we've used it for in a microservices architecture.
What I really like about RabbitMQ is the support bolstering it has gotten in the Spring community with [Spring AMQP](https://spring.io/projects/spring-amqp).

<div class="alert alert-info" role="alert">
  Fun fact, did you know that iMatix, the company founded by Pieter Hintjens, was the initial firm contracted by JPMorgan Chase to build <a href="https://en.wikipedia.org/wiki/Advanced_Message_Queuing_Protocol#History">AMQP</a>?
</div>

What I don't necessarily like is the need for a broker when I'm doing simple communication patterns.
Not that ZeroMQ is only well-suited for simple communication patterns.
But it feels kind of ironic that RabbitMQ's slogan is "Messaging that just works" when you have to set up (or pay for a sidecar service to use) a broker.
By definition, that does not "just work." It requires extra work... to work.
Or extra money.

Anyway, I wanted to create a ZeroMQ programming model in Spring that matched the ease-of-use of Spring AMQP.
While I wouldn't say this library approaches the level of sophistication of spring-amqp, I wanted to lay a foundation built against the same idea.
Namely, when I'm building enterprise software, I don't want to stop to have to think about how to write event loops, socket communication, and complex protocols just to send messages between two pieces.
Not every part of my architecture deserves re-inventing the wheel every time I run into a connectivity problem needing to be solved.
And yet, I don't want to have to be tied to a message broker either.
I want an architecture that can evolve in micro-units, like the ZeroMQ programs Pieter outlines in the ZeroMQ Guide (you really should read it).

## What can I do with this?

Come back next time, for an in-depth look at in-spring-zeromq, a little library for hiding ZeroMQ behind a protective enterprise development blanket (on the JVM).

<div class="alert alert-success" role="alert">
  Check out the library now! <a href="https://github.com/InSourceSoftware/in-spring-zeromq">Spring ZeroMQ</a>
</div>

## Conclusion

In this article, we've learned... well, nothing really. Hopefully we'll get to learn a bunch next time. Thanks for watching!
