# 功能优化审查报告

> 审查范围：近期借鉴优秀功能开发 + 三项低风险项优化  
> 审查时间：2026-03-03

---

## 一、近期改动概览

| 模块 | 改动内容 | 状态 |
|------|----------|------|
| `rate-limiter.js` | `executeUrgent` 真正跳过排队，直接消耗令牌执行 | ✅ 已优化 |
| `security.js` | 定时器可清理（`startLoginLockCleanup` / `stopLoginLockCleanup`） | ✅ 已优化 |
| `common.js` | `withTimeout` 文档说明（超时不取消原任务） | ✅ 已优化 |
| `friend.js` | 三阶段巡查、封禁自动加黑、过滤统计 | ✅ 已实现 |
| `config-validator.js` | Schema 校验、`friend_three_phase` / `auto_blacklist_banned` | ✅ 已实现 |
| `admin.js` | 设置保存接入 `validateSettings` | ✅ 已接入 |

---

## 二、已发现的问题

### 1. `rate-limiter.js` 未被实际使用

**现象**：`core/src/services/rate-limiter.js` 中的 `RateLimiter`、`getDefaultLimiter`、`executeUrgent` 等均未被任何模块引用。

**影响**：当前限流逻辑由 `network.js` 内置的令牌桶（`enqueueSend` / `enqueueSendUrgent`）承担，`rate-limiter.js` 为独立实现，处于未接入状态。

**建议**：
- **方案 A**：若计划统一限流，可将 `network.js` 的限流逻辑迁移到 `rate-limiter.js`，由 `sendMsgAsync` / `sendMsgAsyncUrgent` 调用。
- **方案 B**：若暂不接入，在 `rate-limiter.js` 顶部增加注释说明「预留模块，待接入」，避免误用。

---

### 2. `scheduler-optimized.js` 未被使用

**现象**：`core/src/services/scheduler-optimized.js` 实现了时间轮调度器，但所有模块仍使用 `scheduler.js` 的 `createScheduler`。

**影响**：新增代码未参与运行，增加维护成本。

**建议**：
- 在 `scheduler.js` 中增加配置开关，支持切换到 `scheduler-optimized`；或
- 在 `scheduler-optimized.js` 顶部注明「可选实现，需手动切换」，避免被当作主实现。

---

### 3. `common.js` 工具库未被引用

**现象**：`core/src/utils/common.js` 中的 `withRetry`、`withTimeout`、`withRateLimit`、`createDailyCooldown`、`formatDuration`、`summarizeRewards` 等均未被项目引用。

**影响**：工具函数处于未使用状态，但已补充 `withTimeout` 文档，为后续使用做好准备。

**建议**：在需要重试、超时、限流或格式化时，优先从 `common.js` 引入，减少重复实现。

---

### 4. `config-validator` 缺少 `stealFriendFilter` Schema

**现象**：`SETTINGS_SCHEMA` 中未定义 `stealFriendFilter`，而 `data-provider.saveSettings` 会处理 `body.stealFriendFilter`。

**影响**：`stealFriendFilter` 会通过 `validate` 的兜底逻辑（`coerced[key] = input[key]`）透传，但不会被校验类型和结构。

**建议**：在 `config-validator.js` 中补充 `STEAL_FRIEND_FILTER_SCHEMA`，并在 `SETTINGS_SCHEMA` 中增加 `stealFriendFilter` 字段，保证结构一致、可校验。

---

## 三、潜在风险

### 1. `executeUrgent` 无超时保护

**说明**：`executeUrgent` 直接调用 `bucket.waitForToken()`，令牌耗尽时会一直等待，没有类似 `RequestQueue` 的 `timeoutMs` 限制。

**影响**：在令牌长期不足时，紧急请求可能长时间阻塞。

**建议**：若接入 `rate-limiter`，可为 `executeUrgent` 增加可选 `timeoutMs`，超时后 reject，避免无限等待。

---

### 2. `stopLoginLockCleanup` 与热重载

**说明**：热重载会重新加载模块，新实例会再次调用 `startLoginLockCleanup()`，旧实例的 `setInterval` 无法被新实例清除。

**影响**：热重载后可能同时存在多个清理定时器。

**建议**：在测试或热重载前显式调用 `stopLoginLockCleanup()`；若使用单例模式，可将 `_cleanupIntervalHandle` 放在全局对象上，便于跨实例清理。

---

### 3. 配置校验失败仍会保存

**说明**：`/api/settings/save` 在 `validation.valid === false` 时仅打日志，仍使用 `validation.coerced` 保存。

**影响**：部分字段类型错误时，会按默认值或强制转换后的值保存，可能不符合用户预期。

**建议**：若希望严格拦截错误配置，可增加「校验失败时拒绝保存」的模式；当前行为适合「尽量修正并保存」的策略，可保持现状并在文档中说明。

---

## 四、优化建议

### 1. 补充 `stealFriendFilter` Schema（推荐）

在 `config-validator.js` 中增加：

```javascript
const STEAL_FRIEND_FILTER_SCHEMA = {
    enabled: { type: 'boolean', default: false, label: '好友过滤开关' },
    mode: { type: 'string', enum: ['blacklist', 'whitelist'], default: 'blacklist', label: '好友过滤模式' },
    friendIds: { type: 'array', default: [], label: '好友ID列表' },
};

// 在 SETTINGS_SCHEMA 中增加：
stealFriendFilter: {
    type: 'object',
    label: '好友偷菜过滤',
    properties: STEAL_FRIEND_FILTER_SCHEMA,
},
```

---

### 2. 为 `executeUrgent` 增加可选超时（可选）

