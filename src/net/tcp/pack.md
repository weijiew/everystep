# TCP 粘包拆包问题

面试的时候被问到了，TCP 粘包和拆包问题，之前的项目中也涉及了这部分内容，写篇文章系统的总结一下。

### 什么是 TCP 粘包？

TCP粘包问题是由于TCP是一个面向字节流的协议，数据在传输过程中，可能会将多个数据包合并为一个数据包进行发送，这就是所谓的TCP粘包问题。这种情况通常发生在发送端的数据发送速度大于接收端的数据处理速度时。

```
发送端                      网络                     接收端
  |                         |                         |
  | -- 数据包A -->           |                         |
  | -- 数据包B -->           |                         |
  |                         | -- 数据包C (A + B) -->   |
  |                         |                         | -- 处理数据包C，分离出A和B
  |                         |                         |
```

在这个示例中，发送端连续发送了两个数据包A和B。由于网络的原因，这两个数据包在到达接收端时被合并为一个数据包C。接收端在接收数据时，期望能够按照数据包的边界接收数据，即先接收数据包A，然后接收数据包B。但是由于TCP的粘包问题，接收端实际上接收到的是一个大的数据包C，这就需要接收端自己去处理如何从数据包C中分离出原来的数据包A和B。

### TCP 粘包是怎么产生的？

TCP粘包问题是由于TCP协议的特性导致的。TCP是一个面向字节流的协议，这意味着TCP并不关心数据的边界，它只负责将数据作为一个连续的字节流发送出去。因此，当发送端连续发送多个数据包时，这些数据包可能会被合并为一个大的数据包进行发送，这就是所谓的TCP粘包问题。

相比之下，UDP是一个面向报文的协议，每个UDP数据包都是独立的，UDP保证了数据包的边界。在UDP中，数据包的边界是由UDP协议自身来保证的。每个UDP数据包都是独立的，包含了源端口号、目标端口号、长度和校验和等信息。当接收端接收到UDP数据包时，它可以通过这些信息来确定数据包的边界。因此，UDP不存在粘包问题。当接收端接收到UDP数据包时，它可以明确知道数据包的边界在哪里。

具体来说，UDP数据包的长度字段表示了UDP头部和数据部分的总长度，接收端可以通过这个长度字段来确定数据包的边界。因此，UDP不存在像TCP那样的粘包问题。

总的来说，TCP和UDP的主要区别在于TCP是面向连接的，提供可靠的数据传输服务，而UDP是无连接的，提供不可靠的数据传输服务。这也导致了TCP存在粘包问题，而UDP不存在粘包问题。

### 如何解决 TCP 粘包问题？

解决TCP粘包问题的常见方法有：

1. 在数据包之间添加特殊的分隔符，使得接收端可以通过这些分隔符来识别数据包的边界。

2. 在每个数据包的头部添加长度字段，表示数据包的长度，接收端通过读取长度字段，可以知道每个数据包的边界在哪里。

3. 使用固定长度的数据包，这样接收端可以直接通过数据包的长度来确定数据包的边界。

### 如何在数据包之间添加特殊的分隔符？

在这个示例中，发送端在每个数据包的末尾添加了一个特殊的分隔符'#'，接收端可以通过这个分隔符来识别数据包的边界。

```
发送端                      网络                     接收端
  |                         |                         |
  | -- 数据包A# -->          |                         |
  | -- 数据包B# -->          |                         |
  |                         | -- 数据包A#B# -->        |
  |                         |                         | -- 分离出数据包A和B
  |                         |                         |
```

这种方法常见于文本协议，如HTTP和SMTP。在这些协议中，数据包之间通常使用特殊的字符（如换行符或空格）作为分隔符。例如，HTTP协议中的请求和响应头部就是通过换行符来分隔的。当接收端接收到数据时，它可以通过这些分隔符来识别数据包的边界。

以下结合HTTP协议的报文结构来讲解这种方法：

HTTP协议是一种文本协议，它的报文结构主要包括起始行、头部字段和消息体三部分。起始行和头部字段之间、头部字段和消息体之间、以及头部字段之间都是通过换行符来分隔的。

