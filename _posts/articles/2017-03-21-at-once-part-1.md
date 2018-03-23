---
layout: post
subheadline: Open Source Series
title: "@Once - The Server - Part 1"
teaser: "@Once is a real-time communication and notifications framework for highly distributed networks."
categories:
  - articles
tags:
  - open source
  - real time
  - zeromq
related:
  - /blog/articles/introducing-at-once.html
  - /blog/articles/at-once-part-2.html
  - /blog/articles/at-once-part-3.html
image:
    title: stock/sunrise-1920.jpg
    thumb: stock/sunrise-1920-t.jpg
    homepage: stock/sunrise-1920.jpg
---

[sjohnr][1] / [at-once][2]

<a class="github-button" href="https://github.com/sjohnr/at-once" data-icon="octicon-star" data-style="mega" data-count-href="/sjohnr/at-once/stargazers" data-count-api="/repos/sjohnr/at-once#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star sjohnr/at-once on GitHub">Star</a>
<a class="github-button" href="https://github.com/sjohnr/at-once/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/sjohnr/at-once/network" data-count-api="/repos/sjohnr/at-once#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork sjohnr/at-once on GitHub">Fork</a>

Ahhh, the server. Such a useful concept. So convenient. So powerful. So... expensive? Why would anything so useful have the potential to be the enemy of the good?

I recently read an article (which I won't bother to link to), which described peer-to-peer messaging as an anti-pattern. The rationale goes like this:

**Problem:**

Messaging is hard. Baking low-level communication details and logic into my application makes it hard to maintain. And besides, I'm extremely lazy.

**Solution:**

Outsource that crap.

**T.L.D.R.**

Specifically, outsource my messaging needs to a monstrously complex middleware system so I never have to think about it again. I simply have to run all the connectedness of my entire ecosystem through a single choke-point so that I gain control, auditing capabilities, horizontally scalability, and the illustrious ability to queue messages in "The Message Broker" so my application can't have problems due to messaging.

Basically, it's too hard to bother with, so we'll just buy something.

### The new (old) model

It's true that the client-server paradigm is extremely powerful. It's been used to great effect to create powerful new capabilities where client software is simple, pretty, lightweight, and effective. In this model, we simply let the server software take the brunt of everything, as it becomes complex, heavyweight, burdensome, overloaded, difficult to scale, and in constant need of refactoring, rearchitecting, evolving, and ultimately sending to the scrapheap so we can start over.

It works. Well, in fact. The web, perhaps the most powerful illustration of the client-server model, is now a portal to a million desktop apps. And yet, as well as it works, apps running natively on your phone or tablet are all the rage. Why is that, I wonder? Has the web failed in its only real calling, to deliver us from the shackles of thick client native software, and allow us to simply download what we need from the cloud on demand, so we can trash it and leave it to expire from our RAM buffers two seconds later?

Under the client-server model, the server is a very important piece of the formula, the most important piece. In order to make your software truly great, you must build out a truly remarkable server implementation. Only then can you get customers (clients) to use your app. Once you have customers, your server begins to get taxed, and you quickly find the problems, change the server, refactor, fix, scale, etc. It's fine. Things are fine. REALLY!

But the reality is, things are not fine. For the most successful, this is where the real hurt comes from. The only way to maintain the complexity and needs of the server environment are to hire massive teams of engineers and operations people to manage the code and infrastructure, learn really fast how to build truly scalable systems, buy millions upon millions of dollars worth of hardware, expend vast resources on third party systems to enhance your homegrown masterpiece and give you the capabilities you need to keep up, and then simply keep going until the sun goes supernova.

Very few organizations have the resources necessary to make this work.

### The (old) new model

So, when we talk about the next generation of technological capabilities, is the server in the picture?

The answer is, absolutely. But we need to think differently about the role the server plays in our architecture. Depending on the system in question, this thought process can play out very differently. But when it comes to real-time communication (you can call it messaging if you want, I won't blame you... much), all of that cost can safely be considered waste.

We can leverage the capabilities of the connected devices themselves to much greater effect, and diminish or even potentially eliminate the role of the server in many cases.

### Mr. Server, meet the peer-to-peer architecture

So what role does the server play in peer-to-peer?

I'm glad you asked.

If the server had only one role to play and couldn't be used for anything else, it would have to play the role of discovery. After all, the web is a big place, and we have to start somewhere. So if we can build a system that uses the simplest server known to man (a single program with one responsibility), we would probably be winning already. All it has to do is allow us to perform a basic handshake to announce ourselves, possibly verify our identity if we care about such things as security and privacy, and then tell us who else is around looking for friends, we can bounce and do our own thing from then on.

Sound too good to be true? Sound like an anti-pattern otherwise known as peer-to-peer? Great, let's build it!

*If you want to learn more about this topic, come back soon for Part 2 this mini-series on server, where we'll discuss building out the server implementation for a p2p communication architecture. If you'd like to follow along with our progress, check out our GitHub repository below.*

[sjohnr][1] / [at-once][2]

<a class="github-button" href="https://github.com/sjohnr/at-once" data-icon="octicon-star" data-style="mega" data-count-href="/sjohnr/at-once/stargazers" data-count-api="/repos/sjohnr/at-once#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star sjohnr/at-once on GitHub">Star</a>
<a class="github-button" href="https://github.com/sjohnr/at-once/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/sjohnr/at-once/network" data-count-api="/repos/sjohnr/at-once#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork sjohnr/at-once on GitHub">Fork</a>
<script async defer src="https://buttons.github.io/buttons.js"></script>

 [1]: https://github.com/sjohnr
 [2]: https://github.com/sjohnr/at-once
 [3]: #
 [4]: #
 [5]: #
 [6]: #
 [7]: #
 [8]: #
 [9]: #
 [10]: #