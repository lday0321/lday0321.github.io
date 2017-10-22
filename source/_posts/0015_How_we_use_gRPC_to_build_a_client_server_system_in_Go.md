---
title: 我们如何在Go中使用gRPC构建C/S结构系统
date: 2017-10-22 23:14:14
tags: 
	- golang
	- gRPC
---

RPC作为分布式系统节点中较为主流的一种通信方式，因其简单、方便的调用模式而深受开发者的喜爱。gRPC作为google推出的RPC调用框架，更是受到大家的关注。gRPC原生提供了对Golang的支持，最近，我也在关注gRPC的使用，本文是我整理的*[《How we use gRPC to build a client/server system in Go》](https://medium.com/pantomath/how-we-use-grpc-to-build-a-client-server-system-in-go-dd20045fa1c2)*翻译:

<!-- more -->

这篇文章是一篇关于我们如何在Go中使用[gRPC](https://grpc.io/)(和[Protobuf](https://developers.google.com/protocol-buffers/))构建鲁棒的C/S结构系统的技术介绍。

我不会详细介绍为什么我们选择gRPC作为我们的客户端与服务器端之间的主要通信协议，已经有很多非常好的文章涵盖了这一主题（像是[这一篇](https://grpc.io/blog/vendastagrpc)，[这一篇](https://stackoverflow.com/questions/43682366/how-is-grpc-different-from-rest)，或是[这一篇](https://thenewstack.io/grpc-lean-mean-communication-protocol-microservices/)）。但我们可以从整体上来看一看：我们正在使用Go构建我们的C/S结构系统，我们需要他足够快，足够可靠，足够具有伸缩性（这也是为什么我们选择gRPC）。我们希望客户端与服务端之间的通信越小越好，越安全越好，同时，C/S两端的使用模式要一致（这也是为什么我们选择Protobuf）

我们也希望能够在服务端暴露其他类型的接口，以防有些客户端类型无法使用gRPC：例如暴露传统的REST接口。我们希望这一要求（几乎）没有额外的代价。

## 说明

我们将开始使用Go构建一个非常简单的C/S系统，系统将会在客户端、服务端之间交换虚拟的消息。在完成第一步，客户端、服务端之间理解了对方的消息后，我们将会加入其它的特性，例如TLS支持，认证，以及一个REST API。

文章的接下去部分假设你具有基本的Go语言编程能力。同时，也假设你已经安装了`protobuf`包，`protoc`命令是可用的（再次说明，已经有很多文章涵盖了介绍如何安装的主题，同时，这里也有一份[官方文档](https://developers.google.com/protocol-buffers/docs/proto3)）。

你也需要安装Go依赖库，例如[protobuf的go实现](http://github.com/golang/protobuf)，还有[gRPC网关](https://github.com/grpc-ecosystem/grpc-gateway)


这篇文章中的所有代码在：[https://gitlab.com/pantomath-io/demo-grpc](https://gitlab.com/pantomath-io/demo-grpc)。你可以随意使用这个仓库，并通过标签来导航、定位。这个仓库应当放置在你`$GOPATH`的`src`目录:

```bash
$ go get -v -d gitlab.com/pantomath-io/demo-grpc
$ cd $GOPATH/src/gitlab.com/pantomath-io/demo-grpc
```

## 定义协议

首先，我们需要定义协议。例如，定义在客户端和服务端我们能交互些什么，以及如何交互。这就是Protobuf发挥作用的地方。它让你定义两类东西：服务(Service)和消息(Message)。一个`service`是服务端的一组动作集合，服务端针对客户端的请求执行并产生不同的响应。一个`message`就是一个客户端请求的内容。简单来说，你可以认为`service`定义了**动作**，而`message`定义了**对象**

在[api/api.proto](https://gitlab.com/pantomath-io/demo-grpc/blob/init-protobuf-definition/api/api.proto)中写入如下内容：

```
syntax = "proto3";
package api;
message PingMessage {
  string greeting = 1;
}
service Ping {
  rpc SayHello(PingMessage) returns (PingMessage) {}
}
```

你定义了两个东西：一个叫做Ping的`service`，他暴露一个`SayHello`的方法，该方法接收一个`PingMessage`的输入参数，并返回一个`PingMessage`；一个叫做`PingMessage`的`message`，这个消息由一个单一的域`greeting`组成，这个域的类型为`string`

你也同时说明了，你使用`proto3`语法，与`proto2`语法想区别（参考[文档](https://developers.google.com/protocol-buffers/docs/proto3)）

实际上，上述这个文件现在是不可用的：他需要被编译。编译proto文件的意思是生成你选择的目标语言的代码，你的目标语言也就是你的应用中实际使用的编程语言。

在shell中，`cd`到你项目的根目录，并执行如下命令：

```bash
$ protoc -I api/ \
    -I${GOPATH}/src \
    --go_out=plugins=grpc:api \
    api/api.proto
```

这条命令会生成文件`api/api.pb.go`，一个实现了你的应用需要的gRPC代码的Go源代码，你可以查看并使用它，但是你不能去人工修改他（因为，每次你执行`protoc`命令，他都将被覆盖掉）

你也需要定义一个供`service Ping`调用的函数，所以创建一个文件`api/handler.go`:

```golang
package api

import (
  "log"
  "golang.org/x/net/context"
)

// Server represents the gRPC server
type Server struct {
}

// SayHello generates response to a Ping request
func (s *Server) SayHello(ctx context.Context, in *PingMessage) (*PingMessage, error) {
  log.Printf("Receive message %s", in.Greeting)
  return &PingMessage{Greeting: "bar"}, nil
}
```

`Server struct`仅仅是服务器的一个抽象。他允许你的server"附加"一些资源，使得这些资源在RPC调用过程中可用。

`SayHello`函数就是Protobuf文件中定义的`Ping service`的接口实现，如果你不实现该接口，你就无法创建gRPC server。

`SayHello`以一个`PingMessage`作为输入参数，并且会返回一个`PingMessage`。`PingMessage struct`定义在从`api.proto`自动生成的`api.pb.go`文件中。这个函数还有一个`Context`参数（想进一步了解原因，可参见[官方文档](https://blog.golang.org/context)）。后续你会知道通过`Context`我们能做什么。另一方面，这个函数还返回一个`error`，以防一些糟糕的情况发生。


## 创建一个最简单的服务端

现在，你已经有了一个适当的协议，你可以创建一个简单的服务器实现这个`service`，并理解他的`message`。用你最喜欢的编辑器，创建文件`server/main.go`:

```golang
package main

import (
  "fmt"
  "log"
  "net"
  "gitlab.com/pantomath-io/demo-grpc/api"
  "google.golang.org/grpc"
)

// main start a gRPC server and waits for connection
func main() {
  // create a listener on TCP port 7777
  lis, err := net.Listen("tcp", fmt.Sprintf(":%d", 7777))
  if err != nil {
    log.Fatalf("failed to listen: %v", err)
  }
  
  // create a server instance
  s := api.Server{}
  
  // create a gRPC server object
  grpcServer := grpc.NewServer()
  
  // attach the Ping service to the server
  api.RegisterPingServer(grpcServer, &s)
  
  // start the server
  if err := grpcServer.Serve(lis); err != nil {
    log.Fatalf("failed to serve: %s", err)
  }
}
```

让我分解一下代码，使得理解更清晰：

* 注意你需要引入`api`包，使得Protobuf的`service`处理句柄以及`Server struct`是可用的
* `main`函数以创建一个TCP侦听端口开始，这个侦听端口将由你绑定到你的gRPC服务器上
* 后续部分就是很直接的了：你创建一个你的`Server`的实例，创建一个gRCP服务实例，并在gRPC服务实例上注册你的`service`，并启动gRPC服务

你可以编译你的代码，并获得你服务端的二进制：

```bash
$ go build -i -v -o bin/server gitlab.com/pantomath-io/demo-grpc/server
```

## 创建一个最简单的客户端

客户端也引入`api`包，使得`message`和`service`可用，因此创建文件`client/main.go`:

```golang
package main

import (
  "log"
  "gitlab.com/pantomath-io/demo-grpc/api"
  "golang.org/x/net/context"
  "google.golang.org/grpc"
)

func main() {
  var conn *grpc.ClientConn
  
  conn, err := grpc.Dial(":7777", grpc.WithInsecure())
  if err != nil {
    log.Fatalf("did not connect: %s", err)
  }
  defer conn.Close()
  
  c := api.NewPingClient(conn)
  
  response, err := c.SayHello(context.Background(), &api.PingMessage{Greeting: "foo"})
  if err != nil {
    log.Fatalf("Error when calling SayHello: %s", err)
  }
  log.Printf("Response from server: %s", response.Greeting)
}
```

再一次，代码的分解很直观：

* `main`函数实例化一个客户端链接，端口指向服务端绑定的端口
* 注意`defer`调用，用于在函数返回时正确的关闭链接
* `c`变量是`Ping`服务的一个客户端，他能够调用`SayHello`方法，并向他传递`PingMessage`参数

你可以编译你的代码并获得客户端的二级制文件
```bash
$ go build -i -v -o bin/client gitlab.com/pantomath-io/demo-grpc/client
```

## 让他们交流

你已经构建了客户端和服务端，那么就在两个终端里分别启动他们，测试他们：

```bash
$ bin/server
2006/01/02 15:04:05 Receive message foo

$ bin/client
2006/01/02 15:04:05 Response from server: bar
```

## 使用工具让你的工作更轻松

现在，API、客户端、服务端都已经可以工作了，你可能想要一个Makefile来编译你的代码，清理你的文件夹，管理你的依赖，等等。

所以在项目根目录创建一个[`Makefile`](https://gitlab.com/pantomath-io/demo-grpc/blob/init-makefile/Makefile)。解释这个文件超出了这篇文章的范围，它主要使用以前已经产生的编译命令。

要使用这个文件`Makefile`，尝试使用下面的命令：

```bash
$ make help
api                            Auto-generate grpc go sources
build_client                   Build the binary file for client
build_server                   Build the binary file for server
clean                          Remove previous builds
dep                            Get the dependencies
help                           Display this help screen
```

## 让通信更安全

客户端与服务端之间的通信基于[HTTP/2](https://http2.github.io/)(gRPC的传输层)。消息都是二进制数据（多亏了Protobuf)，但是整个交互是明文的。幸运的是，gRPC集成了SSL/TLS，可用于对服务器进行身份验证（从客户端的角度），并加密消息交换。

你不需要在协议上做任何改变：他保持原样。只是在客户端以及服务端各自创建gRPC对象时要做些改变。需要注意的是，如果你只在客户端或者服务端一段做了改变，那整个链接将无法工作。

在改变你的代码之前， 你需要先创建一个[自签名的SSL证书](https://en.wikipedia.org/wiki/Self-signed_certificate)。这篇文章的目的不是来教你如何创建这个证书，[OpenSSL的官方文档](https://www.openssl.org/docs/manmaster/man1/)(genrsa, req, x509)可以回答你关于自签名SSL证书的问题。（DigitalOcean也有一个关于这部分内容的[非常好，非常完整的教程](https://www.digitalocean.com/community/tutorials/openssl-essentials-working-with-ssl-certificates-private-keys-and-csrs)）。当然，你也可以直接使用`cert`目录下提供的文件。下面的命令用于生成这些文件：

```bash
$ openssl genrsa -out cert/server.key 2048
$ openssl req -new -x509 -sha256 -key cert/server.key -out cert/server.crt -days 3650
$ openssl req -new -sha256 -key cert/server.key -out cert/server.csr
$ openssl x509 -req -sha256 -in cert/server.csr -signkey cert/server.key -out cert/server.crt -days 3650
```

您可以继续并更新服务器定义以使用证书和密钥:

```golang
package main

import (
  "fmt"
  "log"
  "net"
  "gitlab.com/pantomath-io/demo-grpc/api"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials"
)

// main starts a gRPC server and waits for connection
func main() {
  // create a listener on TCP port 7777
  lis, err := net.Listen("tcp", fmt.Sprintf("%s:%d", "localhost", 7777))
  if err != nil {
    log.Fatalf("failed to listen: %v", err)
  }
  
  // create a server instance
  s := api.Server{}
  
  // Create the TLS credentials
  creds, err := credentials.NewServerTLSFromFile("cert/server.crt", "cert/server.key")
  if err != nil {
    log.Fatalf("could not load TLS keys: %s", err)
  }
  
  // Create an array of gRPC options with the credentials
  opts := []grpc.ServerOption{grpc.Creds(creds)}
  
  // create a gRPC server object
  grpcServer := grpc.NewServer(opts...)
  
  // attach the Ping service to the server
  api.RegisterPingServer(grpcServer, &s)
  
  // start the server
  if err := grpcServer.Serve(lis); err != nil {
    log.Fatalf("failed to serve: %s", err)
  }
}
```

那么，有哪些变化呢？

* 通过你的证书和秘钥，你创建了一个[`credentials`](https://godoc.org/google.golang.org/grpc/credentials)对象（叫做`creds`）
* 你创建了一个[`grpc.ServerOption`](https://godoc.org/google.golang.org/grpc#ServerOption)数组，并将你的`credentials`对象置于其中。
* 当创建grpc server的时候，你向构造函数提供了你的`grpc.ServerOption`数组
* 你需要注意，你得正确提供你的server需要绑定的IP地址，这个地址要和你证书中FQDN使用的地址匹配

注意`grpc.NewServer()`是一个变参函数，你可以向他传入任意数量的参数。你创建了一个选项数组，后续你可以向这个数组中添加其他参数。

如果你现在编译你的服务端程序，并使用之前编译好的客户端程序，那他们两者之间的连接将无法工作，他们两边都会抛出error

* 服务端报错说客户端没有执行带TLS的握手

```bash
2006/01/02 15:04:05 grpc: Server.Serve failed to complete security handshake from "localhost:64018": tls: first record does not look like a TLS handshake
```

* 客户端则是在他可以做任何事之前连接被关闭

```bash
2006/01/02 15:04:05 transport: http2Client.notifyError got notified that the client transport was broken read tcp localhost:64018->127.0.0.1:7777: read: connection reset by peer.
2006/01/02 15:04:05 Error when calling SayHello: rpc error: code = Internal desc = transport is closing
```

你需要在客户端使用同样的证书文件。那么编辑你的`client/main.go`文件：

```golang
package main

import (
  "log"
  "gitlab.com/pantomath-io/demo-grpc/api"
  "golang.org/x/net/context"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials"
)

func main() {

  var conn *grpc.ClientConn
// Create the client TLS credentials
  creds, err := credentials.NewClientTLSFromFile("cert/server.crt", "")
  if err != nil {
    log.Fatalf("could not load tls cert: %s", err)
  }
  
  // Initiate a connection with the server
  conn, err = grpc.Dial("localhost:7777", grpc.WithTransportCredentials(creds))
  if err != nil {
    log.Fatalf("did not connect: %s", err)
  }
  defer conn.Close()
  
  c := api.NewPingClient(conn)
  
  response, err := c.SayHello(context.Background(), &api.PingMessage{Greeting: "foo"})
  if err != nil {
    log.Fatalf("error when calling SayHello: %s", err)
  }
  log.Printf("Response from server: %s", response.Greeting)
}
```

客户端做出的改变和服务端做出的改变非常类似：

* 你通过证书文件创建了一个[`credentials`](https://godoc.org/google.golang.org/grpc/credentials)对象，注意客户端**没有使用**证书密钥，密钥是服务端私有的。
* 你在`grpc.Dial()`函数上增添了选项：使用你的credentials对象。注意`grpc.Dial()`函数同样是一个可变参数的函数，他可以接受任意数量的选项。
* 与服务端一样，同样需要注意的是：你需要使用证书FQDN中相同的地址来链接服务端，否则传输层认证的握手过程将会失败

两边都使用了credentials，因此他们能像之前一样，相互通信了。但是现在是以加密的方式实现的。你可以编译你的代码：

```bash
$ make
```

并在各自的终端中执行：
```bash
$ bin/server
2006/01/02 15:04:05 Receive message foo

$ bin/client
2006/01/02 15:04:05 Response from server: bar
```

## 识别客户端

gRPC服务的另外一个有趣的特性是他具有从客户端拦截请求的能力。客户端可以在传输层插入一些信息。你可以使用这个特性来识别你的客户端，因为SSL的实现仅仅认证了服务端（通过证书），而没有认证客户端（所有的客户端使用了相同的证书）

因此你将更新你的客户端，在每一个调用上插入元数据信息，由服务端在处理每一个进来的调用时校验这些认证信息。

在客户端，你仅仅需要在你的`grpc.Dial()`上设置一个`DialOption`。但是，这个`DialOption`有一些限制，编辑你的`client/main.go`文件：

```golang
package main

import (
  "log"
  "gitlab.com/pantomath-io/demo-grpc/api"
  "golang.org/x/net/context"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials"
)

// Authentication holds the login/password
type Authentication struct {
  Login    string
  Password string
}

// GetRequestMetadata gets the current request metadata
func (a *Authentication) GetRequestMetadata(context.Context, ...string) (map[string]string, error) {
  return map[string]string{
    "login":    a.Login,
    "password": a.Password,
  }, nil
}

// RequireTransportSecurity indicates whether the credentials requires transport security
func (a *Authentication) RequireTransportSecurity() bool {
  return true
}

func main() {
  var conn *grpc.ClientConn
  
// Create the client TLS credentials
  creds, err := credentials.NewClientTLSFromFile("cert/server.crt", "")
  if err != nil {
    log.Fatalf("could not load tls cert: %s", err)
 }
 
  // Setup the login/pass
  auth := Authentication{
    Login:    "john",
    Password: "doe",
  }
  
  // Initiate a connection with the server
  conn, err = grpc.Dial("localhost:7777", grpc.WithTransportCredentials(creds), grpc.WithPerRPCCredentials(&auth))
  if err != nil {
    log.Fatalf("did not connect: %s", err)
   }
   defer conn.Close()
   
  c := api.NewPingClient(conn)
  
  response, err := c.SayHello(context.Background(), &api.PingMessage{Greeting: "foo"})
  if err != nil {
    log.Fatalf("error when calling SayHello: %s", err)
  }
  log.Printf("Response from server: %s", response.Greeting)
}
```

* 你定义了一个接口来存放你想要插入到rpc调用中的所有字段信息。在我们的示例中，仅仅是login和password，但你能定义任何你想要的字段
* `auth`变量存放了你用到的内容
* 你使用`grpc.WithPerRPCCredentials()`函数来创建`DialOption`对象，这个对象将在`grpc.Dial()`调用上使用
* 注意`grpc.WithPerRPCCredentials()`以一个接口作为参数，所以你的`Authentication`接口必须实现这个接口，从[文档](https://godoc.org/google.golang.org/grpc/credentials#PerRPCCredentials)中，你知道你需要在你的结构上实现两个方法：`GetRequestMetadata`和`RequireTransportSecurity`
* 因此，你定义了`GetRequestMetadata`方法，他返回你`Authentication`结构的一个map
* 最后，你定义了`RequireTransportSecurity`方法，他告诉你的grpc client是否需要将元数据插入到传输层。在我们当前的示例中，他始终返回true，但是你可以让他根据配置信息返回不同的值。

客户端已经更新，在他想服务端发起rpc调用时，会将额外的数据一起推送过去，但是服务端目前还不知道有这些额外信息。因此，我们需要告诉服务端，让他去校验这些元数据信息。

打开`server/maingo`，并更新他：

```golang
package main

import (
  "fmt"
  "log"
  "net"
  "strings"
  "golang.org/x/net/context"
  "gitlab.com/pantomath-io/demo-grpc/api"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials"
  "google.golang.org/grpc/metadata"
)

// private type for Context keys
type contextKey int

const (
  clientIDKey contextKey = iota
)

// authenticateAgent check the client credentials
func authenticateClient(ctx context.Context, s *api.Server) (string, error) {
  if md, ok := metadata.FromIncomingContext(ctx); ok {
    clientLogin := strings.Join(md["login"], "")
    clientPassword := strings.Join(md["password"], "")
    
    if clientLogin != "john" {
      return "", fmt.Errorf("unknown user %s", clientLogin)
    }
    if clientPassword != "doe" {
      return "", fmt.Errorf("bad password %s", clientPassword)
    }
    
    log.Printf("authenticated client: %s", clientLogin)
    return "42", nil
  }
  
  return "", fmt.Errorf("missing credentials")
}

// unaryInterceptor calls authenticateClient with current context
func unaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
  s, ok := info.Server.(*api.Server)
  if !ok {
    return nil, fmt.Errorf("unable to cast server")
  }
  clientID, err := authenticateClient(ctx, s)
  if err != nil {
    return nil, err
  }
  
  ctx = context.WithValue(ctx, clientIDKey, clientID)
  return handler(ctx, req)
}

// main start a gRPC server and waits for connection
func main() {
  // create a listener on TCP port 7777
  lis, err := net.Listen("tcp", fmt.Sprintf("%s:%d", "localhost", 7777))
  if err != nil {
    log.Fatalf("failed to listen: %v", err)
  }
  
  // create a server instance
  s := api.Server{}
  
  // Create the TLS credentials
  creds, err := credentials.NewServerTLSFromFile("cert/server.crt", "cert/server.key")
  if err != nil {
    log.Fatalf("could not load TLS keys: %s", err)
  }
  
  // Create an array of gRPC options with the credentials
  opts := []grpc.ServerOption{grpc.Creds(creds),
    grpc.UnaryInterceptor(unaryInterceptor)}
    
  // create a gRPC server object
  grpcServer := grpc.NewServer(opts...)
  
  // attach the Ping service to the server
  api.RegisterPingServer(grpcServer, &s)
  
  // start the server
  if err := grpcServer.Serve(lis); err != nil {
    log.Fatalf("failed to serve: %s", err)
  }
}
```

再一次，让我来为你分解一下：

* 你在你之前创建的数组里面添加了一个新的`grpc.ServerOption`(现在知道为什么他是一个数组了吧？)：`grpc.UnaryInterceptor`。并且，你想那个函数传入了一个[函数引用](https://godoc.org/google.golang.org/grpc#UnaryServerInterceptor)，这样他就知道去调用谁。main函数的剩余部分没有变化。
* 你必须要定义一个`unaryInterceptor`函数，他接收一堆参数
 1. 一个[`context.Context`](https://godoc.org/golang.org/x/net/context)对象，包含了你的数据，他在整个请求生命周期内都存在
 2. 一个`interface{}`，他是rpc调用的入参接口
 3. 一个[`UnaryServerInfo`](https://godoc.org/google.golang.org/grpc#UnaryServerInfo)结构，他包含了若干关于本地调用的信息（例如，`Server`抽象对象，以及客户端调用的具体方法）
 4. 一个[`UnaryHandler`](https://godoc.org/google.golang.org/grpc#UnaryHandler)结构，他被`UnaryServerInterceptor`调用用于完成正常的一元RPC调用（例如：在`UnaryInterceptor`返回前进行一些处理）

* `unaryInterceptor`函数确保`grpc.UnaryServerInfo`拥有正确的server抽象，并且会调用认证函数`authenticateClient`

* 你定义了包含你认证逻辑的认证函数：`authenticateClient`函数--在这个示例中非常非常简单。你可以看到，他接收一个`context.Context`参数，并从该参数中抽取元数据信息。他校验用户信息，并且返回他的ID（以`string`的形式，和一个错误，如果有的话）

* 如果`unaryInterceptor`从`authenticateClient`调用中返回一切正常，他会将`clientID`信息写入到`context.Context`对象，这样的话，后续的调用链上的函数都能够使用它（记得吗，`handler`都以`context.Context`作为入参）

* 注意，你创建了你自己的`type`和`const`，用于在`context.Context`的映射表中引用`clientID`。这只是为避免命名冲突，并允许不断引用的一种方便的方法。

你能用这样来编译代码：

```bash
$ make
```

并在各自终端中运行客户端和服务端：

```bash
$ bin/server
2006/01/02 15:04:05 authenticated client: john
2006/01/02 15:04:05 Receive message foo

$ bin/client
2006/01/02 15:04:05 Response from server: bar
```

显然，你的认证逻辑可以是更加聪明的，可以使用数据库，而不是使用凭证。方便的做法是，你的认真该函数获取到你的`Server`对象，而在你的这个`Server`结构中，能够保存了你数据库的句柄。

## 向REST开放

你已经拥有了一个非常漂亮、简洁的服务端，客户端以及协议；有序列化，加解密以及认证。但还有最后一点事情，也是一个很重要的限制：你的客户端需要是gRPC兼容的，也就是说，你必须在这份[支持平台列表](https://grpc.io/about/#osp)中，为了避免这个限制，我们可以向一个REST的网关开放我们的服务，允许REST客户端向我们发起请求。幸运的是，有这么一个gRPC的[protoc插件](https://github.com/grpc-ecosystem/grpc-gateway)，它能够生成一个方向代理服务端，通过这个服务端将一个RESTFul的JSON API请求转换成一个gRPC调用，我们可以使用少量的Go代码就能实现一个这样的反向代理服务

让我们在你的`api/api.proto`文件中加入一些额外信息

```
syntax = "proto3";
package api;
import "google/api/annotations.proto";
message PingMessage {
  string greeting = 1;
}
service Ping {
  rpc SayHello(PingMessage) returns (PingMessage) {
    option (google.api.http) = {
      post: "/1/ping"
      body: "*"
    };
  }
}
```

引入的`annotations.proto`能够让`protoc`能够理解在文件后面设置的`option`。`option`则定义了这个方法对应方法调用路径

更新`Makefile`文件，加入新的编译目标：

```makefile
SERVER_OUT := "bin/server"
CLIENT_OUT := "bin/client"
API_OUT := "api/api.pb.go"
API_REST_OUT := "api/api.pb.gw.go"
PKG := "gitlab.com/pantomath-io/demo-grpc"
SERVER_PKG_BUILD := "${PKG}/server"
CLIENT_PKG_BUILD := "${PKG}/client"
PKG_LIST := $(shell go list ${PKG}/... | grep -v /vendor/)

.PHONY: all api server client

all: server client

api/api.pb.go: api/api.proto
  @protoc -I api/ \
    -I${GOPATH}/src \
    -I${GOPATH}/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
    --go_out=plugins=grpc:api \
    api/api.proto
    
api/api.pb.gw.go: api/api.proto
  @protoc -I api/ \
    -I${GOPATH}/src \
    -I${GOPATH}/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
    --grpc-gateway_out=logtostderr=true:api \
    api/api.proto
    
api: api/api.pb.go api/api.pb.gw.go ## Auto-generate grpc go sources

dep: ## Get the dependencies
  @go get -v -d ./...
  
server: dep api ## Build the binary file for server
  @go build -i -v -o $(SERVER_OUT) $(SERVER_PKG_BUILD)
  
client: dep api ## Build the binary file for client
  @go build -i -v -o $(CLIENT_OUT) $(CLIENT_PKG_BUILD)
  
clean: ## Remove previous builds
  @rm $(SERVER_OUT) $(CLIENT_OUT) $(API_OUT) $(API_REST_OUT)
  
help: ## Display this help screen
  @grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'
```

生成为gateway准备的Go代码（和`api/api.pb.go`类似，将会生成`api/api.pb.gw.go`文件 -- 不要编辑它，在编译的时候他会自动更新）

```bash
$ make api
```

服务端的改变更重要。`grpc.Server()`是一个阻塞型调用，他只会在发生错误时返回（或者是被信号kill掉的时候）。因为我们需要启动另外一个服务端（提供REST服务），因此我们需要是的这个调用是非阻塞的。幸运的是，我们可以使用**goroutines**来达到这个目的。并且在认证的时候，也有一些小技巧。因为，REST gateway仅仅是一个反向代理，在gRPC的角度来看，他实际上是一个gRPC的客户端，因此，当他与服务端建立链接的时候，也需要使用`WithPerRPCCredentials`选项。

下面是你的`server/main.go`:

```golang
package main

import (
  "fmt"
  "log"
  "net"
  "net/http"
  "strings"
  "github.com/grpc-ecosystem/grpc-gateway/runtime"
  "golang.org/x/net/context"
  "gitlab.com/pantomath-io/demo-grpc/api"
  "google.golang.org/grpc"
  "google.golang.org/grpc/credentials"
  "google.golang.org/grpc/metadata"
)

// private type for Context keys
type contextKey int

const (
  clientIDKey contextKey = iota
)

func credMatcher(headerName string) (mdName string, ok bool) {
  if headerName == "Login" || headerName == "Password" {
    return headerName, true
  }
  return "", false
}

// authenticateAgent check the client credentials
func authenticateClient(ctx context.Context, s *api.Server) (string, error) {
  if md, ok := metadata.FromIncomingContext(ctx); ok {
    clientLogin := strings.Join(md["login"], "")
    clientPassword := strings.Join(md["password"], "")
    
    if clientLogin != "john" {
      return "", fmt.Errorf("unknown user %s", clientLogin)
    }
    if clientPassword != "doe" {
      return "", fmt.Errorf("bad password %s", clientPassword)
    }
    
    log.Printf("authenticated client: %s", clientLogin)
    return "42", nil
  }
  return "", fmt.Errorf("missing credentials")
}

// unaryInterceptor call authenticateClient with current context
func unaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
  s, ok := info.Server.(*api.Server)
  if !ok {
    return nil, fmt.Errorf("unable to cast server")
  }
  clientID, err := authenticateClient(ctx, s)
  if err != nil {
    return nil, err
  }
  
  ctx = context.WithValue(ctx, clientIDKey, clientID)
  return handler(ctx, req)
}

func startGRPCServer(address, certFile, keyFile string) error {
  // create a listener on TCP port
  lis, err := net.Listen("tcp", address)
  if err != nil {
    return fmt.Errorf("failed to listen: %v", err)
  }
  
  // create a server instance
  s := api.Server{}
  
  // Create the TLS credentials
  creds, err := credentials.NewServerTLSFromFile(certFile, keyFile)
  if err != nil {
    return fmt.Errorf("could not load TLS keys: %s", err)
  }
  
  // Create an array of gRPC options with the credentials
  opts := []grpc.ServerOption{grpc.Creds(creds),
    grpc.UnaryInterceptor(unaryInterceptor)}
    
  // create a gRPC server object
  grpcServer := grpc.NewServer(opts...)
  
  // attach the Ping service to the server
  api.RegisterPingServer(grpcServer, &s)
  
  // start the server
  log.Printf("starting HTTP/2 gRPC server on %s", address)
  if err := grpcServer.Serve(lis); err != nil {
    return fmt.Errorf("failed to serve: %s", err)
  }
  
  return nil
}

func startRESTServer(address, grpcAddress, certFile string) error {
  ctx := context.Background()
  ctx, cancel := context.WithCancel(ctx)
  defer cancel()
  
  mux := runtime.NewServeMux(runtime.WithIncomingHeaderMatcher(credMatcher))
  
  creds, err := credentials.NewClientTLSFromFile(certFile, "")
  if err != nil {
    return fmt.Errorf("could not load TLS certificate: %s", err)
  }
  
  // Setup the client gRPC options
  opts := []grpc.DialOption{grpc.WithTransportCredentials(creds)}
  
  // Register ping
  err = api.RegisterPingHandlerFromEndpoint(ctx, mux, grpcAddress, opts)
  if err != nil {
    return fmt.Errorf("could not register service Ping: %s", err)
  }
  
  log.Printf("starting HTTP/1.1 REST server on %s", address)
  http.ListenAndServe(address, mux)
  
  return nil
}

// main start a gRPC server and waits for connection
func main() {
  grpcAddress := fmt.Sprintf("%s:%d", "localhost", 7777)
  restAddress := fmt.Sprintf("%s:%d", "localhost", 7778)
  certFile := "cert/server.crt"
  keyFile := "cert/server.key"
  
  // fire the gRPC server in a goroutine
  go func() {
    err := startGRPCServer(grpcAddress, certFile, keyFile)
    if err != nil {
      log.Fatalf("failed to start gRPC server: %s", err)
    }
  }()
  
  // fire the REST server in a goroutine
  go func() {
    err := startRESTServer(restAddress, grpcAddress, certFile)
    if err != nil {
      log.Fatalf("failed to start gRPC server: %s", err)
    }
  }()
  
  // infinite loop
  log.Printf("Entering infinite loop")
  select {}
}
```

那究竟发生了什么呢？

* 你将gRPC服务端创建需要的所有代码一到了一个goroutine中，并使用一个函数包装（`startGRPCServer`)，这样他就不会阻塞`main`。

* 你也创建了一个goroutine，使用了一个包装函数（`startRESTServer`），在这个函数里面，你创建了一个HTTP/1.1服务

* 在你创建的REST gateway的`startRESTServer`函数中，你首先获得了一个`context.Context`对象（例如，context树的根）。接着你创建了一个请求复用器对象，`mux`，并使用了一个参数:`runtime.WithIncomingHeaderMatcher`。这个参数已一个函数引用作为参数，`credMatch`，这个函数将在每一个进入的请求针对HTTP头被调用。这个函数用于判断对应的HTTP头信息是否应该装换成gRPC的上下文context

* 你定义了一个`credMatch`函数，用于匹配证书头，允许他们成为gRPC上下文中的元数据。这也是为什么你的认证能工作的原因，因为反向代理在于后端gRPC服务连接时，使用HTTP头转换成gRPC的上线文元数据，

* 你也创建了一个`credentials.NewClientTLSFromFile`，用于`grpc.DialOption`，正如你之前在客户端做的一样

* 你注册你的api访问路径，例如，你让你的请求复用器和你的gRPC服务通过上下文context和gRPC选项给连接起来

* 最后，你启动你了的HTTP服务，并等待有连接请求进来

* 除了使用goroutine，你还是用了一个阻塞的`select`调用，这样做可以防止你的程序立马退出

现在，构建你的整个项目，你可以测试你的REST接口了：

```bash
$ make
```

在不用的终端中分别运行客户端和服务端：

```bash
$ bin/server
2006/01/02 15:04:05 Entering infinite loop
2006/01/02 15:04:05 starting HTTP/1.1 REST server on localhost:7778
2006/01/02 15:04:05 starting HTTP/2 gRPC server on localhost:7777
2006/01/02 15:04:05 authenticated client: john
2006/01/02 15:04:05 Receive message foo

$ curl -H "login:john" -H "password:doe" -X POST -d '{"greeting":"foo"}' 'http://localhost:7778/1/ping'
{"greeting":"bar"}
```

## 最后是swag...

REST形式的网关是很cool的，但是，如果能直接从他生成文档，岂不是更cool，对吗？

通过使用[`protoc`插件](https://github.com/grpc-ecosystem/grpc-gateway)来生成[swagger](https://swagger.io/)json文件，你可以很容易的做到：

```bash
protoc -I api/ \
  -I${GOPATH}/src \
  -I${GOPATH}/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --swagger_out=logtostderr=true:api \
  api/api.proto
```

这将会生成`api/api.swagger.json`文件。像其他有Protobuf编译生成的文件一样，你不应该手动编辑它，但你可以使用它，同时，你也可以通过修改你的定义文件来编译更新他。

你可以把上述编译命令放入到`Makefile`中。

## 结论

你已经拥有了一个完整功能的gRPC客户端和服务端，他具有SSL加密、身份认证、客户端标识，以及REST网关（并包含swagger文件）等功能。那接下来，应该干什么呢？

你可以在REST网关上再添加一些新的功能，让他支持HTTPS，而不是HTTP。显然，你还可以在你的Protobuf上添加更加复杂的数据结构，增加更多的`service`。你也可以从HTTP/2的特性中获益，例如从客户端到服务端，或者从服务端到客户端，甚至是双向的[流式](https://grpc.io/docs/guides/concepts.html#bidirectional-streaming-rpc)特性。（当然，这个特性是仅仅针对gRPC的，REST是基于HTTP/1.1，无此特性）

非常感谢[Charles Francoise](https://medium.com/@loderunnr)，他和我一同完成这篇文章，并编写了示例代码：https://gitlab.com/pantomath-io/demo-grpc.

