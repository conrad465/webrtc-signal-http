# webrtc-signal-http

[![Build Status](https://travis-ci.org/bengreenier/webrtc-signal-http.svg?branch=master)](https://travis-ci.org/bengreenier/webrtc-signal-http)

[![Deploy to Azure](https://azuredeploy.net/deploybutton.png)](https://azuredeploy.net/)

opinionated webrtc signal provider using `http` as a protocol :spider_web: :signal_strength:

![logo gif](./readme_example.gif)

We needed a simple to use, easy to extend [WebRTC](https://webrtc.org/) signaling server that communicated over regular old `HTTP/1.1` for [3dtoolkit](https://github.com/catalystcode/3dtoolkit) - this is it. It's designed to mirror [the WebRTC example server](https://github.com/svn2github/webrtc/tree/master/talk/examples/peerconnection/server) at an API level, while allowing developers to consume and extend the base functionality.

## Getting started

> Learn about the RESTful API via the OpenAPI doc ([raw](./swagger.yml) or [hosted](https://rebilly.github.io/ReDoc/?url=https://github.com/bengreenier/webrtc-signal-http/blob/master/swagger.yml)) to understand how clients should interact with the service.

To install the server cli `npm install -g webrtc-signal-http`. To run it, just use `webrtc-signal-http` from the command line, using the `PORT` environment variable to configure it's listening port.

To consume this server as a basis but add some extended functionality, `npm install webrtc-signal-http` and then run some code like the following:

```
const signalAppCreator = require('webrtc-signal-http')
const useBunyanLogger = true

const app = signalAppCreator(useBunyanLogger)

app.get('/new-endpoint', (req, res) => { res.send('hello') })

app.listen(process.env.PORT || 3000)
```

## RESTful API

For example clients, see the following:
+ [webrtc-native-peerconnection](https://github.com/svn2github/webrtc/tree/master/talk/examples/peerconnection/client)

### GET /sign_in

> Takes `peer_name` query parameter

Indicates a peer is available to peer with. The response will contain the unique peer_id assigned to the caller in the `Pragma` header, and a `csv` formatted list of peers in the `body`.

```
GET http://localhost:3000/sign_in?peer_name=test HTTP/1.1
Host: localhost:3000

=>

HTTP/1.1 200 OK
Pragma: 1
Content-Type: text/plain; charset=utf-8
Content-Length: 8

test,1,1
```

### GET /sign_out

> Takes `peer_id` query parameter

Indicates a peer is no longer available to peer with. It is expected this method be called when a peer is no longer intending to use signaling. The response will be empty.

```
GET http://localhost:3000/sign_out?peer_id=1 HTTP/1.1
Host: localhost:3000

=>

HTTP/1.1 200 OK
Content-Length: 0
```

### POST /message

> Takes `peer_id` (indicating the caller id) and `to` (indicating whom we're sending to)

Provides a messaging mechanism for one peer to send data to another. There are no requirements around the type of data that can be sent. The message may be buffered until the receiving peer connects to the service. The response will be empty.

```
POST http://localhost:3000/message?peer_id=2&to=3 HTTP/1.1
Host: localhost:3000
Content-Type: text/plain
Content-Length: 12

test content

=>

HTTP/1.1 200 OK
Content-Length: 0
```

### GET /wait

> Takes `peer_id` query parameter

Provides a mechanism for simulated server push, using vanilla http long polling. That is, the TCP socket behind this request will remain open to the server until there is content the server needs to send. In the event of a TCP timeout the client should reconnect. Messages that contain a `Pragma` value that matches the client `peer_id` are peer status updates and should be handled the same as the status update provided in the `GET /sign_in` response. `Content-Type` headers will not reflect the type of the original content.

Peer status update:

```
GET http://localhost:3000/wait?peer_id=2 HTTP/1.1
Host: localhost:3000

=>

HTTP/1.1 200 OK
Pragma: 2
Content-Type: text/html; charset=utf-8
Content-Length: 18

test2,3,1
test,2,0
```

Peer message:

```
GET http://localhost:3000/wait?peer_id=2 HTTP/1.1
Host: localhost:3000

=>

HTTP/1.1 200 OK
Pragma: 3
Content-Type: text/html; charset=utf-8
Content-Length: 12

test content
```

## Extension API

For example extensions, see the following:
+ [webrtc-signal-http-heartbeat](https://github.com/bengreenier/webrtc-signal-http-heartbeat)

### module.exports

> This is the exported behavior, you access it with `require('webrtc-signal-http)`

[Function] - takes `boolean` indicating if the bunyan logger should be enabled. __Returns__ an [express](https://expressjs.com) `app` object.

#### app.get('peerList')

[Object] - can be used to retrieve a `PeerList` from the express `app`. __Returns__ a [PeerList](#peerlist) object.

### PeerList

[Class] - Represents a collection of WebRTC peers on which signaling operations are possible.

#### addPeer

[Function] - takes `name` (a string), and `res` (a http.Response object). Creates a representation of the peer for signaling. __Returns__ a `Number` that shall be used as a unique id for the peer.

#### removePeer

[Function] - takes `id` (a Number). Removes the representation of the peer from signaling. __Returns__ nothing.

#### getPeer

[Function] - takes `id` (a Number). Retrieves the representation of the peer from signaling. __Returns__ a [Peer](#peer) object.

#### getPeerIds

[Function] - takes nothing. Retrieves all the peer id's in the PeerList. __Returns__ an [Array] of id's (Numbers).

#### setPeerSocket

[Function] - takes `id` (a Number), and `res` (a http.Response object). Updates a representation of the peer with a new response object for signaling. __Returns__ nothing.

#### pushPeerData

[Function] - takes `srcId` (a Number), `destId` (a Number), `data` (an Object). Pushs arbitrary data onto a stack for a particular destination peer. __Returns__ nothing.

#### popPeerData

[Function] - takes `id` (a Number). Retrives arbitrary data from the stack for the particular peer. __Returns__ a [PeerData](#peerdata) object.

#### format

[Function] - takes nothing. Formats a csv (without headers) list of the peers in the format of `peerName, peerId, peerConnectionStatusAsInt`. It's worth noting that `peerConnectionStatusAsInt` is a `0` or `1` literal that represents if the peer is currently connected to the server. __Returns__ a `string`.

### PeerData

[Object] - Represents arbitrary data to be sent to a peer on behalf of another peer.

#### srcId

[Number] - the peer id that sent the data (as defined when `pushPeerData` was called).

#### data

[Object] - the peer data to be sent (as defined when `pushPeerData` was called).

### Peer

[Object] - Represents a peer for signaling. Contains both interally used data properties as well as externally exposed data properties.

#### name

[String] - the name of the peer.

#### id

[Number] - the unique id of the peer.

#### buffer

[Array] - an internally used stack that stores arbitrary data (see `pushPeerData` and `popPeerData`).

#### res

[http.Response] - the response object of the peer. Used to send 'push' data to a hanging socket.

#### status

[Function] - takes nothing. Determines if the peer's `res` is able to be written to. __Returns__ a `bool`.

## License

MIT