例如，一个HTTP请求报文可能如下所示：

```
GET /index.html HTTP/1.1\r\n
Host: www.example.com\r\n
Connection: keep-alive\r\n
\r\n
```

在这个例子中，`GET /index.html HTTP/1.1`是起始行，`Host: www.example.com`和`Connection: keep-alive`是头部字段，它们之间都是通过`\r\n`（换行符）来分隔的。头部字段和消息体之间的空行（即连续的两个换行符）表示头部字段的结束和消息体的开始。

当接收端接收到这个HTTP请求报文时，它可以通过这些换行符来识别数据包的边界。例如，它可以先找到第一个换行符，然后读取起始行；然后再找到下一个换行符，读取第一个头部字段；以此类推，直到读取到连续的两个换行符，表示头部字段的结束和消息体的开始。

### 在头部设置长度

在这个示例中，发送端在每个数据包的头部添加了一个长度字段，表示数据包的长度，接收端通过读取长度字段，可以知道每个数据包的边界在哪里。

```
发送端                      网络                     接收端
  |                         |                         |
  | -- 数据包(3,A) -->       |                         |
  | -- 数据包(3,B) -->       |                         |
  |                         | -- 数据包(3,A)(3,B) -->  |
  |                         |                         | -- 分离出数据包A和B
  |                         |                         |
```


这种方法常见于二进制协议，如Protocol Buffers和Thrift。在这些协议中，每个数据包的头部通常会包含一个表示数据包长度的字段。当接收端接收到数据时，它可以通过读取这个长度字段来确定数据包的边界。例如，Protocol Buffers协议中的消息就是通过在消息头部添加一个表示消息长度的字段来解决粘包问题的。

以下是一个具体的例子，结合Protocol Buffers协议的报文结构来讲解这种方法：

Protocol Buffers（简称protobuf）是一种二进制协议，它的报文结构主要包括一个长度字段和一个数据字段。长度字段表示数据字段的长度。

例如，一个protobuf报文可能如下所示：

```
+----------------+------------------+
| 长度 (2 bytes)  | 数据 (n bytes)   |
+----------------+------------------+
```

在这个例子中，长度字段是2字节，表示数据字段的长度。数据字段是n字节，表示实际的数据。
当接收端接收到这个protobuf报文时，它可以先读取长度字段，然后根据长度字段的值来读取数据字段。这样，接收端就可以通过长度字段来确定数据包的边界。

这就是protobuf协议如何通过在每个数据包的头部添加长度字段来解决粘包问题的。

### 如何设置包长度固定

在这个示例中，发送端使用固定长度的数据包，这样接收端可以直接通过数据包的长度来确定数据包的边界。

```
发送端                      网络                     接收端
  |                         |                         |
  | -- 数据包A(5) -->        |                         |
  | -- 数据包B(5) -->        |                         |
  |                         | -- 数据包A(5)B(5) -->    |
  |                         |                         | -- 分离出数据包A和B
  |                         |                         |
```

这种方法在一些特定的场景中可能会被使用，例如在一些实时通信的协议中。在这些协议中，为了简化处理过程，所有的数据包都会被设计为固定长度。当接收端接收到数据时，它可以直接通过数据包的长度来确定数据包的边界。例如，一些音频流协议就可能会使用这种方法来解决粘包问题。

以下是一个具体的例子，结合音频流协议（如RTP）的报文结构来讲解这种方法：

实时传输协议（RTP）是一种面向数据包的协议，常用于音频和视频的实时传输。在RTP协议中，所有的数据包都被设计为固定长度，以简化处理过程。

例如，一个RTP数据包的结构可能如下所示：

```
+----------------+------------------+------------------+
| RTP头部 (12字节) | 有效载荷 (固定长度) | RTP尾部 (可选)   |
+----------------+------------------+------------------+
```

在这个例子中，RTP头部是12字节，有效载荷是固定长度，RTP尾部是可选的。当接收端接收到这个RTP数据包时，它可以直接通过数据包的长度来确定数据包的边界。

