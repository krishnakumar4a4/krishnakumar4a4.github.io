+++
authors = [
    "Krishna Kumar Thokala",
]
title = "Websocket fever for IOT"
description = "Websocket fever for IOT"
date = 2018-07-28T09:43:12+05:30
tags = [
    "IOT",
    "Websocket",
    "Https",
    "Http Request",
    "Browsers",
]
categories = [
    "iot",
]
series = ["iot"]
aliases = ["iot"]
images = []
+++

Internet of things is a buzz now. In simpler terms, connecting all the things to the internet so that they are monitored and controlled from anywhere.

What is the most common protocol of internet. It is http. You open a browser enter a url and then you see content on your browser. How is that working? When you enter a url, your browser makes http get request to the site and fetches the content.

Likewise multiple methods are supported for http like post, put, delete etc.

##### How does http work internally?
Http is equipped with lot of headers to actually reach it's destination over the internet. Http runs on tcp protocol and every request makes a connection and closes it after the response.

##### How does WebSocket works?
WebSocket is a simple tcp based protocol which does the initial handshake by http and keeps a persistent tcp connection to the server.

Unlike http, it is a duplex connection that allows client and server push frames from either side. Hence allowing the client and server to communicate in more real-time.

Except the typical initial handhsake which will be in http, rest of the communication is headerless and lightweight.

WebSockets are supported on every browser thus allowing you to fetch data in real time without using traditional http polling mechanisms that works by pull approach where the client asks for any updates server has and makes and breaks connection for every single request, which is overhead for both client and servers.

Tcp connection is not a standard for browsers and hence having a protocol as light as tcp other than the traditional http is very useful.

Http has certain advantages like requests can be cached by intermediate proxy servers there by reducing the load on the actual server when serving the same response to multiple clients. However this kind of caching is not supported by websockets, but it is highly unusable for a scenario where websockets has to be used. When things change in real time, caching is of least importance.

##### WebSockets for IOT:
Given the need for visualizing lot of details on web dashboards in real time. Advent of cloud platforms for IOT etc. WebSockets would be a ideal choice for having lightweight communication between server and client.
###### A typical home IOT environment:
You might be having lot of devices at your home which needed to be visualized and controlled when you are away from your home or you might be lazy getting up from your couch to turn off something or you want your grandparents to have complete control of your home at their fingers and so on.
In a typical gateway based IOT architecture, your gateway is always connected to your server to see the current status of devices over the internet. Your server can send commands back to the gateway only if there is a two way communication. Doing the same thing over http will involve your gateway constantly polling for commands from server. WebSockets makes your life easier.
Let's say you wanted to offer your IOT system for all the residents of your apartment as a service. WebSocket would be a right choice in maintaining just one connection per gateway per flat. Otherwise, using http, your server will be flooded with lot of connections and disconnections for every poll per gateway per flat more than the data/commands to be sent across.
###### Real-time audio streaming environment:
I have built an application that allows people within a wifi network to talk from their mobile web browser. This requires a real low latency audio streaming from the browser itself. WebSockets are useful here as it needed binary data streaming in real time. Using a traditional http for each and every frame would impact latency as it makes and breaks connection for each and every frame, WebSockets on the other hand makes connection once and streams data on that connection continuously. Also, the same WebSocket can be used to push messages,notifications and even transpiled string asynchronously to all the clients from the server.

A self driving car, pizza delivery service, realtime ambulance guiding system to accident spot etc. There are many usecase where websockets can be or will be or being used. See if your next usecase needs a WebSocket.