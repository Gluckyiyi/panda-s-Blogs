# 初识grpc(一）——mac下的初体验

## 前言

最近在做这样一个需求，一个PC客户端应用拥有一个system权限下的服务，这样的一个服务在执行高权限方面确实很有优势，解决了很多的权限难题，但任何事情都有两面性，在拥有高权限的同时也丧失了一些需要用户权限执行的功能，比如要获取显示器信息的时候，因为system的session下面是没有用户界面的，所以就无法准确获取到，想要安装证书到当前用户下面的时候也是无能无力，等等随着这样的场景越来越多，对于一个用户权限下的进程需求就愈发的明显。又由于历史性的问题，核心的通信框架在这个agent中，所有的和远程server的交互都是agent执行的，所以我们需要的是一个用户权限下的进程，需要和agent进行通信，来执行agent的指令，关于这个通信的实现综合考虑了很多解决方案，后来在朋友的推荐下了解了grpc，因为是谷歌出品，同时又拥有golang的版本，和我的agent(golang实现的)可以无缝对接，决定一试。

## grpc简介

### 1、grpc

grpc是由谷歌开发的，一款语言中立、平台中立、开源的远程过程调用(RPC)系统。

在 gRPC 里*客户端*应用可以像调用本地对象一样直接调用另一台不同的机器上*服务端*应用的方法，使得您能够更容易地创建分布式应用和服务。与许多 RPC 系统类似，gRPC 也是基于以下理念：定义一个*服务*，指定其能够被远程调用的方法（包含参数和返回类型）。在服务端实现这个接口，并运行一个 gRPC 服务器来处理客户端调用。在客户端拥有一个*存根*能够像服务端一样的方法。

![图1](http://www.grpc.io/img/grpc_concept_diagram_00.png)

gRPC 客户端和服务端可以在多种环境中运行和交互 - 从 google 内部的服务器到你自己的笔记本，并且可以用任何 gRPC [支持的语言](http://doc.oschina.net/grpc?t=58008#quickstart)来编写。所以，你可以很容易地用 Java 创建一个 gRPC 服务端，用 Go、Python、Ruby 来创建客户端。此外，Google 最新 API 将有 gRPC 版本的接口，使你很容易地将 Google 的功能集成到你的应用里。

###2、protocol buffers

gRPC 默认使用 *protocol buffers*，这是 Google 开源的一套成熟的结构数据序列化机制（当然也可以使用其他数据格式如 JSON）。正如你将在下方例子里所看到的，你用 *proto files* 创建 gRPC 服务，用 protocol buffers 消息类型来定义方法参数和返回类型。你可以在 [Protocol Buffers 文档](http://doc.oschina.net/https%EF%BC%9A//developers.google.com/protocol-buffers/docs/overview)找到更多关于 Protocol Buffers 的资料。

## 话不多说，开撸。Show me the code ！

> 以下内容是基于Mac平台下搭建的环境，运行的Demo。语言是golang

### 1、搭建环境

####1.1 安装grpc-go

#### golang下的grpc是[grpc-go](<https://github.com/grpc/grpc-go>)，首先我们需要下载grpc-go包，使用命令

```go get -u google.golang.org/grpc```

> 注：本人尝试这条命令是需要科学上网，需要朋友们自己解决了。

如果实在没办法解决的，下面的命令也是可行的

```git clone https://github.com/grpc/grpc-go.git $GOPATH/src/google.golang.org/grpc```

#### 1.2安装protobuf

[protobuf](<https://github.com/golang/protobuf>)的安装方式有多种，有极客精神的同学应该是下载源码，自己编译再设置到环境变量中或者移动到bin/sbin之类的目录下。这里为了快速我采用了直接下载release版本的方式，mac版本的下载地址：[protobuf release下载](<<https://github.com/protocolbuffers/protobuf/releases>>)，目前的最新release版本是3.7.1。如下图:

![image-20190417230854010](/Users/panda/Library/Application Support/typora-user-images/image-20190417230854010.png)

然后将目录加入到环境变量，中或者移动到bin/sbin之类的目录下。

然后安装golang protobuf直接使用golang的get即可

```golang
go get -u github.com/golang/protobuf/proto // golang protobuf 库
go get -u github.com/golang/protobuf/protoc-gen-go //protoc --go_out 工具
```

这时候就完成了protoc的安装，在命令行中输入命令，应该会看到如下图的界面：

![image-20190417231138160](/Users/panda/Library/Application Support/typora-user-images/image-20190417231138160.png)

2、开撸代码

既然是初体验，我们当然是从hello world开始啦。

我的目录结构大体是这样的：

![image-20190417231741972](/Users/panda/Library/Application Support/typora-user-images/image-20190417231741972.png)

1. 首先在proto目录下创建hello.proto文件，内容如下：

```go
syntax = "proto3";

package hello;

service Hello {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
    string Name = 1;
}

message HelloReply {
    string Message = 1;
}
```

> 这里的**syntax**表示的是proto的版本，**package**：是指定输出的包名称。**service**关键字是创建一个服务，**message**定义消息结构体

2.切换到proto目录下，执行如下命令：

```protoc --go_out=plugins:. hello.proto```

格式为protoc --go_out=plugins:{go文件输出路径} {proto文件路径}

这个时候会看到proto目录下多了一个hello.pb.go文件

![image-20190417232524350](/Users/panda/Library/Application Support/typora-user-images/image-20190417232524350.png)

3.下面我们创建Server。

代码:

```go
const (
   address = ":8848"
)

type Server struct {

}

func (s *Server)SayHello(ctx context.Context,in *hello.HelloRequest)(*hello.HelloReply,error){
   return &hello.HelloReply{
      Message:"hello," + in.Name,
   }, nil
}
func main() {
   conn, err := net.Listen("tcp", address)
   if err != nil {
      log.Fatal(err)
   }
   fmt.Println("grpc server listening at: 8848 port")
   server := grpc.NewServer()
   hello.RegisterHelloServer(server, &Server{})
   server.Serve(conn)
}
```

这份代码中我们首先实现了Hello服务接口，在客户端传过来的名字前面添加hello.

```go
hello.RegisterHelloServer(server, &Server{})
```

这句代码是向grpc服务做注册的。

3.创建client

```go
const (
	address = "localhost:8848"
	defaultName = "world"
)
func main() {
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	client := hello.NewHelloClient(conn)

	name := defaultName
	if len(os.Args) > 1 {
		name = os.Args[1]
	}
	request, err := client.SayHello(context.Background(), &hello.HelloRequest{Name:name})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(request.Message)
}
```

4.激动人心的时刻，运行看效果

切换到server目录下，`go run server.go`

![image-20190417233450762](/Users/panda/Library/Application Support/typora-user-images/image-20190417233450762.png)

Bingo,server成功的运行在8848的端口上了。

切换到client目录下，`go run client.go`

![image-20190417233745420](/Users/panda/Library/Application Support/typora-user-images/image-20190417233745420.png)

Hello,world多么美妙的一句话，至此我们已经在mac下面搭载好了grpc的环境并向世界say了hello。

## 结尾

万事开头难，万里长征的第一步已经迈出去了。明天会在windows下面来搭建和测试system下和user下的两个进程使用grpc的通信。