---
title: "WSL2 + etcd + etcdkeeper 环境搭建踩坑记录"
date: 2026-03-05 21:00:00 +0800
categories: [Java, 学习记录, rpc]
tags: [WSL2, etcd, etcdkeeper, Java, RPC, 环境搭建, 踩坑]
---

## 前言

最近在写一个自己的 RPC 框架，使用 etcd 作为注册中心，etcdkeeper 作为 Web 管理界面。之前一直能正常使用，某天突然发现服务提供方无法注册服务，etcdkeeper 网页端也一直加载不出来，折腾了整整一个晚上才彻底搞清楚原因。本文记录一下完整的踩坑过程和解决方案，希望能帮到遇到同样问题的朋友。

---

## 一、环境说明

| 组件 | 运行环境 | 地址 |
|------|----------|------|
| etcd 3.6.8 | WSL2 (Ubuntu 24.04) | `0.0.0.0:2379` |
| etcdkeeper v0.7.8 | Windows | `0.0.0.0:8083` |
| Java 后端 | Windows (IDEA) | 连接 `localhost:2379` |

---

## 二、踩坑过程

### 坑一：WSL2 终端代理环境变量残留

在 WSL2 里执行 `curl http://127.0.0.1:2379/version` 时报错：

```
curl: (7) Failed to connect to 127.0.0.1 port 7897 after 0 ms: Couldn't connect to server
```

请求没有直接连 etcd，而是被转发到了 `127.0.0.1:7897`（Clash 代理端口）。原因是 WSL2 终端里存在代理环境变量残留：

```bash
echo $http_proxy   # http://127.0.0.1:7897
echo $https_proxy  # http://127.0.0.1:7897
```

**解决方法：​**

```bash
unset http_proxy
unset https_proxy
unset ALL_PROXY
```

要彻底解决，检查 `~/.bashrc` 或 `~/.zshrc` 里是否有代理相关的 export 语句，有的话删掉：

```bash
grep -i proxy ~/.bashrc ~/.zshrc 2>/dev/null
```

---

### 坑二：etcd 进程被 wsl --shutdown 杀掉后没有重启

etcd 不是系统服务，WSL2 重启或执行 `wsl --shutdown` 后进程会消失。确认 etcd 是否在运行：

```bash
ps aux | grep etcd
```

如果没有 etcd 进程（只有 grep 自身），说明 etcd 已经停了，需要重新手动启动。

---

### 坑三：WSL2 NAT 模式下 Windows 无法通过 localhost 访问 WSL2 内的服务

**这是本次问题的根本原因。​**

WSL2 默认使用 NAT 网络模式，在这种模式下：

- WSL2 有独立的虚拟 IP（如 `172.x.x.x`）
- Windows 进程（etcdkeeper、Java 后端）用 `localhost` 或 `127.0.0.1` **无法稳定访问** WSL2 内部的服务
- etcd 虽然监听在 `0.0.0.0:2379`，但 Windows 侧的端口转发并不总是可靠

**具体现象：​**

- etcdkeeper 网页一直空白，F12 Network 面板里请求全部 pending
- Java 后端启动后卡在注册中心初始化阶段，日志停在这里不动：

```
INFO io.dangzitou.rpc.RpcApplication - rpc registry init, config:RegistryConfig(registry=etcd, address=http://localhost:2379 ...)
```

- Windows PowerShell 里执行 `curl http://localhost:2379/version` 无法连接

**解决方法：开启 WSL2 镜像网络模式（mirrored networking）​**

在 Windows PowerShell 里执行：

```powershell
notepad "$env:USERPROFILE\.wslconfig"
```

写入以下内容并保存：

```ini
[wsl2]
networkingMode=mirrored
```

然后重启 WSL2：

```powershell
wsl --shutdown
```

重启后 WSL2 和 Windows 共享同一个网络栈，`localhost` 在两侧完全互通。

验证是否生效：

```powershell
wsl --status
# 输出中应包含 networkingMode: mirrored
```

> **注意：​** `wsl --shutdown` 会杀掉所有 WSL2 进程，包括 etcd，重启后需要重新启动 etcd，否则 `curl http://localhost:2379/version` 依然无响应。这也是第一次操作后感觉没生效的原因——不是镜像模式没起作用，而是忘了重启 etcd。

---

### 坑四：每次启动脚本都删除 etcd 数据目录

原启动脚本里有这一行：

