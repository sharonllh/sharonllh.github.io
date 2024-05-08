---
layout: post
title: DDIA阅读笔记(4)：编码与演化
date: 2024-03-20 +0800
tags: [系统设计, DDIA]
categories: [系统设计]
---

## 编码与解码

程序通常有两种形式的数据：
- 在内存中，数据保存在类、数组、散列表等数据结构中。
- 在把数据写入文件或通过网络传输时，需要编码成字节序列。

编码（Encoding）：把内存对象转换成字节序列，又称序列化（Serialization）。

解码（Decoding）：把字节序列转换成内存对象，又称反序列化（Deserialization）。

要想把数据发送到不共享内存的另一个进程中，例如写入文件或网络传输，都需要编码。

## 编码格式

### JSON和XML

JSON和XML属于文本格式，可读性很好。它们的缺点：

- 数字的模糊性
  - XML无法区分数字和碰巧由数字组成的字符串。
  - JSON虽然区分数字和字符串，但是不区分整数和浮点数，并且不能指定精度。

- 不支持二进制数据：JSON和XML都不支持二进制数据，需要用Base64编码把二进制数据转成文本。

- 模式
  - XML的模式比较复杂。
  - JSON虽然有可选模式支持，但是许多基于JSON的工具不会使用模式。

### JSON和XML的二进制变体

JSON的二进制变体：MessagePack、BSON、BJSON、UBJSON、BISON、Smile

XML的二进制变体：WBXML、Fast Infoset

这些二进制变体使用二进制格式，但是没有改变JSON和XML的数据模型，通常只能节省一点点空间，使用并不广泛。

### Thrift和Protobuf

Thrift和Protobuf是基于相同原理的二进制编码方式。Thrift最初是Facebook开发的，Protobuf（Protocol Buffers）最初是Google开发的。

以下面的数据为例，采用JSON编码时有81个字节（删除了空白字符）。

```json
{
    "userName": "Martin",
    "favoriteNumber": 1337,
    "interests": ["daydreaming", "hacking"]
}
```

#### 定义模式

采用Thrift和Protobuf编码，首先需要使用接口定义语言（IDL）来定义模式。

Thrift的模式定义如下：

```c
struct Person {
    1: required string       userName,
    2: optional i64          favoriteNumber,
    3: optional list<string> interests
}
```

Protobuf的模式定义如下：

```protobuf
message Person {
    required string user_name       = 1;
    optional int64  favorite_number = 2;
    repeated string interests       = 3;
}
```

把字段定义成`required`还是`optional`不会影响编码的结果，只会影响对字段的检查。

自带的代码生成工具会根据模式定义生成相应编程语言中的类，供应用程序调用，从而进行编解码。

#### 编码方式

Thrift和Protobuf编码后，会生成二进制格式的字节序列。Thrift主要有两种二进制格式：BinaryProtocol和CompactProtocol，而Protobuf只有一种二进制格式。

Thrift的BinaryProtocol格式如下，有59个字节：

![Thrift BinaryProtocol](/assets/img/DDIA_Thrift_BinaryProtocol.png)

- type：数据类型
- field tag：字段编号，即模式定义中的数字
- length：字段长度

没有使用字段名，而是使用字段编号，所以会更加紧凑。

字段的值被编码为ASCII或UTF-8。

Thrift的CompactProtocol格式如下，有34个字节：

![Thrift CompactProtocol](/assets/img/DDIA_Thrift_CompactProtocol.png)

相比于BinaryProtocol，CompactProtocol有以下优化：
- field tag和type合并到一个字节中
- 可变长度整数：数字1337没有使用全部八个字节，而只用了两个字节

Protobuf格式如下，有33个字节：

![Protobuf](/assets/img/DDIA_Protobuf.png)

Protobuf和Thrift CompactProtocol很像，只差了一个字节。主要区别是，Thrift有专门的列表类型，而Protobuf没有列表类型，它用`repeated`表示字段是重复的。