这就是RTP协议如何通过使用固定长度的数据包来解决粘包问题的。

### 什么是 TCP 拆包？

TCP 拆包是指 TCP 协议在传输数据时，将大的数据包拆分为多个小的数据包进行发送的过程。这是因为网络中的每个链路可能有不同的最大传输单元（MTU），超过 MTU 大小的数据包需要被拆分才能进行传输。

例如，假设我们有一个大小为 3000 字节的数据包需要通过 TCP 发送，而网络的 MTU 为 1500 字节。在这种情况下，TCP 协议会将这个数据包拆分为两个大小分别为 1500 字节的数据包进行发送。

```
发送方
+---------------------+
|     数据包 (3000字节) |
+---------------------+
          |
          | TCP 拆包
          v
+-----------------+   +-----------------+
| 数据包1 (1500字节)|  | 数据包2 (1500字节)|
+-----------------+   +-----------------+
          |
          | 发送
          v
接收方
+-----------------+   +-----------------+
| 数据包1 (1500字节)|  | 数据包2 (1500字节) |
+-----------------+   +-----------------+
          |
          | TCP 粘包
          v
+---------------------+
|     数据包 (3000字节) |
+---------------------+
```

在接收端，TCP 协议会将接收到的所有数据包重新组装成原始的数据包。这个过程被称为 TCP 粘包。

### 什么原因导致了 TCP 拆包？

TCP 拆包主要是由于网络中的每个链路可能有不同的最大传输单元（MTU）导致的。MTU 是指网络中一次能够传输的最大数据包大小。如果一个数据包的大小超过了 MTU，那么这个数据包就需要被拆分成多个小的数据包才能进行传输。

例如，假设我们有一个大小为 3000 字节的数据包需要通过 TCP 发送，而网络的 MTU 为 1500 字节。在这种情况下，TCP 协议会将这个数据包拆分为两个大小分别为 1500 字节的数据包进行发送。

这样做的原因是，如果一个数据包的大小超过了 MTU，那么这个数据包在传输过程中可能会被丢弃，导致数据无法成功传输。通过将大的数据包拆分为多个小的数据包，可以避免这种情况，确保数据能够成功传输。

此外，TCP 拆包还可以提高网络的利用率。如果一个大的数据包在传输过程中出现了错误，那么整个数据包都需要被重新传输。而如果将大的数据包拆分为多个小的数据包，那么即使其中一个小的数据包出现了错误，也只需要重新传输这个小的数据包，而不需要重新传输整个大的数据包。这样可以减少不必要的重传，提高网络的利用率。

### TCP 拆包是透明的吗？

TCP 拆包是由 TCP 协议自身处理的。当 TCP 协议在发送数据时，如果数据包的大小超过了网络的最大传输单元（MTU），TCP 协议会自动将数据包拆分为多个小的数据包进行发送。这个过程是完全透明的，对于上层协议和应用程序来说，它们并不需要知道数据包是否被拆分，也不需要进行任何额外的处理。

同样地，当 TCP 协议在接收数据时，如果接收到了多个被拆分的数据包，TCP 协议会自动将这些数据包重新组装成原始的数据包。这个过程也是完全透明的，对于上层协议和应用程序来说，它们只会看到完整的数据包，而不会看到被拆分的数据包。

因此，TCP 拆包和粘包是由 TCP 协议自身处理的，不需要上层协议进行进一步的解决。

### 总结

TCP粘包是由于TCP协议的字节流特性，导致在数据传输过程中，多个数据包可能被合并为一个数据包发送。这通常发生在发送端的数据发送速度大于接收端的数据处理速度时。解决TCP粘包问题的常见方法有：添加特殊的分隔符，设置长度字段，或使用固定长度的数据包。

TCP拆包则是由于网络中的每个链路可能有不同的最大传输单元（MTU），导致超过MTU大小的数据包需要被拆分为多个小的数据包进行传输。这个过程是由TCP协议自身处理的，对于上层协议和应用程序来说，是完全透明的。