```bat
wsl -d Ubuntu-24.04 bash -c "pkill etcd 2>/dev/null; rm -rf ~/etcd-data"
```

`rm -rf ~/etcd-data` 会删除 etcd 的全部数据，导致每次启动后之前注册的服务全部丢失。除非需要重置环境，否则应该去掉这一行，只保留 `pkill etcd`。

---

## 三、最终一键启动脚本

```bat
@echo off
chcp 65001 >nul
title ETCD 控制台 + 命令行客户端

echo ========================================
echo   ETCD + ETCDKEEPER + CLI 一键启动
echo ========================================
echo.

:: --- 第一步：清理旧 etcd 进程（保留数据） ---
echo [1/4] 正在清理旧环境...
wsl -d Ubuntu-24.04 bash -c "/usr/bin/pkill -f etcd 2>/dev/null; sleep 1"
echo 清理完成

:: --- 第二步：启动 WSL2 中的 etcd ---
echo.
echo [2/4] 正在启动 Etcd Server (新窗口)...
start "WSL-ETCD (服务端-请勿关闭)" wsl -d Ubuntu-24.04 bash -c ^
"/usr/local/bin/etcd ^
  --data-dir=~/etcd-data ^
  --listen-client-urls=http://0.0.0.0:2379 ^
  --advertise-client-urls=http://127.0.0.1:2379 ^
  --listen-peer-urls=http://0.0.0.0:2380 ^
  --initial-advertise-peer-urls=http://127.0.0.1:2380 ^
  --initial-cluster=default=http://127.0.0.1:2380"

echo 等待 etcd 启动...
timeout /t 4 /nobreak >nul

:: 验证 etcd 是否启动成功
wsl -d Ubuntu-24.04 bash -c "curl -s http://127.0.0.1:2379/version" >nul 2>&1
if %errorlevel% neq 0 (
    echo etcd 启动失败，请检查 WSL-ETCD 窗口
    pause
    exit /b 1
)
echo etcd 启动成功

:: --- 第三步：启动 Windows 上的 etcdkeeper ---
echo.
echo [3/4] 正在启动 EtcdKeeper...
set "KEEPER_DIR=E:\Edge Download\etcdkeeper-v0.7.8-windows_x86_64"

if not exist "%KEEPER_DIR%\etcdkeeper.exe" (
    echo 错误：找不到 etcdkeeper.exe
    pause
    exit /b 1
)

start "EtcdKeeper" /D "%KEEPER_DIR%" "%KEEPER_DIR%\etcdkeeper.exe" -p 8083 -e http://127.0.0.1:2379

timeout /t 2 /nobreak >nul
start http://localhost:8083/etcdkeeper/
echo EtcdKeeper 已启动

:: --- 第四步：进入 WSL2 命令行 ---
echo.
echo [4/4] 进入命令行模式...
echo -------------------------------------------------------
echo  已进入 Ubuntu 命令行
echo  ETCDCTL_API=3，直接使用 etcdctl 命令
echo.
echo  常用命令:
echo     etcdctl put name tom
echo     etcdctl get name
echo     exit  退出窗口
echo -------------------------------------------------------
echo.

wsl -d Ubuntu-24.04 bash -c "export ETCDCTL_API=3 && export ETCDCTL_ENDPOINTS=http://127.0.0.1:2379 && exec bash"
```

---

## 四、环境恢复 Checklist

每次重新开机或 WSL2 重启后，按以下顺序检查：

1. 确认 WSL2 网络模式：`wsl --status`，看是否有 `networkingMode: mirrored`
2. 确认 etcd 进程存在：`ps aux | grep etcd`
3. 确认 etcd 在 WSL2 内响应正常：`curl http://127.0.0.1:2379/version`
4. 确认 Windows 侧可访问（PowerShell）：`curl http://localhost:2379/version`
5. 启动 etcdkeeper，访问 `http://localhost:8083/etcdkeeper/`

---

## 五、总结

WSL2 的网络模式是这次问题的根本原因。默认的 NAT 模式下，Windows 和 WSL2 的网络是隔离的，`localhost` 并不互通，导致运行在 Windows 的 etcdkeeper 和 Java 后端无法访问 WSL2 里的 etcd。开启镜像网络模式（`networkingMode=mirrored`）后，两侧共享网络栈，问题彻底解决。

如果你也在用 WSL2 做开发，强烈建议第一步就把镜像网络模式开起来，省去很多不必要的麻烦。
