---
layout: post
author: "Baiju Muthukadan"
title: "Using Gorilla Websocket"
date: 2015-09-21
categories: web golang
---

## Introduction

Displaying data in real time is a common practice in web applications.
Also sending data back and forth between web client and server is
widely used.  Until recently long polling was a common technique used
for these type applications.  As part of the HTML5, a new application
[protocal](https://tools.ietf.org/html/rfc6455) called Websocket and
corresponding [API](http://www.w3.org/TR/websockets) has been
standardized.  Websocket provides a full-duplex single socket
connection for two-way communication between client and server.
Websocket is [supported](http://caniuse.com/#feat=websockets) in most
of the web browsers.

Go is very good for
[performance](http://dave.cheney.net/2015/08/08/performance-without-the-event-loop)
critical web applications.  This blog gives you an introduction to
implementing Websocket backend in Go using [Gorilla
websocket](http://godoc.org/github.com/gorilla/websocket) package.  An
alternative package would be:
[golang.org/x/net/websocket](http://godoc.org/golang.org/x/net/websocket).

## Installation

To install the package, you can use `go get`:

{% highlight bash %}
go get github.com/gorilla/websocket
{% endhighlight %}

## Client Side API

One of the fundamental abstraction in client side is a Websocket
connection object.  You can create a new connection object using the
WebSocket object.  You need to pass the URL end point for the
websocket.

Then you can define call back functions for handling various events
like `onmessage`, `onopen`, `onerror`, and `onclose`.  These handlers
can registered with the Websocket connection object by setting
attribues.

To understand the event handler registration, see this example:

{% highlight javascript %}
var conn = new WebSocket("ws://localhost:8081/ws")
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>
<html>
  <head><title>Websocket Example</title></head>
  <body>
  </body>
  <script type="text/javascript">
  (function() {
    function onmessage(event) {
      console.log(event.data);
    };
    var conn = new WebSocket("ws://localhost:8081/ws")
    conn.onmessage = onmessage
  })();
  </script>
</html>
{% endhighlight %}

In the above example, we defined a call back function name
`onmessage`.  As you can see this function will be callend with the
event object as the argument.  The data attributes gives the pay load
send by the server.  The payload could be a Unicode string or binary.
We are registering the `onmessage` function to the WebSocket
connection object by setting the attribute named `onmessage`.  (Note:
The call back function could be named anything).

Inside the call back function, we are writing the data send from
server into the console log.  You can open the browser console window
to see the message.

You can save the above HTML into a file and open it your browser.  In
the next section, we will go though the server side code.

## Backend

You can import `websocket` package like this:

{% highlight go %}
import "github.com/gorilla/websocket"
{% endhighlight %}

The `websocket` package provides an upgrader object using which you
can convert a regular HTTP handler to a Websocket handler.  In this
example, we are setting a `CheckOrigin` attribute which will be called
for every request.  The `CheckOrigin` is function used to check the
origin.  If the `CheckOrigin` function returns `false`, then the
`Upgrade` method fails the WebSocket handshake with HTTP status 403.

{% highlight go %}
var upgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool { return true },
}

You can register Websocket handler similar to an HTTP handler.  The
TCP socket that is used by HTTP will also be used for Websocket.

As you can see in the below code, the webserver is listening on `8081`
port and the URL end point for Websocket is `/ws`.

In the handler, a `writer` function is invoked as a goroutine.  This
function will be sending the data to client.  The reader is the
blocking function which waits for any message from the client.

{% endhighlight %}
func serveWs(w http.ResponseWriter, r *http.Request) {
	ws, _ := upgrader.Upgrade(w, r, nil)
	go writer(ws)
	reader(ws)
}

func main() {
	http.HandleFunc("/ws", serveWs)
	http.ListenAndServe(":8081", nil)
}
{% endhighlight %}





{% highlight go %}

{% endhighlight %}



{% highlight go %}

{% endhighlight %}


Here is the complate example::

{% highlight go %}
package main

import (
	"net/http"
	"time"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool { return true },
}

func reader(ws *websocket.Conn) {
	defer ws.Close()
	for {
		if _, _, err := ws.ReadMessage(); err != nil {
			break
		}
	}
}

func writer(ws *websocket.Conn) {
	msgTicker := time.NewTicker(5 * time.Second)
	defer func() {
		msgTicker.Stop()
		ws.Close()
	}()

	for {
		select {
		case <-msgTicker.C:
			if err := ws.WriteMessage(websocket.TextMessage, []byte("hello")); err != nil {
				return
			}
		}
	}
}

func serveWs(w http.ResponseWriter, r *http.Request) {
	ws, _ := upgrader.Upgrade(w, r, nil)
	go writer(ws)
	reader(ws)
}

func main() {
	http.HandleFunc("/ws", serveWs)
	http.ListenAndServe(":8081", nil)
}
{% endhighlight %}
