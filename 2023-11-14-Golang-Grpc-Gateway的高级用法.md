---
layout: 	post
title: 	    Golang Grpc-Gateway 如何读取HttpHeader请求头信息
subTitle:   关于Grpc-Gateway中http header和grpc metadata的转换问题
date: 		2023-11-14
author:     Yong
tags:
    - Go
    - Grpc
---

## 前言
在我们使用grpc对外提供rpc接口的时候, 鉴于复杂的外部环境 gRPC 并不是万能的工具。在某些情况下，我们仍然希望提供传统的 HTTP/JSON API，来满足维护向后兼容性或者那些不支持 gRPC 的客户端。

但是为我们的RPC服务再编写另一个服务只是为了对外提供一个 HTTP/JSON API，这是一项相当耗时和乏味的任务。

GRPC-Gateway 能帮助你同时提供 gRPC 和 RESTful 风格的 API. 它读取 gRPC 服务定义并生成一个反向代理服务器，该服务器将 RESTFUL JSON API 转换为 gRPC。

在前不久的一个项目中, 我需要读取HTTP协议请求中请求头的参数来做一些校验, 那在 gRpc 的函数中, 我们要怎么获取的到HTTP请求头中的参数呢?

## 一个基本的GRPC服务 
### proto文件编写
新建一个`example.proto`文件, 添加如下内容, 定义一个基本的Grpc服务
```proto
syntax = "proto3";

option go_package = "./examplepb";

import "google/api/annotations.proto";

package examplepb;


service ExampleHelloService {
	rpc SayHello(HelloRequest) returns (HelloResponse) {
		option (google.api.http) = {
			post: "/hello"
			body: "*"
		};
	};
}

message HelloRequest {
	string name = 1;
}

message HelloResponse {
	string message = 1;
}
```
使用如下命令, 生成`.go`代码文件
```shell
protoc pb/example.proto \
    --go_out=. \
    --go-grpc_out=. \
    --proto_path $GOPATH/pkg/mod/github.com/grpc-ecosystem/grpc-gateway@v1.16.0/third_party/googleapis \
    --proto_path $GOPATH/pkg/mod/github.com/protocolbuffers/protobuf@v4.23.2+incompatible/src \
    --proto_path=. \
    --grpc-gateway_out=.
```

### 启动GRPC服务 
在`main.go`中注册GRPC服务并运行
```go
type exampleServer struct {
	examplepb.UnimplementedExampleHelloServiceServer
}

func (es *exampleServer) SayHello(ctx context.Context, in *examplepb.HelloRequest) (*examplepb.HelloResponse, error) {	
	return &examplepb.HelloResponse{
		Message: "hello",
	}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":29919")
	if err != nil {
		fmt.Printf("failed to listen: %v", err)
		return
	}

	s := grpc.NewServer()
	examplepb.RegisterExampleHelloServiceServer(s, &exampleServer{})

	cm := cmux.New(lis)
	grpcListener := cm.MatchWithWriters(cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"))
	httpListener := cm.Match(cmux.HTTP1Fast(), cmux.HTTP2())

	mux := runtime.NewServeMux()
	if err := examplepb.RegisterExampleHelloServiceHandlerServer(context.Background(), mux, &exampleServer{}); err != nil {
		fmt.Printf("register service handler error: %v", err)
		return
	}

	grp := errgroup.Group{}
	grp.Go(func() error {
		return cm.Serve()
	})
	grp.Go(func() error {
		return s.Serve(grpcListener)
	})
	grp.Go(func() error {
		return http.Serve(httpListener, mux)
	})
	fmt.Println(grp.Wait())
}
```
此时目录结构如下:
```shell
.
├── examplepb
│   ├── example.pb.go
│   ├── example.pb.gw.go
│   └── example_grpc.pb.go
├── go.mod
├── go.sum
├── main.go
└── pb
    └── example.proto
```
到这里, 一个最简单的GRPC服务就启动起来了, 它使用`cmux`启动了一个多路复用器, 将http服务和Grpc服务都绑定在一个同一个端口中.

## Grpc metadata

我们可以轻松的使用`GRPC`的方式或者`HTTP`的方式请求到我们需要的接口

不管是`GRPC`还是`HTTP`我们有时候都需要对接口进行校验