#### 怎么处理模式更改？

模式更改后，需要保证两个兼容性：
- 向前兼容：旧代码可以处理新数据（新代码写入的数据）
- 向后兼容：新代码可以处理旧数据（旧代码写入的数据）

针对不同的模式更改方式，有不同的处理方法。具体如下：

- 修改字段

  - 可以修改字段名称，因为编码不会用到字段名称。

  - 不可以修改字段编号，否则会导致旧数据无效。

  - 可以修改字段的数据类型，但是可能会导致值的精度下降。例如，把一个32位的整数改成64位时，旧代码处理新数据时会截断数据，导致精度下降，而新代码处理旧数据时会填充缺失的位，没有精度损失。

- 增加字段

  - 新字段必须有新的编号，不能用之前用过的编号（即使对应字段已经被删除），避免和旧数据混淆。
  - 新字段必须是可选的或者有默认值的。
  - 旧代码处理新数据时，会忽略新加的字段。新代码处理旧数据时，新字段要么是空（可选的），要么是默认值。

- 删除字段

  - 只能删除可选字段，不能删除必需字段。
  - 旧代码处理新数据时，被删字段为空。新代码处理旧数据时，忽略被删字段。

### Avro

Avro也使用二进制格式，与Thrift和Protobuf的原理不同。

#### 定义模式

Avro有两种定义模式的方式：

1. Avro IDL（方便人工编辑）

    ```c
    record Person {
        string                userName;
        union { null, long }  favoriteNumber = null;
        array<string>         interests;
    }
   ```

2. 基于JSON（方便机器读取）

    ```json
    {
        "type": "record",
        "name": "Person",
        "fields": [
            {"name": "userName", "type": "string"},
            {"name": "favoriteNumber", "type": ["null", "long"], "default": null},
            {"name": "interests", "type": {"type": "array", "items": "string"}}
        ]
    }
    ```

与Thrift和Protobuf不同，Avro的模式中没有字段编号。

#### 编码方式

Avro的字节序列如下，有32个字节：

![Avro ByteSequence](/assets/img/DDIA_Avro_ByteSequence.png)

与Thrift和Protobuf不同，Avro的字节序列中没有字段标识和数据类型。在解码时，需要根据模式获取字段的类型。

#### 怎么处理模式更改？

Avro把模式分为两类：
- Writer模式：编码时使用的模式
- Reader模式：解码时使用的模式

Writer模式会以某种方式，随着字节序列一起写入文件或发送至网络，例如：
- 有很多记录的大文件：文件中所有的记录都是按照同一个Writer模式来编码的，文件的开头写入这个Writer模式。
- 数据库：每个记录包含一个版本号，每个版本号对应一个Writer模式，把Writer模式也写入数据库。
- 网络传输：两个进程进行网络通信时，在连接设置上协商模式版本。

解码时，同时查看Writer模式和Reader模式，并把Writer模式转换成Reader模式，从而保证兼容性。例如：

![Avro WriterToReader](/assets/img/DDIA_Avro_WriterToReader.png)

- 通过字段名来匹配字段
- 如果一个字段在Writer模式中有，而Reader模式中没有，就忽略它
- 如果一个字段在Reader模式中有，而Writer模式中没有，就设为默认值

为了保证兼容性，更改模式时也有要求：
- 添加或删除字段：只能添加或删除有默认值的字段
- 修改字段：可以修改字段类型，不能修改字段名称

#### 使用场景

由于Avro不存储字段编号，它非常适用于动态生成的模式。

另外，Avro既可以像Thrift和Protobuf一样，为静态语言提供代码生成功能，又可以不生成代码直接使用，因为Writer模式是自带的。

## 数据流的类型

数据从一个进程流向另一个不共享内存的进程，可以通过多种方式：
- 通过数据库
- 通过服务调用
- 通过异步消息传递

### 数据库中的数据流

在数据库中，写入数据时编码，读取数据时解码。写入的进程和读取的进程可能是同一个，也可能不是。

