---
layout: post
subheadline: Open Source Series
title: "@Once - The Server - Part 3"
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
  - /blog/articles/at-once-part-2.html
image:
    title: stock/architecture-1920.jpg
    thumb: stock/architecture-1920-t.jpg
    homepage: stock/architecture-1920.jpg
mediaplayer: false
---

[sjohnr][1] / [at-once][2]

<a class="github-button" href="https://github.com/sjohnr/at-once" data-icon="octicon-star" data-style="mega" data-count-href="/sjohnr/at-once/stargazers" data-count-api="/repos/sjohnr/at-once#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star sjohnr/at-once on GitHub">Star</a>
<a class="github-button" href="https://github.com/sjohnr/at-once/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/sjohnr/at-once/network" data-count-api="/repos/sjohnr/at-once#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork sjohnr/at-once on GitHub">Fork</a>

Welcome back! Today, I'd like to discuss in more detail the server implementation of a peer-to-peer communication architecture.

### Recap

Last time, we discussed discovery and connectivity using the ZRE protocol over the internet, to add the concept of a distributed ZyRE network. We saw how we could use that to connect to a server, and even how we could connect to other peers. However, we need a mechanism for actually discovering those other peers. That's where the server comes in.

### The @Once server

So what will our server do for us, now that we're connected to it? One of the nice things about ZyRE, is the state management handled by the background thread. It maintains a list of connected peers, and all the necessary information about that for us to write applications on top of, and we don't have to do that state management. We can ask for it at any time, and Zap-Pow, the API of ZyRE gives it to us.

So the server can just be a regular ZyRE peer. Once connected, we can ask it who else is connected, and it can tell us. Obviously, at this point we'd also want to think about security, but there are enough details there that we can save that for a later time as well.

So let's turn our attention to communicating with the server about what peers are connected so that we can connect to them as well. This will fill in the final aspect of our discovery model, and cement the server's role in our design. But at this point, we have two things to consider.

1. We need to build out a new protocol for these types of messages, since ZRE doesn't handle this for us.
2. Since this new protocol will have multiple uses, we need to establish roles for our application components and begin to isolate them.

#### Identifying roles

Let's begin to tackle #2 first since it's the easiest to think about. The main role we're thinking about is the server role. But there will obviously be clients connected to that server, so let's call that out. There will probably be other roles, and we'll need to begin to distinguish each, but let's stick with these two for now. We'll call them server and bridge peer. Why bridge peer? Well, any peer that's connecting to the server is going to be connected to both a local network and a remote network. That sounds a lot like a bridge node to me.

With the bridge peer connecting to the server through a manual handshake process, we have the opportunity for all sorts of weirdness. If we (for testing, say) run the server on the same network as our bridge peer, they wouldn't even have the opportunity to manually handshake, as they'd find each other automatically through UDP beacon broadcast. Only an issue for testing. But further, the server would have to filter out noise coming from all the nodes that it sees automatically, so that it can care about only bridge peer connections, and the messages that will follow. So we can do two things about this:

1. Pick a different port for the server to listen for beacons on. Let's increment ZRE's port by one for now.
2. We need to disable UDP beacons through configuration. This solves our testing use case.

For \#2 would, this requires a change to the ZRE library we're using, but requires no changes to the protocol. We can do that as an enhancement. \#1 is the same type of thing really, but one that the ZyRE maintainers already thought of. Obviously, you might want to use a custom port for your ZyRE network, so we'll use what they give us for this and set our own port.

#### Designing a protocol

Next, let's talk about protocol. We need to start building out a set of our own messages that can be part of @Once. Luckily, we can use the `content` portion of a Whisper or Shout message in the ZRE protocol to design our own. The only limitation we have is that the protocol is not currently multi-frame. So if we want to add a blob of our own data, we'll need to embed that inside a single frame with other data. Later, perhaps we could add frames to the ZRE protocol, but let's not mess with a good thing right now, and let sleeping giants lay.

Our first message needs to be a simple request to get a list of peers. Specifically, we need a list of endpoints, or addresses, that we can connect to so we can send UDP beacons to them, and establish a connection. Obviously, we'll need to a response message with that data as well.

Here's a snippet of a formal grammar for these messages, which we'll expand (and change) later.

```
    ;  Get a list of peers connected to the server.                          

    GET-ENDPOINTS   = signature %d1 version
    version         = number-1              ; Version number (1)

    ;  Send a list of peers connected to the server.                         

    LIST-ENDPOINTS  = signature %d2 version endpoints
    version         = number-1              ; Version number (1)
    endpoints       = strings               ; List of peer endpoints.

    ; A list of string values
    strings         = strings-count *strings-value
    strings-count   = number-4
    strings-value   = longstr

    ; Strings are always length + text contents
    string          = number-1 *VCHAR

    ; Numbers are unsigned integers in network byte order
    number-1        = 1OCTET
    number-4        = 4OCTET
```

That should be good enough. But we have to have a way to send this data. If you've studied the way ZyRE was built, as covered in the [ZeroMQ Guide][5], you know that code generation is the way to go here. Once we start messing with lots of protocol design, we aren't writing that code by hand anymore. But most of the code generated in ZyRE assumes it's being sent directly over a socket. That's just a refactoring issue, but we can't use free tools verbatim. We'll have to design a bit of our own tooling.

I've been working with [jzmq-api][6] and [jyre][7] to do this. It's still general purpose enough that it should be useful for the community, so I'll hopefully be able to contribute that back as a new codec. Basically, we need a codec layer that can serialize/deserialize data, independent of the socket layer which sends/receives that data. So we can get a `Message` back, and hand that off to the ZRE interface. We can use the Whisper message to communicate directly with the server when sending a request, and asynchronously receive a response, as we would with the `DEALER-ROUTER` pattern.

And there you have it! We've got a good start on our server implementation, and can even begin building out the bridge peer.

*Next time, we'll discuss the bridge peer in more detail, and see how we can enhance this design to bridge a local network together with a remote network discovered via a server. If you want to learn more about this topic, come back soon for Part 1 of a new mini-series, where we'll discuss building out the bridge peer implementation for a p2p communication architecture. If you'd like to follow along with our progress, check out our GitHub repository below.*

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