---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

## 关于我

你好，我是 **邓梓滔 (Zitao Deng)**。  
厦门大学计算机科学与技术专业在读本科生（2023 - 2027），现居深圳。

我在做的事情大致有两类，一类是后端与分布式系统的工程实践，另一类是把前沿 AI 能力做成真正能用的东西。比起停在 demo 阶段的项目，我更在意工具是否可检查、可回放、真正解决问题。

这个博客记录我折腾过的东西，开源贡献的复盘、自研项目的设计取舍，还有一些踩坑记录。最近在写的几篇：

- [二十个 PR，九个合并，剩下的各有各的死法](/posts/octop-pr-anatomy/) —— 在腾讯 Octop 打工两个月的完整复盘
- [我给腾讯 27k star 的记忆系统提了个 PR，教它「越用越准」](/posts/tdai-agent-memory-l1-feedback/) —— 拆解 TencentDB-Agent-Memory 的 L0-L3 记忆管线
- [我给自己写了个不会说谎的 coding agent](/posts/keel-agent-event-sourcing/) —— keel-agent 的设计动机与 Apache Maka 的对照

---

## 教育背景

- 厦门大学  计算机科学与技术 本科  2023 - 2027

---

## 技术方向

**后端与基础设施**

- Java（Spring Boot / MyBatis-Plus）、Go、Python、TypeScript
- MySQL / Redis（ZSet / Set / List / GEO / Bitmap）与缓存设计
- 分布式系统（Redis Stream、Redisson、Etcd / ZooKeeper）
- Netty / Vert.x、TCP 通信与 RPC 框架原理

**AI 工程**

- Agent 运行时与 harness 设计（事件溯源、工具策略、证据驱动的完成契约）
- LLM 应用工程（多 Agent 协作、RAG、记忆系统、流式协议）
- 本地优先（local-first）与自托管部署

---

# 核心项目

这些项目是我实现的系统项目。

### [zyro-rpc](https://github.com/dangzitou/zyro-rpc)
轻量级高可用 RPC 框架

- Java + Vert.x + Etcd
- 自定义协议 + 动态代理
- SPI 扩展机制
- 负载均衡与容错策略

---

### [zyro-go](https://github.com/dangzitou/zyro-go)
高并发本地生活服务平台

- Spring Boot + MySQL + Redis
- Redis Stream 异步削峰
- Lua 原子校验
- Redisson 分布式锁
- 缓存优化策略

---

# 开源项目

### [keel-agent](https://github.com/dangzitou/keel-agent)
可回放、可验证、可审计的 coding agent CLI

- 会话即 append-only 事件流，历史 / 花费 / 验证状态全部是对日志的纯推导
- 证据驱动的完成契约，没跑通验证就以非零退出码结束，CI 可直接依赖
- 策略先行，路径黑名单、命令审批、预算熔断，判定本身也落事件
- 零 npm 运行时依赖，支持 OpenAI / Anthropic / Responses 三种线协议

---

# 开源贡献

相信「可验证的痕迹」比简历上的形容词更值钱，日常给上游项目提 PR、修 bug、写文档。

**已合并**

- [TencentCloud/Octop](https://github.com/TencentCloud/Octop)（~6.5k star，自托管多 Agent 助手平台）21 个 PR，9 个合并，覆盖 OAuth 安全修复、多 Agent 失败语义、备份导入、连接器功能等
- [TencentCloud/octop-harness](https://github.com/TencentCloud/octop-harness)（Octop 的 Agent 运行时）流式 thinking 分块投影修复
- [TencentCloud/TencentDB-Agent-Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory)（~27.7k star，Agent 记忆中枢）若干修复与功能 PR

**评审中**

- TencentDB-Agent-Memory：L1 记忆使用反馈强化（提取练习 + 遗忘曲线的召回重排）、L1 记忆证据链维护、ZCode 记忆接入等 PR
- [Tencent/BrowserSkill](https://github.com/Tencent/BrowserSkill)：diagnostics 导出、并行任务冲突预检、Agent Window 首页防护等
- [apache/flink-agents](https://github.com/apache/flink-agents)：有界内部子 Agent 批量执行

更多细节见 [我在 Octop 的 PR 复盘](/posts/octop-pr-anatomy/) 与 [我的 GitHub 主页](https://github.com/dangzitou)。

---

# 联系方式

- **Email**: dengzitao888@163.com
- **GitHub**: [https://github.com/dangzitou](https://github.com/dangzitou)
- **个人网站**: [https://dangzitou.github.io](https://dangzitou.github.io)

---

> Done is Better than Perfect
{: .prompt-tip }
