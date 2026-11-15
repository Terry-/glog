---
title: '单日 Worker 调用激增 21 万次：一次边缘多云网关的 CC 攻击排查与治理实录'
description: '通过 Cloudflare GraphQL Analytics API 构建数据证据链，定位海外单一节点高频恶意爬取行为，揭示多云路由逻辑带来的“调用放大效应”，并通过边缘 WAF 实现秒级阻断的工程治理全过程。'
pubDate: '2026-11-15 12:00:00'
series: '嵌入式与边缘网络实战'
tags: ['Cloudflare', 'Cloudflare Workers', 'Edge Computing', 'WAF', 'CC攻击防护', '多云架构', '运维排障']
author: "尹鸿文（霈泽）"
---

在多云架构与边缘计算（Edge Computing）日常运维中，监控看板的异常毛刺往往是系统健康度或外部安全风险的直接映射。近期在巡检 Cloudflare 运维看板时，发现过去 24 小时内 Workers 调用量出现指数级异常激增，从接近平稳基线骤增至 **21.61 万次**（环比增长 13,226%），同时全站缓存命中率被拉低至 **0.05%**。

本文记录如何从排除内部重定向配置死循环开始，通过 Cloudflare GraphQL Analytics API 构建数据证据链，定位海外单一节点高频恶意爬取行为，揭示多云路由逻辑带来的“调用放大效应”，并通过边缘安全规则（WAF）实现秒级阻断的完整工程治理过程。

---

## 一、异常现象与排查假设

监控面板直观呈现了四项关键异常指标：
- **请求总数**：239.63k（环比增长 904.1%）；
- **Worker 调用量**：216.16k（环比增长 13,226.6%）；
- **缓存命中率**：0.05%（下跌 93.7%）；
- **CPU 时间 P90**：稳定在 2 ms。

```text
流量激增曲线形态：
[Requests/hr]
 40k ┤                                     ╭───────╮
 30k ┤                                     │       │
 20k ┤                                     │       │
 10k ┤                                     │       │
   0 ┼─────────────────────────────────────╯       ╰──
     00:00                 12:00                 24:00
```

### 初始排查假设
此前系统刚刚完成跨域 301/308 重定向与静态资源缓存规则治理，出现类似陡增时，第一怀疑点通常是：
1. **重定向环路（Redirect Loop）**：根域名与子域名之间的跳转规则是否存在配置冲突，导致客户端或边缘节点陷入死循环。
2. **探针自激震荡**：监控探针高频轮询叠加未命中缓存，导致调用线性累积。

---

## 二、数据核验与证据链构建

为了避免盲目调整线上路由，排查优先通过 Cloudflare GraphQL Analytics API 抽取底层明细数据。

### 1. 锁定异常 Worker 实例
通过调用 `workersInvocationsAdaptive` 数据集查询各 Worker 脚本在过去 24 小时的调用分布：

```graphql
query($accountTag: String!, $since: Time!) {
  viewer {
    accounts(filter: {accountTag: $accountTag}) {
      workersInvocationsAdaptive(limit: 10, filter: {datetime_geq: $since}) {
        dimensions { scriptName, status }
        sum { requests, subrequests, errors }
      }
    }
  }
}
```

**统计结果**：
- `afn-official-gateway`：成功调用 220,549 次，发出子请求（subrequests）220,550 次；
- 其余业务 Worker（如 `wuxian-ai-router` 等）：调用量均在几百次正常水位。

排查确认：**所有突增调用 100% 集中于挂载在 `agentfriendly.network/*` 的网关 Worker。**

### 2. 溯源请求特征（IP、UA 与路由）
进一步深入 `httpRequestsAdaptiveGroups` 数据集，按国家、客户端 IP、User-Agent 及请求路径进行聚合分析：

```graphql
query($zoneTag: String!, $since: Time!) {
  viewer {
    zones(filter: {zoneTag: $zoneTag}) {
      httpRequestsAdaptiveGroups(limit: 20, filter: {datetime_geq: $since}, orderBy: [count_DESC]) {
        count
        dimensions {
          clientRequestHTTPHost
          clientRequestPath
          clientRequestHTTPMethodName
          userAgent
          clientIP
        }
      }
    }
  }
}
```

分析提取出核心事实：
1. **单一来源 IP**：超过 99.8% 的高频请求来自日本软银（SoftBank）家庭宽带/住宅出口单一 IP `126.74.***.***`；
2. **伪造 User-Agent**：请求头携带伪造的 `Mozilla/5.0 ... Chrome/156.0.0.0 Safari/537.36`（当前主流 Chrome 大版本远未达到 156）以及 `fl2 early hints` / `nginx-ssl early hints` 自动化探针特征；
3. **固定目标路径**：针对站点 4 个前台路由进行循环轰击：
   - `/standards`（5.6 万次）
   - `/projects`（5.5 万次）
   - `/agent-native`（5.5 万次）
   - `/members`（5.5 万次）
4. **请求速率**：短时间内瞬时并发达到约 **70 QPS**，在过去 15 分钟内产生逾 6 万次请求，并已导致部分上游响应触发 `504 Gateway Timeout`。

排除结果：该现象与治理阶段配置的 301 重定向规则无关，确认为针对前台核心页面的高频爬虫与 CC 探测行为。

---

## 三、架构根因：多云故障转移逻辑的“调用放大效应”

为什么外部针对源站的刷量，会引发 Worker 调用量与子请求数的完全等比暴涨？

