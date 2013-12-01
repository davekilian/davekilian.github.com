---
layout: post
title: Setting Up WebRTC The Hard Way
author: Dave
draft: true
---

It may not immediately sound like it, but setting up a connection with WebRTC
is more challenging than you'd first think...

WebRTC alone is not able to establish a connection between two peers.
Instead, to establish a connection WebRTC requires the help of a "signaling 
server" (or **broker**), accessed through your app.
Most WebRTC wrapper libraries either handle this by implementing a broker
server for you to run (a la [p2p](https://github.com/js-platform/p2p)) or 
running a broker server in the cloud for you (a la 
[PeerJS](http://peerjs.com/)).

## Why Can't WebRTC Do It Alone?

The short answer: it has to do with the way the 'net has evolved over the
years. 
The long answer: well, ...

The Internet is a peer-to-peer system: every computer has an IP 
address, and can use the Internet to send packets to any other computer with 
an IP address. 
However, the World Wide Web (by far the most popular service
that the Internet supports) isn't peer-to-peer at all! 
Instead, the
would-be 'peers' are clients, who connect to a special 'peer' called a server.
But this is just an abstraction: the web's client/server model builds on top of
the Internet's peer-to-peer system.

The web is so popular that the terms 'World Wide Web' and 'Internet' have
almost become synonymous. 
As an indirect result, the Internet has evolved in ways that tailor to the web 
(and other client/server models), leaving peer-to-peer models (like those
WebRTC enables) out to dry.
Most notable of these developments is the ubiquity of 
[NAT routers](http://en.wikipedia.org/wiki/Network_address_translation). 

NAT routers were a hack that allowed the web to continue growing back when we 
were almost out of IPv4 addresses and IPv6 wasn't ready for primetime. 
NAT allows multiple computers to share a single IP address, but only allows 
these computers to act as clients (i.e. by making outgoing connection 
requests). 
Computers behind a NAT router can't receive inbound connections.

WebRTC implements workarounds that help computers behind NAT routers talk to
each other. 
Unfortunately, these techniques rely on peers sharing information in order to 
connect to each other; thus, WebRTC needs a way for peers to communicate with 
each other _before_ establishing a connection between them.
This is where your application, and the broker, come into play.

## What the Broker Does

As previously mentioned, WebRTC peers can work around NAT limitations, but only
by communicating through your broker before establishing the connection.
Specifically, WebRTC needs help from the broker to perform three tasks:

1. Pair two peers and prepare them to connect to each other (via **offers**
   and **answers**)

2. Make sure both peers are using the same connection options (via a **session
   description**)

3. Figure out how to connect the peers (via the **[ICE 
   Protocol](http://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment)**)

We'll talk about each of these three steps in turn. 
At each step, your app will use the broker to relay some piece of information.
Fortunately, you just need to relay the information, not interpret it!

## Starting the Process (Offers and Answers)

First, a bit of terminology: let the **caller** be the WebRTC peer that is
initiating the connection, and let the **callee** be the WebRTC peer that
accepts it, thereby establishing the connection. 

Assuming both the caller and callee have created their respective
`RTCPeerConnection` objects, starting the connection process is pretty simple.
The caller starts by calling `createOffer` on its `RTCPeerConnection`, and
sending the resulting `desc` object to the callee via the broker:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

pc.createOffer(function(desc) {
    ...
    broker.sendOffer(desc);
});
```

The `desc` argument will be discussed in more detail below. 
When the callee receives the message from the broker, it creates its own answer 
to the caller's offer:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

pc.createAnwser(function(desc) { ... });
```

## Synchronizing Session Descriptions

WebRTC uses the `RTCSessionDescription` object to describe a connection's
real-time streaming capabilities. 
This object wraps an implementation of the
[Session Description Protocol](http://en.wikipedia.org/wiki/Session_Description_Protocol),
or SDP.
The `desc` argument passed into the offer/answer callbacks above is an
`RTCSessionDescription`.

Each `RTCPeerConnection` knows two session descriptions: that of itself (the
**local** session description) and that of the peer at the other end of the
connection (the **remote** session description). 
During connection setup, WebRTC requires your app to set both.
This is done by calling `setLocalDescription` and `setRemoteDescription`,
both of which have the same type signature:

```js
set{Local|Remote}Description(desc, onsuccess, onfailure)
```

* `desc` is the `RTCSessionDescription` object
* `onsuccess()` is called if the operation suceeds
* `onfailure(error)` is called if the operation fails

During the offer/answer process, both peers must set the local and
remote session descriptions on their respective `RTCPeerConnection` objects.
The callee's remote description must match the caller's local description, and
vice versa. 

99% of the time, the `RTCSessionDescription` produced by `createOffer` or
`createAnswer` is acceptable as-is.
In this case, the app needs to do two things in the offer/answer callback:

* Set the local description: `setLocalDescription(desc)`
* Via the broker, send `desc` to the other peer. The other peer must call
  `setRemoteDescription()` on the description received.

Integrating this with the offer/answer handshake, the caller looks like this:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

broker.onanswer = function(desc) {
    pc.setRemoteDescription(desc, function() { }, myLogError);
}

pc.createOffer(function(desc) {
    pc.setLocalDescription(desc, function() {
        broker.sendOffer(desc);
    }, myLogError);
});
```

While the callee looks like this:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

broker.onoffer = function(desc) {
    pc.setRemoteDescription(desc, function() {
        pc.createAnswer(function(desc) {
            pc.setLocalDescription(desc, function(desc) {
                broker.sendAnswer(desc);
            }, myLogError);
        }, myLogError);
    }, myLogError);
}
```

To unroll what's going above:

1.  The caller calls `createOffer`, and receives a `desc`
2.  The caller calls `setLocalDescription(desc)`
3.  The caller sends the description via the broker
4.  The callee receives the description via the broker
5.  The callee calls `setRemoteDescription(desc)`
6.  The callee calls `createAnswer`, and receives a `desc`
7.  The callee calls `setLocalDescription(desc)`
8.  The callee sends the description via the broker
9.  The caller receives the description via the broker
10. The caller calls `setRemoteDescription(desc)`.
    The caller/callee session descriptions are now synchronized!

## Choosing the Connection Method

After the session descriptions have been synchronized, WebRTC uses the **[ICE 
Protocol](http://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment)**
to determine how to establish the connection between the peers.
This is the part of setup responsible for working around NAT limitations.

The contract for ICE is simple:

* Whenever WebRTC produces an ICE candidate (by firing the `onicecandidate`
  event of `RTCPeerConnection`), the app must send the candidate to the other
  peer via the broker.

* Whenever the app receives a candidate from the broker, the app must give the
  candidate to WebRTC locally by calling `RTCPeerConnection.addIceCandate`.

We can easily work this into our existing example (the new code is the same for
both the caller and the callee).

caller:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

pc.onicecandidate = function(event) {
    if (event.candidate) {
        broker.sendIce(event.candidate);
    }
}

broker.onice = function(candidate) {
    pc.addIceCandidate(candidate);
}

broker.onanswer = function(desc) {
    pc.setRemoteDescription(desc, function() { }, myLogError);
}

pc.createOffer(function(desc) {
    pc.setLocalDescription(desc, function() {
        broker.sendOffer(desc);
    }, myLogError);
});
```

callee:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

pc.onicecandidate = function(event) {
    if (event.candidate) {
        broker.sendIce(event.candidate);
    }
}

broker.onice = function(candidate) {
    pc.addIceCandidate(candidate);
}

broker.onoffer = function(desc) {
    pc.setRemoteDescription(desc, function() {
        pc.createAnswer(function(desc) {
            pc.setLocalDescription(desc, function(desc) {
                broker.sendAnswer(desc);
            }, myLogError);
        }, myLogError);
    }, myLogError);
}
```

Once this negotiation is complete, WebRTC automatically connects to the remote
peer using whatever connection mechanism was decided upon.
Voila!

## The Broker Object

Throughout this post, the examples have relied on a couple of 'magic' functions
provided by a broker object. 
Turns out this broker object doesn't need to be very magical, but it's worth
showing a sample implementation anyways...

```js
function Broker() {
    var ws = <create WebSocket connection to broker server>;

    ws.onmessage = function(e) {
        var msg = e.data;
        var data = JSON.parse(msg.data);
        
        if (msg.type == 'offer' && this.onoffer) {
            var desc = new RTCSessionDescription(data);
            this.onoffer(desc);
        }
        else if (msg.type == 'answer' && this.onanswer) {
            var desc = new RTCSessionDescription(data);
            this.onanswer(desc);
        }
        else if (msg.type == 'ice' && this.onice) {
            var candidate = new RTCIceCandidate(data);
            this.onice(candidate);
        }
    }
}

Broker.prototype.sendOffer = function(desc) {
    var msg = { type: 'offer', data: desc };
    this.ws.send(JSON.stringify(msg));
}

Broker.prototype.sendAnswer = function(desc) {
    var msg = { type: 'answer', data: desc };
    this.ws.send(JSON.stringify(msg));
}

Broker.prototype.sendIce = function(candidate) {
    var msg = { type: 'ice', data: candidate };
    this.ws.send(JSON.stringify(msg));
}

function myConnecToBroker() {
    return new Broker();
}
```

This example assumes the existence of some server at the other end of the
WebSocket. 
This server need not be very complicated either; it just needs some way to know
which peers want to talk to each other and track their open WebSockets.
Then, whenever one of the peers sends a message via its WebSocket, the server 
just needs to send the same message to the other peer. 
Easy!

## Bypassing the Broker

If you're just experimenting with WebRTC, setting up a broker may sound like a 
bit much overhead just to write Hello World!
For these sorts of debugging purposes, it's often sufficient to create two
WebRTC peers in the same browser tab.
Since both peer connections are in memory, no broker is needed at all;
instead, you can just directly call the session description/ICE methods
directly.
This is what many [WebRTC examples](http://simpl.info/rtcdatachannel/) do.

```js
var caller = new RTCPeerConnection(...);
var callee = new RTCPeerConnection(...);

// ... (set ICE callbacks, etc)

// Establish the connection
caller.createOffer(function(desc) {

// Set the caller's session description
caller.setLocalDescription(desc, function() {
callee.setRemoteDescription(desc, function() {

// Accept the connection
callee.createAnswer(function (desc) {

// Set the callee's session description
callee.setLocalDescription(desc, function() {
caller.setRemoteDescription(desc, function() { 

}, myLogError);
}, myLogError);
}, myLogError);
}, myLogError);
}, myLogError);
}, myLogError);

```

## Using the Connection

With the session description and ICE exchange steps complete, the session is 
up and ready! 
We can leverage the connection for audio, voice and/or data streaming.
In all three cases, the model is the same:

* One peer creates the stream using its `RTCPeerConnection` object
  (`addStream()` for audio/video streams, `createDataChannel()` for data)

* The other peer receives a matching stream via an event from its
  `RTCPeerConnection` (`onaddstream` for audio/video, `ondatachannel` for data)

Since these streams are part of the session description,
your app _must_ have called `addStream()`/`createDataChannel()` before
`createOffer()`.
Otherwise the stream won't be part of the connection
description, and the `onaddstream`/`ondatachannel` event won't get called on
the other peer!

## Reference

That's it for this tutorial, but there's lots of information online!
Here are a few WebRTC resources I have personally found useful:

* [WebRTC server-based signaling in much more
  depth](http://www.html5rocks.com/en/tutorials/webrtc/infrastructure/)
* [W3C draft of the WebRTC spec (current at the time of
  writing)](http://dev.w3.org/2011/webrtc/editor/webrtc.html)
* [A WebRTC audio/video streaming
  example](http://simpl.info/rtcpeerconnection/)
* [A data channel
  example](http://simpl.info/rtcdatachannel/)

