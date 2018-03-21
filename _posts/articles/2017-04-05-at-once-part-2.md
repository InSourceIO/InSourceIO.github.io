---
layout: post
subheadline: Open Source Series
title: "@Once - The Server - Part 2"
teaser: "@Once is a real-time communication and notifications framework for highly distributed networks."
categories:
  - articles
tags:
  - open source
  - real time
  - zeromq
related:
  - /blog/articles/introducing-at-once.html
  - /blog/articles/at-once-part-1.html
  - /blog/articles/at-once-part-3.html
image:
    title: stock/cars-1920.jpeg
    thumb: stock/cars-1920-t.jpeg
    homepage: stock/cars-1920.jpeg
mediaplayer: false
---

[sjohnr][1] / [at-once][2]

<a class="github-button" href="https://github.com/sjohnr/at-once" data-icon="octicon-star" data-style="mega" data-count-href="/sjohnr/at-once/stargazers" data-count-api="/repos/sjohnr/at-once#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star sjohnr/at-once on GitHub">Star</a>
<a class="github-button" href="https://github.com/sjohnr/at-once/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/sjohnr/at-once/network" data-count-api="/repos/sjohnr/at-once#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork sjohnr/at-once on GitHub">Fork</a>

Welcome back! Today, I'd like to discuss the server implementation of a peer-to-peer communication architecture. For background, let's start with the serverless design of ZyRE that we're building on with the @Once framework.

### ZyRE discovery

The protocol of ZyRE is described as follows on the [RFC page][3]:

> The ZeroMQ Realtime Exchange Protocol (ZRE) governs how a group of peers on a network discover each other, organize into groups, and send each other events. ZRE runs over the [ZeroMQ Message Transfer Protocol (ZMTP)][4].

The discovery mechanism used for ZyRE is UDP beacons. Here's how the [RFC page][3] describes it:

> ZRE uses UDP IPv4 beacon broadcasts to discover nodes. Each ZRE node SHALL listen to the ZRE discovery service which is UDP port 5670 (ZRE-DISC port assigned by IANA). Each ZRE node SHALL broadcast, at regular intervals, on UDP port 5670 a beacon that identifies itself to any listening nodes on the network.

So with a local area network, we can easily discover other nodes wanting to connect, by listening on a standard port for a fixed-size beacon, a packet of data that contains identifying information about a peer, and how to connect to them.

This is great for a LAN, but how can we do this over the internet?

### @Once discovery

As it turns out, ZyRE should work perfectly fine over the internet. The problem is that discovery doesn't work. So we need to come up with a way to handshake with peers without a UDP broadcast.

Perhaps we could send a TCP message that's equivalent to our UDP broadcast, but to a specific peer. This is probably the best option, since it's guaranteed to arrive, and has a certain appeal to it from a conceptual point of view. But there are a few too many problems with this approach. For starters, listening to a UDP socket and listening to a TCP socket are two different things. We'd have to change a fundamental part of ZyRE to make that work. Further, the structure of the beacon sent over ZyRE as specified in the RFC is not similar enough to the other message types. It's specifically designed to be succinct, to the point of being terse. We could write some manual code to do it, as code generation for one message won't have good ROI. And we could use ZMTP to send it. But why? We already have UDP beacons working with ZyRE.

Perhaps we could send a Hello message from the protocol instead. After all, according to the protocol the Hello message can be used as a means of establishing a connection. But we have a problem here too. The only way to know how to send a Hello message to a peer is if we receive a UDP beacon first. Why? Because the UDP beacon contains information about the actual port the peer is listening on for ZRE messages. If you read the specification, this is to allow for multiple peers on the same machine, same NIC even. Peers simply stack up on subsequent ports. So there's no way to tell what port a given peer will be listening on. Perhaps for the server we want to connect to (more on this later) it's OK, as we can "assume" the server is listening on a fixed port. But it's pretty weak tea to make that assumption just for the convenience of it. This would prevent us from hosting multiple server instances on the same node.

### @Once connectivity

Nope, there's nothing else for it but to go back to UDP beacons. But without broadcast, do we just send it to a specific address and hope for the best? Perhaps a bit inelegant, and certainly it has the potential for problems, especially if the UDP message gets lost as there's no guarantee of delivery. But we can easily handle retry for connectivity, if it's the only time we have to do it. We may have to think about security, but let's leave that for another time. Crawl first, and all that.

So it's settled, we'll connect to the server via a one-time (or perhaps retryable) UDP beacon. Once received, the server will turn around and connect to us with a TCP Hello message, since it knows what TCP port we're listening on. We'll turn around and connect to it, and send our own Hello message. And what's more, we can use this same mechanism to connect to other peers as well. We do have one last problem with this approach, however, so let's address that before moving on.

The problem with this approach is that UDP beacons are also used to maintain connectivity, as they continually announce "presence" to other peers on the network.

We could certainly add our own custom timer to invoke the same connect sequence regularly, which would in effect enable manual beacons per connected peer. This could be a long term solution, if needed, but we'd have to bake it into another layer around the ZRE interface we use in our applications, so it doesn't have to be re-invented each time @Once is used. Let's leave this as a last resort.

Thankfully, ZyRE has a solution for the problem of UDP beacons getting lost, so perhaps we could rely on that. The concept in play here is referred to as "evasive", as in a node that is having trouble sending its own beacons, or a network that is so congested it loses the beacons to other traffic. In that case, we rely on a less frequent (and usually one-time) TCP keep-alive message, or a Ping message in the ZRE protocol. Normally in ZyRE networks, the evasive state is an unusual circumstance. But we can simply let it handle our presence and call it good enough.

With that, we have a decent connectivity model for discovering our server, and other peers, which we'll see later.

*If you want to learn more about this topic, come back soon for Part 3 of this mini-series on server, where we'll discuss more about the server design and protocol for a p2p communication architecture. If you'd like to follow along with our progress, check out our GitHub repository below.*

[sjohnr][1] / [at-once][2]

<a class="github-button" href="https://github.com/sjohnr/at-once" data-icon="octicon-star" data-style="mega" data-count-href="/sjohnr/at-once/stargazers" data-count-api="/repos/sjohnr/at-once#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star sjohnr/at-once on GitHub">Star</a>
<a class="github-button" href="https://github.com/sjohnr/at-once/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/sjohnr/at-once/network" data-count-api="/repos/sjohnr/at-once#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork sjohnr/at-once on GitHub">Fork</a>
<script async defer src="https://buttons.github.io/buttons.js"></script>

 [1]: https://github.com/sjohnr
 [2]: https://github.com/sjohnr/at-once
 [3]: https://rfc.zeromq.org/spec:36/ZRE
 [4]: http://rfc.zeromq.org/spec:23/ZMTP
 [5]: http://zguide.zeromq.com
 [6]: https://github.com/zeromq/jzmq-api/blob/master/src/main/java/org/zeromq/api/Message.java
 [7]: https://github.com/sjohnr/jyre/blob/master/model/zmq_socket.gsl
 [8]: #
 [9]: #
 [10]: #