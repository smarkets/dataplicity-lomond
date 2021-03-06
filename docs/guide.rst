Introduction
------------

Lomond is a websocket client library designed to make adding websocket
support to your application as tranquil as the `Scottish Loch
<https://en.wikipedia.org/wiki/Loch_Lomond>`_ it was named after.


Installing
----------

You can install Lomond with ``pip`` as follows::

    pip install lomond

Or to upgrade to the most recent version::

    pip install lomond --upgrade

Alternatively, if you would like to install from source, check
out `the code from Github <https://github.com/wildfoundry/dataplicity-
lomond>`_.

You may wish to install `wsaccel`, which is a C module containing
optimizations for some websocket operations. Lomond will use it if
available::

    pip install wsaccel

Basic Usage
-----------

To connect to a websocket server, first construct a
:class:`~lomond.websocket.WebSocket` object, with a `ws://` or `wss://` URL.
Here is an example::

    from lomond.websocket import WebSocket
    ws = WebSocket('wss://echo.websocket.org')

No socket connection is made by a freshly constructed WebSocket object.
To connect and interact with a websocket server, iterate over the
WebSocket instance, which will yield a number of
:class:`~lomond.event.Event` objects. Here's an example::

    for event in ws:
        print(event)

Here is an example of the output you might get from the above
code::

    Connecting(url='wss://echo.websocket.org')
    Connected(url='wss://echo.websocket.org')
    Ready(<response HTTP/1.1 101 Web Socket Protocol Handshake>, protocol=None, extensions=set([]))

The :class:`~lomond.events.Ready` event indicates a successful
connection to a websocket server. You may now use the
:meth:`~lomond.websocket.WebSocket.send_text` and
:meth:`~lomond.websocket.WebSocket.send_binary` methods to send data to
the server.

When you receive data from the server, a :class:`~lomond.events.Text` or
:class:`~lomond.events.Binary` event will be generated.

Iterating over the WebSocket instance in this way calls
:meth:`~lomond.websocket.WebSocket.connect` with default parameters, i.e. it is
equivalent to the following::

    for event in ws.connect():
        print(event)

You may want to call `connect` manually to change the default behavior.

Events
------

Events inform your application when data is received from the server or
when the websocket state changes.

All events are derived from :class:`~lomond.events.Event` and will
contain at least 2 attributes; `received_time` is the epoch time the
event was received, and `name` is the name of the event. Some events
have additional attributes with more information. See the :ref:`events`
for details.

When handling events, you can either check the type with `isinstance` or
by looking at the `name` attribute.

For example, the following two lines are equivalent::

    if isinstance(event, events.Ready):

or::

    if event.name == "ready":

.. note::
    The `isinstance` method is possibly uglier, but has the advantage
    that you are less likely to introduce a bug with a typo in the event
    name.

If an event is generated that you aren't familiar with, then you should
simply ignore it. This is important for backwards compatibility; future
versions of Lomond may introduce new event types.

Be careful with code that responds to events. Should there be an
unhandled exception within the event loop, Lomond will disconnect the
socket without sending a close packet. It's up to your application to
ensure that programming errors don't prevent the websocket from
closing gracefully.

You may wish to adopt an defensive approach to handling WebSocket
events, such as the following::

    for event in websocket:
        try:
            on_event(event)
        except:
            log.exception('error handling %r', event)
            websocket.close()


Closing the Websocket
---------------------

The websocket protocol specifies how to close the websocket cleanly. The
procedure for handling closes, depends on whether it is initiated by the
client or the server.

Client
++++++

To close a websocket, call the :meth:`~lomond.websocket.Websocket.close`
method to initiate a *websocket close handshake*. You may call this
method from within the websocket loop, or from another thread.

When you call :meth:`~lomond.websocket.Websocket.close`, Lomond sends a
close packet to the server. The server will respond by sending a close
packet of its own. Only when this echoed close packet is received will
the WebSocket close the underlaying tcp/ip socket. This allows both ends
of the connection to finish what they are doing without worrying the
remote end has stopped responding to messages.

.. note::
    When you call the `close()` method, you will no longer be able to
    *send* data, but you may still *receive* packets from the server
    until the close has completed.

