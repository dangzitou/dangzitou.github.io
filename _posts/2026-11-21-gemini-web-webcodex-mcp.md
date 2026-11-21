---
title: "让 Gemini 网页版连接服务器：WebCodex、MCP 与 OAuth 实战"
date: 2026-11-21 12:00:00 +0800
categories: [AI, 开发工具]
tags: [Gemini, MCP, WebCodex, Linux, OAuth, 自托管]
description: "记录在 Linux 服务器上部署 WebCodex，并让 Gemini 网页版通过 HTTPS 和 OAuth 访问开发环境的过程。"
---

Gemini 网页版可以通过远程 MCP 连接自建工具。本文记录一次在 ARM64 Linux 云服务器上接入 [WebCodex](https://github.com/yyjeqhc/webcodex) 的过程，包括 HTTPS、OAuth 授权、Runner 的运行环境，以及两个实际遇到的兼容性问题。

实验使用 WebCodex v0.5 系列，Gemini 界面和 WebCodex 参数可能随版本变化。部署前应核对所选版本的[完整使用指南](https://github.com/yyjeqhc/webcodex/blob/main/docs/PERSONAL_SETUP.zh-CN.md)、[部署指南](https://github.com/yyjeqhc/webcodex/blob/main/docs/DEPLOYMENT.zh-CN.md)和 [MCP 文档](https://github.com/yyjeqhc/webcodex/blob/main/docs/MCP.zh-CN.md)。

## 架构和权限

```text
Gemini 网页版
    │  HTTPS + OAuth + MCP
    ▼
Caddy / HTTPS 反向代理
    │  127.0.0.1:8080
    ▼
WebCodex Server
    │  WebSocket
    ▼
Runner（项目文件、Git、编译器和测试环境）
```

Server 管理连接和授权，Runner 在持有代码的机器上运行命令并访问项目。公网入口使用受信任证书，Runner 到同机 Server 的连接可以走本地回环地址。

**Runner 的操作系统账号决定了命令能做什么。** 项目根目录限制了 WebCodex 的项目解析与文件工具，但 shell 命令本身继承 Runner 账号的系统权限。root Runner 能读取系统密钥、修改系统配置和删除文件。公网暴露与 OAuth 组合后，拿到有效权限的一方也可能操作这些资源。建议从专用低权限账号和单个项目开始，明确需要管理整台机器时再提升权限。

## 1. 准备 Server 和 Runner

准备一台能够联网的 Linux 服务器、可解析到服务器的域名、Docker Compose 或 WebCodex 官方发布的 Linux Runtime，以及你要授权的项目目录。域名需要能从外网访问，反向代理需要获得受信任的 HTTPS 证书。也可以按官方文档使用 Cloudflare Tunnel；临时 Tunnel 地址可能改变，不适合依赖固定 OAuth 地址的长期部署。

以下是官方 CLI 路径的核心步骤。根据实际 Release 的安装说明安装 `webcodex`，先确认 `webcodex --version`。以下命令中的尖括号都是需要替换的占位符。

```bash
# 初始化并在前台启动 Server，首次调试时保留这个终端
webcodex server init \
  --listen 127.0.0.1:8080 \
  --data-dir "$HOME/.local/share/webcodex" \
  --env-file "$HOME/.config/webcodex/webcodex.env"
webcodex server run --env-file "$HOME/.config/webcodex/webcodex.env"
```

在另一终端创建一次性配对码，并在持有项目的机器登录。以下的 `--env-file` 路径与上一步一致。

```bash
webcodex pairing create \
  --server-url http://127.0.0.1:8080 \
  --env-file "$HOME/.config/webcodex/webcodex.env" \
  --username <用户名> --ttl-secs 600

webcodex login http://127.0.0.1:8080 \
  --code <一次性配对码> \
  --allowed-root /path/to/projects \
  --project /path/to/projects/my-project \
  --print-mcp-config
```

保存登录命令输出的 Runner 配置路径和凭证，按提示启动 Runner。Linux 可以使用官方命令安装用户级常驻服务：

```bash
webcodex runner install --scope user --config <登录输出的runner配置路径>
webcodex runner status --scope user --config <登录输出的runner配置路径>
```

用户级 systemd 服务在注销后能否继续运行取决于用户管理器和 linger 配置。长期 Server 的 Docker Compose 和 systemd 部署详见官方部署指南。不要把配对码、PAT、Runner token、OAuth client secret 或含密钥的配置文件提交到 Git。凭证文件使用 `chmod 600`，并限制目录访问。

## 2. 配置 HTTPS

在 Server 环境文件中启用 OAuth，并将 issuer 设置为你的公网 HTTPS 根地址，例如 `WEBCODEX_OAUTH2_ENABLED=true` 和 `WEBCODEX_OAUTH2_ISSUER=https://mcp.example.com`。使用 Compose 时，将这些值放进 Server 的 `environment`；修改后重新创建 Server，使配置生效。

先将域名解析到服务器公网地址，开放证书申请和访问所需的端口。以同机 Server 为例，Caddy 的基本反向代理配置可以写成：

```caddyfile
mcp.example.com {
    reverse_proxy 127.0.0.1:8080 {
        flush_interval -1
    }
}
```

将 `mcp.example.com` 换成自己的域名，并按实际部署安装、验证和重载 Caddy。若使用 Nginx，需正确转发请求头，并关闭代理缓冲以支持流式响应。DNS、云防火墙和证书都要检查。

```bash
curl -i https://mcp.example.com/mcp
curl -fsS https://mcp.example.com/.well-known/oauth-authorization-server
```

第一条未携带凭证，返回 `401` 可能是正常的鉴权提示。第二条用于检查 OAuth 元数据和 HTTPS 证书，具体端点以实际版本返回为准。不要用 `curl -k` 跳过证书校验。

## 3. 创建 Gemini OAuth 客户端

在 Gemini 的 **Connected Apps → Custom apps** 或 Developer Mode 中创建连接，填写 `https://mcp.example.com/mcp`，展开 **Additional settings**，点击 **Copy redirect URI**。回调地址应以 Gemini 当时显示的完整值为准，每个字符都要与服务端登记的一致。

使用有权限创建客户端的 WebCodex 用户 PAT 注册 OAuth client。下面的示例通过交互式输入读取 PAT 和 Gemini 回调地址，并把包含一次性 client secret 的响应写入仅当前用户可读的文件。此操作使用 Server 的本机回环地址，具体 API 和权限要求以所部署版本为准。

```bash
umask 077
read -rsp 'WebCodex 用户 PAT: ' WEBCODEX_PAT; echo
read -r -p 'Gemini Redirect URI: ' GEMINI_REDIRECT

jq -n --arg uri "$GEMINI_REDIRECT" \
  '{name:"Gemini", redirect_uris:[$uri], allowed_scopes:["runtime:read","session:collaborate","project:read","project:write","job:run"]}' \
  | curl -fsS http://127.0.0.1:8080/api/oauth/clients/create \
      -H "Authorization: Bearer $WEBCODEX_PAT" \
      -H 'Content-Type: application/json' \
      --data-binary @- > "$HOME/gemini-webcodex-oauth.json"
unset WEBCODEX_PAT GEMINI_REDIRECT
chmod 600 "$HOME/gemini-webcodex-oauth.json"
```

客户端密钥通常只在创建时返回一次。安全地读取返回文件中的 Client ID 和 Client Secret，填入 Gemini 的对应字段。接着点击 **Next** 和 **Connect**，在 WebCodex 授权页面输入需要代表的用户 PAT，审阅权限并完成授权。用户 PAT 用于登录，Runner token 用于 Runner 连接 Server；两者用途不同。

## 4. 遇到 `invalid scope`

这次遇到的问题是 Gemini 请求了 Server 元数据公布的所有 scope，而预注册 OAuth client 只允许其中一部分。WebCodex 校验时因此返回 `invalid scope`。

先检查 Gemini 本次请求的 scope 与该 client 的 `allowed_scopes`，不要把完整授权 URL 发到公开场所。`offline_access` 是协议级 scope，由 WebCodex 单独处理。在这次单客户端部署中，我让该域名的 OAuth 元数据只公布这个客户端允许的五项权限和 `offline_access`，然后从 Gemini **重新发起**连接。旧授权页面仍带有旧请求参数，重复提交不会更新它。

一种实现方式是在反向代理上为 `/.well-known/oauth-authorization-server` 提供一份从当前 Server 元数据生成的 JSON，仅替换 `scopes_supported`，保留 issuer、端点及其他字段。下面的命令在 Server 主机执行，先确认本地 Server 的 issuer 已配置成公网 HTTPS 域名：

```bash
sudo mkdir -p /var/lib/webcodex-metadata
curl -fsS http://127.0.0.1:8080/.well-known/oauth-authorization-server \
  | jq '.scopes_supported = ["runtime:read", "session:collaborate", "project:read", "project:write", "job:run", "offline_access"]' \
  | sudo tee /var/lib/webcodex-metadata/oauth.json >/dev/null
```

如果 Caddy 直接运行在宿主机，可把站点配置改成以下形式。容器中的 Caddy 则需要把元数据文件所在目录以只读方式挂载到容器内，并使用容器内路径。

```caddyfile
mcp.example.com {
    handle /.well-known/oauth-authorization-server {
        header Content-Type application/json
        root * /var/lib/webcodex-metadata
        rewrite * /oauth.json
        file_server
    }
    handle {
        reverse_proxy 127.0.0.1:8080 {
            flush_interval -1
        }
    }
}
```

验证并重载 Caddy，再检查公网返回的 `issuer`、端点和 `scopes_supported`。这些字段须与 Server 配置和客户端允许范围一致，然后重新发起授权。

这个方法适合元数据与单个权限集合一致的部署。多个客户端需要不同权限时，应分别设计兼容方案，并在 WebCodex 升级、域名或端点变化后同步检查这份静态元数据。不要为了绕过错误直接授予所有 scope，也不要移除 PKCE 或其他鉴权校验。[WebCodex OAuth 文档](https://github.com/yyjeqhc/webcodex/blob/main/docs/MCP.zh-CN.md)

## 5. 工具调用成功但没有数据

接通后，我在 Gemini 里请求列出项目，工具返回了 `WebCodex tool completed successfully.`，却没有项目列表。WebCodex 默认将完整数据放在 MCP 的 `structuredContent`，而该次 Gemini 会话只向模型呈现了文本部分。

WebCodex 提供兼容开关，让普通 runtime tool result 同时在 `content[0].text` 中输出 JSON：

```yaml
# Docker Compose 中的 Server 环境变量示例
services:
  webcodex:
    environment:
      WEBCODEX_MCP_TEXT_JSON_COMPAT: "true"
```

修改后重建 Server 容器。systemd 部署则在 Server 的环境文件中设置相同变量并重启服务。这个设置会增加返回文本和模型上下文用量。我的服务端直接 MCP 测试已验证文本 JSON 与 `structuredContent` 一致；Gemini 是否能在具体会话中读到完整结果，还要重新发起一次真实工具调用确认。

## 6. 容器 Runner 和宿主机 Runner

本次最初的 Runner 在离线 Docker 网络中，缺少宿主机上安装的编译器、Python 环境和出站网络。因此，即使项目目录已经挂载，容器中的命令也无法直接使用宿主机的开发工具。

后来我按自己的需求将 Runner 切到宿主机，以 systemd 常驻，并开放 root 和 `/`。通过 MCP 验证了 root 身份、宿主机 CMake、Python 虚拟环境、项目文件及 GitHub 出站访问。这是一项高权限选择，创建的文件也可能归 root 所有。实际配置可从低权限宿主机账号开始，按项目需要调整路径、工具和网络权限。

容器部署适合把执行环境和挂载范围单独管理；宿主机 Runner 更容易直接使用现有工具链。无论采用哪种方式，都要检查 Runner 的实际运行账号、项目注册、挂载和网络。新增项目可用登录时输出的 Runner 配置执行：

```bash
webcodex project register --config <runner配置路径> /path/to/another-repo
```

## 7. 连接后如何验证

在 Gemini 中选择已连接的 WebCodex，先进行只读检查：

```text
请列出当前可访问的项目和 Runner 状态，只做只读检查。
```

确认返回的项目和在线状态与实际一致，再在专门的测试仓库中做一次小文件读写、运行测试并检查 `git diff`。如果需要验证提交，先确认测试仓库和目标分支，避免影响已有工作。长任务、包下载、远程 Git 操作和多轮修改应分别验证，具体可用性取决于 Runner 环境和客户端行为。

常用排查方向包括 Server 与 Runner 的健康状态、反向代理日志、OAuth 回调及 scope、客户端是否读取工具结果，以及容器挂载和出站网络。服务名和 Compose 目录由实际部署决定：

```bash
# Compose 部署示例
cd /path/to/webcodex-deployment
docker compose ps
docker compose logs --tail=100 webcodex

# 官方安装的用户级 Runner 示例
webcodex runner status --scope user --config <runner配置路径>
webcodex runner logs --scope user --config <runner配置路径> --lines 100
```