需要考虑向前和向后的兼容性。举个例子，假设新代码添加了一个新字段，并写入数据库，而旧代码读取了记录，更新并写回，此时旧代码应该保证新字段不会丢失。

### 服务调用中的数据流

通过网络进行进程间的通信时，通常有客户端和服务器两个角色。服务器通过网络公开API（该API称为服务），客户端连接到服务器来调用该API。

服务器本身可以是另一个服务的客户端。在微服务架构中，将大型应用程序按照功能区域分解为较小的服务，当一个服务需要另一个服务的某些功能或数据时，就会向另一个服务发起请求。

为了使应用程序更容易更改和维护，需要保证每个服务都可以独立部署和演化。

#### Web服务

如果服务采用HTTP作为底层通信协议，就称为Web服务。Web服务有两种常见的类型：REST和SOAP。

REST不是一个协议，而是一个设计哲学。它的数据格式很简单，用URL来标识资源，用HTTP功能来进行缓存控制、身份验证和内容类型协商。根据REST原则设计的API称为RESTful。

SOAP是一种基于XML的协议。它避免使用大多数的HTTP功能，相关标准很复杂。基于SOAP的API使用WSDL（Web服务描述语言）来描述，需要依赖代码生成工具。

REST风格的API比SOAP更简单，所以更加常用。

#### 远程过程调用（RPC）

RPC的基本思想是，向远程网络中的服务发起请求，看起来就像在调用同一进程中的函数一样。这种抽象称为位置透明。

然而，网络请求和本地函数调用是非常不同的：
- 本地函数调用的结果是可预测的，网络请求却不是，因为会有网络数据丢失、远程服务不可用等不可控因素。
- 重试失败的网络请求需要考虑幂等性（idempotence），因为有可能请求已经完成了，只是响应丢失了而已。本地函数调用就不会有这个问题。
- 本地函数调用的时间是大致相同的，网络请求的时间却变动很大。
- 本地函数调用可以直接传递内存中的对象，网络请求却需要进行编码。
- 网络请求的两端可能使用不同的编程语言，所以RPC框架需要把数据类型从一种语言翻译成另一种语言。本地函数调用就没有这个问题。

RPC框架包括：
- gRPC
- Thrift RPC
- Avro RPC
- Finagle
- Rest.li

RPC和RESTful API的对比：
- 使用二进制编码格式的RPC比使用JSON的RESTful API拥有更好的性能
- RESTful API更加方便测试，支持所有主流的平台和编程语言，生态系统更好，有大量可用的工具

公共API通常用REST，公司内部服务之间的通信可以用RPC。

### 异步消息传递中的数据流

在异步消息传递中，发送方的消息先临时存储到消息队列中，再异步地发送到接收方进行处理。这种通信通常是单向的，发送者发布完消息就结束了，不会等待消息被传递，也不会收到消息的回复。

使用消息队列的好处：
- 如果接收方过载或者不可用，可以充当缓冲区，从而提高系统的可靠性
- 使发送方和接收方解耦，发送方只需要发布消息，不用关系接收方
- 自动重试，防止消息丢失
- 允许把一条消息发送给多个接收方
- 发送方不需要知道接收方的IP地址和端口号，这在云部署中特别有用

消息队列的产品有：
- Kafka
- RabbitMQ
- ActiveMQ
- HornetQ
- NATS

消息队列的使用方式通常为：生产者把消息发送到指定的主题，消息队列确保把消息传递给这个主题的所有消费者。同一个主题可以有多个生产者和消费者。

消息只是包含一些元数据的字节序列，可以使用任何编码格式。需要保证编码的兼容性，从而可以对生产者和消费者的程序进行独立修改和部署。

一个主题的数据流是单向的，但是一个主题的消费者可能会发布消息到另一个主题，此时这个消费者就是另一个主题的生产者。为了保证兼容性，这个消费者在发布消息时需要注意保留未知字段。