When the websocket has been closed, you will receive a
:class:`~lomond.events.Closed` event, followed by a
:class:`~lomond.events.Disconnected` event, and the event loop will
exit.

It's possible a malfunctioning server may not respond to a close packet,
which would leave a WebSocket in a permanent *closing* state. As a
precaution, Lomond will force close the socket after 30 seconds, if the
server doesn't respond to a close packet. You can change or disable this
timeout with the `close_timeout` parameter, on
:meth:`~lomond.websocket.Websocket.connect`.

Server
++++++

The websocket server will send a close packet when it wished to close.
When Lomond receives that packet, a :class:`~lomond.events.Closing`
event will be generated. You may send text or binary messages in
response to the Closing event, but afterwards Lomond echos the close
packet and no further data may be sent. The server will then close the
socket, and you will receive a :class:`~lomond.events.Disconnected`
event, followed by the event loop ending.

Non-graceful Closes
+++++++++++++++++++

A non-graceful close is when a the tcp/ip connection is closed *without*
completing the closing handshake. This can occur if the server is
misbehaving or if connectivity has been interrupted.

The :class:`~lomond.events.Disconnected` event contains a boolean
attribute `graceful`, which will be `False` if the closing handshake was
not completed.

Pings and Pongs
---------------

Both the server and client may send 'ping' packets, which should be
responded to with a 'pong' packet. This allows both ends of the
connection to know if the other end is really listening.

By default, Lomond will send pings packets every 30 seconds. If you wish
to change this rate or disable ping packets entirely, you may use the
:meth:`~lomond.websocket.connect` method.

Here's how you would disable pings::

    websocket = Websocket('wss://ws.example.org')
    for event in WebSocket.connect(ping_rate=0):
        on_event(event)

Lomond will also automatically respond to ping requests. Since this is a
requirement of the websocket specification, you probably don't want to
change this behaviour. But it may be disabled with the `auto_pong` flag
in :meth:`~lomond.websocket.WebSocket.connect`.

Regardless of whether *auto pong* is enabled, a
:class:`~lomond.events.Pong` event will be generated when Lomond
receives a ping packet. If auto pong *is* disabled, you should manually
call :meth:`~lomond.websocket.send_pong` in response to a ping, or the
server may disconnect you.

Polling
-------

Lomond checks for automatic pings and performs other housekeeping tasks
at a regular intervals. This *polling* is exposed as
:class:`~lomond.events.Poll` events. Your application can use these
events to do any processing that needs to be invoked at regular
intervals.

The default poll rate of 5 seconds is granular enough for Lomond's
polling needs, while having negligible impact on CPU. If your
application needs to process at a faster rate, you may set the `poll`
parameter of :meth:`~lomond.websocket.WebSocket.connect`.

.. note::
    If your application needs to be more realtime than polling once a
    second, you should probably use threads in tandem with the event
    loop.

WebSockets and Threading
------------------------

WebSocket objects are *thread safe*, but Lomond does not need to launch
any threads to run a websocket. For many applications, responding to
data and poll events is all you will need. However, if your application
needs to do more than communicate with a websocket server, you may want
to run a websocket in a thread of its own.

Persistent Connections
----------------------

Lomond supports a simple mechanism for persistent connections -- you can
tell Lomond to continually retry a websocket connection if it is dropped
for any reason. This allows an application to maintain a websocket
connection even if there are any outages in connectivity.

To run a persistent connection, wrap a WebSocket instance with
:func:`~lomond.persist.persist`. Here is an example::

    from lomond.persist import persist
    websocket = WebSocket('wss://ws.example.org')
    for event in persist(websocket):
        # handle event

You will receive events as normal with the above loop.

If the connection is dropped for any reason, you will receive
:class:`~lomond.events.Disconnected` as usual, followed by
:class:`~lomond.events.Connecting` when Lomond retries the connection.
Lomond will keep retrying the connection until it is successful, and
a :class:`~lomond.events.Ready` event is generated.

The :func:`~lomond.persist.persist` function implements *exponential
backoff*. If the websocket object fails to connect, it will wait for a
random period between zero seconds and an upper limit. Every time the
connection fails, it will double the upper limit until it connects, or a
maximum delay is reached.

The exponential backoff prevents a client from hammering a server that
may already be overloaded. It also prevents the client from being stuck
in a cpu intensive spin loop.

