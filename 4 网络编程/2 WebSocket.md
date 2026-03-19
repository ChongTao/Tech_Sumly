## 1 WebSocket介绍

WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议。它允许客户端与服务器之间建立持久连接，从而实现低延迟、实时的数据传输。相比传统的 HTTP 请求-响应模式，WebSocket 可以在连接建立后持续进行双向通信，而无需反复建立连接，特点如下：

- 全双工通信：客户端和服务器可以同时发送和接收数据。
- 长连接：连接建立后保持打开状态，避免频繁的 TCP 握手。
- 低延迟：减少 HTTP 轮询带来的开销。
- 轻量协议头：相比 HTTP，WebSocket 数据帧头更小。

## 2 WebSocket 与 HTTP 的区别

| 对比项   | HTTP               | WebSocket |
| -------- | ------------------ | --------- |
| 通信方式 | 单向（请求-响应）  | 双向      |
| 连接方式 | 短连接（或长轮询） | 长连接    |
| 开销     | 较高               | 较低      |
| 实时性   | 一般               | 高        |



## 3 WebSocket 工作原理

### 3.1 建立连接（握手）

WebSocket 使用 HTTP 进行握手，客户端发送 Upgrade 请求：

```html
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: xxxxx
Sec-WebSocket-Version: 13
```

服务器返回：

```html
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: xxxxx
```

握手完成后，协议从 HTTP 升级为 WebSocket。

### 3.2 数据传输

WebSocket 使用帧（Frame）进行通信：

* Text Frame（文本）
* Binary Frame（二进制）
* Ping/Pong（心跳）
* Close（关闭连接）



### 3.3 连接关闭

任意一方发送 Close 帧，双方关闭连接。