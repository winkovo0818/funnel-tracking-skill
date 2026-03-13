---
name: funnel-tracking
description: Design and implement reusable funnel-style business process tracking with unified client/server event contracts, resilient delivery, and raw-plus-summary sinks.
license: Complete terms in LICENSE.txt
---

# Skill: funnel-tracking

这个 skill 用于实现“关键业务流程埋点”：前端只负责在关键节点发统一事件，后端统一校验、补充上下文、异步落库，并同时维护“事件明细 + 流程汇总”两层数据。

适用场景：Web、H5、小程序、App、桌面端，或者任何存在明确漏斗/阶段流转的业务流程，例如注册、下单、申请、测评、提单、支付、审批。

## 何时使用

当你需要：

- 给一个多步骤业务流程补埋点，而不是做全量页面分析
- 保持前后端一套事件口径，而不是各端各记一套
- 让埋点失败不影响主流程
- 既能看每一条事件明细，又能看每个 `flow_id` 的转化结果
- 先做最小可用版本，再逐步加细粒度事件

如果只是接入 GA、Mixpanel、神策这类现成 SDK 的页面浏览统计，这个 skill 不一定是最佳方案。

## 如何使用这个 SKILL

### 用户需要提供的信息

在使用这个 SKILL 之前，请准备以下信息：

#### 1. Webhook 配置（必需）

```text
Webhook URL: https://your-webhook-url.com/api/track
Authorization: Bearer your_token_here（如果需要鉴权）
```

#### 2. 业务流程定义（必需）

明确你的业务流程有哪些关键节点，例如：

```text
我的业务流程：
1. 用户访问首页（visit）
2. 用户登录（login_success）
3. 用户开始测评（assessment_start）
4. 用户完成测评（assessment_complete）
5. 用户提交结果（submit_success / submit_fail）
```

#### 3. 字段定义（可选）

如果你的 Webhook 需要特定字段，请说明：

```text
必填字段：event_id, event_name, event_time, flow_id, session_id, anonymous_id
可选字段：user_id, page_path, source, device_type, assessment_id, submit_id
业务特有字段：order_id, product_id, amount（如果有）
```

如果不提供，AI 会使用本 SKILL 推荐的标准字段。

#### 4. 项目信息（必需）

```text
项目类型：Web / 小程序 / App / H5
技术栈：React / Vue / 原生小程序 / UniApp / Flutter
后端语言：Node.js / Python / Java / Go
```

### AI 会做什么

当你提供上述信息后，AI 会：

1. **分析你的项目结构**
   - 找到关键业务流程的代码位置
   - 识别需要埋点的节点

2. **生成 tracker 模块**
   - 创建统一的埋点工具函数
   - 实现 ID 管理（anonymous_id, session_id, flow_id）
   - 实现 trackEvent 和 trackEventOnce
   - 实现失败重试队列（可选）

3. **在关键节点插入埋点代码**
   - 在登录成功后调用 `trackEventOnce('login_success')`
   - 在提交成功后调用 `trackEvent('submit_success')`
   - 确保埋点失败不阻塞主流程

4. **生成配置文件**
   - 将 Webhook URL 和 Token 写入环境变量
   - 生成事件枚举和字段映射

### 使用示例

**示例 1：最简单的使用方式**

```text
我想给我的项目加埋点，使用飞书多维表格存储。

Webhook URL: https://tvo7pfzu3em.feishu.cn/wiki/OIUewKdMyi2iLtksofmcgTMLnLf?table=tbly1FGwdiaBrwOM&view=vewwAzhS2P
Token: xxxxxxx

业务流程：
1. 访问首页
2. 登录
3. 开始测评
4. 提交结果

项目类型：Web（React）
```

AI 会自动：
- 创建 `src/utils/tracker.js`
- 在 `src/pages/Login.jsx` 的登录成功处加埋点
- 在 `src/pages/Submit.jsx` 的提交成功处加埋点
- 生成 `.env` 配置文件

**示例 2：指定字段的使用方式**

```text
我想给小程序加埋点。

Webhook URL: https://api.example.com/track
Token: abc123

业务流程：
1. 访问首页（visit）
2. 登录（login_success）
3. 下单（order_create）
4. 支付（payment_success / payment_fail）

必填字段：event_id, event_name, event_time, flow_id, user_id, order_id
可选字段：amount, payment_method

项目类型：微信小程序
```

AI 会：
- 创建 `utils/tracker.js`（小程序版本）
- 使用 `wx.setStorageSync` 存储 ID
- 在 `pages/order/order.js` 的下单成功处加埋点
- 在 `pages/payment/payment.js` 的支付成功/失败处加埋点

### 输出内容

AI 完成后会提供：

1. **tracker 模块代码**（完整可运行）
2. **埋点接入代码**（在关键节点的具体调用）
3. **环境变量配置**（.env 或 config.js）
4. **测试建议**（如何验证埋点是否正常工作）
5. **飞书视图配置建议**（如果使用飞书多维表格）

### 注意事项

- 如果你的项目已经有埋点代码，请告诉 AI，它会帮你重构而不是重写
- 如果你的 Webhook 有特殊的请求格式（如嵌套字段），请提供示例
- Token 会写入环境变量，不会硬编码到代码中

## 核心设计

### 1. 只追踪关键漏斗，不追踪一切

第一版只保留最关键的阶段事件。常见最小集合：

- `visit` / `entry`
- `login_success` / `auth_success`
- `flow_start`
- `flow_complete`
- `submit_success`
- `submit_fail`

先把核心转化链路跑通，再决定是否补充 `fail`、`abandon`、`retry`、`view_detail`、`parse_success` 之类的细事件。

### 2. 前端不直接承担分析职责

前端/客户端的职责只有四件事：

- 维护埋点上下文 ID
- 在关键节点发送统一 payload
- 对同一流程中的一次性事件做去重
- 在弱网环境下尽力补发，但不阻塞主流程

