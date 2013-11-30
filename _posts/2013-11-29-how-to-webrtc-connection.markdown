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
server for you (a la [p2p](https://github.com/js-platform/p2p)) or running a 
broker server in the cloud for you (a la [PeerJS](http://peerjs.com/)).

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
(and other client/server models), leaving behind the kinds of peer-to-peer 
models WebRTC enables. 
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
by sharing data with each other before connecting directly to each other. 
Your app, using the broker, is responsible for relaying this information. 
Fortunately, your application does not need to understand the data being 
relayed.

Specifically, WebRTC needs help from the broker to perform three tasks:

1. Pair two peers and prepare them to connect to each other (via **offers**
   and **answers**)

2. Make sure both peers are using the same connection options (via a **session
   description**)

3. Figure out how to connect the peers (via the **[ICE 
   Protocol](http://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment)**)

We'll talk about each of these three steps in turn. 
They are all that's needed to set up a peer-to-peer WebRTC connection.

## Starting the Process (Offers and Answers)

First, a bit of terminology: let the **caller** be the WebRTC peer that is
initiating the connection, and let the **callee** be the WebRTC peer that
accepts it, thereby establishing the connection. 
Also, to keep things simple, we'll establish an `RTCDataChannel` connection for 
data rather than using `getUserMedia()` and starting a call (however, the 
connection semantics are similar for both scenarios).

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
    broker.send(desc);
});
```

The `desc` argument will be discussed in more detail below. When the callee
receives the message from the broker, it creates its own answer to the caller's
offer:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

pc.createAnwser(function(desc) { ... });
```

## Synchronizing Connection Descriptions

In the `createOffer` and `createAnswer` methods above, WebRTC passed a `desc`
argument to the completion callback. This `desc` option is an
`RTCSessionDescription` object, which describes the peer's WebRTC capabilities.
WebRTC expects the broker to relay these session descriptions between peers, so
that connection parameters can be negotiated.

Each `RTCPeerConnection` has a local description and a remote description. The
local description represents this peer's connection parameters, while the rmote
description represents the other peer's connection options. These descriptions
can be set via `RTCPeerConnection.setLocalDescription()` and
`setRemoteDescription()` respectively.

`setLocalDescription()` and `setRemoteDescription()` have the same signature: 

```js
setDescription(desc, onsuccess, onfailure)
```

The `onsuccess` callback is run if the description is acceptable; the
`onfailure` callback is run otherwise.

In order for connection establishment to continue, both the caller's
`RTCPeerConnection` objects must have had their `setLocalDescription()` and
`setRemoteDescription()` methods both called. To expand upon the example above,
the caller-side establishment code becomes:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

broker.ondesc = function(desc) {
    pc.setRemoteDescription(desc, function() { }, myLogError);
}

pc.createOffer(function(desc) {
    pc.setLocalDescription(desc, function() {
        broker.sendOffer(desc);
    }, myLogError);
});
```

Meanwhile, the callee-side connection code becomes:

```js
var broker = myConnectToBroker();
var pc = new RTCPeerConnection(...);

...

broker.onoffer = function(desc) {
    pc.setRemoteDescription(desc, function() {
        pc.createAnswer(function(desc) {
            pc.setLocalDescription(desc, function(desc) {
                broker.sendDesc(desc);
            }, myLogError);
        }, myLogError);
    }, myLogError);
}
```

TODO add a quick trace through the functions to unroll what's going on above

TODO and clean up this section in general

## Choosing the Connection Method

TODO ICE candidate stuff

## The Broker Interface

TODO the client-side broker object

TODO a simple implementation

## Putting It All Together

TODO end-to-end implementation with named source files
