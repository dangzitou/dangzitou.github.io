---
title: "Debugging "Service not found" Issue in RPC Framework"
date: 2026-02-28 10:00:00 +0800
categories: [教程]
tags: [RPC, Java, Etcd, ZooKeeper, 服务注册与发现, 分布式系统, 动态代理, 异常排查, Spring Boot, Maven]
---

## 问题描述

在使用自定义 RPC 框架时，消费者（Consumer）在调用服务时抛出以下异常：

```java
Exception in thread "main" java.lang.RuntimeException: Service not found: io.dangzitou.example.common.service.UserService
    at io.dangzitou.rpc.proxy.ServiceProxy.invoke(ServiceProxy.java:63)
    at jdk.proxy2/jdk.proxy2.$Proxy2.getUser(Unknown Source)
    at io.dangzitou.example.consumer.ConsumerExample.main(ConsumerExample.java:25)
```

这个错误表明消费者在注册中心找不到服务提供者注册的信息。

---

## 问题分析

找不到服务，无非就几种原因：

### 1. 服务端未启动或注册失败

消费者尝试调用服务时，服务端必须先启动并成功将自己注册到注册中心（如 ZooKeeper 或 Etcd）中。如果服务端未启动或注册失败，注册中心中就没有该服务的节点信息。

### 2. 服务键名（Service Key）生成逻辑不一致

服务注册和服务发现必须使用完全相同的键名（Service Key）。

在 `ServiceMetaInfo.java` 中，服务键名的生成逻辑如下：

```java
public String getServiceKey() {
    return String.format("%s/%s", serviceName, serviceVersion);
}
```

生成的键名类似于：`io.dangzitou.example.common.service.UserService/1.0`。

在 `ZooKeeperRegistry.java` 中，注册服务时：

```java
.name(serviceMetaInfo.getServiceKey())
```

发现服务时：

```java
serviceDiscovery.queryForInstances(serviceName);
```

如果键名中包含 `/`，Curator 的 `ServiceDiscovery` 可能会将其视为多级路径，导致注册和查询的路径不一致。

### 3. 注册中心路径分隔符冲突

ZooKeeper 和 Etcd 都使用 `/` 作为路径分隔符。如果服务键名中包含 `/`，可能会与路径分隔符混淆，导致注册和查询失败。

---

## 解决方案

### 1. 修改服务键名生成逻辑

将服务键名中的 `/` 替换为 `:`，避免与路径分隔符冲突。

修改 `ServiceMetaInfo.java`：

```java
public String getServiceKey() {
    return String.format("%s:%s", serviceName, serviceVersion);
}

public String getServiceNodeKey() {
    return String.format("%s:%s/%s:%s", serviceName, serviceVersion, serviceHost, servicePort);
}
```

### 2. 清理注册中心旧数据

修改键名后，需要清理注册中心中的旧数据（如 ZooKeeper 或 Etcd 中的节点），以确保新键名能够正确注册。

### 3. 确保服务端先启动

服务端必须先启动并成功注册服务，消费者才能发现并调用服务。

### 4. 成功解决问题

---

## 总结

这个问题的根本原因是服务键名生成逻辑与注册中心的路径处理规则不匹配。通过统一键名格式（使用 `:` 而非 `/`）并清理旧数据，可以解决该问题。

完整的修复步骤：

1. 修改 `ServiceMetaInfo.java`，将键名中的 `/` 替换为 `:`。
2. 清理注册中心中的旧数据。
3. 确保服务端先启动并成功注册服务。
4. 启动消费者，验证服务调用是否成功。

通过以上步骤，可以确保服务注册和发现逻辑一致，避免类似问题再次发生。