不要在客户端维护复杂分析逻辑，也不要把埋点事件再存一份业务数据库。

### 3. 后端是唯一的埋点入口

推荐统一走一个后端入口，例如：

```text
client action
  -> tracker helper
  -> POST /api/track
  -> tracking service
  -> raw event sink
  -> flow summary sink
```

这样做的价值是：

- 各端事件口径一致
- 枚举、必填规则、容错策略统一
- 可以在服务端补 `ip`、`geo`、`user-agent`
- 后续换存储介质时，不需要重写所有客户端

## ID 体系

至少区分四类 ID，不要混用：

- `event_id`：单条事件唯一 ID，用于幂等/排查
- `anonymous_id`：未登录也稳定存在的访客 ID，通常跨多次访问复用
- `session_id`：一次运行期会话 ID，通常按“标签页 / App 启动 / 小程序启动”生成
- `flow_id`：一次完整业务流程 ID，用来把多个关键事件串成一个漏斗

可选业务 ID：

- `user_id`：登录后用户 ID
- `business_id`：业务实体 ID，例如 `order_id`、`application_id`、`assessment_id`
- `submit_id` / `payment_id` / `ticket_id`：末端动作的业务记录 ID

通用原则：

- `anonymous_id` 稳定，`session_id` 短期，`flow_id` 面向单次流程
- `flow_id` 不等于业务主键，但可以和业务主键关联
- 如果存在“断点继续”，优先复用原来的 `flow_id`

### 默认存储策略

按平台选择最轻量、最稳定的存储方式：

- Web：`anonymous_id` / `flow_id` / once flags 放 `localStorage`，`session_id` 放 `sessionStorage`
- 小程序 / App：`anonymous_id` / `flow_id` / once flags 放持久化存储，`session_id` 走运行时内存并可同步落本地兜底
- 服务端渲染或纯后端流程：由服务端根据请求上下文或业务流程生成 `session_id` / `flow_id`

终态事件成功后，通常要清理 flow 级状态：

- 删除 `flow_id`
- 删除 once flags
- 删除阶段开始时间
- 保留 `anonymous_id`

### 断点续传的 flow_id 复用策略

当业务流程支持"中途退出、稍后继续"时，需要明确 `flow_id` 的复用规则。

#### 方案 A: 纯客户端存储

实现方式：

- `flow_id` 只存在客户端本地存储
- 恢复流程时直接读取本地 `flow_id`

优点：

- 实现简单，无需改动业务表

缺点：

- 换设备后无法续传
- 清除缓存后 `flow_id` 丢失
- 跨端（Web/App/小程序）无法共享

#### 方案 B: 业务表关联（推荐）

实现方式：

在业务主表（如 `sessions`、`orders`、`applications`）增加 `tracking_flow_id` 字段：

- 创建业务记录时生成并保存 `flow_id`
- 断点恢复时从业务记录读取 `tracking_flow_id`
- 客户端同时在本地缓存 `flow_id` 作为快速读取路径

优点：

- 跨设备、跨会话稳定
- 支持多端共享同一 `flow_id`
- 便于后续关联业务数据与埋点数据

实现示例：

```javascript
// 创建业务记录时
async function createBusinessSession() {
  const flowId = ensureFlowId(); // 从本地读取或生成新的
  const session = await api.createSession({
    tracking_flow_id: flowId,
    // ... 其他业务字段
  });
  return session;
}

// 恢复业务记录时
async function resumeBusinessSession(sessionId) {
  const session = await api.getSession(sessionId);
  if (session.tracking_flow_id) {
    // 复用业务记录中的 flow_id
    localStorage.setItem('trackingFlowId', session.tracking_flow_id);
  }
  return session;
}
```

注意事项：

- `tracking_flow_id` 只是关联字段，不是存一份埋点事件
- 业务表仍然只存业务数据，埋点明细仍然走统一埋点表
- 如果业务记录被删除，对应的埋点数据仍然保留在埋点表中

## 推荐事件协议

推荐把公共字段稳定下来：

```json
{
  "event_id": "evt_xxx",
  "event_name": "submit_success",
  "event_time": "2026-03-13T10:00:00.000Z",
  "flow_id": "flow_xxx",
  "session_id": "sess_xxx",
  "anonymous_id": "anon_xxx",
  "user_id": "user_xxx",
  "page_path": "/submit",
  "page_title": "提交页",
  "source": "direct",
  "device_type": "mobile",
  "browser": "Chrome/123",
  "os": "iOS 18",
  "duration_ms": 12345,
  "is_success": true,
  "error_code": "",
  "error_msg": "",
  "extra": {
    "client_platform": "web"
  }
}
```

字段分层建议：

- 必填主键层：`event_id`、`event_name`、`event_time`、`flow_id`、`session_id`、`anonymous_id`
- 身份层：`user_id`
- 页面层：`page_path`、`page_title`
- 设备层：`source`、`device_type`、`browser`、`os`
- 业务层：`business_id`、`submit_id`、`duration_ms`、`is_success`
- 扩展层：`extra`

约束建议：

- `event_time` 用 ISO 字符串
- `event_name`、`source`、`device_type` 使用后端枚举
- `duration_ms` 只能是非负数
- `error_msg` 只能放可公开的错误摘要，不放敏感数据

## 跨端字段映射规范

当同一套埋点协议需要在多个平台（Web、H5、小程序、App）使用时，需要统一字段填充规则。

### source 字段映射

推荐枚举值：

- `direct`：直接访问（无 referrer）
- `wechat`：微信生态（包括微信内 H5、微信小程序）
- `alipay`：支付宝生态
- `feishu`：飞书生态
- `search`：搜索引擎
- `ad`：广告投放
- `other`：其他来源

平台映射规则：

