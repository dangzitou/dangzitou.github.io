---
title: "RPC 自定义协议编解码踩坑记录：bodyLength 读取错误导致越界"
date: 2025-06-13 15:00:00 +0800
categories: [Java, rpc, 学习记录]
tags: [Java, RPC, Vert.x, 网络编程, 协议设计, 调试]
---

## 前言

在实现一个自定义 RPC 框架（zyro-rpc）的过程中，我手动设计了一套应用层通信协议，并实现了对应的编码器（Encoder）和解码器（Decoder）。

在写完编解码器后，跑单元测试时遇到了一个报错：

```
java.lang.IllegalArgumentException: end must be greater or equal than start
```

排查过程不算复杂，但根因挺有代表性——**协议这种东西对字段顺序和偏移量极其敏感，写错了顺序，解码读到的就是一堆乱码**。本文记录一下排查和修复过程。

---

## 一、自定义协议结构设计

协议的 Header 固定长度，结构如下：

| 字段        | 类型   | 长度   | 偏移量  |
|-------------|--------|--------|---------|
| 魔数        | byte   | 1 字节 | 0       |
| 版本号      | byte   | 1 字节 | 1       |
| 序列化算法  | byte   | 1 字节 | 2       |
| 消息类型    | byte   | 1 字节 | 3       |
| 状态        | byte   | 1 字节 | 4       |
| 请求ID      | long   | 8 字节 | 5       |
| body 长度   | int    | 4 字节 | 13      |

Header 总计 17 字节，之后紧跟 body 数据。

解码器读取 body 的方式：
```java
byte[] bodyBytes = buffer.getBytes(17, 17 + header.getBodyLength());
```

逻辑很清晰：从 offset 17 开始，读取 `bodyLength` 个字节。

---

## 二、报错现场

运行 `testEncodeAndDecode` 单元测试，抛出：

```
[main] INFO ...ProtocolMessageEncoder - bodyLength: 351
[main] INFO ...ProtocolMessageDecoder - bodyLength: 11332864

java.lang.IllegalArgumentException: end must be greater or equal than start
    at io.vertx.core.impl.Arguments.require(Arguments.java:29)
    at io.vertx.core.buffer.impl.BufferImpl.getBytes(BufferImpl.java:222)
    at io.dangzitou.rpc.protocol.ProtocolMessageDecoder.decode(ProtocolMessageDecoder.java:35)
```

注意日志里的两个数字：

- 编码时：`bodyLength = 351`（正确）
- 解码时：`bodyLength = 11332864`（完全不对！）

解码器在 offset 13 读到的 bodyLength 是一个离谱的大数，导致 `getBytes(17, 17 + 11332864)` 越界。

---

## 三、定位根因

既然解码读错了 bodyLength，说明 offset 13 处的内容不是 bodyLength。那是什么？

去看编码器的实现：

```java
byte[] bodyBytes = serializer.serialize(protocolMessage.getBody());
buffer.appendBytes(bodyBytes);       // ← 第一次写入 body ！！！
log.info("bodyLength: {}", bodyBytes.length);
buffer.appendInt(bodyBytes.length);  // ← 写入 bodyLength
buffer.appendBytes(bodyBytes);       // ← 第二次写入 body
```

**问题找到了**：body 数据被写入了两次，而且第一次写在了 bodyLength 之前。

实际 Buffer 结构变成了：

```
[magic(1)] [version(1)] [serializer(1)] [type(1)] [status(1)] [requestId(8)]
[body数据(351字节)]        ← 多余的！
[bodyLength(4字节)]
[body数据(351字节)]
```

而解码器期望的结构是：

```
[magic(1)] [version(1)] [serializer(1)] [type(1)] [status(1)] [requestId(8)]
[bodyLength(4字节)]       ← offset 13
[body数据]                ← offset 17
```

解码器去 offset 13 读 int，读到的是 body 序列化数据的前 4 个字节，所以拿到了 `11332864` 这个乱码——再用它做长度去切 buffer，直接越界。

---

## 四、修复方案

编码器里删掉第一次多余的 `appendBytes`，保证写入顺序和协议设计完全一致：

```java
// 修复前
byte[] bodyBytes = serializer.serialize(protocolMessage.getBody());
buffer.appendBytes(bodyBytes);       // ❌ 删掉这行
log.info("bodyLength: {}", bodyBytes.length);
buffer.appendInt(bodyBytes.length);
buffer.appendBytes(bodyBytes);

// 修复后
byte[] bodyBytes = serializer.serialize(protocolMessage.getBody());
log.info("bodyLength: {}", bodyBytes.length);
buffer.appendInt(bodyBytes.length);  // ✅ 先写长度
buffer.appendBytes(bodyBytes);       // ✅ 再写数据
```

修复后再跑测试，编解码均正常，断言通过。

---

## 五、总结与反思

这个 bug 说起来很简单——多写了一行 `appendBytes`——但排查时如果不注意日志里那两个 bodyLength 的对比，很容易把方向搞错，去怀疑解码偏移量算错了。

几点值得记住的经验：

1. **协议编解码，字段写入顺序必须和协议文档严格对应**，任何多写、少写、顺序错误都会导致解码读到错误的字段值，而不会有任何编译期报错。

2. **日志是最快的排查手段**。编码器打出 `351`，解码器打出 `11332864`，这个差异直接指向了"Buffer 里 offset 13 的内容不是 bodyLength"，缩小了排查范围。

3. **写完编解码器后，先跑一遍 encode → decode 的单元测试**，比等到集成测试时排查要省事得多。

---

如果你在实现自定义协议时遇到类似的 `end must be greater or equal than start` 或长度越界问题，不妨先打印一下编解码两端的 bodyLength，看看是否一致——大概率是编码器写入顺序出了问题。
