---
layout: post
title:  "WebSocket"
date:   2020-03-07 00:00:00
categories: protocol

---

### 全双工协议

简而言之，许多应用场景需要全双工的web应用，例如：服务器主动推送和实时交互的应用。这些应用不适合在HTTP协议上实现。最好的方式是使用HTTP建立握手，之后基于TCP做长连接。

与HTTP的关系是：使用了其握手机制和端口号。

> Relationship to TCP and HTTP

>    The WebSocket Protocol is an independent TCP-based protocol.  Its
>    only relationship to HTTP is that its handshake is interpreted by
   HTTP servers as an Upgrade request.

>    By default, the WebSocket Protocol uses port 80 for regular WebSocket
   connections and port 443 for WebSocket connections tunneled over
   Transport Layer Security (TLS) [RFC2818].
   
   [支持WebSocket的浏览器](https://en.wikipedia.org/wiki/Comparison_of_WebSocket_implementations)
   

### 握手

WebSocket通过HTTP/1.1协议的101状态码握手。

**客户端发起握手请求：**

``` http
GET ws://localhost:6060/ws HTTP/1.1
Host: localhost:6060
Connection: Upgrade   # 连接升级
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36
Upgrade: websocket    # 升级到WebSocket
Origin: http://localhost:6060
Sec-WebSocket-Version: 13   # 表示支持的WebSocket版本
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7,ja;q=0.6,zh-TW;q=0.5
Cookie: wp-settings-time-1=1566626634
Sec-WebSocket-Key: Omt7iN7CaEUxYdm+wlVEaA==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

**服务器回应**

``` http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: ggdcU6xoySObbFCh62I2J3eRxu0=
```

* `Sec-WebSocket-Key`是随机的字符串，服务器端会用这些数据来构造出一个SHA-1的信息摘要。把`Sec-WebSocket-Key`加上一个特殊字符串（固定）：`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`，然后计算SHA-1摘要，之后进行BASE-64编码，将结果做为`Sec-WebSocket-Accept`头的值，返回给客户端。如此操作，可以尽量避免普通HTTP请求被误认为Websocket协议。

``` golang
// 固定字符串
var keyGUID = []byte("258EAFA5-E914-47DA-95CA-C5AB0DC85B11")

// 服务器根据客户端请求的随机key计算Sec-WebSocket-Accept字段的值
func computeAcceptKey(challengeKey string) string {
	h := sha1.New()
	h.Write([]byte(challengeKey))
	h.Write(keyGUID)
	return base64.StdEncoding.EncodeToString(h.Sum(nil))
}

// 客户端生成随机key
func generateChallengeKey() (string, error) {
	p := make([]byte, 16)
	if _, err := io.ReadFull(rand.Reader, p); err != nil {
		return "", err
	}
	return base64.StdEncoding.EncodeToString(p), nil
}

```

* `Origin`字段是可选的，通常用来表示在浏览器中发起此Websocket连接所在的页面。


### 协议格式

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

* `FIN`为true表示是最后一个分片。
* `RSV`一般情况都为0。
* `opcode` 定义了`payload data`的类型。
* `mask`为true表示payload是编码处理的。
* `masking-key`编码key。
* `payload length`表示payload的长度。

extend为payload过长时的扩展字段。

客户端发给服务器的包，必须masking：

``` golang

const wordSize = int(unsafe.Sizeof(uintptr(0)))

// 随机生成masking-key
func newMaskKey() [4]byte {
	n := rand.Uint32()
	return [4]byte{byte(n), byte(n >> 8), byte(n >> 16), byte(n >> 24)}
}

func maskBytes(key [4]byte, pos int, b []byte) int {
	// Mask one byte at a time for small buffers.
	if len(b) < 2*wordSize {
		for i := range b {
			b[i] ^= key[pos&3]
			pos++
		}
		return pos & 3
	}

	// Mask one byte at a time to word boundary.
	if n := int(uintptr(unsafe.Pointer(&b[0]))) % wordSize; n != 0 {
		n = wordSize - n
		for i := range b[:n] {
			b[i] ^= key[pos&3]
			pos++
		}
		b = b[n:]
	}

	// Create aligned word size key.
	var k [wordSize]byte
	for i := range k {
		k[i] = key[(pos+i)&3]
	}
	kw := *(*uintptr)(unsafe.Pointer(&k))

	// Mask one word at a time.
	n := (len(b) / wordSize) * wordSize
	for i := 0; i < n; i += wordSize {
		*(*uintptr)(unsafe.Pointer(uintptr(unsafe.Pointer(&b[0])) + uintptr(i))) ^= kw
	}

	// Mask one byte at a time for remaining bytes.
	b = b[n:]
	for i := range b {
		b[i] ^= key[pos&3]
		pos++
	}

	return pos & 3
}
```

### 控制消息

WebSocket定义了3种类型的控制消息（control message）：close, ping and pong。

**close**

`opcode`为0x8

客户端发起的close，`websocket.payload.close.status_code`置为1000，表示normal closure。若不设置，则默认为CloseNoStatusReceived=1005。

服务器收到关闭请求，响应包也发送1000。

客户端收到响应后，执行TCP四次挥手过长，双方关闭TCP连接。

在实际捕获websocket的close code时，应注意以下细节：
* `1006`属于错误码，并不是可从`SetCloseHandler`中捕获到的，因为它没有真正的发送close消息。
* 若websocket由Nginx代理，应考虑Nginx可能会主动发送close的情况，因此应该从读取message的地方去获取断开原因，因为`SetCloseHandler`有可能会接收到Nginx发来的close状态。

**ping**

`opcode`为0x9

发送方 -> 接收方

>  WebSocket为了保持客户端、服务端的实时双向通信，需要确保客户端、服务端之间的TCP通道保持连接没有断开。然而，对于长时间没有数据往来的连接，如果依旧长时间保持着，可能会浪费包括的连接资源。但不排除有些场景，客户端、服务端虽然长时间没有数据往来，但仍需要保持连接。这个时候，可以采用心跳来实现。

> 如果客户端支持ping，最好由客户端发起ping，然后服务器记录时间，超时断开即可。浏览器中没有相关api发送ping给服务器，只能由服务器发ping给浏览器。


**pong**

`opcode`为0xA

接收方 -> 发送方

### 参考

[https://tools.ietf.org/html/rfc6455](https://tools.ietf.org/html/rfc6455)

[https://zh.wikipedia.org/wiki/WebSocket](https://zh.wikipedia.org/wiki/WebSocket)

[https://godoc.org/github.com/gorilla/websocket](https://godoc.org/github.com/gorilla/websocket)

[https://github.com/gorilla/websocket](https://github.com/gorilla/websocket)

[websocket ping/pong timeout](https://github.com/gorilla/websocket/blob/master/examples/chat/client.go)
