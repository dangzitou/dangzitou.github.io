---

title: "Vert.x TCP 服务器处理消息的完整流程详解"

date: 2026-03-05 16:00:00 +0800

categories: [Java, 学习记录, rpc]

tags: [Java, Vert.x, TCP, RPC, 网络编程, 设计模式]

---

## 前言

在学习手写 RPC 框架的过程中，遇到了基于 Vert.x 实现的 TCP 服务器处理消息的一段代码。初看之下，既没有 `while` 循环，又有 Lambda、匿名内部类、事件驱动等多种概念交织在一起，让人摸不着头脑。本文从零开始，通俗地梳理这段代码的完整执行流程，希望能帮助同样在学习 Vert.x 或 RPC 框架的朋友理解这套设计。

---

## 一、整体架构：三层嵌套

整个 TCP 消息处理流程可以分为三层，每一层只负责自己的事情：

```
TcpServerHandler.handle(NetSocket)        ← 第一层：连接级别，负责初始化
    └── TcpBufferHandlerWrapper           ← 第二层：字节流处理，负责拆包
            └── buffer -> { 业务逻辑 }    ← 第三层：完整消息的业务处理
```

这种设计遵循了**单一职责原则**，每一层只关心自己的任务，通过 `Handler<Buffer>` 接口串联起来。

---

## 二、核心代码

### TcpServerHandler（第一层）

```java
public class TcpServerHandler implements Handler<NetSocket> {
    @Override
    public void handle(NetSocket netSocket) {
        TcpBufferHandlerWrapper tcpBufferHandlerWrapper = new TcpBufferHandlerWrapper(buffer -> {
            // 解码、反射调用、编码、写回响应
            ProtocolMessage<RpcRequest> protocolMessage =
                (ProtocolMessage<RpcRequest>) ProtocolMessageDecoder.decode(buffer);
            RpcRequest rpcRequest = protocolMessage.getBody();
            // ... 反射调用服务方法 ...
            Buffer encode = ProtocolMessageEncoder.encode(responseMessage);
            netSocket.write(encode);
        });
        netSocket.handler(tcpBufferHandlerWrapper);
    }
}
```

### TcpBufferHandlerWrapper（第二层）

```java
public class TcpBufferHandlerWrapper implements Handler<Buffer> {
    private final RecordParser recordParser;

    public TcpBufferHandlerWrapper(Handler<Buffer> bufferHandler) {
        recordParser = initRecordParser(bufferHandler);
    }

    @Override
    public void handle(Buffer buffer) {
        recordParser.handle(buffer);
    }

    private RecordParser initRecordParser(Handler<Buffer> bufferHandler) {
        RecordParser parser = RecordParser.newFixed(ProtocolConstant.MESSAGE_HEAD_LENGTH);
        parser.setOutput(new Handler<Buffer>() {
            int size = -1;
            Buffer resultBuffer = Buffer.buffer();

            @Override
            public void handle(Buffer buffer) {
                if (size == -1) {
                    size = buffer.getInt(13);
                    parser.fixedSizeMode(size);
                    resultBuffer.appendBuffer(buffer);
                } else {
                    resultBuffer.appendBuffer(buffer);
                    bufferHandler.handle(resultBuffer); // 交给业务层
                    parser.fixedSizeMode(ProtocolConstant.MESSAGE_HEAD_LENGTH);
                    size = -1;
                    resultBuffer = Buffer.buffer();
                }
            }
        });
        return parser;
    }
}
```

---

## 三、完整流程详解

### 第一阶段：连接建立（只执行一次）

当客户端连接到服务器的瞬间，Vert.x 的 Event Loop 检测到新连接，触发 `TcpServerHandler.handle(NetSocket netSocket)`。

这个方法只做**初始化配置**，不处理任何业务数据：

```
新连接进来
    → 创建 TcpBufferHandlerWrapper（内部同时创建好 RecordParser）
    → 把 buffer -> {业务逻辑} 存入 RecordParser 的 setOutput 里
    → netSocket.handler(tcpBufferHandlerWrapper)，把 wrapper 绑定到这个连接上
    → 初始化完成，静静等待数据
```

此时整个"管道"已经搭好，就像给新员工完成了入职培训，等待客户发来消息。

### 第二阶段：数据到达，读取包头

客户端发送了一条完整的 RPC 消息（假设协议格式为：前 17 字节是包头，后 N 字节是包体）。

```
原始字节流到达 netSocket
    → TcpBufferHandlerWrapper.handle(rawBuffer) 被触发
    → 内部直接调用 recordParser.handle(rawBuffer)，把字节喂给解析器
```

`RecordParser` 当前处于 `fixedSizeMode(17)` 状态，开始积累字节。凑够 17 字节后，触发 `setOutput` 里的 `handle`：

```
setOutput 的 handle 第一次被触发（size == -1）
    → 从这 17 字节里读取 buffer.getInt(13)，得到 body 长度，比如 size = 100
    → 调用 parser.fixedSizeMode(100)，告诉解析器下次凑够 100 字节再来叫我
    → 把这 17 字节的头写入 resultBuffer
    → 本次 handle 结束
```

