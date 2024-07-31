# Introduction to v2ray transports

*2024-08-01*

Translations: [Russian](https://marzban.dev/blog/about_transport/)

## Background

**Note**: The historical details are to the best of my knowledge, but I am not sure about them.

V2ray originally started out with Vmess-TCP only. Over time, Vmess got
detected, and v2ray added some functionality to wrap the underlying TCP socket
in additional layers.

* One of these layers are transports, the main point of this document.

* The other one may be some kind of mux, depending on which fork of v2ray you use.

* TLS is not a "real" layer. The settings are passed into transport code, and
  the transport can do whatever it wants with it. However, all transports treat
  `tlsSettings` in almost the same way, and in fact call the same helper
  functions.

Anyway, back to transports.

Transports in v2ray are *anything* that can emulate (or wrap) a bidirectional
pipe between server and client. TCP sockets are just that, but it doesn't have
to be TCP, or even based on TCP. It just has to be something that transmits
bytes in-order from client to server without losing them, and the other way
around.

After this pipe is provided, Vmess, VLESS or Trojan can be run over that
transport. Or something else entirely.

## Our first transport

We're going to use some Linux commands to showcase what transports can do.
Let's get started with something simple.

Open a terminal and type this:

```
nc -l localhost 6003  # server
```

and in another terminal:

```
nc localhost 6003  # client
```

Write some lines in one terminal and watch them appear on the other. It works in both directions.

This command is equivalent to "TCP transport" in v2ray. You can substitute host
and port in the `client` command to test if a TCP port is open.

## WebSocket

WebSockets are a HTTP-based protocol to literally do the same thing as a TCP
socket. The main difference is that:

1. WebSocket is adding large HTTP-based handshake at the beginning of the
   connection. This is useful for cookies, authentication, serving multiple
   "WebSocket services" under different HTTP paths, and for enforcing that a
   website can only open connections to its own Origin, instead of random
   hosts.

   For the purpose of building transports, some of those features, particularly
   path-based routing, are pretty useful, but the additional back-and-forth at
   the beginning adds additional latency.

2. WebSocket does not transmit bytes, but "messages". For our purposes it means
   that instead of `Write("hello world")` sending `hello world` (like on TCP),
   it actually means that `<frame header>hello world` is being sent. For the
   purpose of building transports, this design is useless overhead.

The reason we put up with both of these things is because certain CDN can
forward WebSockets as-is.

If `nc` is basically a TCP transport, then tools like
[websocat](https://github.com/vi/websocat/) or
[wscat](https://github.com/websockets/wscat) are basically a WebSocket
transport.

In fact, those tools are very useful to test whether websocket is listening
correctly on a particular path:

```
curl https://example.com  # example.com is a valid HTTP service...
websocat wss://example.com  # ...but not actually websocket! this fails
```

If `websocat` fails on your v2ray server but `curl` can open it, it probably
means you messed up some path settings somewhere.

To get a local test server like with the previous `nc` example, you can do this:

```
websocat -s 6003  # server
websocat ws://localhost:6003  # client
```

Once again you can send random lines of text back and forth.

Since WebSocket is just HTTP/1.1, and HTTP/1.1 is just TCP, we can even point `websocat` against `nc` to dump the initial handshake:

```
nc -l localhost 6003  # server
websocat ws://localhost:6003  # client
```

When you run the second command, it will actually print:

```
GET / HTTP/1.1
Host: localhost:6003
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: MOIjFT7/cVsCCr95mkpCtg==


```

This is some of the overhead of WebSockets. The client is still waiting for a response. So let's extract the response too.

Kill both commands with `Ctrl-C`, and start `websocat -s 6003` as server.

Take the above text, and send it directly with `nc`:

```
echo 'GET / HTTP/1.1
Host: localhost:6003
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: MOIjFT7/cVsCCr95mkpCtg==
' | unix2dos | nc localhost 6003 | cat -v
```

1. `echo` just prints the data
2. `unix2dos` converts line-endings, because HTTP/1.1 expects `\r\n` instead of `\n`
3. `nc` sends the data to the server and prints out the response
4. `cat -v` makes special characters (or rather, non-character bytes) visible.

You will see the response:

```
HTTP/1.1 101 Switching Protocols^M
Sec-WebSocket-Accept: HKN6nOSb0JT0jWhszYuKJPUPpHg=^M
Connection: Upgrade^M
Upgrade: websocket^M
```

After this, you can send data from the server to the client by typing into its
terminal window. Type `hello` into the `websocat` server, and observe on the client:

```
M-^A^Fhello
```

The garbage in front is WebSocket message framing.

Sending data client-to-server does not work, because WebSocket expects that
data is wrapped in this additional message framing.

## WebSocket 0-RTT

**(getting rid of connection handshake overhead)**

Where regular TCP just does its handshake, WebSocket sends a HTTP request and
waits for the response. This additional latency is very visible, especially in
this flow:

1. TCP handshake
2. Send HTTP request
3. Wait for HTTP response and read it
4. Send VLESS instruction to connect to some website `pornhub.com`
5. Wait for the answer and read it

Send a little bit of data, wait, send a little bit of data again, wait again.

It would be faster to send a lot of data, and then wait for a lot of data:

1. TCP handshake (can't avoid it... for now...)
2. Send HTTP request together with the first VLESS instruction to connect to `pornhub.com`
3. Wait for the HTTP response and the first bytes of the body at once

If we can achieve this, it would be 1 less roundtrip.

Xray and Sing-Box implement this idea under the name Early Data or sometimes
"0-RTT" in changelogs and PRs. Early data is just whatever data the client
wants to write at Step 2.

* Sing-Box by default sends it as part of the URL, but can be configured to use
  any header name (base64-encoded value).

* In Xray, this is always sent as `Sec-WebSocket-Protocol` header. There is a
  pretty obscure reason called [Browser
  Dialer](https://xtls.github.io/en/config/features/browser_dialer.html) for
  this particular choice of header.

It can be said that sending this data in the URL is more compatible with HTTP
proxies, but headers have more capacity for larger data.

This mostly solves the latency concerns around WebSockets.

## HTTPUpgrade

**(getting rid of data framing overhead)**

Remember? When you send `hello` in WebSocket, this is actually transmitted:

```
M-^A^Fhello
```

What's with all this garbage at the front? That's just a waste of bandwidth. We
previously said the WebSocket standard requires this.

But it actually turns out that many CDN do not care if you send actual
WebSocket data over a WebSocket connection. After the first HTTP request and
HTTP response, it seems the socket can be used directly without any overhead at
all.

This is the entire idea behind
[HTTPUpgrade](https://github.com/v2fly/v2ray-core/pull/2727). Originally this
kind of transport was designed by Tor under the name HTTPT, some time after
v2ray added WebSocket. Later v2fly ported it as a new transport, then it got
copied to Xray.

So the transmission becomes this. From the client:

```
GET / HTTP/1.1
Host: localhost:6003
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: MOIjFT7/cVsCCr95mkpCtg==
```

Then, a response from the server:

```
HTTP/1.1 101 Switching Protocols
Sec-WebSocket-Accept: HKN6nOSb0JT0jWhszYuKJPUPpHg=
Connection: Upgrade
Upgrade: websocket
```

...and then, it's used directly as a normal TCP connection, like the first `nc` example.

## HTTPUpgrade 0-RTT

**(getting rid of both issues)**

It works the same way as WebSocket 0-RTT. Now both annoyances of WebSocket are
eliminated: The handshake overhead (mostly), and the message overhead
(entirely)

## Intermission: Changing learning methods

Until this point, tools like `nc` and `websocat` were sufficient to build some
understanding about how those transports work. That was easy enough to do
because WebSocket and TCP are very standardized and not some strange invention
by anti-censorship researchers.

For the next few transports, it is more difficult to emulate them with standard
tools, and we need some inspection tools to keep learning.

Install [mitmproxy](https://docs.mitmproxy.org/stable/overview-installation/)
for the next steps. I am using `Mitmproxy: 10.0.0`, but for as long as you are
not on Debian stable, I think any version will do fine.

After this, let's try looking at some websocket requests. Run these commands in
separate terminals:

1. `websocat -s 3002`
2. `mitmproxy --mode reverse:http://localhost:3002 -p 3001`
3. `websocat ws://localhost:3001`

Instead of websocat talking to websocat directly, it talks to mitmproxy, and
mitmproxy logs and forwards all traffic.

As before, you can enter messages on both ends. In the `mitmproxy` window, you
can hit `Enter` to view the HTTP request, then hit `Tab` to switch to the
`WebSocket Messages` tab. Hit `q` to get back to the list of requests, and `q`,
then `y` to exit the inspector.

Next let's look at SplitHTTP, just because this one is HTTP-based and a good
target for mitmproxy.

## SplitHTTP

I've done my best at a high-level explanation in [the official Xray
docs](https://xtls.github.io/en/config/transports/splithttp.html), so I'm not
even going to try to explain how this protocol works, just tell you how to
intercept its traffic and see what it does.

There is no such thing as `websocat` for SplitHTTP, but actually you can build
a tool like this with xray. In other words, you can use Xray transports without
VLESS or Vmess. Just raw data.

Save as `client.json`:

```json
{
  "log": {
    "access": "/dev/stdout",
    "error": "/dev/stdout",
    "loglevel": "debug"
  },
  "inbounds": [
    {
      "tag": "dkd",
      "listen": "127.0.0.1",
      "port": 5999,
      "protocol": "dokodemo-door",
      "settings": {
        "address": "127.0.0.1",
        "port": 6000,
        "network": "tcp"
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "streamSettings": {
        "network": "splithttp"
      }
    }
  ]
}
```

Save as `server.json`:

```json
{
  "log": {
    "access": "/dev/stdout",
    "error": "/dev/stdout",
    "loglevel": "debug"
  },
  "inbounds": [
    {
      "tag": "dkd",
      "listen": "127.0.0.1",
      "port": 6001,
      "protocol": "dokodemo-door",
      "settings": {
        "address": "127.0.0.1",
        "port": 6002,
        "network": "tcp"
      },
      "streamSettings": {
        "network": "splithttp"
      }
    }
  ],
  "outbounds": [
    {"protocol": "freedom"}
  ]
}
```

Launch, all in separate terminals again:

1. `xray -c client.json`
2. `xray -c server.json`
3. `mitmproxy --mode reverse:http://localhost:6001 -p 6000 --set stream_large_bodies=0m`
4. `nc -l 6002`
5. `nc localhost 5999`

The traffic flow client-to-server is now this:

```
(nc localhost 5999) -> port 5999 (xray client) -> port 6000 (mitmproxy)
-> port 6001 (xray server) -> nc -l 6002)
```

In this setup, xray (client) acts as a port forwarder, but when forwarding the
traffic, it is encoded as splithttp. Then in mitmproxy you can check out how
SplitHTTP works. In xray (server) SplitHTTP is decoded again and the unwrapped
contents is forwarded to `nc -l 6002`.

Enter `hello` into `nc localhost 5999`, hit `Enter` key, and see what happens
in mitmproxy. Xray (client) should send a GET and a POST request. Navigate to
the POST request with your arrow keys, and you should see `Content-Length: 6`.
This is `hello` plus a newline.

Notice that the `GET` request has not finished yet (there is no response time
in the rightmost column). Anything you type into `nc -l 6002` will be streamed
through this never-ending HTTP response. If you kill the `nc` commands, the
`GET` request finishes.

Unfortunately due to the `stream_large_bodies` option, all request and response
bodies seem to be missing in mitmproxy, at least on my machine.

## GRPC

TODO
