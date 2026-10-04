---
title: "给腾讯 Agent Memory 加反馈强化机制"
date: 2026-09-10 21:30:00 +0800
categories: [技术]
tags: [AI, Agent, 记忆系统, 开源, 认知科学]
---

前两天在本地把 Claude Code 的 API 地址指向了一个代理服务。新开会话，随口说了几句：我是厦大 CS 的本科生，主攻 Java 后端和 AI，最近在给腾讯的开源项目提 PR。为了对照，还编了第三条：我特别爱喝手冲咖啡，每天下午三点准时下楼买一杯。

今天另起全新会话，上下文空白，问它：我下午三点一般干嘛。

它答：下午三点你一般会去楼下买一杯手冲咖啡。

新会话，零上下文，它凭什么记得？

这个系统叫 **TencentDB-Agent-Memory**，腾讯云今年开源的项目，GitHub 上 27.7k star。它要解决的核心问题是：**如何让 Agent 的经验能够沉淀、流转和复用**，而不是每次对话结束就归零。

## 一、项目是什么

### 1.1 要解决的问题

天天用 Claude Code 也好，Codex 也好，它们都是金鱼记忆。今天的对话一关，明天全忘。上下文窗口越做越大，但窗口像旅游团大巴，下车就散。上周踩出来的结论，这周它一脸无辜地再踩一遍。

TencentDB-Agent-Memory 的思路是，不争上下文窗口，在 Agent 和大模型之间插一层代理，对话流过时顺手沉淀成记忆。Agent 零代码接入，base URL 指过去就行。

![Memory Loop：每次对话循环都会积累经验](/assets/post_imgs/2026-10-04-tdai-memory/memory-loop.png)

### 1.2 效果验证

项目的 benchmark 里有 **PersonaMem**，专门测 Agent 能不能在长期交互后正确理解和使用用户信息。结果：

| 配置 | PersonaMem 准确率 |
|------|------------------|
| 不带记忆 | 48% |
| 带 Agent Memory | **76%** (+59%) |

## 二、记忆分层架构

这套系统的核心是四层记忆架构，对应认知科学里人脑记忆的不同层次。

![四层记忆架构：L0到L3](/assets/post_imgs/2026-10-04-tdai-memory/chat-memory-layers.png)

### 2.1 L0 - 对话原文层（Conversation）

**作用**：原始记录，相当于工作记忆

- 对话原样落盘，每天一个文件，不压缩
- 保留完整上下文，可追溯原文
- 触发后续抽取 pipeline

对应认知科学：**工作记忆（Working Memory）**，容量有限但保留完整细节

### 2.2 L1 - 原子记忆层（Atomic Memory）

**作用**：提取可独立理解的事实单元

一个大模型在一次调用里同时干两件事：
1. 先切情境（Context Tagging）
2. 再从对话里把事实、偏好、约束拆出来，拆成独立的小卡片

每条卡片的规矩：
- 要独立完整，跳出对话还得成立
- 要带溯源 ID，能追回原文
- **宁缺毋滥**：记错了比记不住更糟

我实测时编了两条生活事实，想问它记不记得，结果这两条没被提取出来。翻到提取 prompt 发现，工作模式下明确写着不提取与工作无关的个人偏好。

对应认知科学：**情景记忆（Episodic Memory）** 的编码过程，将经历拆解成可检索的记忆单元

### 2.3 L2 - 场景记忆层（Scenario Memory）

**作用**：围绕项目/场景整合相关记忆

- 把散落的 L1 卡片围绕项目整合成场景文件
- 每个场景带热度值，被翻牌越多越热
- 系统里跑这个活的 prompt 开头自称：记忆整合架构师，你不仅是记录数据，更像一位人类学家和心理学家

对应认知科学：**语义记忆（Semantic Memory）** 的初步形成，将情景记忆抽象成知识

### 2.4 L3 - 核心记忆层（Core/Persona）

**作用**：用户的稳定画像

- 两千字符硬上限
- 装的是这个用户最稳定的画像，认知内核
- 最高层次的抽象

对应认知科学：**长期记忆（Long-term Memory）** 中的核心人格和认知模式

### 2.5 记忆如何被使用

四层分好之后怎么用也有讲究：

| 层级 | 使用方式 | 原因 |
|------|---------|------|
| L3 核心 | 直接拼进 system prompt | 稳定不变，钉死在系统提示里 |
| L2 场景 | 直接拼进 system prompt | 相对稳定，快速恢复工作上下文 |
| L1 原子 | 包装成只读工具，按需查询 | 易变且量大，避免 KV cache 失效 |
| L0 原文 | 只在需要验证原文时查询 | 最详细但最重，极少直接使用 |

