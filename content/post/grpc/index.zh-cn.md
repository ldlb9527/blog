+++

author = "旅店老板"
title = "grpc基础"
date = "2023-03-02                                                                                                                                                                                                                                                                                                                                            "
description = "grpc基础学习"
tags = [
	"redis",
]
categories = [
    "redis",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "redis.png"
mermaid = true
+++

## 定义
* **RPC**：RPC(Remote Procedure Call)又名远程过程调用，是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一个地址空间  
（通常为一个开放网络的一台计算机）的子程序，而程序员就像调用本地程序一样，无需关注其细节。RPC是一种Client/Server模式。
* **GRPC**:GRPC是一款基于RPC协议实现的、高性能的、基于protoBuf序列化协议开发的RPC框架。
## RPC调用过程
![rpc调用过程](rpc.png "rpc调用过程")
## proto3语法
### 变量类型与各平台映射
| .proto   | 说明                                                         | C++    | Java       | Python      | Go      | Ruby                           | C#         | PHP            |
| -------- | ------------------------------------------------------------ | ------ | ---------- | ----------- | ------- | ------------------------------ | ---------- | -------------- |
| double   |                                                              | double | double     | float       | float64 | Float                          | double     | float          |
| float    |                                                              | float  | float      | float       | float32 | Float                          | float      | float          |
| int32    | 使用变长编码，对负数编码效率低，<br>如果你的变量可能是负数，可以使用sint32 | int32  | int        | int         | int32   | Fixnum or Bignum (as required) | int        | integer        |
| int64    | 使用变长编码，对负数编码效率低，<br>如果你的变量可能是负数，可以使用sint64 | int64  | long       | int/long    | int64   | Bignum                         | long       | integer/string |
| uint32   | 使用变长编码                                                 | uint32 | int        | int/long    | uint32  | Fixnum or Bignum (as required) | uint       | integer        |
| uint64   | 使用变长编码                                                 | uint64 | long       | int/long    | uint64  | Bignum                         | ulong      | integer/string |
| sint32   | 使用变长编码，带符号的int类型，<br>对负数编码比int32高效         | int32  | int        | int         | int32   | Fixnum or Bignum (as required) | int        | integer        |
| sint64   | 使用变长编码，带符号的int类型，<br>对负数编码比int64高效         | int64  | long       | int/long    | int64   | Bignum                         | long       | integer/string |
| fixed32  | 4字节编码，如果变量经常大于的<br>话，会比uint32高效             | uint32 | int        | int         | int32   | Fixnum or Bignum (as required) | uint       | integer        |
| fixed64  | 8字节编码，如果变量经常大于的<br>话，会比uint64高效             | uint64 | long       | int/long    | uint64  | Bignum                         | ulong      | integer/string |
| sfixed32 | 4字节编码                                                    | int32  | int        | int         | int32   | Fixnum or Bignum (as required) | int        | integer        |
| sfixed64 | 8字节编码                                                    | int64  | long       | int/long    | int64   | Bignum                         | long       | integer/string |
| bool     |                                                              | bool   | boolean    | bool        | bool    | TrueClass/FalseClass           | bool       | boolean        |
| string   | 必须包含utf-8编码或者7-bit ASCII text                        | string | String     | str/unicode | string  | String (UTF-8)                 | string     | string         |
| bytes    | 任意的字节序列                                               | string | ByteString | str         | []byte  | String (ASCII-8BIT)            | ByteString | string         |

### 定义一个Message
```protobuf
syntax = "proto3"; //指定使用proto3，如果不指定的话，编译器会使用proto2去编译

message SearchRequests {
    // 定义SearchRequests的成员变量，需要指定：变量类型、变量名、变量Tag
    string query = 1;
    int32 page_number = 2;
    int32 result_per_page = 3;
}
```
* SearchRequests有三个字段，两个整形和一个string类型
* 在消息定义中，每个字段都有唯一标识符Tag。用来在消息的二进制格式中识别各个字段，1-15在编码时占用一个字节，16-2047在编码时占用2个字节
* 最小标识号为1，最大标识号为2<sup>29</sup>-1。19000－19999为 Protobuf协议的预留标识号
***
### 定义多个Message
* 方式一：
```protobuf
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message SearchResponse {
  repeated string result = 1;
}
```
***
* 方式二：
```protobuf
syntax = "proto3";
message SearchResponse {
    message Result {
        string url = 1;
        string title = 2;
        repeated string snippets = 3;
    }
    repeated Result results = 1;
}

message SomeOtherMessage {
  SearchResponse.Result result = 1; //定义在内部的message的使用
}
```
* message可以嵌套定义，比如message可以定义在另一个message内部
***
### 定义枚举
```protobuf
syntax = "proto3";
message SearchRequest {
    string query = 1;
    int32 page_number = 2; // 页码
    int32 page_size = 3; // 每页条数
    enum Corpus {
        UNIVERSAL = 0;
        WEB = 1;
        IMAGES = 2;
        LOCAL = 3;
        NEWS = 4;
        PRODUCTS = 5;
        VIDEO = 6;
    }
    Corpus corpus = 4;
}
```
* 枚举可以定义在一个message内部或message外部
* 如果枚举是定义在message内部，外部message可以通过 `MessageType.EnumType` 的方式引用
* proto3第一个枚举值必须是0，枚举值不能重复，除非使用 `option allow_alias = true` 选项来开启别名
```protobuf
syntax = "proto3";
enum EnumAllowingAlias {
    option allow_alias = true;
    UNKNOWN = 0;
    STARTED = 1;
    RUNNING = 1;
}
```
***
### 指定变量规则
* 在proto3中，可以给变量指定两个规则：`singular`:可选字段（0或者1个，但不能多于1个),**proto3语法的默认字段规则，显示使用会报错**;`repeated`:可重复字段（任意数量，包括0）
* 在proto2中，规则为：`required`：必须具有这个字段;`optional`：可选字段（0或者1个）;`repeated`：可重复字段（任意数量，包括0）
```protobuf
syntax = "proto3";
message SearchRequests {
  string query = 1;
  repeated int32 page_number = 2; //列表
  map<string, int32> projects = 3; //map的key必须是string或者int,value可以是任意类型
}
```
* map类型不能使repeated
***
### 定义Service
```protobuf
syntax = "proto3";

service User {
  rpc UserIndex(UserIndexRequest) returns (UserIndexResponse) {}
}

message UserIndexRequest {
  int32 page = 1;
  int32 page_size = 2;
}

message UserIndexResponse {
  int32 err = 1;
  string msg = 2;
  // 返回一个 UserEntity 对象的列表数据
  repeated UserEntity data = 3;
}

message UserEntity {
  string name = 1;
  int32 age = 2;
}
```

