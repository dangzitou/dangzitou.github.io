---
title: "手写 RPC 框架踩坑记录：自定义 TCP 协议调试全过程"
date: 2026-03-04 20:27:00 +0800
categories: [Java, 学习笔记, rpc]
tags: [RPC, Java, Vert.x, TCP, 自定义协议, 踩坑]

---

## 前言

最近在跟着编程导航鱼皮的教程手写 RPC 框架，第 7 章要求将原有的 HTTP 传输改为自定义 TCP 协议。改完之后消费者一直报错，折腾了不少时间，在此记录一下完整的排查和修复过程，希望能帮到同样踩坑的朋友。

---

## 一、背景介绍

本项目基于 Vert.x 构建 RPC 框架，第 7 章的目标是：

- 将原有的 `VertxHttpServer` 替换为 `VertxTcpServer`
- 自定义消息结构（17 字节消息头 + 消息体）
- 实现 `ProtocolMessageEncoder` 和 `ProtocolMessageDecoder`
- 实现 `TcpServerHandler` 处理请求并回写响应

---

## 二、坑一：`Invalid magic number: 72`

### 报错现象

消费者调用服务时，抛出异常：

```
RuntimeException: 消息 magic 非法 (Invalid magic number: 72)
```

### 排查过程

72 转成十六进制是 `0x48`，对应 ASCII 字符 `'H'`。联想到 `VertxTcpServer` 中有一个示例方法：

```java
private byte[] handleRequest(byte[] requestData) {
    return "Hello, client!".getBytes(); // 'H' = 72
}
```

服务端返回的不是自定义协议格式的响应，而是这个示例字符串。消费者的解码器读取第一个字节时，拿到的是 `'H'` 而不是协议魔数 `0x1`，因此报 magic 非法。

### 根本原因

`VertxTcpServer` 的 `connectHandler` 里仍然在调用旧的 `handleRequest()` 示例逻辑，没有替换成真正的 `TcpServerHandler`。

### 解决方案

将 `VertxTcpServer` 的 `connectHandler` 改为：

```java
server.connectHandler(new TcpServerHandler());
```

---

## 三、坑二：消费者超时 `TimeoutException`，但 Provider 日志显示服务调用成功

### 报错现象

修复坑一之后，Provider 日志打印了正确的业务结果（例如"用户名:dangzitou"），说明服务调用成功。但消费者侧等待 5 秒后抛出：

```
java.util.concurrent.TimeoutException
```

### 排查过程

既然 Provider 业务逻辑执行成功了，那问题一定出在**响应的编码和回写**环节。仔细检查 `TcpServerHandler` 的代码，发现了如下问题：

```java
// ❌ 错误代码
ProtocolMessage.Header header = protocolMessage.getHeader();
header.setType((byte) ProtocolMessageTypeEnum.RESPONSE.getKey());

ProtocolMessage<RpcResponse> responseMessage = new ProtocolMessage<>(); // body 和 header 都是 null！

try {
    Buffer encode = ProtocolMessageEncoder.encode(responseMessage);
    netSocket.write(encode);
} catch (Exception e) {
    throw new RuntimeException("Failed to encode protocol message");
}
```

`responseMessage` 是一个空对象，`header` 和 `body` 都没有设置进去。

### 根本原因

编码器 `ProtocolMessageEncoder.encode()` 内部有如下判断：

```java
if (protocolMessage == null || protocolMessage.getHeader() == null) {
    return Buffer.buffer(); // 直接返回空 Buffer！
}
```

由于 `responseMessage` 的 `header` 为 `null`，编码器直接返回了空 Buffer。`netSocket.write(Buffer.buffer())` 向客户端写了一个空内容，消费者自然收不到任何有效数据，一直等到超时。

### 解决方案

构建 `responseMessage` 时，把 `header` 和 `rpcResponse` 都传入：

```java
// ✅ 正确代码
ProtocolMessage.Header header = protocolMessage.getHeader();
header.setType((byte) ProtocolMessageTypeEnum.RESPONSE.getKey());

ProtocolMessage<RpcResponse> responseMessage = new ProtocolMessage<>(header, rpcResponse);

try {
    Buffer encode = ProtocolMessageEncoder.encode(responseMessage);
    netSocket.write(encode);
} catch (Exception e) {
    throw new RuntimeException("Failed to encode protocol message: " + e.getMessage(), e);
}
```

---

## 四、完整修复后的 `TcpServerHandler`

```java
public class TcpServerHandler implements Handler<NetSocket> {
    @Override
    public void handle(NetSocket netSocket) {
        netSocket.handler(buffer -> {
            // 解码请求
            ProtocolMessage<RpcRequest> protocolMessage;
            try {
                protocolMessage = (ProtocolMessage<RpcRequest>) ProtocolMessageDecoder.decode(buffer);
            } catch (Exception e) {
                throw new RuntimeException("Failed to decode protocol message: " + e.getMessage(), e);
            }
            RpcRequest rpcRequest = protocolMessage.getBody();

            // 反射调用服务
            RpcResponse rpcResponse = new RpcResponse();
            try {
                Class<?> implClass = LocalRegistry.getService(rpcRequest.getServiceName());
                Method method = implClass.getMethod(rpcRequest.getMethodName(), rpcRequest.getParamTypes());
                Object result = method.invoke(implClass.getDeclaredConstructor().newInstance(), rpcRequest.getParams());
                rpcResponse.setData(result);
                rpcResponse.setDataType(method.getReturnType());
                rpcResponse.setMessage("ok");
            } catch (Exception e) {
                e.printStackTrace();
                rpcResponse.setMessage(e.getMessage());
                rpcResponse.setException(e);
            }

            // 编码并回写响应（注意：header 和 body 都要设置！）
            ProtocolMessage.Header header = protocolMessage.getHeader();
            header.setType((byte) ProtocolMessageTypeEnum.RESPONSE.getKey());
            ProtocolMessage<RpcResponse> responseMessage = new ProtocolMessage<>(header, rpcResponse);
            try {
                Buffer encode = ProtocolMessageEncoder.encode(responseMessage);
                netSocket.write(encode);
            } catch (Exception e) {
                throw new RuntimeException("Failed to encode protocol message: " + e.getMessage(), e);
            }
        });
    }
}
```

---

## 五、经验总结

**坑一的教训**：教程中的示例代码（如 `"Hello, client!"` 这类占位逻辑）在推进到下一步时一定要及时替换，否则会产生非常迷惑的报错。72 这个魔数值如果不了解 ASCII 编码，很难第一时间联想到原因。

**坑二的教训**：在构建响应消息对象时，一定要检查所有字段是否都已正确赋值再传入编码器。编码器对 `null` 的防御性处理（返回空 Buffer）不会抛异常，这导致问题被静默掩盖，表现为消费者超时而非明显报错，排查起来非常隐蔽。

---
