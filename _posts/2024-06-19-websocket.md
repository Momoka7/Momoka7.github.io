---
layout: post
title: "Cloudflare系列（二）：Cloudflare KV & D1"
subtitle: "cloudflare KV 和 D1 的使用"
date: 2024-06-17 16:50:00
author: "Momoka7"
header-style: text
catalog: true
tags:
  - Cloudflare
  - worker
  - WebSocket
---

# worker 服务端

```java
export default {
	async fetch(request, env, ctx) {
		const upgradeHeader = request.headers.get('Upgrade');
		console.log('An request is received');
		//若不是wss协议则不处理
		if (!upgradeHeader || upgradeHeader !== 'websocket') {
			return new Response('Expected Upgrade: websocket', { status: 426 });
		}
		const webSocketPair = new WebSocketPair();
		//获取客户端-服务端对
		const [client, server] = Object.values(webSocketPair);
		server.accept();
		server.addEventListener('message', (event) => {
			console.log(event.data);
			//使用server向客户端发送数据
			server.send(event.data);
		});
    //切换协议
		return new Response(null, {
			status: 101,
			webSocket: client,
		});
		// return new Response('Hello World!');
	},
};
```

# 客户端

**⚠️ 注意：**在 workers 中部署和测试中，若为**本地 dev 测试**，只能使用`ws`协议连接。

若要部署到公网上，无法直接使用 wokers 所提供的默认路由（ws 和 wss 都无法使用），

需要绑定一个子域名才能使用（在子域名添加 dns 记录后会颁发 ssl/tls 证书）。

```javascript
const WebSocket = require("ws");

const websocket = new WebSocket("ws://127.0.0.1:8787");

websocket.addEventListener("message", (event) => {
  console.log("Message received from server");
  console.log(event.data);
});

websocket.addEventListener("open", (event) => {
  console.log("Connected to server");
  websocket.send("Hello!");
});

websocket.addEventListener("error", (err) => {
  console.log(err);
});

websocket.addEventListener("close", (e) => {
  console.log(e);
});

// 定时发送消息
setInterval(() => {
  websocket.send("periodic message");
}, 1000);
```

# 其他

若要持久化链接对象，可以使用 cloudflare workers 的 Durable Objects 服务（需要氪金的力量）