- **Web H5**：根据 `document.referrer` 或 URL 参数 `?source=xxx` 判断
- **微信小程序**：统一使用 `wechat`（如需区分可在 `extra.client_platform` 标注 `miniprogram`）
- **支付宝小程序**：使用 `alipay`
- **原生 App**：使用 `app_ios` / `app_android`（或统一为 `app`，在 `os` 字段区分）
- **桌面端应用**：使用 `desktop_app`

### device_type 字段映射

推荐枚举值：

- `mobile`：移动设备
- `desktop`：桌面设备

判断规则：

- **Web**：根据 `window.innerWidth` 或 `navigator.userAgent` 判断
- **小程序**：通过 `wx.getSystemInfoSync().platform` 判断，通常为 `mobile`
- **App**：根据设备类型固定填充

### browser 字段映射

推荐填充规则：

- **Web**：`navigator.userAgent` 的浏览器部分，如 `Chrome/123`、`Safari/17`
- **微信小程序**：固定为 `wechat-miniprogram`
- **支付宝小程序**：固定为 `alipay-miniprogram`
- **原生 App**：固定为 `native-app` 或 `app-ios` / `app-android`

### os 字段映射

推荐填充规则：

- **Web**：从 `navigator.userAgent` 解析，如 `Windows`、`macOS`、`iOS 18`、`Android 14`
- **小程序**：从 `wx.getSystemInfoSync().system` 获取，如 `iOS 18.3`、`Android 13`
- **App**：从系统 API 获取完整版本号

### page_path 字段映射

推荐填充规则：

- **Web SPA**：`window.location.pathname`，如 `/assessment`
- **Web MPA**：完整路径，如 `/pages/submit.html`
- **小程序**：小程序页面路由，如 `/pages/assessment/assessment`
- **App**：屏幕标识符，如 `AssessmentScreen` 或 `/assessment`

### extra 字段扩展

当标准字段无法区分平台细节时，使用 `extra` 字段补充：

```json
{
  "extra": {
    "client_platform": "miniprogram",
    "miniprogram_version": "1.2.3",
    "app_version": "2.0.1",
    "sdk_version": "1.0.0"
  }
}
```

### 跨端一致性原则

- 优先使用标准枚举值，避免自造词汇
- 同一业务流程在不同端的 `event_name` 必须一致
- `flow_id` 生成规则在各端保持一致（格式、长度、前缀）
- 如果某个字段在某端无法获取，填 `undefined` 而不是空字符串或占位符

## 前端实现模式

前端建议抽一个统一 tracker，而不是散落各页面自己拼请求。

至少提供这些能力：

```text
init()
ensureAnonymousId()
ensureRuntimeSessionId()
ensureFlowId()
markStageStarted()
getStageDurationMs()
trackEvent()
trackEventOnce()
flushQueue()
resetFlow()
```

### 一次性事件与普通事件

推荐区分两类发送方式：

- `trackEventOnce(eventName)`：同一 `flow_id` 内只应出现一次的里程碑事件
- `trackEvent(eventName)`：允许重复出现的结果事件或失败事件

常见映射：

- 用 `trackEventOnce`：`visit`、`login_success`、`flow_start`、`flow_complete`
- 用 `trackEvent`：`submit_success`、`submit_fail`、`payment_fail`、`retry`

本地去重 key 推荐：

```text
trackingFlag:{flowId}:{eventName}
```

### 基础 payload 组装

客户端的 `buildPayload()` 通常负责：

- 自动补 `event_id`
- 自动补 `event_time`
- 自动补 `flow_id` / `session_id` / `anonymous_id`
- 自动补页面、来源、设备信息
- 再合并业务特有字段

### 流程重置与继续

客户端需要明确两件事：

- 什么情况下算“继续同一流程”，应该复用原 `flow_id`
- 什么情况下算“开启新流程”，应该调用 `resetFlow()` 并生成新的 `flow_id`

常见默认规则：

- 成功提交 / 支付成功 / 流程显式完成后重置 flow
- 用户中途返回但业务记录仍处于进行中时，继续复用 flow
- 用户主动点击“重新开始”时，显式重置 flow

### 不要让埋点影响主流程

必须遵守：

- 埋点失败不能阻塞页面跳转
- 埋点失败不能回滚业务提交
- 埋点调用使用短超时
- UI 侧最多打印日志，不向用户暴露“埋点失败”

### 可靠性分层

基础版：

- 直接 fire-and-forget
- 失败只记日志

增强版：

- 本地维护一个失败重试队列
- 关键事件失败后入队
- 在 `onShow`、网络恢复、登录成功、关键动作完成前后触发 `flushQueue()`
- `trackEventOnce` 只有发送成功后才写 once flag

队列建议字段：

```json
{
  "payload": {},
  "onceKey": "trackingFlag:flow_xxx:flow_start",
  "retryCount": 0,
  "createdAt": 1710000000000,
  "critical": true
}
```

### 弱网重试队列实现细节

在移动端、小程序等弱网环境下，埋点请求容易失败。推荐实现本地重试队列。

#### 队列项结构

```json
{
  "payload": {
    "event_id": "evt_xxx",
    "event_name": "assessment_start",
    "flow_id": "flow_xxx",
    ...
  },
  "onceKey": "trackingFlag:flow_xxx:assessment_start",
  "retryCount": 0,
  "createdAt": 1710000000000,
  "critical": true
}
```

字段说明：

- `payload`：原始埋点 payload，完整保留
- `onceKey`：仅 once 事件需要，用于补发成功后写入 flag
- `retryCount`：当前重试次数，超过阈值后丢弃
- `createdAt`：入队时间戳，用于过期清理
- `critical`：是否关键事件（如 `submit_success`），关键事件优先补发

#### 入队规则

`trackEvent` 流程：

1. 组装 payload
2. 直接调用 `/api/track`
3. 成功则结束
4. 失败则写入本地队列

