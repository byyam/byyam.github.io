---
layout: post
title:  "Mediasoup"
date:   2021-04-10 00:00:00
categories: media

---

# mediasoup


## 基本概念

v3版本源码实现了SFU的基本转发功能，由C++部分的worker和TS部分的信令组成。这两部分之间用Unix domain socket通信，是进程之间的全双工通信方式，基于文件系统，不需要走协议栈，因此必须同机部署。

**peer**

逻辑层抽象，代表了通话的成员。

**room** 

逻辑层通常和router绑定，代表了通话的房间。

**router**

模型层代表了媒体流转发的单元，一个router内包含了transport, producer, consumer这些模型的实例。
记录了transport和流的包含关系，还有producer和consumer的映射关系，主要负责从transport中获取数据包和producer/consumer的流媒体转发。

**transport**

数据层抽象，代表了一路socket层面的数据流，可以承载多个RTP stream。通常一个典型的通话里，上行和下行需要分别建立两个socket连接，就是两个transport。客户端与mediasoup之间的transport通常是`webrtcTransport`。mediasoup之间的routers也可以建立transport实现级联关系，通常是`pipeTransport`。发给GStreamer和ffmepg的为`plainTransport`。

基于socket的webrtcTransport的ICE只支持Lite模式，一方面是因为作者并非将它设计为商业用途，另一方面mediasoup的STUN把自己也当成一个host，简单实现了协议功能。

**producer**

媒体流的生产者。

**consumer**

媒体流的消费者。



## 源码结构

[mediasoup](/images/svg/mediasoup.svg)
<img src="/images/svg/mediasoup.svg">


## 实现协议

WebRTC实现连接的ICE和能力协商的SDP都属于描述性协议，并不严格规定具体的实现。因此信令层面由mediasoup官方提供的[demo](https://github.com/versatica/mediasoup-demo)和[client-cpp](https://github.com/versatica/libmediasoupclient),[client-js](https://github.com/versatica/mediasoup-client)等实现具体基于webrtc的调用，这部分代码主要在demo。数据层则是实现了RTP/RTCP协议的基本传输和反馈能力，这部分代码主要在worker。client-cpp提供了调用webrtc-native的API，client-js则是调用支持webrtc的浏览器端提供的API。client和demo/worker通过私有信令和标准协议通信实现了基本的RTC功能。



``` plantuml
skinparam NoteBackgroundColor white

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
## 动手实现一个Mediasoup client

使用pion提供的webrtc库实现一个可以对接mediasoup的client小工具，使用的语言主要为golang。

### 建立媒体传输通道
因为mediasoup把SDP拆成了几条信令在websocket上协商，因此需要把SDP中的字段拆解成所需要的字段封装在client端和给mediasoup server。

#### ICE

从webrtc transport信令中拿到candidate列表，因为mediasoup是lite的ICE，client端是controlling，mediasoup是controlled，只需要从client去建立连接即可。

``` golang

// 创建ice agent
iceAgent, err := ice.NewAgent(&ice.AgentConfig{
    NetworkTypes: []ice.NetworkType{ice.NetworkTypeUDP4},
})

// 把candidate加入ice的列表里
iceAgent.AddRemoteCandidate(candidate)

// 建立连接
iceConn, err := iceAgent.Dial(context.Background(), rsp.Answer.IceParameters.UsernameFragment, rsp.Answer.IceParameters.Password)

```

至此，和webrtctransport的ice连接就建立了一半，接下来在ice的基础上做dtls密钥交换。

#### DTLS

mediasoup支持双向的dtls连接，dtls之后将密钥导出到ice的connection，对rtp的payload进行加密，媒体数据走的是srtp标准协议。

生成dtls证书和指纹

``` golang

    // 不使用pem证书文件，自签名生成证书，对证书计算摘要
	var tlsCerts []tls.Certificate
	certificate, err := selfsign.GenerateSelfSigned()
	if err != nil {
		return mediasoupFPs, tlsCerts, err
	}
	x509cert, err := x509.ParseCertificate(certificate.Certificate[0])
	if err != nil {
		return mediasoupFPs, tlsCerts, err
	}
	actualSHA256, err := fingerprint.Fingerprint(x509cert, crypto.SHA256)
	if err != nil {
		return mediasoupFPs, tlsCerts, err
	}

```

将dtls指纹参数通过信令发送给mediasoup，然后进行dtls连接即可，dtls可使用server也可以使用client，通过信令参数均可调整。

``` golang

	config := &dtls.Config{
		Certificates:         tlsCerts,
		ExtendedMasterSecret: dtls.RequireExtendedMasterSecret,
		// Create timeout context for accepted connection.
		ConnectContextMaker: func() (context.Context, func()) {
			return context.WithTimeout(context.Background(), 30*time.Second)
		},
		SRTPProtectionProfiles: []dtls.SRTPProtectionProfile{dtls.SRTP_AES128_CM_HMAC_SHA1_80},
	}

    dtlsConn, err := dtls.Server(iceConn, config) // or Client

```

至此，媒体通道webrtc transport已经建立。


### 媒体传输

srtp不需要对整个数据包进行加密，因此收发数据仍然是在ice的connection上进行，但是将dtls的密钥导出用于加密每一个rtp的payload。

``` golang

    // 导出dtls密钥
	srtpConfig := &srtp.Config{
		Profile: srtp.ProtectionProfileAes128CmHmacSha1_80,
	}

	connState := dtlsConn.ConnectionState()
	if err := srtpConfig.ExtractSessionKeysFromDTLS(&connState, false); err != nil {
		return nil, fmt.Errorf("errDtlsKeyExtractionFailed: %v", err)
	}

    // 建立srtp session
    srtpSession, err := srtp.NewSessionSRTP(iceConn, srtpConfig)

    // 建立srtp读写handler
    rtpWriteStream, err := srtpSession.OpenWriteStream()

    // 建立srtcp session
    srtcpSession, err := srtp.NewSessionSRTCP(iceConn, srtpConfig)

    // 建立srtcp读写handler
    rtcpReadStream, err := srtcpSession.OpenReadStream(opt.SSRC)

```

将RTP包的header和payload分别填入handler即可在之前建立的媒体通道上进行收发媒体。





## 参考资料

[源码实现](https://github.com/versatica/mediasoup)

[官网](https://mediasoup.org/)

[官方讨论组](https://mediasoup.discourse.group/)

[golang实现信令层](https://github.com/jiyeyuran/mediasoup-go)

