---
layout: post
title:  "Java 8 hibernate troubleshoot"
date:   2016-08-31 14:17:06 +0530
category: troubleshooting
tags: [java, java 8, jvm, hibernate]
---


# Mysql connection issue
**NoRouteToHostException exception raised**
>Signals that an error occurred while attempting to connect a socket to a remote address and port. Typically, the remote host cannot be reached because of an intervening firewall, or if an intermediate router is down.

Error
> The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.

```java
Caused by: java.net.NoRouteToHostException: Cannot assign requested address
    at java.net.PlainSocketImpl.socketConnect(Native Method)
    at java.net.PlainSocketImpl.doConnect(PlainSocketImpl.java:333)
```

Fix: 
```bash
echo "1" >/proc/sys/net/ipv4/tcp_tw_reuse
    and/or
echo "1" >/proc/sys/net/ipv4/tcp_tw_recycle
```