`trackEventOnce` 流程：

1. 检查 once flag 是否已存在
2. 已存在则跳过
3. 不存在则尝试发送
4. 发送成功后再写 once flag
5. 发送失败则入队，但**不写** once flag（重要：避免事件永久丢失）

#### 补发时机

建议在以下时机调用 `flushQueue()`：

- 应用/小程序回到前台时（`onShow` / `onResume`）
- 网络从断开恢复到可用时（监听网络状态变化）
- 登录成功后（此时可能有 `user_id` 补齐需求）
- 关键动作完成前（如提交成功弹窗前，确保埋点已发送）

#### 队列容量控制

推荐策略：

- 队列上限：50 条（避免占用过多存储）
- 单条事件最大重试次数：3 次
- 事件过期时间：24 小时（超过 24 小时的事件直接丢弃）
- 超过上限时：丢弃最旧的非关键事件，保留关键事件

#### 补发逻辑

```javascript
async function flushQueue() {
  const queue = getLocalQueue();
  if (queue.length === 0) return;

  const now = Date.now();
  const toRetry = queue.filter(item => {
    // 过期事件直接丢弃
    if (now - item.createdAt > 24 * 60 * 60 * 1000) return false;
    // 超过重试次数的事件丢弃
    if (item.retryCount >= 3) return false;
    return true;
  });

  // 关键事件优先
  toRetry.sort((a, b) => {
    if (a.critical && !b.critical) return -1;
    if (!a.critical && b.critical) return 1;
    return a.createdAt - b.createdAt;
  });

  const remaining = [];
  for (const item of toRetry) {
    try {
      await sendTrackingEvent(item.payload);
      // 成功后写入 once flag（如果是 once 事件）
      if (item.onceKey) {
        localStorage.setItem(item.onceKey, '1');
      }
    } catch (error) {
      // 失败则增加重试次数并重新入队
      item.retryCount += 1;
      remaining.push(item);
    }
  }

  saveLocalQueue(remaining);
}
```

#### 存储方式

- **Web**：`localStorage.setItem('trackingPendingQueue', JSON.stringify(queue))`
- **小程序**：`wx.setStorageSync('trackingPendingQueue', queue)`
- **App**：使用平台持久化存储 API

#### 注意事项

- 补发时不要阻塞主线程，使用异步批量发送
- 补发失败不要弹窗提示用户
- 队列满时优先保留关键事件（`submit_success`、`payment_success`）
- 定期清理过期事件，避免队列无限增长

## 后端实现模式

后端 tracking 服务建议拆成四层：

- `route/controller`：接收请求，提取 IP / user-agent，快速返回
- `validator/normalizer`：校验枚举、必填字段、类型，并规范化时间与空字符串
- `processor`：异步补充上下文并写入存储
- `sink adapter`：对接飞书、多维表、SQL、OLAP、消息队列等最终落点

### 推荐入口行为

- `POST /api/track`
- 对外尽量无状态
- 限流，避免被滥打
- 先快速验参，再异步处理
- 返回 `queued` / `processing` / `dropped` 之类的处理状态，便于观测

如果埋点量不大，可以同步写；如果埋点量持续增长，优先上“有界内存队列 + 后台任务”，再演进到消息队列。

默认推荐一个受控的后台处理模型：

- 有界队列，避免高峰期无限堆积
- 固定并发，避免打爆下游存储
- 队列满时可丢弃低价值事件，但要打印告警
- 主请求快速返回，实际写入在后台完成

### 服务端校验

后端至少要校验：

- 必填字段存在且非空
- `event_name` 属于允许枚举
- `source` / `device_type` 属于允许枚举
- `event_time` 可解析
- `duration_ms` 合法
- 某些事件必须带业务字段

事件级约束要按业务定义，例如：

- `auth_success` 必须带 `user_id`
- `flow_start` / `flow_complete` 必须带 `business_id`
- `submit_success` / `submit_fail` 必须带 `submit_id`

不要把所有事件强行塞成一套完全相同的必填规则。

### 服务端补充上下文

后端补充的数据通常包括：

- 请求 IP
- 用户代理
- IP 归属地/城市
- 服务端接收时间

注意：

- Geo enrich 是增强项，不应影响主链路
- 第三方 IP 服务失败时应降级为 `undefined`

## 存储设计：明细表 + 汇总表

这类埋点最稳定的结构不是单表，而是两层：

### 1. 事件明细表 `event_logs`

一行一条事件，用于：

- 排查问题
- 看完整时间线
- 看失败样本
- 回溯某个 `flow_id` 的真实轨迹

建议字段：

- 主键：`event_id`
- 时间：`event_time`、`event_date`
- 过程：`event_name`、`step_order`
- 关联：`flow_id`、`session_id`、`anonymous_id`、`user_id`
- 上下文：`page_path`、`page_title`、`source`、`device_type`、`browser`、`os`、`ip_city`
- 业务：`business_id`、`submit_id`、`is_success`、`duration_ms`
- 诊断：`error_code`、`error_msg`、`extra`

### 2. 流程汇总表 `flow_summary`

一行一个 `flow_id`，用于：

- 看漏斗转化
- 看当前走到哪一步
- 看流程是否结束
- 看成功/失败/未完成状态

建议字段：

- `flow_id`
- `start_time` / `end_time`
- `event_date`
- `anonymous_id` / `user_id` / `session_id`
- `source` / `device_type` / `browser` / `os` / `ip_city`
- 各阶段布尔值，例如 `is_visited`、`is_logged_in`、`is_started`、`is_completed`、`is_submitted`
- `current_step`
- `lost_step`
- `result` / `submit_result`
- `fail_reason`
- `stage_duration_ms` / `flow_duration_ms`
- `event_count`

### 汇总合并规则

按 `flow_id` upsert 时，推荐：

