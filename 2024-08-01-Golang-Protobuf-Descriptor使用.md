---
layout: 	post
title: 	    Protobuf Descriptor.proto使用	
subTitle:	关于descriptor.proto使用、拓展descriptor以及自定义插件处理器详解
date:		2024-08-01	
author:     Yong
tags:
    - Go
    - Protobuf
---

## 前言
最近在研究 [syncthing](https://github.com/syncthing/syncthing) 源码时发现，他们的各种传输结构都是使用 `protobuf` 定义的

并且其中很多字段都使用了 `descriptor` 进行拓展

```proto
syntax = "proto3";

package config;

import "lib/protocol/bep.proto";
import "lib/config/observed.proto";

import "ext.proto";

message DeviceConfiguration {
    bytes                   device_id                  = 1 [(ext.goname) = "DeviceID", (ext.xml) = "id,attr", (ext.json) = "deviceID", (ext.device_id) = true, (ext.nodefault) = true];
    string                  name                       = 2 [(ext.xml) = "name,attr,omitempty"];
    repeated string         addresses                  = 3 [(ext.xml) = "address,omitempty"];
    protocol.Compression    compression                = 4 [(ext.xml) = "compression,attr"];
    string                  cert_name                  = 5 [(ext.xml) = "certName,attr,omitempty"];
    bool                    introducer                 = 6 [(ext.xml) = "introducer,attr"];
    bool                    skip_introduction_removals = 7 [(ext.xml) = "skipIntroductionRemovals,attr"];
    bytes                   introduced_by              = 8 [(ext.xml) = "introducedBy,attr", (ext.device_id) = true, (ext.nodefault) = true];
    bool                    paused                     = 9;
    repeated string         allowed_networks           = 10 [(ext.xml) = "allowedNetwork,omitempty"];
    bool                    auto_accept_folders        = 11;
    int32                   max_send_kbps              = 12;
    int32                   max_recv_kbps              = 13;
    repeated ObservedFolder ignored_folders            = 14;
    repeated ObservedFolder pending_folders            = 15 [deprecated = true];
    int32                   max_request_kib            = 16 [(ext.goname) = "MaxRequestKiB", (ext.xml) = "maxRequestKiB", (ext.json) = "maxRequestKiB"];
    bool                    untrusted                  = 17;
    int32                   remote_gui_port            = 18 [(ext.goname) = "RemoteGUIPort", (ext.xml) = "remoteGUIPort", (ext.json) = "remoteGUIPort"];
    int32                   num_connections            = 19 [(ext.goname) = "RawNumConnections"]; // attempt to establish this many connections to the device
}
```

特此记录一下 `descriptor.proto` 的使用以及过程中遇到的一些问题

## Descriptor.proto
`descriptor.proto` 是 Protocol Buffers（protobuf）中的一个核心文件，它定义了描述 protobuf 消息类型、枚举类型、服务等的元数据结构。

具体来说，它包含了 `FileDescriptorProto`、`DescriptorProto`、`FieldDescriptorProto` 等描述符，这些描述符用于描述 .proto 文件中的所有元素。

主要用途: 

1. **描述 .proto 文件结构**：`descriptor.proto` 文件中的描述符定义了 .proto 文件的结构，包括消息类型、字段、枚举、服务等。

2. **动态消息处理**：通过这些描述符，protobuf 可以在运行时动态地解析和处理消息，而不需要预先生成代码。这对于需要处理未知或动态消息类型的应用非常有用。

3. **代码生成**：protobuf 编译器（`protoc`）使用这些描述符来生成相应的代码，以便在不同编程语言中使用 protobuf。

在正常情况下使用 protobuf 我们并不怎么会使用到 `descriptor.proto` 的东西。

但是当我们想要更好地控制生成代码的行为和消息的序列化方式的时候，我们就可以使用它。

使用它的方式大概有:

1. **内置选项**: `descriptor.proto` 文件中定义了许多内置选项，用于描述和配置 protobuf 消息、字段、枚举等。

2. **拓展选项**: 当内置的选项不符合我们使用场景的时候，我们可以通过拓展它的选项并结合自定义插件来根据不同的选项生成不同的代码文件

## descriptor.proto 内置选项

`descriptor.proto` 文件中定义了许多内置选项，用于描述和配置 protobuf 消息、字段、枚举等。以下是一些常见的内置选项：

### 文件级选项
在 .proto 文件中，文件级选项用于配置生成代码的行为和其他全局设置。以下是一些常见的文件级选项及其使用示例：

#### 常见文件级选项
- java_package
- java_outer_classname
- optimize_for
- cc_enable_arenas
- objc_class_prefix

#### 使用示例
```proto
syntax = "proto3";

package example;

option java_package = "com.example.foo";
option java_outer_classname = "ExampleProto";
option optimize_for = SPEED;
option cc_enable_arenas = true;
option objc_class_prefix = "EX";

message SampleMessage {
    string name = 1;
    int32 id = 2;
}
```

### 消息级选项
在 .proto 文件中，消息级选项用于配置特定消息的行为和属性。以下是一些常见的消息级选项及其使用示例：

#### 常见消息级选项
- message_set_wire_format
- no_standard_descriptor_accessor
- deprecated

#### 使用示例
```proto
syntax = "proto3";

package example;

message SampleMessage {
    option message_set_wire_format = true;
    option no_standard_descriptor_accessor = true;
    option deprecated = true;

    string name = 1;
    int32 id = 2;
}
```

### 字段级选项
在 .proto 文件中，字段级选项用于配置特定字段的行为和属性。以下是一些常见的字段级选项及其使用示例：

#### 常见字段级选项
- deprecated
- packed
- jstype
- ctype

#### 使用示例
```proto
syntax = "proto3";

package example;

message SampleMessage {
    int32 old_field = 1 [deprecated = true];
    repeated int32 ids = 2 [packed = true];
    int64 big_number = 3 [jstype = JS_STRING];
    string name = 4 [ctype = CORD];
}
```

### 枚举级选项
在 .proto 文件中，枚举级选项用于配置特定枚举的行为和属性。以下是一些常见的枚举级选项及其使用示例：

#### 常见枚举级选项
- allow_alias
- deprecated
- enum_value_option

#### 使用示例
```proto
syntax = "proto3";

package example;

enum Status {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  IN_PROGRESS = 1; // 别名
  COMPLETED = 2 [deprecated = true]; // 标记为已弃用
}

message Task {
  string name = 1;
  Status status = 2;
}
```

### 服务级选项
在 .proto 文件中，服务级选项用于配置特定服务的行为和属性。以下是一些常见的服务级选项及其使用示例：

#### 常见服务级选项
- deprecated
- uninterpreted_option

#### 使用示例
以下是一个包含服务级选项的 .proto 文件示例：
```proto
syntax = "proto3";

package example;

service MyService {
  option deprecated = true;
  option (custom_option) = "example";

  rpc MyMethod (Request) returns (Response);
}

message Request {
  string name = 1;
}

message Response {
  string message = 1;
}
```

### 方法级选项
在 .proto 文件中，方法级选项用于配置特定方法的行为和属性。以下是一些常见的方法级选项及其使用示例：

#### 常见方法级选项
- deprecated
- idempotency_level

#### 使用示例
以下是一个包含方法级选项的 .proto 文件示例：
```proto
syntax = "proto3";

package example;

service MyService {
  rpc MyMethod (Request) returns (Response) {
    option deprecated = true;
    option idempotency_level = IDEMPOTENT;
  }
}

message Request {
  string name = 1;
}

message Response {
  string message = 1;
}
```

## descriptor.proto 拓展选项
当内置的选项不符合我们的使用需求的时候，我们可以拓展我们需要的自定义选项

但是通常这是大多数人用不到的功能，且自定义选项仅支持proto3。

由于"选项" 是由 `google/protobuf/descriptor.proto` 中定义的消息定义的（例如FileOptions或FieldOptions），因此自定义选项仅是扩展这些消息而已。

拓展示例:
```proto3
syntax = "proto3";

package testpb;
option go_package = "./testpb";

import "google/protobuf/descriptor.proto";

// 定义一个扩展选项
extend google.protobuf.FieldOptions {
  string my_custom_option = 50000;
}

// 定义一个消息
message MyMessage {
  string name = 1 [(my_custom_option) = "example"];
}
```

然后使用以下命令生成代码文件

`protoc test.proto --proto_path=$GOPATH/src/github.com/gogo/protobuf/protobuf --proto_path=. --go_out=Mgoogle/protobuf/descriptor.proto=github.com/gogo/protobuf/protoc-gen-gogo/descriptor,paths=source_relative:. `

注意: 这里使用的是 `protoc-gen-go` 代码生成器, 并且确保clonse了`github.com/gogo/protobuf/protoc-gen-gogo/descriptor/descriptor.proto`在你的GOPATH下

这个使用，我们就可以使用以下代码获取到 MyMessage 的自定义选项的值了
```go
package main 

import (
	"fmt"

	"github.com/superwhys/haowen/golang/proto-test/testpb"
	"google.golang.org/protobuf/proto"
)

func main() {
    // 创建一个示例消息
	msg := &testpb.MyMessage{
		Name: "test",
	}

	// 获取消息的描述符
	msgDescriptor := msg.ProtoReflect().Descriptor()

	// 获取字段描述符
	fieldDescriptor := msgDescriptor.Fields().ByName("name")

	// 获取自定义选项
	if proto.HasExtension(fieldDescriptor.Options(), testpb.E_MyCustomOption) {
		ext := proto.GetExtension(fieldDescriptor.Options(), testpb.E_MyCustomOption)
		customOption := ext.(string)
		fmt.Printf("Custom option value: %s\n", customOption)
	} else {
		fmt.Println("Custom option not found")
	}
}
```

### 使用自定义插件
但是，我们一般不会就这么使用自定义选项，我们通常会根据不同的选项来生成不同的代码文件。

下面演示使用 `protoc-gen-gogo` 生成器来通过自定义插件生成代码

这里使用 `protoc-gen-gogo` 是因为它用来写自定义插件比较方便

#### ext.proto
定义一个 `ext.proto`, 我们把拓展选项放到这个 proto 文件中，因为我们需要先生成这个的代码文件，然后再编写插件代码

```proto2
syntax = "proto2";

package ext;

option go_package = "github.com/superwhys/haowen/golang/proto-test/ext";

import "google/protobuf/descriptor.proto";

// 定义一个扩展选项
extend google.protobuf.FieldOptions {
  optional string my_custom_option = 50000;
}
```
 
使用命令生成代码文件

`protoc -I . -I $GOPATH/src/github.com/gogo/protobuf/protobuf --gogofast_out=Mgoogle/protobuf/descriptor.proto=github.com/gogo/protobuf/protoc-gen-gogo/descriptor,paths=source_relative:ext ext.proto`

#### 自定义插件protoc-gen-test

接下来就可以编写自定义插件的代码了
```go
package main

import (
	"fmt"
	
	"github.com/gogo/protobuf/gogoproto"
	"github.com/gogo/protobuf/proto"
	"github.com/gogo/protobuf/protoc-gen-gogo/descriptor"
	"github.com/gogo/protobuf/vanity"
	"github.com/gogo/protobuf/vanity/command"
	"github.com/superwhys/haowen/golang/proto-test/ext"
)

func main() {
    // 读取传进来的proto文件
	req := command.Read()
	
	files := req.GetProtoFile()
	files = vanity.FilterFiles(files, vanity.NotGoogleProtobufDescriptorProto)

    // 一次处理每一个proto文件
	vanity.ForEachFile(files, handleFile)
	
	resp := command.Generate(req)
	command.Write(resp)
}

func handleFile(file *descriptor.FileDescriptorProto) {
    // 一次处理文件中的每个message
	vanity.ForEachMessageInFiles([]*descriptor.FileDescriptorProto{file}, handleMessage)
}

func handleMessage(msg *descriptor.DescriptorProto) {
    // 一次处理message中的每个字段
	vanity.ForEachField([]*descriptor.DescriptorProto{msg}, func(field *descriptor.FieldDescriptorProto) {
		if field.Options == nil {
			field.Options = &descriptor.FieldOptions{}
		}
		
		current := ""
		if v, ok := getFieldStringExtension(field, gogoproto.E_Moretags); ok {
			current = v
		}

        // 获取我们的自定义拓展选项
		val, ok := getFieldStringExtension(field, ext.E_MyCustomOption)
		if ok {
            // 写成一个tag的形式
            // 这里我们还可以有很多的操作
            // 1. 通过自定义的选项操作内置的选项。内置的选项在生成代码文件时会有自己的生成逻辑
            // gogoproto.E_Customname, gogoproto.E_Customtype, gogoproto.E_Jsontag, gogoproto.E_Moretags等
            // 2. 通过自定义选项来决定其他的生成逻辑
			current += fmt.Sprintf(`my_custom_option:"%v"`, val)
		}
		
		if err := proto.SetExtension(field.Options, gogoproto.E_Moretags, &current); err != nil {
			panic(err)
		}
	})
}

func getFieldStringExtension(field *descriptor.FieldDescriptorProto, extension *proto.ExtensionDesc) (string, bool) {
	if field.Options == nil {
		return "", false
	}
	value, err := proto.GetExtension(field.Options, extension)
	if err != nil {
		return "", false
	}
	if value == nil {
		return "", false
	}
	if v, ok := value.(*string); !ok || v == nil {
		return "", false
	} else {
		return *v, true
	}
}

```
将插件编译成 `protoc-gen-xxx` 的形式

#### 修改test.proto
然后修改一下 `test.proto`

```proto3
syntax = "proto3";

package testfastpb;

import "ext.proto";

// 定义一个消息
message MyMessage {
  string name = 1 [(ext.my_custom_option) = "example"];
}
```

#### 生成代码
执行命令生成代码

`protoc -I . -I $GOPATH/src/github.com/gogo/protobuf/protobuf --plugin=protoc-gen-xxx=plugins/protoc-gen-xxx --xxx_out=paths=source_relative:testpluginpb test.proto`

然后我们就能够看到对应 `xxx.pb.go` 的文件按照我们的逻辑生成出来

```go
// .....
type MyMessage struct {
	Name                 string   `protobuf:"bytes,1,opt,name=name,proto3" json:"name,omitempty" my_custom_option:"example"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
// .....
```