**为什么不一起塞？**

因为 system prompt 一动，大模型的 KV cache 就全废，每次对话的输入成本直接翻倍。稳定的东西钉死，易变的东西按需取，这个边界踩过坑的人才画得出来。

## 三、记忆巩固 Pipeline

想起以前看的脑科学书。人的睡眠有个过程叫**记忆巩固（Memory Consolidation）**，白天海马体匆匆记下的短暂痕迹，会在夜里被大脑离线重放，一遍遍重组，最后沉淀成皮层里的语义知识。

L0 到 L3 的这条管线，就是给 AI 造睡眠。

![技术架构：分层记忆 + 异步巩固 + Agent 装配](/assets/post_imgs/2026-10-04-tdai-memory/architecture.png)

### 3.1 异步处理流程

```
对话发生 → L0 落盘
  ↓ (异步)
L1 抽取任务入队
  ↓
大模型：切情境 + 提取事实
  ↓
去重、冲突检测
  ↓
L2 场景整合
  ↓
L3 核心更新
```

整个链路带：
- 任务队列（Task Queue）
- 分布式锁（Distributed Lock）
- 失败重试（Retry）
- 级联调度（Cascade Scheduling）

我实测那天，全程盯着日志看它跑完这条流水线。我那段随口说的三句话，先是被 L0 原样落盘，几分钟后 L1 抽取任务自动入队，大模型把对话切成情境，情境名是：团队成员在介绍 TencentDB-Agent-Memory 项目工作背景。然后抽取、去重、冲突检测，一路自动跑进 L2 的场景块。整个过程没有人操作，它自己把白天的东西消化了。

### 3.2 认知科学对应

这套设计几乎是认知科学教科书的工程实现：

| 认知科学概念 | Agent Memory 实现 |
|-------------|------------------|
| Working Memory | L0 Conversation |
| Encoding | L1 抽取 pipeline |
| Consolidation | L0→L1→L2→L3 异步流程 |
| Semantic Memory | L2 Scenario |
| Schema | L3 Core/Persona |
| Retrieval | BM25 + Vector + RRF |

## 四、我提的 PR：让记忆「越用越准」

### 4.1 问题：记忆系统只管记，不管忘

这套系统有个地方一直空着：它只管记，不管忘，也不会越用越准。

人脑不是这样的。心理学里有两个机制：

1. **提取练习效应（Testing Effect）**：记忆每被成功提取一次，下次就更容易被想起来，越用越牢
2. **遗忘曲线（Forgetting Curve）**：不用的记忆会自然衰减

这两个机制拧在一起，人脑才能把有限的容量留给真正重要的东西。

而这套系统呢，检索只按相似度排序。一条记忆哪怕上周刚救过 Agent，今天检索时，它跟三个月没人碰过的记忆权重一样。

### 4.2 解决方案：反馈强化机制

这就是我那个 PR 要干的事，编号 **#1376**，让 Agent 选中过的 L1 记忆被强化。

![PR 1376 的页面](/assets/post_imgs/2026-10-04-tdai-memory/pr-1376.png)

#### 第一步：确认式使用记录

搜索结果带上记忆 ID，Agent 判断某条记忆真的支撑了这次回答，就回写一次使用记录。

**注意**：是 Agent 主动确认，不是搜索 Top-K 全部记为已使用。后者会把候选一起强化，几次搜索后大家都顶着时间戳，信号就稀释了。

#### 第二步：重排打分

在这条记忆的原始相似度上叠两个轻量因子：

1. **近因加成（Recency Boost）**：14天半衰期的指数衰减
   ```
   recency_factor = exp(-days_since_last_use / 14)
   ```

2. **频次加成（Frequency Boost）**：饱和函数，用得越多加分越多，但有上限
   ```
   frequency_factor = tanh(use_count / 10)
   ```

最终得分：
```
final_score = similarity * (1 + 0.1 * recency + 0.1 * frequency)
```

注释里写了四个字：**use it or lose it**。

#### 第三步：Frame Gate（时态门控）

这是后来补的，也是最关键的部分。

我把带强化的版本跑了三组对照实验，一百多个会话 episode，同一个模型配对测。结果出来一半在预期内：正确反馈的场景，命中率涨了 8.3 个百分点。