- `start_time` 取最早
- `end_time` 只在终态事件出现时更新
- `browser` / `os` / `ip_city` 取首个非空值
- `user_id` 允许后续事件补齐
- `current_step` 取最新阶段
- `result` 终态覆盖 `pending`
- `event_count` 每次加一

### 3. 使用飞书多维表格作为存储

飞书多维表格是一个轻量、可视化的埋点存储方案，适合中小规模埋点数据。

#### 为什么选择飞书多维表格

优势：

- 无需搭建数据库，开箱即用
- 可视化界面，非技术人员也能查看数据
- 支持筛选、分组、视图，便于快速分析
- 支持 Webhook 写入，配置简单
- 免费额度足够中小团队使用

适用场景：

- 日均埋点量 < 1 万条
- 团队规模 < 50 人
- 需要快速验证埋点方案
- 暂时没有专业 BI 工具

不适用场景：

- 日均埋点量 > 10 万条（建议用 ClickHouse、Doris 等 OLAP）
- 需要实时大屏（建议用 Grafana + 时序数据库）
- 需要复杂 SQL 分析（建议用 PostgreSQL、MySQL）

#### 单表 vs 双表方案对比

| 维度 | 单表（event_logs） | 双表（+ flow_summary） |
|---|---|---|
| 实施复杂度 | ★★★★★ 一个 webhook 搞定 | ★★☆☆☆ 需要两个 webhook 或同步逻辑 |
| 数据一致性 | ★★★★★ 天然一致 | ★★☆☆☆ 需要保证两表同步 |
| 查询性能 | ★★★★☆ 小数据量够用 | ★★★★★ 预聚合快 |
| 维护成本 | ★★★★★ 只维护一个表 | ★★☆☆☆ 维护两个表 |
| 灵活性 | ★★★★★ 可随时重算 | ★★★☆☆ 字段固定 |

**推荐方案：单表（event_logs）**

理由：

1. **简单可靠**：只需要一个 webhook，避免了两表同步的一致性问题
2. **性能够用**：日均 < 1 万条的场景下，飞书多维表格的查询性能完全够用
3. **维护成本低**：只需要维护一个表，字段变更、数据备份都更简单
4. **灵活性高**：所有分析都基于原始事件，可以随时重新计算任何维度
5. **演进友好**：如果未来需要 flow_summary，可以随时从 event_logs 重新计算

**何时需要双表方案**：

- 已经在使用专业数据库（PostgreSQL、ClickHouse）
- 日均埋点量 > 1 万条，需要预聚合提升性能
- 需要频繁查看实时漏斗转化
- 有专门的数据团队维护

#### 单表方案的飞书字段设计

**event_logs 表字段**：

| 字段名 | 飞书字段类型 | 是否必填 | 示例值 | 说明 |
|---|---|---:|---|---|
| event_id | 文本 | 是 | `evt_20260312_001` | 事件唯一 ID |
| event_time | 日期（包含时间） | 是 | `2026-03-12 10:00:00` | 事件发生时间 |
| event_date | 日期（不含时间） | 是 | `2026-03-12` | 方便按天筛选 |
| event_name | 单选 | 是 | `visit` | 事件名称 |
| step_order | 数字 | 是 | `1` | 流程步骤顺序 |
| flow_id | 文本 | 是 | `flow_abc123` | 完整流程 ID |
| session_id | 文本 | 是 | `sess_xyz123` | 会话 ID |
| anonymous_id | 文本 | 是 | `anon_u83hd2` | 匿名访客 ID |
| user_id | 文本 | 否 | `user_10086` | 登录后用户 ID |
| page_path | 文本 | 否 | `/assessment` | 页面路径 |
| page_title | 文本 | 否 | `测评页` | 页面标题 |
| source | 单选 | 否 | `wechat` | 流量来源 |
| device_type | 单选 | 否 | `mobile` | 设备类型 |
| browser | 文本 | 否 | `Chrome` | 浏览器 |
| os | 文本 | 否 | `iOS` | 操作系统 |
| ip_city | 文本 | 否 | `Shanghai` | IP 城市 |
| assessment_id | 文本 | 否 | `assess_001` | 业务 ID |
| submit_id | 文本 | 否 | `submit_001` | 提交 ID |
| is_success | 复选框 | 否 | `true` | 是否成功 |
| error_msg | 多行文本 | 否 | `timeout` | 错误信息 |
| duration_ms | 数字 | 否 | `32500` | 耗时（毫秒） |
| extra | 多行文本 | 否 | `{"version":1}` | 扩展字段 |

#### 推荐视图配置

基于单表 event_logs，可以创建以下视图满足常见分析需求：

**1. 今日事件总览**
- 筛选：`event_date = 今天`
- 分组：按 `event_name` 分组
- 统计：统计每个事件的数量

**2. 今日漏斗转化**
- 筛选：`event_date = 今天`
- 手动统计：
  - 访问数：`event_name = visit` 的 `flow_id` 去重数
  - 登录数：`event_name = login_success` 的 `flow_id` 去重数
  - 提交数：`event_name = submit_success` 的 `flow_id` 去重数

**3. 失败事件**
- 筛选：`is_success = false`
- 排序：按 `event_time` 倒序

**4. 按来源分析**
- 分组：按 `source` 分组
- 统计：统计每个来源的事件数

**5. 按 flow 查看**
- 分组：按 `flow_id` 分组
- 排序：按 `event_time` 升序
- 用途：查看某个 flow 的完整事件链

**6. 用户行为追踪**
- 筛选：输入特定 `user_id` 或 `anonymous_id`
- 排序：按 `event_time` 升序
- 用途：排查用户问题

#### 飞书集成方式：Webhook 工作流（推荐）

**为什么推荐 Webhook**：

- ✅ 无需管理飞书 API token（无需处理 token 刷新、过期）
- ✅ 无需处理飞书 API 限流
- ✅ 配置简单，5 分钟搞定
- ✅ 后端只需发送标准 HTTP POST 请求
- ✅ 单表方案只需要一个 webhook，避免一致性问题