***
### 引用其他 proto 文件
```protobuf
// 在1.proto中
//import "2.proto";

// 在2.proto中
//import "3.proto";   //1中导入2，2中导入3，此时1无法使用3中定义的内容

// 在1.proto中
//import "2.proto";

// 在2.proto中
//import public "3.proto";   //1中导入2，2中导入3，此时1可以使用3中定义的内容
//2都能使用3中定义的内容
```
***
### Options
Options分为**file-level options**（只能出现在最顶层，不能在消息、枚举、服务内部使用）、 
**message-level options**（只能在消息内部使用）、**field-level options**（只能在变量定义时使用）。
* **file-level options**：  
**1**.  java_package：指定生成类的包名，如果没有指定此选项，将由关键字package指定包名，只在生成java代码时有效  
**2**.  java_multiple_files:如果为true，定义在最外层的message 、enum、service将作为单独的类存在   
**3**.java_outer_classname:指定最外层class的类名，如果不指定，将会以文件名作为类名
### Demo
```protobuf
syntax = "proto3";

import "google/protobuf/any.proto";

service UserService {
  rpc Query(QueryRequest) returns (Response) {}
  rpc Delete(DeleteRequest) returns (Response) {}
}

message QueryRequest {
  int32 page = 1;
  int32 size = 2;
}

message DeleteRequest {
  int32 id = 1;
}

message Response {
  int32 err = 1;
  string msg = 2;
  google.protobuf.Any data = 3;
}

message User {
  string name = 1;
  int32 age = 2;
}

```
* `protoc -I . ./user.proto --go_out=plugins=grpc:.`


