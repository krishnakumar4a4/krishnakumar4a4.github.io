---
title: TcpProxy
date: 2019-10-06
images: []
tags: ["golang"]
githubUrl: "https://github.com/krishnakumar4a4/tcpproxy"
summary: "A simple TCP interceptor proxy for HTTP/S requests with network speed caclulator"
---
A simple proxy server for HTTP/S requests which can calculate the network upload and download speed in bytes/sec.

```./tcpproxy --help``` for usage

```./tcpproxy --port 2345``` to listen on particular port with chained proxy assumed to be running localhost:3128

```./tcpproxy --proxy --no-proxy``` to listen on particular port with no chained proxy