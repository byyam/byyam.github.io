---
layout: post
title:  "HttpLongPolling"
date:   2021-05-09 00:00:00
categories: protocol

---


### HTTP推送技术 Push technology 

传统的HTTP请求都是client主动发起到server的请求，若server有通知需要即时通知client时，需要推送技术。

#### 常规轮询 Regular Polling

每隔一段时间pull/get消息。

* 没有消息时浪费资源。
* 有消息时不能即时推送。

#### 长轮询 Long Polling

1. 请求
   
   client发起请求并把keepalive时间设的较长，服务器收到消息并不立即返回，而是hold一段时间。

2. 响应
   
   当有消息推送时，server立即返回消息。或server的connection hang timeout时，返回消息。server返回消息后，将连接关闭。

3. 循环
   
    client收到response后，应立即再次发起请求。

### Long Polling vs Websocket

Long Polling并不是一种全新的协议，而是基于XMLHttpRequest实现了一种推送机制。最大的优点是过渡期的兼容性好，client无需新增协议，容易适配。
同个client若发起多个request连接，消息只返回一次，应通过本地缓存共享推送消息。

websocket协议相对较晚出现，需要client和server端都支持websocket协议，尽管它的握手还是基于HTTP协议，但它在HTTP协议上实现了全双工通信。

websocket几乎解决了Long Polling的所有缺点，只是被断开后不能自恢复。


### Apollo在Long polling的实践

Apollo是携程开源的配置中心，提供了带缓存和不带缓存的pull接口，和long polling即时变更推送接口。

为了避免long polling机制的缺陷，使用了版本号等方案。

* Long polling获取的消息时，需要带上本地缓存的notificationId，server端判断client带过来的notificationId是否为最新，若需要更新则立即返回消息，否则等配置有更新才会返回，返回消息携带notificationId，client端更新缓存。这样做的好处是，即使中间断开连接，恢复连接后也可以对比本地是否需要更新配置信息。

* client使用pull接口获取配置时，可填参数IP表示client的唯一Id，当需要灰度推送时，根据client携带的IP判断是否给它灰度配置。这样做的好处是，使用逻辑上的Id（IP），相同IP的client可以获取到相同的配置。

* client使用pull接口获取配置时，可填参数releaseKey。Apollo每次发布配置时，会为新版本的配置生成releaseKey，使用常规轮询时server端无需每次都查表，可根据releaseKey判断是否需要读取配置。client也无需每次都更新本地缓存的配置。


``` plantuml

start 

repeat :LongPolling;
if (Apollo config normal response?) then (yes)
    switch (http status code?) 
    case (200)
        : update notificationId;
        switch (HTTP get config newest?)
            case (200)
                : update local cache;
            case (304)
                : no change;
        endswitch
    case (304)
        : no change;
    case (other)
        : other;
    endswitch
else (no)
    : Http error;
endif

backward :sleep;
repeat while ()
stop


```

流程图不需要考虑很严谨的逻辑，因为有版本号，即使中间有异常，下次轮询的时候也可以恢复。


### refer

[https://ably.com/blog/websockets-vs-long-polling](https://ably.com/blog/websockets-vs-long-polling)

