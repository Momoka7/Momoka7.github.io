---
layout: post
title: "gRPC入门"
subtitle: "gRPC基础使用"
date: 2024-06-05 16:41:00
author: "Momoka7"
header-style: text
catalog: true
tags:
  - gRPC
  - Python
  - Javascript
---

# 原理

提供函数接口，使得调用其他服务器中的服务就像调用本地代码一样

![截图](/img/in-post/gRPC/3b18db4c5040ed5a091c108bc04f15d9.png)

![截图](/img/in-post/gRPC/6a60a4fb5b27deab3bbac2afa78dfe90.png)

## proto 文件格式

![截图](/img/in-post/gRPC/9a073a31367b0abe447797d3659a1e27.png)

**IDL 文件定义服务和消息类型： **开发人员使用 gRPC 的 IDL（Interface Definition Language）文件来定义服务和消息类型。这些文件使用 Protocol Buffers 语言编写，其中包含了服务的方法以及方法参数和返回值的消息类型定义。

**代码生成器生成客户端和服务器代码： **开发人员使用 gRPC 工具的代码生成器，将 IDL 文件编译成各种编程语言的客户端和服务器代码。这些生成的代码包括客户端和服务器端的 stub 类，这些 stub 类使得开发人员可以直接调用远程服务方法。

**基于 HTTP/2 的双向流式传输：** gRPC 使用 HTTP/2 作为传输协议。HTTP/2 支持双向流式传输，可以在单个连接上并行发送和接收多个消息。这使得 gRPC 可以高效地处理大量的请求和响应，并且支持实时通信等场景。

**支持多种序列化格式：** gRPC 使用 Protocol Buffers 作为默认的序列化格式，这使得数据在传输过程中可以更紧凑地编码。此外，gRPC 也支持其他的序列化格式，如 JSON，以满足不同语言和应用的需求。

**支持多语言：** gRPC 支持多种编程语言，包括但不限于 C/C++, Java, Python, Go, JavaScript 等。这意味着您可以在不同的编程语言中使用相同的服务定义，并且可以相互调用。

# 示例

js 作为客户端，python 作为服务端

## proto 文件

```protobuf
// greeter.proto
syntax = "proto3";

package greeter;

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
    string name = 1;
}

message HelloReply {
    string message = 1;
}

```

## js

需安装@grpc/grpc-js

```javascript
const grpc = require("@grpc/grpc-js");
const protoLoader = require("@grpc/proto-loader");

const packageDefinition = protoLoader.loadSync("test.proto", {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
}); //读取proto文件

const testDef = grpc.loadPackageDefinition(packageDefinition);
const test = testDef.greeter; //解析proto文件，指定package

const client = new test.Greeter(
  "localhost:50051",
  grpc.credentials.createInsecure()
); //指定ip和端口连接
const name = "afdafa"; //参数

client.SayHello({ name }, (err, response) => {
  //调用rpc方法
  if (!err) {
    console.log("Greeting:", response.message);
  } else {
    console.error("Error:", err);
  }
});
```

## python

安装 grpcio 和 grpcio-tool

使用 protoc 编译器和 gRPC 插件来生成 Python 文件。运行以下命令：

```
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. greeter.proto
```

对`greeter.proto`文件生成协议文件：消息类型的定义（在 greeter_pb2.py 中）和 gRPC 服务的定义（在 greeter_pb2_grpc.py 中）

**构建 gRPC 服务端：**

这里使用 test.proto 生成的 gRPC 协议来编写一个 gRPC 服务

```python
import grpc
import test_pb2
import test_pb2_grpc
from concurrent import futures

class Greeter(test_pb2_grpc.GreeterServicer): # 继承服务类
    def SayHello(self, request, context): # 实现服务中的方法
        # 返回的消息类型定义在test_pb2中
        # 客户端请求的参数在request中
        return test_pb2.HelloReply(message='Hello, %s!' % request.name)

def serve(): # 启动服务
    # 创建服务器，处理并发请求
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    # 添加实现了 GreeterServicer 接口的 Greeter实例到gRPC服务器中
    test_pb2_grpc.add_GreeterServicer_to_server(Greeter(), server)
    # 指定服务器监听的端口
    server.add_insecure_port('[::]:50051') # 指定端口
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

### 空类型的处理

在 protobuf 中：

```protobuf
import "google/protobuf/empty.proto"; // 导入 Empty 类型

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
    // 使用 Empty 类型作为请求消息类型
    rpc GetList (google.protobuf.Empty) returns (ListReply) {}
}
```