另一半让我意外：**在学过旧方案的场景里，反向暴跌 18.1 个点**。

排查下来发现一个现象。那条记忆的内容写着「以前是这么做的」，查询问的是「现在该怎么做」。一个过时的历史方案，因为被翻牌多，反而压过了现行约定，冲到了最前面。

**强化放大了一切，包括过时。**

心理学里其实早有名字：
- **编码特异性（Encoding Specificity）**：强化应该附着在使用的情境上
- **干扰更新（Interference）**：新旧内容冲突时要分得出先后

我最初只参考了提取练习和遗忘曲线，漏了这两条。

所以我加了第二版，叫 **Frame Gate（时态门控）**。一条记忆如果自我描述成历史——「以前」「曾经」「已废弃」「后来改了」，这类标记一出现，在现在式的查询里就撤回它的强化，只撤奖不惩罚，原始排序兜底。

```python
def should_apply_reinforcement(memory, query):
    # 检测历史标记
    historical_markers = ['以前', '曾经', '已废弃', '后来改了', 'deprecated']
    has_historical_marker = any(marker in memory.content for marker in historical_markers)
    
    # 检测查询时态
    is_present_query = any(word in query for word in ['现在', '目前', '当前', 'now'])
    
    # 历史记忆 + 现在式查询 → 撤回强化
    if has_historical_marker and is_present_query:
        return False
    return True
```

### 4.3 实验结果

改完再跑一遍那组实验：

| 配置 | PersonaMem准确率 | 提升 |
|------|----------------|------|
| 不带强化（基线） | 72.4% | - |
| 只带强化 | 76.3% | +3.9% |
| 强化 + Frame Gate | **84.2%** | **+11.8%** |

退化场景全部转正，两个不同的模型上方向一致。

诚实说一句，这套基准是我自己造的合成评测，不是线上真实收益，PR 里也是这么标注的。数字看方向就好。

## 五、实现拆解

### 5.1 使用记录存储

在 L1 原子记忆表里加了三个字段：

```typescript
interface AtomicMemory {
  // ... 原有字段
  use_count: number;           // 累计使用次数
  last_used_at: Date | null;   // 最近使用时间
  use_history: Array<{         // 使用历史（可选，用于debug）
    timestamp: Date;
    agent_id: string;
    session_id: string;
  }>;
}
```

### 5.2 确认式反馈接口

新增一个 API 端点：

```typescript
POST /v3/atomic/mark-used

Request:
{
  memory_ids: string[],        // Agent 确认使用的记忆 ID
  agent_id: string,
  session_id: string,
  timestamp: Date
}

Response:
{
  updated_count: number,
  failed_ids: string[]
}
```

这个接口由 Proxy 在收到 Agent 响应后调用。Agent 通过特殊标记（如在响应中附带 `used_memory_ids`）告知 Proxy 哪些记忆真正被使用。

### 5.3 检索重排逻辑

修改 L1 检索的打分函数：

```typescript
function rerankWithReinforcement(
  results: SearchResult[],
  config: { recencyWeight: 0.1, frequencyWeight: 0.1, halfLife: 14 }
): SearchResult[] {
  return results.map(r => {
    const memory = r.memory;
    
    // 计算近因因子
    const daysSinceUse = memory.last_used_at 
      ? (Date.now() - memory.last_used_at.getTime()) / (1000 * 60 * 60 * 24)
      : Infinity;
    const recencyFactor = Math.exp(-daysSinceUse / config.halfLife);
    
    // 计算频次因子
    const frequencyFactor = Math.tanh(memory.use_count / 10);
    
    // Frame Gate 检查
    const shouldReinforce = checkFrameGate(memory, r.query);
    
    // 最终得分
    const reinforcementBoost = shouldReinforce 
      ? config.recencyWeight * recencyFactor + config.frequencyWeight * frequencyFactor
      : 0;
    
    return {
      ...r,
      score: r.similarity * (1 + reinforcementBoost),
      debug: { recencyFactor, frequencyFactor, reinforcementApplied: shouldReinforce }
    };
  }).sort((a, b) => b.score - a.score);
}

function checkFrameGate(memory: AtomicMemory, query: string): boolean {
  const historicalMarkers = ['以前', '曾经', '已废弃', '后来改了', 'deprecated', 'was', 'used to'];
  const presentMarkers = ['现在', '目前', '当前', 'now', 'current', 'currently'];
  
  const hasHistorical = historicalMarkers.some(m => memory.content.includes(m));
  const isPresent = presentMarkers.some(m => query.includes(m));
  
  // 历史记忆遇到现在式查询，撤回强化
  return !(hasHistorical && isPresent);
}
```