审视网关脚本核心路由逻辑：
```javascript
const BACKENDS = [
  { name: "vercel",  url: "https://vercel.example.com",  region: "GLOBAL" },
  { name: "netlify", url: "https://netlify.example.com", region: "AMER/EMEA" },
  { name: "qcloud",  url: "https://qcloud.example.com",  region: "APAC" }
];

const GEO_ROUTING = {
  'JP': 'qcloud', // 日本访客默认调度至腾讯云 EdgeOne 亚太加速节点
  'CN': 'vercel',
  // ...
};

async function handleRequest(request) {
  const country = request.cf ? request.cf.country : 'US';
  let preferred = BACKENDS[0];
  if (GEO_ROUTING[country]) {
    preferred = BACKENDS.find(b => b.name === GEO_ROUTING[country]);
  }
  // 发起上游子请求并支持故障降级
  const response = await fetch(new URL(url.pathname, preferred.url), { ... });
  return response;
}
```

在该网关架构下：
1. **执行层级倒置**：域名路由通配绑定了 Worker 路由（`example.com/*` -> Worker），外部请求在未经过边缘安全过滤前，首选唤醒 Worker 运行时；
2. **地理路由映射**：攻击源 IP 归属日本（JP），触发 `GEO_ROUTING['JP'] === 'qcloud'`；
3. **调用被动翻倍**：每一次外部进入的恶意 GET 请求，Worker 内部均执行一次 `fetch('https://qcloud.example.com/...')` 向上游拉取内容。

```text
攻击流量注入路径：
[恶意 IP: 126.74.***.***] (70 QPS, Chrome/156)
               │
               ▼
   [Cloudflare 边缘接入]
               │ (无前置 WAF 拦截)
               ▼
[Worker: afn-official-gateway] (消耗 1 次 Invocations)
               │
               ├─► 判定为 JP 流量
               │
               ▼
  [Subrequest: fetch(qcloud)]  (消耗 1 次 Subrequest + 穿透后端)
```

最终形成 **1 次入站请求 = 1 次 Worker 调用 + 1 次外发子请求** 的级联消耗链，导致 Worker 配额短时间内剧烈消耗，缓存命中率被拉低至 0.05%。

---

## 四、治理与防御实施

阻断思路遵循纵深防御原则：**安全过滤必须前置于计算执行层，直接在 Cloudflare 最外层边缘拦截恶意请求，杜绝其下发至 Worker 运行时。**

### 1. WAF 自定义规则配置
在 Cloudflare 域内配置一条高优先级自定义防火墙规则（Custom Rule）：

- **规则名称**：`Block Aggressive Crawler`
- **匹配表达式**：
  ```text
  (ip.src eq 126.74.***.***) or (http.user_agent contains "Chrome/156")
  ```
- **执行操作**：**阻止 (Block)**（直接返回 403 Forbidden 并终止后续评估）

在规则创建界面，Cloudflare 回测引擎即时显示该规则涵盖了过去 30 天内约 **23.2 万次**（近 90%）的请求，印证了规则特征的准确性。

```text
优化后的边缘拦截链路：
[外部请求] ──► [Cloudflare WAF 边缘防御层]
                        │
         ┌──────────────┴──────────────┐
         ▼                             ▼
    命中恶意特征                  正常用户流量
  [直接返回 403]                  [唤醒 Worker 网关]
(零 Worker 开销)             (执行多云健康调度与缓存)
```

### 2. 部署后即时验证
规则下发后，通过终端探针对不同场景进行快速冒烟测试：

- **场景 A：伪造 UA 探测验证**
  ```bash
  curl -sI -A "Mozilla/5.0 ... Chrome/156.0.0.0 Safari/537.36" https://agentfriendly.network/standards
  ```
  **返回**：`HTTP/1.1 403 Forbidden`（边缘毫秒级拦截，未到达源站与 Worker）。

- **场景 B：正常用户访问验证**
  ```bash
  curl -sI https://agentfriendly.network/standards
  ```
  **返回**：`HTTP/1.1 200 OK`（业务流量正常放行，未受阻断影响）。

- **场景 C：流量与调用监控**
  规则生效 60 秒内，来自该 IP 的请求全部收敛为 403，`afn-official-gateway` 的 Invocations 从几万级峰值**直接归零**，后端 `504` 超时告警完全消除。

---

## 五、架构与运维反思

通过本次排查与治理，沉淀出以下多云边缘网关架构原则：

1. **防护前置于计算（Security Before Compute）**：
   在 Serverless / Edge Worker 网关架构中，Worker 属于按量计费的计算资源。绝不能将 CC 防护、频控（Rate Limiting）完全寄托于 Worker 代码内部实现。必须利用边缘平台第一道 WAF 防线进行阻断，避免算力被无意义消耗。
2. **数据证据链优先于主观猜测**：
   监控指标突增时，优先使用 GraphQL 等细粒度 API 抽取 IP、UA、Path、Status 的多维交叉数据。本次排查若未先行抓出特定 IP 与伪造 UA，容易误判为重定向规则失效而进行多余回滚。
3. **边缘子请求限速与健康熔断**：
   对于根据地理位置执行分流的网关脚本，需在 Worker 内部增加针对同 IP/同路由的轻量限频及边缘 Cache API 兜底，防止单点故障引发后端源站级联击穿。

---

> **环境与凭据说明**：本文中涉及的所有物理 IP 均已使用通用短掩码进行脱敏，配置与代码遵循生产脱敏规约。
