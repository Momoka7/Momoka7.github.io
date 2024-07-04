---
layout: post
title: "Springboot中的一些Websocket实现"
subtitle: "各种Websocket实现方式"
date: 2024-07-04 14:38:00
author: "Momoka7"
header-style: text
catalog: true
tags:
  - Java
  - Springboot
  - Websocket
  - SocketIO
  - Stomp
---

# WebSocket

## spring-boot-starter-websocket

spring 官方提供的 websocket 实现，**_注意无法直接与 socket.io 连接_**

### 构建连接端点

使用`@ServerEndpoint(value = "/echo")`注解来标注端点路径

```java
import java.io.IOException;
import java.time.Instant;

import jakarta.websocket.CloseReason;
import jakarta.websocket.EndpointConfig;
import jakarta.websocket.OnClose;
import jakarta.websocket.OnError;
import jakarta.websocket.OnMessage;
import jakarta.websocket.OnOpen;
import jakarta.websocket.Session;
import jakarta.websocket.server.ServerEndpoint;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@ServerEndpoint(value = "/echo")
public class EchoChannel {

    private Session session;

    @OnMessage
    public void onMessage(String message) throws IOException {

        log.info("[websocket] 收到消息：id={}，message={}", this.session.getId(), message);

        if (message.equalsIgnoreCase("bye")) {
            // 由服务器主动关闭连接。状态码为 NORMAL_CLOSURE（正常关闭）。
            this.session.close(new CloseReason(CloseReason.CloseCodes.NORMAL_CLOSURE, "Bye"));
            return;
        }

        this.session.getAsyncRemote().sendText("[" + Instant.now().toEpochMilli() + "] Hello " + message);
    }

    @OnOpen
    public void onOpen(Session session, EndpointConfig endpointConfig) {
        // 保存 session 到对象
        this.session = session;
        log.info("[websocket] 新的连接：id={}", this.session.getId());
    }

    // 连接关闭
    @OnClose
    public void onClose(CloseReason closeReason) {
        log.info("[websocket] 连接断开：id={}，reason={}", this.session.getId(), closeReason);
    }

    // 连接异常
    @OnError
    public void onError(Throwable throwable) throws IOException {

        log.info("[websocket] 连接异常：id={}，throwable={}", this.session.getId(), throwable.getMessage());

        // 关闭连接。状态码为 UNEXPECTED_CONDITION（意料之外的异常）
        this.session.close(new CloseReason(CloseReason.CloseCodes.UNEXPECTED_CONDITION, throwable.getMessage()));
    }
}

```

### 注册端点

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

@Configuration
public class WSConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {

        ServerEndpointExporter exporter = new ServerEndpointExporter();

        // 手动注册 WebSocket 端点
        exporter.setAnnotatedEndpointClasses(EchoChannel.class);

        return exporter;
    }
}
```

### 客户端连接

这里使用的是 ws 库

```javascript
// const { WebSocket } = require("");
const path = require("path");
const { WebSocket } = require("ws");
console.log(path.dirname(__filename));

let websocket = new WebSocket("ws://localhost:8888/echo");

// 连接断开
websocket.onclose = (e) => {
  console.log(`连接关闭: code=${e.code}, reason=${e.reason}`);
};
// 收到消息
websocket.onmessage = (e) => {
  console.log(`收到消息：${e.data}`);
};
// 异常
websocket.onerror = (e) => {
  console.log("连接异常");
  console.error(e);
};
// 连接打开
websocket.onopen = (e) => {
  console.log("连接打开");

  // 创建连接后，往服务器连续写入3条消息
  websocket.send("sprigdoc.cn");
  websocket.send("sprigdoc.cn");
  websocket.send("sprigdoc.cn");

  // 最后发送 bye，由服务器断开连接
  websocket.send("bye");

  // 也可以由客户端主动断开
  // websocket.close();
};
```

## Socket.io

并非完全标准的 websocket 实现，java 可使用 netty-socketio，js 使用 socket.io

**_注意版本非常重要！！！_**

### 配置 Controller

`messageSendToUser`为**消息类型**，通过`socketServer.addEventListener`来添加监听器

**可配置广播、房间等功能，待探究**

```java
package com.example.demo;

import com.corundumstudio.socketio.AckRequest;
import com.corundumstudio.socketio.SocketIOClient;
import com.corundumstudio.socketio.SocketIOServer;
import com.corundumstudio.socketio.listener.ConnectListener;
import com.corundumstudio.socketio.listener.DataListener;
import com.corundumstudio.socketio.listener.DisconnectListener;
import lombok.extern.log4j.Log4j2;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
@Log4j2
public class SocketIOController {

    @Autowired
    private SocketIOServer socketServer;