对于`GRPC`来说, 我们都知道(假设大家都知道😜), 可以在客户端调用服务端接口时通过上下文（Context）传入 metadata
```go
client := examplepb.NewExampleHelloServiceClient(conn)

md := metadata.Pairs("key1", "value1", "key2", "value2")
ctx := metadata.NewOutgoingContext(context.Background(), md)

resp, err := client.SayHello(ctx, &examplepb.HelloRequest{
	Name: "name",
})
```
此时我们在服务端中就可以通过读取上下文（Context）中的 metadata, 获取到客户端传过来的数据
```go
md, ok := metadata.FromIncomingContext(ctx)
fmt.Println(md, ok)
// map[:authority:[localhost:29919] content-type:[application/grpc] key1:[value1] key2:[value2] user-agent:[grpc-go/1.59.0]] true
```

但是怎么在HTTP调用的方式下获取到请求头的数据呢 ? 我们先在HTTP调用的时候在请求头中加点数据, 然后在服务端中打印一下 metadata 看看
```shell
curl -X POST 'http://localhost:29919/hello' --header 'rpc-http-header-test: hello-test'
```
此时服务端日志结果为: 
```shell
map[grpcgateway-accept:[*/*] grpcgateway-user-agent:[curl/8.1.2] x-forwarded-for:[127.0.0.1] x-forwarded-host:[localhost:29919]] true
```
可以看到我们请求的头的数据并没进入到GRPC的 metadata 中

### Grpc Gateway HeaderMatcher
按道理来说, 我们能够获取到`httpHeader`数据的地方应该只有上下文的`metadata`了,
但是到底是什么原因导致的`httpHeader`中的数据没有进入到上下文的`metadata`呢

在查看`RegisterExampleHelloServiceHandlerServer`源码后发现有一个比较可疑的函数`runtime.AnnotateIncomingContext`
```go
// AnnotateIncomingContext adds context information such as metadata from the request.
// Attach metadata as incoming context.
func AnnotateIncomingContext(ctx context.Context, mux *ServeMux, req *http.Request, rpcMethodName string, options ...AnnotateContextOption) (context.Context, error) {
	ctx, md, err := annotateContext(ctx, mux, req, rpcMethodName, options...)
	if err != nil {
		return nil, err
	}
	if md == nil {
		return ctx, nil
	}

	return metadata.NewIncomingContext(ctx, md), nil
}
```
这个函数中把`*http.Request`传入进去, 读取请求中的请求头并通过`metadata`放到`context (上下文)`中

再看到它里面的`annotateContext`, 这就是它具体的处理请求头的逻辑了

可以看到, 里面有一个`incomingHeaderMatcher`, 用来控制解析指定的请求头到`metadata`中
```go
for key, vals := range req.Header {
    ...
	for _, val := range vals {
		// For backwards-compatibility, pass through 'authorization' header with no prefix.
		if key == "Authorization" {
			pairs = append(pairs, "authorization", val)
		}
		if h, ok := mux.incomingHeaderMatcher(key); ok {
			...	
            pairs = append(pairs, h, val)
		}
	}
}
```
我们可以通过`runtime.WithIncomingHeaderMatcher`来设置我们的匹配器, 如果我们没有指定的话, 它就会使用默认的匹配器
```go
// DefaultHeaderMatcher is used to pass http request headers to/from gRPC context. This adds permanent HTTP header
// keys (as specified by the IANA, e.g: Accept, Cookie, Host) to the gRPC metadata with the grpcgateway- prefix. If you want to know which headers are considered permanent, you can view the isPermanentHTTPHeader function.
// HTTP headers that start with 'Grpc-Metadata-' are mapped to gRPC metadata after removing the prefix 'Grpc-Metadata-'.
// Other headers are not added to the gRPC metadata.
func DefaultHeaderMatcher(key string) (string, bool) {
	switch key = textproto.CanonicalMIMEHeaderKey(key); {
	case isPermanentHTTPHeader(key):
		return MetadataPrefix + key, true
	case strings.HasPrefix(key, MetadataHeaderPrefix):
		return key[len(MetadataHeaderPrefix):], true
	}
	return "", false
}
```
在这个方法的注释中我们可以看到, 对于带有前缀`Grpc-Metadata-`的请求头, 它都可以自动的解析到metadata中

当然我们也可以自己指定我们想要的匹配逻辑.....
