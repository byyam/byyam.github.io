---
layout: post
title:  "Mediasoup"
date:   2021-04-10 00:00:00
categories: media

---

# mediasoup


## 基本概念

v3版本源码实现了SFU的基本转发功能，由C++部分的worker和TS部分的信令组成。这两部分之间用Unix domain socket通信，是进程之间的全双工通信方式，基于文件系统，不需要走协议栈，因此必须同机部署。

**router**

逻辑层抽象，代表了通话的房间。

**peer**

逻辑层抽象，代表了通话的成员。

**transport**

数据层抽象，代表了一路socket层面的数据流，可以承载多个RTP stream。通常一个典型的通话里，上行和下行需要分别建立两个socket连接，就是两个transport。客户端与mediasoup之间的transport通常是`webrtcTransport`。mediasoup之间的routers也可以建立transport实现级联关系，通常是`pipeTransport`。发给GStreamer和ffmepg的为`plainTransport`。

基于socket的transport在实际业务中非常浪费端口资源，一方面是因为作者并非将它设计为商业用途，另一方面mediasoup的STUN把自己也当成一个host，简单实现了协议功能。

## 源码结构

[mediasoup](/images/svg/mediasoup.svg)
<img src="/images/svg/mediasoup.svg">


## 实现协议

WebRTC实现连接的ICE和能力协商的SDP都属于描述性协议，并不严格规定具体的实现。因此信令层面由mediasoup官方提供的[demo](https://github.com/versatica/mediasoup-demo)和[client-cpp](https://github.com/versatica/libmediasoupclient),[client-js](https://github.com/versatica/mediasoup-client)等实现具体基于webrtc的调用，这部分代码主要在demo。数据层则是实现了RTP/RTCP协议的基本传输和反馈能力，这部分代码主要在worker。client-cpp提供了调用webrtc-native的API，client-js则是调用支持webrtc的浏览器端提供的API。client和demo/worker通过私有信令和标准协议通信实现了基本的RTC功能。



``` plantuml
webrtc --> client : getMediaDevice 
client -> demo : getRouterRtpCapabilities
activate demo
demo -> client : 返回mediasoup支持的RTP能力
deactivate demo
client --> webrtc : set SDP

client -> demo : createWebRtcTransport(up/down)
activate demo
demo -> mediasoup : router.createWebRtcTransport
activate mediasoup
note right : 创建上下行两个Transport
mediasoup -> demo : ICE/DTLS/SCTP参数
deactivate mediasoup
demo -> client : ICE/DTLS/SCTP参数
deactivate demo
client --> webrtc : set SDP

client -> demo : join
activate demo
demo -> client : other peers
demo -> mediasoup : transport.consume
deactivate demo
activate mediasoup
note right : 在Transport上建立consumer的映射
mediasoup -> demo : producer和consumer状态
deactivate mediasoup
demo -> client : 发给自己newConsumer(audio/video)
note right : 把新进房成员当做其他已经进房成员的下行

webrtc --> client : DTLS参数
client -> demo : connectWebRtcTransport(up/down)
activate demo
demo -> mediasoup : transport.connect
activate mediasoup
note right : 设置DTLS参数
mediasoup -> demo : ok
deactivate mediasoup
demo -> client : ok
deactivate demo
client --> webrtc : ok

client -> demo : produce(audio/video)
activate demo
demo -> mediasoup : transport.produce
activate mediasoup
mediasoup -> demo : producer RTP参数
deactivate mediasoup
demo -> client : RTP参数
demo -> mediasoup : transport.consume
deactivate demo
activate mediasoup
note right : 在Transport上建立consumer的映射
mediasoup -> demo : producer和consumer状态
deactivate mediasoup
demo -> client : 通知其他成员newConsumer(audio/video)
note right : 有新上行，给其他成员创建下行转发


```


## 参考资料

[源码实现](https://github.com/versatica/mediasoup)

[官网](https://mediasoup.org/)

[官方讨论组](https://mediasoup.discourse.group/)

[golang实现信令层](https://github.com/jiyeyuran/mediasoup-go)