    SocketIOController(SocketIOServer socketServer) {
        this.socketServer = socketServer;

        this.socketServer.addConnectListener(onUserConnectWithSocket);
        this.socketServer.addDisconnectListener(onUserDisconnectWithSocket);

        /**
         * Here we create only one event listener
         * but we can create any number of listener
         * messageSendToUser is socket end point after socket connection user have to
         * send message payload on messageSendToUser event
         */
        this.socketServer.addEventListener("messageSendToUser", String.class, onSendMessage);

    }

    public ConnectListener onUserConnectWithSocket = new ConnectListener() {
        @Override
        public void onConnect(SocketIOClient client) {
            log.info("Perform operation on user connect in controller");
        }
    };

    public DisconnectListener onUserDisconnectWithSocket = new DisconnectListener() {
        @Override
        public void onDisconnect(SocketIOClient client) {
            log.info("Perform operation on user disconnect in controller");
        }
    };

    public DataListener<String> onSendMessage = new DataListener<String>() {
        @Override
        public void onData(SocketIOClient client, String message, AckRequest acknowledge) throws Exception {

            /**
             * Sending message to target user
             * target user should subscribe the socket event with his/her name.
             * Send the same payload to user
             */

            log.info(" user send message to user " + message);
            // socketServer.getBroadcastOperations().sendEvent("messageSendToUser", client,
            // message);
            client.getNamespace().getBroadcastOperations().sendEvent("messageSendToUser", message);

            /**
             * After sending message to target user we can send acknowledge to sender
             */
            acknowledge.sendAckData("Message send to target user successfully");
        }
    };

}
```

### 配置服务器

使用这种方式配置的 socketio 服务器和 springboot 的 ws 不一样，需要额外设置**主机和端口**，一般为自机 ip 和指定的端口，在**配置文件中设置**

```java
package com.example.demo;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.web.bind.annotation.CrossOrigin;

import com.corundumstudio.socketio.Configuration;
import com.corundumstudio.socketio.SocketIOClient;
import com.corundumstudio.socketio.SocketIOServer;
import com.corundumstudio.socketio.listener.ConnectListener;
import com.corundumstudio.socketio.listener.DisconnectListener;

import jakarta.annotation.PreDestroy;
import lombok.extern.log4j.Log4j2;

@CrossOrigin
@Log4j2
@org.springframework.context.annotation.Configuration
public class SocketIOConfig {

    @Value("${socket.host}")
    private String SOCKETHOST;

    @Value("${socket.port}")
    private int SOCKETPORT;
    private SocketIOServer server;

    @Bean
    public SocketIOServer socketIOServer() {
        Configuration config = new Configuration();
        config.setHostname(SOCKETHOST);
        config.setPort(SOCKETPORT);
        server = new SocketIOServer(config);
        server.start();
        server.addConnectListener(new ConnectListener() {
            @Override
            public void onConnect(SocketIOClient client) {
                log.info("new user connected with socket " + client.getSessionId());
            }
        });

        server.addDisconnectListener(new DisconnectListener() {
            @Override
            public void onDisconnect(SocketIOClient client) {
                client.getNamespace().getAllClients().stream().forEach(data -> {
                    log.info("user disconnected " + data.getSessionId().toString());
                });
            }
        });
        return server;
    }

    @PreDestroy
    public void stopSocketIOServer() {
        this.server.stop();
    }
}
```

### 客户端

可通过**消息类型**来收发消息

```javascript
var socket = io.connect("http://127.0.0.1:8889");

socket.on("connect", function () {
  console.log("connected");
});

socket.emit("messageSendToUser", "123"); //发送消息到服务端

socket.on("messageSendToUser", function (data) {
  console.log(data);
});
```

### 命名空间

#### 服务端

通过`SocketIOServer`来添加命名空间

```java
SocketIOServer socketServer;
this.chatNamespace = socketServer.addNamespace("/chat");
```

使用命名空间：一样具有添加监听器等方法

```java
this.chatNamespace.addConnectListener((client) -> {
    log.info("new user connected to chat " + client.getSessionId());
});

this.chatNamespace.addEventListener("chat", String.class, (client, data, ackRequest) -> {
    log.info("user " + client.getSessionId() + " sent " + data);
    this.chatNamespace.getBroadcastOperations().sendEvent("chat", data);
});
```

#### 客户端

在连接时指定连接命名空间，一个实例对应一个命名空间

**默认都会连接到`/`命名空间**

```javascript
import { io } from "socket.io-client";

var socket = io.connect("http://127.0.0.1:8889/chat");

socket.on("connect", function () {
  console.log("connected");
});
```

### 广播/房间

**广播**

对于某个`namspace`来说，广播到命名空间里的客户端

```javascript
this.chatNamespace.getBroadcastOperations().sendEvent("chat", data);
```

对于`SocketIOServer`实例来说，可广播到所有连接客户端

```java
this.socketServer.getBroadcastOperations().sendEvent("chat", data);
```

**房间**

可再缩小广播的范围，需要加入房间的客户端才会收到消息

```java
this.chatNamespace.getRoomOperations("chat")
  .sendEvent("chat", data);
