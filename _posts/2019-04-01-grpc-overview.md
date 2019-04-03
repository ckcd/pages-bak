---
title: gRPC Overview
tags: gRPC RPC Golang
key: grpc-overview
---

本文结合gRPC和Golang，通过具体的实例对gRPC的基本原理、使用方法、代码组成等进行介绍。

<!--more-->

# 什么是gRPC

gRPC由 google 开发，是一款语言中立、平台中立、开源的远程过程调用(RPC)系统。

在 gRPC 里客户端应用可以像调用本地对象一样直接调用另一台不同的机器上服务端应用的方法，使得用户能够更容易地创建分布式应用和服务。与许多 RPC 系统类似，gRPC 也是基于以下理念：定义一个服务，指定其能够被远程调用的方法（包含参数和返回类型）。在服务端实现这个接口，并运行一个 gRPC 服务器来处理客户端调用。在客户端拥有一个存根能够像服务端一样的方法。

![gRPC](https://grpc.io/img/landing-2.svg)

## 特性

- 基于HTTP/2：HTTP/2 提供了连接多路复用、双向流、服务器推送、请求优先级、首部压缩等机制。可以节省带宽、降低TCP链接次数、节省CPU，帮助移动设备延长电池寿命等。gRPC 的协议设计上使用了HTTP2 现有的语义，请求和响应的数据使用HTTP Body 发送，其他的控制信息则用Header 表示。
- IDL使用ProtoBuf：gRPC使用ProtoBuf来定义服务，ProtoBuf是由Google开发的一种数据序列化协议（类似于XML、JSON、hessian）。ProtoBuf能够将数据进行序列化，并广泛应用在数据存储、通信协议等方面。压缩和传输效率高，语法简单，表达力强。
- 多语言支持(C, C++, Python, PHP, Nodejs, C#, Objective-C、Golang、Java)：gRPC支持多种语言，并能够基于语言自动生成客户端和服务端功能库。目前已提供了C版本grpc、Java版本grpc-java 和 Go版本grpc-go，其它语言的版本正在积极开发中，其中，grpc支持C、C++、Node.js、Python、Ruby、Objective-C、PHP和C#等语言，grpc-java已经支持Android开发。

## 优点

- protobuf二进制消息，性能好/效率高（空间和时间效率都很不错） 
- proto文件生成目标代码，简单易用 
- 序列化反序列化直接对应程序中的数据类，不需要解析后在进行映射(XML,JSON都是这种方式) 
- 支持向前兼容（新加字段采用默认值）和向后兼容（忽略新加字段），简化升级 
- 支持多种语言（可以把proto文件看做IDL文件） 
- Netty等一些框架集成

## 缺点

- GRPC尚未提供连接池，需要自行实现 
- 尚未提供“服务发现”、“负载均衡”机制 
- 因为基于HTTP2，绝大部多数HTTP Server、Nginx都尚不支持，即Nginx不能将GRPC请求作为HTTP请求来负载均衡，而是作为普通的TCP请求。（nginx1.9版本已支持） 
- Protobuf二进制可读性差（貌似提供了Text_Fromat功能） 
- 默认不具备动态特性（可以通过动态定义生成消息类型或者动态编译支持）

# gRPC通信方式

## Simple RPC

A *simple RPC* where the client sends a request to the server using the stub and waits for a response to come back, just like a normal function call.

```protobuf
// Obtains the feature at a given position.
rpc GetFeature(Point) returns (Feature) {}
```

## server-side streaming RPC

A *server-side streaming RPC* where the client sends a request to the server and gets a stream to read a sequence of messages back. The client reads from the returned stream until there are no more messages. As you can see in our example, you specify a server-side streaming method by placing the stream keyword before the response type.

```protobuf
// Obtains the Features available within the given Rectangle.  Results are
// streamed rather than returned at once (e.g. in a response message with a
// repeated field), as the rectangle may cover a large area and contain a
// huge number of features.
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

## client-side streaming RPC

A *client-side streaming RPC* where the client writes a sequence of messages and sends them to the server, again using a provided stream. Once the client has finished writing the messages, it waits for the server to read them all and return its response. You specify a client-side streaming method by placing the stream keyword before the request type.

```protobuf
// Accepts a stream of Points on a route being traversed, returning a
// RouteSummary when traversal is completed.
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

## bidirectional streaming RPC

A *bidirectional streaming RPC* where both sides send a sequence of messages using a read-write stream. The two streams operate independently, so clients and servers can read and write in whatever order they like: for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes. The order of messages in each stream is preserved. You specify this type of method by placing the stream keyword before both the request and the response.

```protobuf
// Accepts a stream of RouteNotes sent while a route is being traversed,
// while receiving other RouteNotes (e.g. from other users).
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

# Golang 实例

## Hello World

本实例来自官网介绍：[Quick Start](https://grpc.io/docs/quickstart/go.html#install-grpc)

### `proto`

首先是定义的`.proto`文件，其中包含了我们所需的请求（`HelloRequest`）和响应（`HelloRequest`）的字段内容，以及一个将请求和响应连接起来的`RPC Service`。

$ helloworld.proto
```protobuf
syntax = "proto3";

option java_multiple_files = true;
option java_package = "io.grpc.examples.helloworld";
option java_outer_classname = "HelloWorldProto";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

使用`protoc`工具编译该文件，从而生成对应的`.pb.go`的文件。

> 依赖项：   
> ```go
> // 安装 gRPC
> go get -u google.golang.org/grpc
> // 安装protoc的Go插件: protoc-gen-go
> go get -u github.com/golang/protobuf/protoc-gen-go
> // 确保该插件在GOPATH中
> export PATH=$PATH:$GOPATH/bin
> ```

```bash
protoc -I helloworld/ helloworld/helloworld.proto --go_out=plugins=grpc:helloworld
```

### `server`

server端应该实现两部分内容：

1. 实现上一步中生成的 helloworld.GreeterServer 接口；
2. 监听；

helloworld.GreeterServer 接口在本实例中即 `SayHello` 方法：

```go
// server is used to implement helloworld.GreeterServer.
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.Name)
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}
```

监听指定的端口，并且调用`grpc`的`NewServer`方法在该端口提供服务。

```go
func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

### `client`

在客户端，首先根据IP地址和端口建立连接，使用`proto`中生成的代码封装该连接之后，只需要像调用本地的方法一样来调用gRPC远程的方法。

建立连接：

```go
func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)
}
```

调用方法：

```go
func main() {
	// Contact the server and print out its response.
	name := defaultName
	if len(os.Args) > 1 {
		name = os.Args[1]
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.Message)
}
```