### 第三阶段：数据到达，读取包体

`RecordParser` 切换到等待 100 字节的状态，继续积累字节流中剩余的数据。凑够 100 字节后，再次触发 `setOutput` 里的 `handle`：

```
setOutput 的 handle 第二次被触发（size == 100，不是 -1）
    → 把这 100 字节的体追加到 resultBuffer
    → 此时 resultBuffer 里装着完整的一条消息（17字节头 + 100字节体）
    → 调用 bufferHandler.handle(resultBuffer)，把完整消息交给业务层
    → 重置状态：parser.fixedSizeMode(17)，size = -1，resultBuffer = 新的空 Buffer
    → 等待下一条消息
```

### 第四阶段：业务处理

`bufferHandler.handle(resultBuffer)` 触发了 `TcpServerHandler` 里传入的那个 Lambda：

```
拿到完整的 resultBuffer
    → ProtocolMessageDecoder.decode(buffer)，解码成 ProtocolMessage<RpcRequest>
    → 取出 RpcRequest，拿到 serviceName、methodName、paramTypes、params
    → LocalRegistry.getService(serviceName)，从本地注册表找到实现类
    → 通过反射获取对应的 Method 对象并执行，拿到返回值 result
    → 把 result 封装进 RpcResponse
    → 把 header 的 type 改为 RESPONSE 类型
    → ProtocolMessageEncoder.encode(responseMessage)，编码成字节
    → netSocket.write(encode)，把响应写回客户端
```

### 完整流程总览

```
客户端发送字节流
        ↓
TcpBufferHandlerWrapper.handle()
        ↓
RecordParser 积累字节
        ↓ 凑够 17 字节（包头）
setOutput.handle()【第一次】
  读取 body 长度 → 切换 fixedSizeMode(N)
        ↓ 凑够 N 字节（包体）
setOutput.handle()【第二次】
  拼装完整消息 → 调用 bufferHandler.handle()
        ↓
业务逻辑 Lambda
  解码 → 反射调用 → 封装响应 → 编码 → 写回客户端
        ↓
状态重置，等待下一条消息
```

---

## 四、几个容易困惑的问题

### 为什么没有 `while` 循环，`handle` 却会被反复执行？

这是 Vert.x **事件驱动模型**的核心。循环并不是不存在，而是**藏在框架内部**。Vert.x 底层基于 Netty，有一个叫做 **Event Loop（事件循环）​** 的线程，它内部有一个真实的 `while` 循环，不断地监听所有连接的 IO 事件。一旦某个连接有数据到来，它就会调用你注册的 `handle` 方法。你看不到循环，是因为框架把"等待"和"处理"解耦了，你只需要专心写"处理"的逻辑。

### `size` 定义在 `Handler` 内部，为什么能跨调用保留值？

`size` 写在了 `handle` 方法的**外面**，是匿名 `Handler` 对象的**成员变量**，而不是方法内部的局部变量。这个匿名类对象只在 `setOutput` 被调用时创建**一次**，之后每次 `handle` 被触发，操作的都是**同一个对象的同一个字段**，因此 `size` 的值会一直保留，直到连接断开、对象被 GC 回收。

### `new TcpBufferHandlerWrapper(buffer -> {...})` 里的 Lambda 是什么？

这个 Lambda 的类型是 `Handler<Buffer>`，但它并不是处理原始字节的，而是**处理已经拼装完整的消息**的。它在构造函数里被传入，最终存放在 `setOutput` 的 `else` 分支里，只有当 `RecordParser` 凑齐了一条完整消息之后，才会被调用。

### Lambda 和匿名内部类有什么区别？

两者都能实现函数式接口，但有几个关键区别：

| | 匿名内部类 | Lambda |
|---|---|---|
| 能否有成员变量 | 能 | 不能 |
| `this` 指向 | 匿名类自身 | 外部类 |
| 底层实现 | 独立 `.class` 文件 | `invokedynamic` 指令 |

这也是为什么 `setOutput` 这里用了匿名内部类而不是 Lambda——因为需要 `size` 和 `resultBuffer` 这两个跨调用保留状态的成员变量，Lambda 做不到这一点。

---

## 五、设计亮点总结

这套代码把三个关注点完全分离开了：

- **​`TcpBufferHandlerWrapper`​**：只管拆包，解决 TCP 粘包/拆包问题，不关心消息内容是什么。
- **​`RecordParser` + `setOutput`​**：只管按协议格式切割字节，维护解析状态，不关心业务逻辑。
- **​`buffer -> { 业务逻辑 }`​**：只管处理一条完整的消息，不关心字节是怎么凑齐的。

每一层都只做自己的事，职责清晰，扩展方便。如果将来要换一套协议格式，只需要修改 `TcpBufferHandlerWrapper` 里的解析规则，业务层完全不需要动。

---
