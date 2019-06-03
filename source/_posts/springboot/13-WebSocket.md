title: WebSocket（待续）

tags:
  - SpringBoot

categories:
  - SpringBoot
---
## 诞生背景
对于需要实时响应、高并发的应用，传统的请求-响应模式的 Web的效率不是很好。HTML5规范中的(有 Web TCP 之称的) WebSocket ，就是一种高效节能的双向通信机制来保证数据的实时传输。

## 运行原理
WebSocket 是 HTML5 一种新的协议。它建立在 TCP 之上，实现了客户端和服务端全双工异步通信，和 HTTP 最大不同是：
  - WebSocket服务器和 Browser/Client Agent 都能主动的向对方发送或接收数据；
  - WebSocket 需要类似TCP的客户端和服务器端通过握手连接，连接成功后才能相互通信。
传统http每次请求-应答都需要客户端与服务端建立连接，如图：
![](/img/springboot/http.png)

WebSocket一旦连接建立后，后续数据都以帧序列的形式传输。在客户端断开 WebSocket连接或 Server端断掉连接前，不需要客户端和服务端重新发起连接请求，这样保证websocket的性能优势，实时性优势明显。如图：
![](/img/springboot/websocket.png)

## 链接
- [SpringBoot+websocket+定时任务（如何及时实时响应服务端数据）](https://segmentfault.com/a/1190000016201055)