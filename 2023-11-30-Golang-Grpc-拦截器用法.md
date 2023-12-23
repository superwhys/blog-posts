---
layout: 	post
title: 	    Golang Grpc 拦截器用法详解 
subTitle:   Golang Grpc 拦截器用法详解
date: 		2023-11-30
author:     Yong
tags:
    - Go
    - Grpc
---

在常规的`http服务`开发的过程中,我们都使用过`中间件`这一类的东西。那么在`grpc服务`中,我们也有对应的技术,那就是`拦截器`。

## 1. 什么是拦截器
在常规的 HTTP 服务器中，我们可以设置有一个中间件将我们的处理程序包装在服务器上。此中间件可用于在实际提供正确内容之前执行服务器想要执行的任何操作，它可以是身份验证或日志记录或任何东西。

对于中间件:

它可以在拦截到发送给 handler 的请求，且可以拦截 handler 返回给客户端的响应

但GRPC的拦截器不同:

它允许在服务器和客户端都使用拦截器。

- 服务器端拦截器是 gRPC 服务器在到达实际 RPC 方法之前调用的函数。它可以用于多种用途，例如日志记录、跟踪、速率限制、身份验证和授权。
- 客户端拦截器是 gRPC 客户端在调用实际 RPC 之前调用的函数。

## 2. 拦截器类型
在 gRPC 中，根据拦截器拦截的 RPC 调用的类型，拦截器在分类上可以分为如下两种：

- 一元拦截器（Unary Interceptor）：拦截和处理一元 RPC 调用。
- 流拦截器（Stream Interceptor）：拦截和处理流式 RPC 调用。

虽然总的来说是只有两种拦截器分类，但是再细分下去，客户端和服务端每一个都有其自己的一元和流拦截器的具体类型。因此，gRPC 中也可以说总共有四种不同类型的拦截器。

## 3. 客户端拦截器
### 3.1 一元拦截器
客户端的一元拦截器类型为 `UnaryClientInterceptor`，方法原型如下：
```go
type UnaryClientInterceptor func(
    ctx context.Context, 
    method string, 
    req, 
    reply interface{}, 
    cc *ClientConn, 
    invoker UnaryInvoker, 
    opts ...CallOption,
) error
```

### 3.2 流拦截器
客户端的流拦截器类型为 `StreamClientInterceptor`，方法原型如下：
```go
type StreamClientInterceptor func(
    ctx context.Context, 
    desc *StreamDesc, 
    cc *ClientConn, 
    method string, 
    streamer Streamer, 
    opts ...CallOption,
) (ClientStream, error)
```

## 4. 服务端拦截器
### 4.1 一元拦截器
服务端一元拦截器类型为 `UnaryServerInterceptor`，方法原型如下：
```go
type UnaryServerInterceptor func(
    ctx context.Context, 
    req any, 
    info *UnaryServerInfo, 
    handler UnaryHandler,
) (resp any, err error)
```

### 4.2 流拦截器
服务端流拦截器类型为 `StreamServerInterceptor`，方法原型如下：
```go
type StreamServerInterceptor func(
    srv any, 
    ss ServerStream, 
    info *StreamServerInfo, 
    handler StreamHandler,
) error
```

## 5. 使用案例
### 5.1 服务端拦截器
#### 一元拦截器
```go

func exampleInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
	start := time.Now()
	resp, err = handler(ctx, req)
	fmt.Printf("spend time: %v\n", time.Since(start))
	return resp, err
}

// ...

s := grpc.NewServer(grpc.UnaryInterceptor(exampleInterceptor))

// ...

```

#### 流拦截器
```go

type wrappedStream struct {
	grpc.ServerStream
}

func newWrappedStream(s grpc.ServerStream) grpc.ServerStream {
	return &wrappedStream{s}
}

func (w *wrappedStream) RecvMsg(m interface{}) error {
	logger("Receive a message (Type: %T) at %s", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.RecvMsg(m)
}

func (w *wrappedStream) SendMsg(m interface{}) error {
	logger("Send a message (Type: %T) at %v", m, time.Now().Format(time.RFC3339))
	return w.ServerStream.SendMsg(m)
}

func exampleStreamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
	// 包装 grpc.ServerStream 以替换 RecvMsg SendMsg这两个方法。
	err := handler(srv, newWrappedStream(ss))
	if err != nil {
		logger("RPC failed with error %v", err)
	}
	return err
}

// ...

s := grpc.NewServer(grpc.StreamInterceptor(exampleStreamInterceptor))

// ...

```

### 5.2 客户端拦截器
#### 一元拦截器
```go
func exampleClientInterceptor(ctx context.Context, method string, req, reply any, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
	// pre-processing
	start := time.Now()
	err := invoker(ctx, method, req, reply, cc, opts...) // invoking RPC method
	// post-processing
	fmt.Printf("invoke remote method=%s req=%v reply=%v duration=%s error=%v\n", method, req, reply, time.Since(start), err)
	return err
}

conn, err := grpc.DialContext(
		context.TODO(),
		"localhost:55771",
		grpc.UnaryClientInterceptor(exampleClientInterceptor),
)

client := examplepb.NewExampleHelloServiceClient(conn)
resp, err := client.SayHello(ctx, &examplepb.HelloRequest{
	Name: "name",
})

// ...
```