**实施步骤**：

1. 在飞书多维表格中创建 `event_logs` 表（按上面的字段设计）
2. 点击表格右上角"自动化" → "创建自动化"
3. 触发条件选择"当收到 Webhook 请求时"
4. 动作选择"新增记录"
5. 配置字段映射（将 Webhook payload 的字段映射到表格字段）
6. 保存后获取 Webhook URL（类似 `https://open.feishu.cn/open-apis/bot/v2/hook/xxx`）

**后端实现示例**：

```typescript
// 发送埋点事件到飞书多维表格
async function sendTrackingEvent(payload: TrackingEventPayload) {
  try {
    const response = await axios.post(
      process.env.FEISHU_WEBHOOK_URL, // 飞书多维表格 webhook URL
      {
        event_id: payload.event_id,
        event_time: payload.event_time,
        event_date: payload.event_time.slice(0, 10), // 提取日期部分 "2026-03-13"
        event_name: payload.event_name,
        step_order: STEP_ORDER_MAP[payload.event_name],
        flow_id: payload.flow_id,
        session_id: payload.session_id,
        anonymous_id: payload.anonymous_id,
        user_id: payload.user_id || '',
        page_path: payload.page_path || '',
        page_title: payload.page_title || '',
        source: payload.source || 'direct',
        device_type: payload.device_type || 'desktop',
        browser: payload.browser || '',
        os: payload.os || '',
        ip_city: payload.ip_city || '',
        assessment_id: payload.assessment_id || '',
        submit_id: payload.submit_id || '',
        is_success: payload.is_success ?? true,
        error_code: payload.error_code || '',
        error_msg: payload.error_msg || '',
        duration_ms: payload.duration_ms || 0,
        extra: JSON.stringify(payload.extra || {}),
      },
      {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${process.env.FEISHU_WEBHOOK_TOKEN}`, // 如果需要鉴权
        },
      }
    );

    console.log('[Tracking] Event sent successfully:', payload.event_id);
  } catch (error) {
    console.error('[Tracking] Failed to send to Feishu:', error);
    // 失败不影响主流程，可以入队重试
  }
}
```

**完整请求示例**：

```bash
# 请求 URL
POST https://tvo7pfzu3em.feishu.cn/wiki/OIUewKdMyi2iLtksofmcgTMLnLf?table=tbly1FGwdiaBrwOM&view=vewwAzhS2P

# 请求头
Content-Type: application/json
Authorization: Bearer xxxxxxx

# 请求体（完整示例）
{
  "event_id": "evt_20260313_abc123",
  "event_time": "2026-03-13T10:30:45.000Z",
  "event_date": "2026-03-13",
  "event_name": "submit_success",
  "step_order": 5,
  "flow_id": "flow_xyz789",
  "session_id": "sess_def456",
  "anonymous_id": "anon_u83hd2k9",
  "user_id": "user_10086",
  "page_path": "/submit",
  "page_title": "提交页",
  "source": "wechat",
  "device_type": "mobile",
  "browser": "wechat-miniprogram",
  "os": "iOS 18.3",
  "ip_city": "Shanghai",
  "assessment_id": "assess_001",
  "submit_id": "submit_001",
  "is_success": true,
  "error_code": "",
  "error_msg": "",
  "duration_ms": 32500,
  "extra": "{\"client_platform\":\"miniprogram\",\"version\":\"1.2.3\"}"
}
```

**Node.js 完整示例**：

```javascript
const axios = require('axios');

// 环境变量配置
const FEISHU_WEBHOOK_URL = 'https://tvo7pfzu3em.feishu.cn/wiki/OIUewKdMyi2iLtksofmcgTMLnLf?table=tbly1FGwdiaBrwOM&view=vewwAzhS2P';
const FEISHU_WEBHOOK_TOKEN = 'xxxxxxx'; // 你的 token

// 事件步骤映射
const STEP_ORDER_MAP = {
  'visit': 1,
  'login_success': 2,
  'assessment_start': 3,
  'assessment_complete': 4,
  'submit_success': 5,
  'submit_fail': 5,
};