若后续接入 `rate-limiter`，可考虑：

```javascript
async executeUrgent(fn, { timeoutMs = 30000 } = {}) {
    if (!this._enabled) return fn();
    await Promise.race([
        this.bucket.waitForToken(),
        new Promise((_, rej) => setTimeout(() => rej(new Error('紧急请求等待令牌超时')), timeoutMs)),
    ]);
    return fn();
}
```

---

### 3. 在 `rate-limiter.js` 顶部增加说明（可选）

```javascript
/**
 * 令牌桶 + 优先级队列限流器
 *
 * 注意：本模块当前未被 network.js 等调用方接入，为预留实现。
 * 实际限流由 network.js 内置的 enqueueSend / enqueueSendUrgent 承担。
 */
```

---

### 4. 统一使用 `common.js` 工具函数（可选）

在需要重试、超时、限流或格式化时，优先从 `common.js` 引入，例如：

```javascript
const { withRetry, withTimeout, formatDuration } = require('../utils/common');
```

---

## 五、总结

| 类别 | 数量 | 说明 |
|------|------|------|
| 已修复/已优化 | 3 | `executeUrgent`、`security` 定时器、`withTimeout` 文档 |
| 未接入/未使用 | 3 | `rate-limiter`、`scheduler-optimized`、`common.js` |
| Schema 缺失 | 1 | `stealFriendFilter` 未在 config-validator 中定义 |
| 潜在风险 | 3 | `executeUrgent` 无超时、热重载定时器、校验失败仍保存 |

**结论**：近期功能优化未引入明显运行问题，主要改进点在于：将未使用模块标注清楚、补全 Schema、在接入 `rate-limiter` 时增加超时保护。当前实现可正常使用，建议按优先级逐步落实上述优化建议。

---

## 六、已落实优化（2026-03-03）

| 建议 | 状态 |
|------|------|
| 补充 stealFriendFilter Schema | ✅ 已落实 |
| executeUrgent 可选超时 | ✅ 已落实（timeoutMs 默认 30s） |
| rate-limiter 预留说明 | ✅ 已落实 |
| scheduler-optimized 可选说明 | ✅ 已落实 |
| security 热重载说明 | ✅ 已落实 |
| 配置校验严格模式 | ✅ 已落实（strictValidation 参数） |

---

## 七、二次审查（2026-03-03）

### 7.1 近期优化未引入新问题

对 2026-03-03 落地的 6 项优化逐项检查，未发现回归或逻辑错误：
- `stealFriendFilter` Schema 与 store 结构一致（enabled、mode、friendIds）
- `strictValidation` 使用 `=== true` 严格判断，避免字符串误触发
- `executeUrgent` 超时逻辑正确，`timeoutMs` 为 0 时保持原行为

### 7.2 新发现：data-provider 未处理 workflowConfig

**现象**：`Workflow.vue` 调用 `/api/settings/save` 时传入 `{ workflowConfig: config.value }`，但 `data-provider.saveSettings` 未将 `workflowConfig` 加入 `snapshot`。

**影响**：工作流编排保存请求会返回 200，但配置不会写入 store，用户修改的工作流会丢失。

**建议**：在 `data-provider.js` 的 `saveSettings` 中增加：
```javascript
if (body.workflowConfig !== undefined) {
    snapshot.workflowConfig = body.workflowConfig;
}
```

### 7.3 executeUrgent 超时后的 Promise 泄漏

**现象**：`Promise.race` 超时 reject 后，`bucket.waitForToken()` 仍会继续执行，直至获取令牌，造成轻微资源浪费。

**影响**：rate-limiter 当前未被使用，影响可忽略；若后续接入，可考虑用 `AbortController` 或封装可取消的 `waitForToken`。

### 7.4 前端与 /api/settings/save 的职责划分

**说明**：前端通过两条路径保存配置：
- `/api/settings/save`：plantingStrategy、intervals、friendQuietHours、stakeoutSteal、workflowConfig
- `/api/automation`：automation 开关、stealFilter、stealFriendFilter（扁平字段）

`stealFriendFilter` 经 `/api/automation` 保存，不经过 `/api/settings/save` 的 Schema 校验，当前设计如此，无问题。

### 7.5 优化建议汇总

| 优先级 | 建议 | 说明 | 状态 |
|--------|------|------|------|
| 高 | 修复 workflowConfig 保存 | 在 data-provider 中补充 workflowConfig 到 snapshot | ✅ 已落实 |
| 低 | 将 workflowConfig 加入 SETTINGS_SCHEMA | 便于统一校验与文档化 | ✅ 已落实 |

---

## 八、三次审查（2026-03-03）

### 8.1 偷菜设置点击不显示（RouterView 懒加载空白）✅ 已修复

**现象**：点击左侧「偷菜设置」时，右侧内容区域空白，需手动刷新才能显示。

**原因**：`StealSettings.vue` 使用 `defineAsyncComponent` 懒加载；客户端导航时，Vue Router 先开始导航，再异步加载组件。在组件加载完成前，`RouterView` 的 slot 中 `Component` 为 `undefined`，`<component :is="undefined" />` 渲染为空。

**修复**：在 `DefaultLayout.vue` 中，当 `Component` 为 `undefined` 时显示加载占位；将 `key` 从 `route.path` 改为 `route.fullPath`，确保嵌套路由下正确触发重渲染。

### 8.2 复核结论

- 近期功能优化未引入新的运行问题
- 已修复项（workflowConfig、stealFriendFilter、loading 闪烁等）均已正确落地
- 详细二次审查报告见：`docs/VIKING_CONTEXT_二次审查_2026-03-03.md`