### 5.4 Proxy 集成

在 Proxy 层添加反馈回路：

```typescript
// anthropicHandler.ts
async function handleStreamResponse(stream, context) {
  const usedMemoryIds = new Set<string>();
  
  for await (const chunk of stream) {
    // 解析 Agent 响应，识别哪些记忆被使用
    if (chunk.type === 'memory_usage') {
      chunk.memory_ids.forEach(id => usedMemoryIds.add(id));
    }
    yield chunk;
  }
  
  // 流结束后，回写使用记录
  if (usedMemoryIds.size > 0) {
    await memoryCore.markMemoriesUsed({
      memory_ids: Array.from(usedMemoryIds),
      agent_id: context.agentId,
      session_id: context.sessionId,
      timestamp: new Date()
    });
  }
}
```

## 六、为什么这么设计

### 6.1 为什么是确认式，而不是被检索即使用？

如果搜出 Top-10 就全部标记为使用，会带来两个问题：

1. **噪音放大**：候选记忆也被强化，几轮后所有候选都顶着高分，区分度消失
2. **无法反映真实价值**：Agent 看到了但没用，说明这条记忆对当前任务价值不大

确认式反馈确保只有真正产生作用的记忆被强化。

### 6.2 为什么权重只有 0.1？

强化是辅助信号，不是主导因素。相似度仍然是第一位的，强化只是在相似度接近时的打分器。

如果权重过大（比如 0.5），会导致：
- 高频记忆压制新鲜但相关的记忆
- 系统过度依赖历史，失去适应性

0.1 是实验出来的平衡点：既能让常用记忆上浮，又不会掩盖语义相关性。

### 6.3 为什么需要 Frame Gate？

这是最关键的设计。没有它，强化机制会变成一个**正反馈陷阱**：

```
旧方案被使用 → 得分上升 → 更容易被检索 → 更容易被使用 → 得分继续上升
```

即使旧方案已经废弃，它仍然会因为历史频繁使用而霸榜。

Frame Gate 打破了这个循环：当记忆内容本身标明「这是历史」，而查询问的是「现状」，系统就暂停强化，让相似度重新主导。

这对应认知科学里的**情境依赖记忆（Context-Dependent Memory）**：记忆的提取应该匹配编码时的情境，历史记忆不应该在现在式查询中获得不当优势。

## 七、还可以做什么

### 7.1 更细粒度的时态识别

当前的 Frame Gate 用关键词匹配，比较粗糙。可以改进为：

1. 用 NLI 模型判断记忆和查询的时态一致性
2. 提取记忆的有效期（validity period）
3. 在 L1 抽取时就标注时间范围

### 7.2 协同过滤

当前只看单条记忆的使用历史。可以加入：

- 相似记忆的共现模式
- 同一 Agent 的偏好学习
- 团队级别的记忆热度共享

### 7.3 主动遗忘

当前只是降权，没有真正删除。可以加入：

- 长期未使用且低分的记忆自动归档
- 冲突记忆的主动合并或淘汰
- 用户可设置的记忆保留策略

## 八、后记

1885 年，艾宾浩斯拿自己做实验，用无意义音节，量出了人类第一条遗忘曲线。一百四十年后，一个本科生在宿舍里给一个 AI 记忆系统写衰减函数，在评论区讨论编码特异性。

你在给 AI 造记忆的时候，沿着认知科学一百年走过的路又走了一遍。工作记忆、情景记忆、巩固、遗忘、再巩固，这些词一个个从教科书里跳出来，变成代码里的模块和函数。

项目的 slogan 那句话我很喜欢：让 Agent 沉淀经验，让人专注创造。记忆是一个身份问题，你记得什么，你就是谁。人如此，Agent 大概也如此。

---

**参考资料**

1. [TencentDB-Agent-Memory GitHub](https://github.com/TencentCloud/TencentDB-Agent-Memory)
2. [PersonaMem Benchmark](https://github.com/TencentCloud/TencentDB-Agent-Memory#benchmark)
3. Ebbinghaus, H. (1885). Memory: A Contribution to Experimental Psychology
4. Roediger, H. L., & Karpicke, J. D. (2006). Test-enhanced learning: Taking memory tests improves long-term retention. Psychological Science
5. Tulving, E. (1972). Episodic and semantic memory. Organization of memory