async function trackEvent(eventData) {
  try {
    const payload = {
      event_id: eventData.event_id,
      event_time: eventData.event_time,
      event_date: eventData.event_time.slice(0, 10),
      event_name: eventData.event_name,
      step_order: STEP_ORDER_MAP[eventData.event_name] || 0,
      flow_id: eventData.flow_id,
      session_id: eventData.session_id,
      anonymous_id: eventData.anonymous_id,
      user_id: eventData.user_id || '',
      page_path: eventData.page_path || '',
      page_title: eventData.page_title || '',
      source: eventData.source || 'direct',
      device_type: eventData.device_type || 'desktop',
      browser: eventData.browser || '',
      os: eventData.os || '',
      ip_city: eventData.ip_city || '',
      assessment_id: eventData.assessment_id || '',
      submit_id: eventData.submit_id || '',
      is_success: eventData.is_success !== false, // 默认 true
      error_code: eventData.error_code || '',
      error_msg: eventData.error_msg || '',
      duration_ms: eventData.duration_ms || 0,
      extra: JSON.stringify(eventData.extra || {}),
    };

    const response = await axios.post(FEISHU_WEBHOOK_URL, payload, {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${FEISHU_WEBHOOK_TOKEN}`,
      },
      timeout: 5000, // 5秒超时
    });

    console.log('[Tracking] Success:', response.data);
    return { success: true };
  } catch (error) {
    console.error('[Tracking] Failed:', error.message);
    // 失败不影响主流程
    return { success: false, error: error.message };
  }
}

// 使用示例
trackEvent({
  event_id: 'evt_' + Date.now(),
  event_time: new Date().toISOString(),
  event_name: 'submit_success',
  flow_id: 'flow_test123',
  session_id: 'sess_test456',
  anonymous_id: 'anon_test789',
  user_id: 'user_10086',
  page_path: '/submit',
  page_title: '提交页',
  source: 'wechat',
  device_type: 'mobile',
  browser: 'wechat-miniprogram',
  os: 'iOS 18.3',
  assessment_id: 'assess_001',
  submit_id: 'submit_001',
  is_success: true,
  duration_ms: 32500,
  extra: { client_platform: 'miniprogram' },
});
```

**安全建议**：

- Webhook URL 包含密钥，不要泄露到公开代码仓库
- 通过环境变量配置：`FEISHU_WEBHOOK_URL=https://...`
- 如果需要验证请求来源，可以在 payload 中加签名字段

**性能优化**：

- 使用后端队列批量发送，避免阻塞主请求
- Webhook 有频率限制（通常 100 次/分钟），需实现限流和重试
- 关键事件（如 `submit_success`）失败时重试，非关键事件失败可丢弃

**替代方案：飞书开放平台 API**

如果需要以下能力，可以考虑使用飞书 API：

- 批量写入（一次写入多条记录）
- 读取飞书表格数据
- 复杂的 upsert 逻辑（先查询，存在则更新）

但需要额外处理：

- 申请飞书应用，获取 app_id 和 app_secret
- 管理 access_token 刷新
- 处理 API 限流和重试
- 维护表 ID、字段 ID 映射关系

对于单表方案，Webhook 已经完全够用。

#### 注意事项

- 飞书多维表格单表行数上限为 50 万行，需定期归档
- Webhook 和 API 都有频率限制，需实现队列和重试
- 敏感字段（如手机号）不要直接写入飞书表格
- 建议每月导出数据备份到对象存储
- Webhook URL 需要保密，建议通过环境变量配置

## 事件命名与步骤建模

不要直接照搬某个业务的名字。先抽象出“阶段”和“结果”：

- 入口事件：`visit` / `entry`
- 身份事件：`login_success` / `auth_success`
- 流程开始：`application_start` / `checkout_start` / `assessment_start`
- 阶段完成：`application_complete` / `checkout_complete` / `assessment_complete`
- 终态结果：`submit_success` / `payment_success` / `submit_fail`

如果要做漏斗，最好给每个事件映射一个 `step_order`。

原则：

- 顺序要稳定
- 同一业务阶段只保留一个主事件名
- `fail` 事件可以与成功事件共享同一层级

## 鉴权策略

追踪接口应尽量与主业务鉴权解耦。

推荐策略：

- 默认允许匿名打点，但靠后端限流和字段校验防滥用
- 如果必须鉴权，也不要让用户 token 失效导致整个埋点体系失真
- 前端调用埋点接口时，避免把过期登录态处理逻辑绑在埋点请求上

实践上常见做法：

- 业务接口带 `Authorization`
- 埋点接口不带 `Authorization`
- `user_id` 作为业务上下文字段显式上传

## 隐私与边界

严禁直接写入：

- 手机号
- 身份证号
- token / cookie / session secret
- 完整对话文本
- 图片原文
- 任意高敏原始输入

建议只写：

- 匿名 ID
- 用户 ID
- 业务记录 ID
- 经过脱敏或摘要化的错误信息
- 必要的渠道和设备上下文

### 隐私合规检查清单

在实施埋点前，必须完成以下隐私合规检查。

#### 必须脱敏的字段

| 原始数据 | 处理方式 | 示例 |
|---|---|---|
| 手机号 | 不记录，或仅记录后四位 hash | `hash('1380')` |
| 身份证号 | 不记录 | - |
| 邮箱 | 不记录，或仅记录域名 | `@gmail.com` |
| 真实姓名 | 不记录，或使用 `user_id` 代替 | `user_12345` |
| IP 地址 | 仅记录城市级别 | `Shanghai` 而非 `123.45.67.89` |
| 完整对话内容 | 不记录，仅记录统计信息 | `message_count: 10, total_chars: 1500` |
| 上传图片 | 不记录原图，仅记录文件 ID | `file_abc123` |
| 支付金额 | 可记录，但需脱敏显示 | 记录真实金额，展示时脱敏 |

#### 数据保留策略

推荐保留时长：

- `event_logs`：90 天（用于问题排查）
- `flow_summary`：1 年（用于趋势分析）
- 敏感字段（`ip_city`、`browser`）：30 天后清空或匿名化
- 测试数据：7 天后删除

实施方式：

- 定期运行清理脚本
- 使用数据库 TTL 功能自动过期
- 归档到冷存储（如对象存储）后删除热数据

#### 用户权益保障

必须提供：

- 用户可查询自己的埋点数据（通过 `user_id` 或 `anonymous_id`）
- 用户可请求删除自己的埋点数据
- 埋点数据不用于用户画像或精准营销（除非明确告知并获得同意）

#### 合规文档

建议准备：

- 《埋点数据隐私政策》：说明收集哪些数据、用途、保留时长
- 《数据字段清单》：列出所有埋点字段及敏感级别
- 《数据访问权限表》：明确谁可以访问哪些数据

#### 开发阶段注意事项

- 测试环境使用脱敏数据或模拟数据
- 代码中不要硬编码真实手机号、身份证号
- 日志输出时自动脱敏敏感字段
- 定期审计埋点代码，确保没有新增敏感字段

## 演进策略

推荐按三个阶段推进：

### 阶段一：最小可用

- 打通统一事件协议
- 接入最小事件集合
- 建 raw log + flow summary
- 确保失败不阻塞主流程

### 阶段二：可靠性增强

- 增加失败重试队列
- 支持断点恢复时复用 `flow_id`
- 支持更多失败/放弃事件

### 阶段三：分析能力增强

- 增加 finer-grained 行为事件
- 补充归因字段
- 接入 BI / 仓库 / 实时看板

## 实施清单

实现时按这个顺序做：

1. 先画出业务漏斗，明确只追踪哪些阶段
2. 定义事件枚举和事件级必填字段
3. 定义 `anonymous_id` / `session_id` / `flow_id` 生成与存储策略
4. 实现客户端 tracker helper
5. 实现后端 `/api/track` 和统一校验
6. 实现 raw event sink
7. 实现 `flow_id` 维度的 summary upsert
8. 为关键事件补充 duration、result、fail reason
9. 做弱网、重复点击、断点恢复、重复提交验证

## 验收标准

完成后至少应满足：

- 同一 `flow_id` 能串起完整关键路径
- 一次性事件不会在同一流程内重复发送
- 失败事件不会阻塞主流程
- 后端能拒绝非法枚举和缺失必填字段
- 原始事件可回放，汇总表可直接看漏斗状态
- 替换存储介质时，客户端无需重写协议

### 测试用例清单

实施埋点后，必须完成以下测试用例验证。

#### 基础流程测试

- [ ] **完整流程**：`visit` → `login_success` → `flow_start` → `flow_complete` → `submit_success`
  - 验证：所有事件都成功发送，`flow_id` 一致
  - 验证：`flow_summary` 中 `is_submitted = true`

- [ ] **中途放弃**：`visit` → `login_success` → `flow_start`（无 `flow_complete`）
  - 验证：`flow_summary` 中 `current_step = flow_start`
  - 验证：`lost_step` 正确标记流失阶段

- [ ] **提交失败**：`visit` → ... → `submit_fail`
  - 验证：`submit_fail` 事件包含 `error_msg`
  - 验证：`flow_summary` 中 `submit_result = fail`

#### 边界测试

- [ ] **重复点击提交按钮**
  - 验证：`trackEventOnce` 的事件只发送一次
  - 验证：`trackEvent` 的事件可以发送多次

- [ ] **弱网环境下埋点失败**
  - 验证：埋点失败不阻塞业务提交
  - 验证：失败事件进入重试队列
  - 验证：网络恢复后自动补发

- [ ] **断点续传后 flow_id 保持一致**
  - 验证：恢复业务记录后，`flow_id` 与之前一致
  - 验证：续传后的事件与之前的事件在同一 `flow_id` 下

- [ ] **同一用户多次流程 flow_id 不同**
  - 验证：完成一次流程后，新流程生成新的 `flow_id`
  - 验证：`resetFlow()` 正确清理了旧的 `flow_id` 和 once flags

#### 数据一致性测试

- [ ] **event_logs 中同一 flow_id 的事件顺序正确**
  - 验证：按 `event_time` 排序后，`step_order` 递增
  - 验证：不存在 `step_order` 倒序的情况

- [ ] **flow_summary 的布尔字段与 event_logs 一致**
  - 验证：`is_logged_in = true` 时，`event_logs` 中存在 `login_success`
  - 验证：`is_submitted = true` 时，`event_logs` 中存在 `submit_success`

- [ ] **duration_ms 计算准确**
  - 验证：`assessment_complete` 的 `duration_ms` = 完成时间 - 开始时间
  - 验证：`submit_success` 的 `duration_ms` = 提交时间 - 开始时间

#### 字段校验测试

- [ ] **必填字段缺失时拒绝**
  - 验证：缺少 `event_id` 时返回 400 错误
  - 验证：缺少 `flow_id` 时返回 400 错误

- [ ] **枚举字段非法值时拒绝**
  - 验证：`event_name = 'invalid_event'` 时返回 400 错误
  - 验证：`source = 'unknown_source'` 时返回 400 错误

- [ ] **事件级约束校验**
  - 验证：`login_success` 缺少 `user_id` 时返回 400 错误
  - 验证：`submit_success` 缺少 `submit_id` 时返回 400 错误

#### 性能测试

- [ ] **高并发下不丢事件**
  - 验证：100 QPS 下，所有事件都成功入队
  - 验证：队列满时，低优先级事件被丢弃，关键事件保留

- [ ] **埋点接口响应时间 < 200ms**
  - 验证：P99 响应时间 < 200ms
  - 验证：异步处理不阻塞主请求

#### 隐私合规测试

- [ ] **敏感字段未写入**
  - 验证：`event_logs` 中不存在手机号、身份证号
  - 验证：`error_msg` 中不包含 token、密码

- [ ] **用户可查询自己的数据**
  - 验证：通过 `user_id` 可查询到该用户的所有埋点
  - 验证：通过 `anonymous_id` 可查询到匿名用户的埋点

#### 跨端一致性测试

- [ ] **Web 和小程序的 flow_id 格式一致**
  - 验证：两端生成的 `flow_id` 都是 `flow_` 前缀
  - 验证：长度和字符集一致

- [ ] **不同端的 event_name 枚举一致**
  - 验证：Web 和小程序都使用 `submit_success` 而非 `submitSuccess`
  - 验证：事件名大小写一致

## 常见反模式

- 每个页面自己拼埋点 payload
- 没有 `flow_id`，导致漏斗只能靠时间猜
- 把业务 `session_id`、登录 session、tracking session 混为一谈
- once 事件发送失败也直接置位，导致真实事件永远丢失
- 埋点失败阻塞提交、支付、下单等关键动作
- 在多个地方重复存 tracking 明细，最后口径对不上
- 把敏感原始数据直接写入埋点表

## 产出要求

当用户要求你“实现这套埋点”时，默认应产出：

- 一份事件协议与字段清单
- 一个客户端 tracker 模块
- 一个后端 tracking endpoint/service
- 一个 raw event 存储模型
- 一个 `flow_summary` 汇总模型
- 一份验收与排查说明

优先复用项目现有业务节点和数据实体命名，但保持这套追踪架构本身通用、可迁移、可替换落点。
