---
layout: post
subheadline: Open Source Series
title: "Introducing @Once"
teaser: "@Once is a real-time communication and notifications framework for highly distributed networks."
categories:
  - articles
tags:
  - open source
  - real time
  - zeromq
related:
  - /blog/articles/at-once-part-1.html
  - /blog/articles/at-once-part-2.html
  - /blog/articles/at-once-part-3.html
image:
    title: stock/cars-1920.jpeg
    thumb: stock/cars-1920-t.jpeg
    homepage: stock/cars-1920.jpeg
---

[sjohnr][1] / [at-once][2]

<a class="github-button" href="https://github.com/sjohnr/at-once" data-icon="octicon-star" data-style="mega" data-count-href="/sjohnr/at-once/stargazers" data-count-api="/repos/sjohnr/at-once#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star sjohnr/at-once on GitHub">Star</a>
<a class="github-button" href="https://github.com/sjohnr/at-once/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/sjohnr/at-once/network" data-count-api="/repos/sjohnr/at-once#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork sjohnr/at-once on GitHub">Fork</a>

If you haven't heard of the <abbr title="ZeroMQ Realtime Exchange">ZRE</abbr> protocol, or ZyRE, you should head over to [Github][3] to read about it.

@Once (pronounced "at once") is an open source framework based on ZyRE. The basic idea behind zyre is that you can create ad-hoc communication networks on a local network without a coordinating server, using presence discovery (UDP beacons or possibly the Gossip Protocol). Once connected, peers can send messages to other peers directly (whisper), or send group messages to many peers (shout). Sound familiar? It should, as you probably use this every day at your workplace, with tools like Slack, Skype, or Google Hangouts. What's cool about ZyRE is the automatic discovery aspect. Otherwise, it's just a high-level framework for clustering and communication, which is no small thing, but you have lots of those to choose from.

However, what would be cooler is if you could use something like ZyRE over the internet. Here again, you're probably thinking "I've heard of this, it's called..." fill in the blank. Apple, Google, and a zillion others have networks that are used for realtime communication. In addition, there are numerous cool new technologies for secure communication, the most modern of which add privacy to the mix for end-to-end encrypted communication that even the server (hosted by somebody you may not trust to know your communication habits) can't know about. Tools like Signal or WhatsApp come to mind. Or perhaps you're thinking about the new focus on mesh, making it possible to communicate without internet. This would be something like FireChat.

The truth is, there are lot of instant communication tools out there. So why do we need a new one? Well, the answer is simple. In the spirit of Ø, we need a framework for building reusable pieces. As great as all the above mentioned technologies and many like them are, very few have a focus on creating a framework that allows for any type of application to utilize such connected capabilities. The tools out there today tend to be specifically connecting users together in a single type of experience, trying to create the next social phenomenon, or solve a very specific problem, thus limiting their usefulness in the larger scheme. There's nothing wrong with this, but in the modern era of technology, we can do better.

So I've created @Once to attempt to address such a problem. Instead of building a set of massively complex libraries that are strung together to produce a single result (the next killer iPhone app, for example), I want to create a framework for realtime, and specifically realtime peer-to-peer, communication.

Many have overlooked ØMQ as a viable technology, as the enterprise world is obsessed with reliable broker-based communication technologies, which unbeknownst to them (until it's too late), tend not to scale well. Perhaps more surprising, the non-enterprise world has continued to do things the hard way, supposing that rolling your own encryption and communication protocols, and then building tools on top of them, makes you a really smart engineer. It probably does. But it seems to me, this is a foolhardy endeavor. The complexity of something so home-grown makes it totally impossible to reuse. Sometimes, with a little care, the protocols themselves can be somewhat reusable. Though the maturity of the community and the tools used by it are not sufficient to get deep adoption across multiple platforms, let alone with relative ease.

ØMQ has been around a long time now. And though it has not had explosive growth, the community has continued to mature, and has a very sophisticated set of tools, many of which were established by the late Pieter Hintjens. These include tools like cross-platform and cross-language code generation, allowing any library written in C to be consumed through generated language bindings for many popular languages. Not to mention the wide array of ports available for the myriad of libraries and protocols within the ØMQ ecosystem. Unfortunately, such useful and practical approaches to software engineering just aren't sexy enough to attract that much attention.

Well, thanks for sticking with me this far. If you're interested in hearing more about what @Once is all about and what it will be able to do, stay tuned. It's still in its infancy, so there's a lot of development to be done. However, we should be able to get a decently reusable library for one tenth the cost (in terms of <abbr title="Lines of Code">LoC</abbr>) of other libraries, by utilizing the sophistocated <abbr title="ZeroMQ Message Transport Protocol">ZMTP</abbr>, Curve, and <abbr title="ZeroMQ Realtime Exchange">ZRE</abbr> protocols, and a few sound engineering techniques that seem lost to the annals of time. Follow us on GitHub and check back to see progress. Should be fun!

[sjohnr][1] / [at-once][2]

<a class="github-button" href="https://github.com/sjohnr/at-once" data-icon="octicon-star" data-style="mega" data-count-href="/sjohnr/at-once/stargazers" data-count-api="/repos/sjohnr/at-once#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star sjohnr/at-once on GitHub">Star</a>
<a class="github-button" href="https://github.com/sjohnr/at-once/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/sjohnr/at-once/network" data-count-api="/repos/sjohnr/at-once#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork sjohnr/at-once on GitHub">Fork</a>
<script async defer src="https://buttons.github.io/buttons.js"></script>

 [1]: https://github.com/sjohnr
 [2]: https://github.com/sjohnr/at-once
 [3]: https://github.com/zeromq/zyre
 [4]: #
 [5]: #
 [6]: #
 [7]: #
 [8]: #
 [9]: #
 [10]: #