---
title: "Spring Boot + 阿里云 OSS 上传文件时报 `Access key id should not be null or empty` 问题记录"
date: 2025-09-21 23:41:21 +0800
categories: [Java, 学习记录]
tags: [Java, SpringBoot, 阿里云OSS, 环境变量]
---

## 问题描述

在 Spring Boot 项目中集成阿里云 OSS 上传功能时，启动日志正常，但在上传文件时报错：
```shell
2025-09-21 22:53:56.720 [http-nio-8080-exec-6] INFO  io.dangzitou.controller.UploadController-文件上传: image (1).png
2025-09-21 22:53:56.733 [http-nio-8080-exec-6] ERROR o.a.c.c.C.[.[localhost].[/].[dispatcherServlet]-Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed: com.aliyun.oss.common.auth.InvalidCredentialsException: Access key id should not be null or empty.] with root cause
com.aliyun.oss.common.auth.InvalidCredentialsException: Access key id should not be null or empty.
	at com.aliyun.oss.common.auth.EnvironmentVariableCredentialsProvider.getCredentials(EnvironmentVariableCredentialsProvider.java:44)
	...
```
说明程序并没有正确读取到配置的 `OSS_ACCESS_KEY_ID`。

## 排查过程

### 1. 设置环境变量

一开始我是通过 **CMD 命令行**(通过管理员身份打开) 来设置环境变量的：

```bat
set OSS_ACCESS_KEY_ID=xxxxxxxxxxxx
set OSS_ACCESS_KEY_SECRET=xxxxxxxxxxxx

setx OSS_ACCESS_KEY_ID "%OSS_ACCESS_KEY_ID%"
setx OSS_ACCESS_KEY_SECRET "%OSS_ACCESS_KEY_SECRET%"
```
然后验证：

```bat
echo %OSS_ACCESS_KEY_ID%
echo %OSS_ACCESS_KEY_SECRET%
```
在 CMD 下是能正常显示的。

## 2. 为什么程序里是 null？
后来发现，我把变量写在了**用户变量**里，而不是**系统变量**。

区别在于：

- 用户变量：只对当前登录的用户有效。

- 系统变量：对所有用户有效，包括以服务方式运行的程序（例如 Tomcat）。

而 Tomcat/IDEA 很可能不是在我的用户上下文下运行的，所以自然读取不到。

## 3.解决过程

#### 把ACCESSKEY配置成系统变量

1. `Win + R` → 输入 `sysdm.cpl` → 打开 系统属性。
2. 切换到 高级 → 点击 环境变量。
3. 在 系统变量 部分新增：
```ini
OSS_ACCESS_KEY_ID=你的id
OSS_ACCESS_KEY_SECRET=你的secret
```
4. 保存后，重启电脑。

## 4.总结

报错原因是 阿里云 OSS SDK 没能读取到 OSS_ACCESS_KEY_ID 和 OSS_ACCESS_KEY_SECRET，而我只设置了 用户变量，Tomcat/IDE 并没有继承到。

解决方法有三种：

- 改成系统变量（快速解决）。

- 写进 Tomcat 启动脚本。

- 直接写在 Spring Boot 的配置文件里，方便管理和部署。

后面两种方法还没尝试过，有时间试试。

---