```

由服务端控制客户端加入或离开房间

```java
client.joinRoom("chat");
client.leaveRoom("chat");
```

### 数据传输

### 服务端

在监听器中指定接受数据类型

对于 Java 内置类型，可以直接直接使用**Integer.class、String.class**等

```java
addEventListener("data", String.class, (client, data, ackRequest)=>{}
```

对于复杂类型，须在服务端定义 entity，**字段名和类型需要和客户端发来的 json 一致**

在收到事件消息和数据后，数据会被自动封装

```java
addEventListener("data", Person.class, (client, data, ackRequest)=>{})
```

### 客户端

接受到的数据若为复杂类型，则会以`json`格式的形式封装好，可以直接使用

```java
socket.on("updatacount", (data) => {//这里data可以是json数据
  if (data.owner != selfCount.owner) {
    otherCount.count = data.count;
    otherText.text = otherCount.count;
  }
});
```

### 配置 ipv6 连接地址

```properties
server.port=8888
server.address=::

socket.host = ::
socket.port = 8889
```

在访问时使用`[]`将 ipv6 地址包含起来即可

```
http://[fe80::484b:afb2:19b7:3ac9]:8889/
```

诸如`fe80::484b:afb2:19b7:3ac9%10`这样的 ipv6 形式

网络接口标识符（Scope ID），**用 % 分隔开**。Scope ID 标识了 IPv6 地址所属的特定网络接口，这在特定的网络配置中可能会使用到。

在使用 Socket.IO 或其他网络通信库连接 IPv6 地址时，通常**不需要在地址后面加上网络接口标识符（Scope ID）**

# STOMP

## 是什么

`STOMP（Streaming Text Orientated Message Protocol）`是应用层协议，可以基于 Websocket 或 TCP，可以像`http`协议一样定义协议格式（如 Header）。

客户端可以使用 `SEND `或 `SUBSCRIBE `命令来发送或订阅消息，以及描述消息内容和谁应该收到它的 destination header。这就实现了一个简单的发布-订阅机制，你可以用它来通过 broker 向其他连接的客户端发送消息，或者向服务器发送消息以请求执行某些工作。

使用 Spring 的 STOMP 支持时，Spring WebSocket 应用程序充当客户的 STOMP broker。消息被路由到 @Controller 消息处理方法或简单的内存中 broker，该 broker 跟踪订阅并将消息广播给订阅用户。也可以将 Spring 配置为与专门的 STOMP broker（如 RabbitMQ、ActiveMQ 等）合作，进行消息的实际广播。

使用 STOMP 作为子协议可以让 Spring 框架和 Spring Security 提供更丰富的编程模型，而不是使用原始 WebSockets。

## 使用

在`Springboot`中，`spring-boot-starter-websocket`模块原生支持了 Stomp 功能。

### 配置

消息代理`Broker`的配置，定义连接端口，前缀等

> enableSimpleBroker("/topic")：启用一个简单的内存消息代理来处理以/topic 为前缀的消息。客户端可以订阅这些消息。
>
> setApplicationDestinationPrefixes("/app")：设置应用程序目的地前缀。当客户端发送消息到服务器时，消息应以/app 为前缀，后面的部分将映射到控制器的@MessageMapping 注解的方法。

```java
@Configuration
@EnableWebSocketMessageBroker
public class StompWebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").withSockJS();
    }
}

```

### 消息处理器 Controller

> @MessageMapping("/hello")：映射客户端发送到/app/hello 的消息到这个方法。这里的/app 前缀是通过配置类中的 setApplicationDestinationPrefixes("/app")配置的。
>
> @SendTo("/topic/greetings")：将方法的返回值发送到订阅了/topic/greetings 的客户端。

```javascript
@Controller
public class WebSocketController {

    @MessageMapping("/hello")
    @SendTo("/topic/greetings")
    public String greeting(String message) {
        System.out.println("Received message: " + message);
        return "Hello, " + message + "!";
    }
}
```

### 前端

安装

```
npm install socket-client
npm install stompjs
```

连接

```javascript
const SockJS = require("sockjs-client");
const Stomp = require("stompjs");

var stompClient = null;
var socket = new SockJS("http://localhost:8080/ws");
stompClient = Stomp.over(socket);
stompClient.connect({}, function (frame) {
  console.log("Connected: " + frame);
  setInterval(function () {
    console.log("sending");
    stompClient.send("/app/hello", {}, { a: 1, b: 2 });
  }, 1000);
  stompClient.subscribe("/topic/greetings", function (message) {
    // showMessage(message.body);
    console.log("Received: " + message.body);
  });
});
```
