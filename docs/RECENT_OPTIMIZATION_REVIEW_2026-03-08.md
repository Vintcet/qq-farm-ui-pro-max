# 近期优化复盘与公告同步记录 (2026-03-08)

## 1. 记录范围

本轮复盘覆盖以下近期优化链路：

- 公告系统与 `Update.log` 同步
- 设置页「经营汇报历史」筛选、统计与分页
- UI 全局配置持久化与多端同步
- 登录页背景预览相关构建链路
- 账号列表可读性与细节体验

## 2. 本轮确认过的更新

### 2.1 公告链路

- 本地公告源继续以 `logs/development/Update.log` 为准。
- 服务端公告数据继续由 `announcements` 表承载。
- 为降低人工维护风险，`Update.log` 解析已增强为按日期标题行切段，而不是强依赖空行。

### 2.2 设置页经营汇报历史

- 新增后端统计接口 `/api/reports/history/stats`，可返回总数、成功数、失败数、测试汇报数、小时汇报数、日报数。
- 前端支持结果筛选、关键词搜索、导出、批量删除、分页和本地视图偏好保留。
- 切换账号时的重复刷新已去掉，减少无效请求和闪动。

### 2.3 UI 配置持久化

- `theme / loginBackground / overlayOpacity / blur / colorTheme / performanceMode / timestamp` 已统一进入 `normalizeUIConfig()` 做后端归一化。
- `auto` 主题现已允许在服务端落库与回读，多端同步时不会再被强制改写成 `dark`。

## 3. 已发现的问题、影响与处理

### 3.1 `/api/announcement` 被全局鉴权误拦

- 现象：代码注释写明“无需认证，公开接口”，但真实访问返回 `401 Unauthorized`。
- 影响：登录页/公开区域无法直接读取公告；线上排查时也必须借管理员态绕过。
- 处理：已将 `/announcement` 加入 `PUBLIC_PATHS`。

### 3.2 `auto` 主题在服务端持久化时失真

- 现象：前端可以切到“自动跟随”，但服务端规范化逻辑只接受 `light` / `dark`，导致回读时被压回深色。
- 影响：多端或刷新后，管理员设置的自动模式会丢失。
- 处理：已更新 `normalizeUIConfig()` 与 `setUITheme()`，允许 `auto` 持久化。

### 3.3 经营汇报历史在切账号时重复拉取

- 现象：`currentAccountId` 变化时，`loadData()` 会刷新一次汇报历史，同时筛选 watcher 也会再次刷新。
- 影响：每次切换账号都会多打一轮 `/api/reports/history` 和 `/api/reports/history/stats`，增加噪声和潜在闪动。
- 处理：已将筛选 watcher 改为只监听筛选项，不再监听 `currentAccountId`。

### 3.4 `Update.log` 对人工排版过于敏感

- 现象：若两个公告块之间缺少空行，旧解析器会把相邻条目合并，导致同步少条目。
- 影响：公告数据库会遗漏版本记录，前台展示顺序和数量都可能异常。
- 处理：
  - 已补齐当前 `Update.log` 中的缺失分隔。
  - 已将解析逻辑改为按日期标题行切段。

### 3.5 登录背景编辑器脚本阻断构建

- 现象：登录背景预览相关方法尚未完全接入模板，但脚本已存在，`vue-tsc` 将其判定为未使用。
- 影响：前端构建失败，影响打包与部署。
- 处理：已采用最小保留方式让这组逻辑继续存在但不阻塞编译，未改业务行为。

## 4. 当前残余风险

- 公告同步目前仍以“标题 + 版本”或“标题 + 日期”去重。若未来同一天发布同标题但正文不同的补充公告，可能仍需更稳定的指纹去重策略。
- `Update.log` 现在虽然更稳健，但仍属于人工维护文本。若后续公告频率继续上升，建议引入脚本化校验或直接从结构化源生成。
- 设置页经营汇报统计当前是“基于当前筛选结果集”的统计口径。如果后续想显示“全量汇报总览”，需要和当前筛选统计拆成两套接口。

## 5. 下一步优化建议

- 为 `parseUpdateLog()` 增加单元测试，覆盖“缺失空行”“同一天多条公告”“无版本号条目”等情况。
- 为公告同步增加稳定哈希去重键，例如 `date + title + content hash`，避免只靠标题和版本。
- 为经营汇报历史请求增加取消前一请求的能力，防止快速切换筛选时旧响应覆盖新状态。
- 若后续准备再次上线服务器版本，建议把本轮 3 个代码修复一并发布：公开公告读取、`auto` 主题持久化、汇报历史去重拉取。

## 6. 本轮验证

- `node -c core/src/controllers/admin.js`
- `node -c core/src/models/store.js`
- `pnpm -C web exec vue-tsc -b`
- `pnpm -C web build`

## 7. 本轮补充优化（登录背景 / 汇报统计 / 精细出售）

### 7.1 已纳入的新增功能

- 登录页背景新增了内置预设、本地上传、遮罩透明度、模糊度四项能力。
- 主界面现在可以按 `backgroundScope` 继承同一张背景，并单独配置业务页的遮罩强度与模糊度。
- 主题抽屉支持一键套用匹配的主题背景预设，颜色主题和氛围背景能同时切换。
- 经营汇报历史新增统计卡片、排序、视图偏好记忆以及“最新失败”快捷入口。
- 背包出售链路由“按合并显示项出售”升级为“按原始背包条目和 UID 拆分出售”。
- 启动账号时，数据库中的完整账号记录现在会覆盖列表缓存，避免新登录态被旧快照回填。

### 7.2 已确认的发布风险

- `core/src/cluster/worker-client.js` 使用了 `socket.io-client`，但之前 `core/package.json` 没声明该运行时依赖。
- 影响：Docker / 二进制发布在集群 Worker 场景下，存在运行时缺模块风险。
- 处理：本轮已补到 `core/package.json`。
- 补充确认：本地执行 `pnpm install` 刷新 workspace 依赖后，`pnpm -C core build:release` 已无该项 `pkg` 警告。

### 7.3 当前影响判断

- 登录背景上传能力会把文件落到 `data/ui-backgrounds/`，目前没有自动清理旧背景文件的机制。
- 外链背景预设仍依赖第三方图床可访问性，若远端防盗链或失效，预设图可能无法显示。
- 本轮已发现并补齐樱花与赛博主题缺失的内置 SVG 资源，否则主题抽屉的一键背景功能会出现静态资源 404。
- 汇报历史统计当前是“基于当前筛选结果”的统计，不是全量总览，这一点需要在后续文案和接口设计上继续保持清晰。

### 7.4 建议

- 为登录背景上传增加“删除旧自定义背景”或“定期清理未引用文件”的机制，避免 `data/ui-backgrounds/` 长期膨胀。
- 将外链背景预设逐步替换为本地托管资源，减少第三方图片站的可用性波动。
- 若后续继续扩充经营汇报分析，建议增加“全量统计”和“当前筛选统计”两套独立口径，避免用户误解。

## 8. 本轮热修补丁（前端 lint / CI）

- GitHub `main` 分支本轮首次推送后，失败点已定位在 `pnpm -C web lint`，不是前端构建或后端打包。
- 处理方式：对 `Settings.vue` 的主题联动卡片绑定、`ui-appearance.ts` 的类型声明、若干 Vue 文件的 UnoCSS/样式顺序做了最小规范收口。
- 结果：`pnpm -C web lint`、`pnpm -C web build`、`pnpm -C core build:release` 已重新通过。
- 结论：本轮新增功能链路本身可用，`v4.5.10` 主要是把远端 CI 与当前前端规范基线重新对齐。

## 9. 二次复查补充（2026-03-08 23:10）

### 9.1 新确认的问题

- **主题整套联动在重构后参数不完整**:
  `getThemeAppearanceConfig()` 一度只返回登录页背景与登录页遮罩/模糊参数，导致“5 套主题联动方案”和“主题锁定背景”在切换主题后，主界面遮罩/模糊参数并不会一起更新。
- **主界面视觉预设处于半接入状态**:
  `workspaceVisualPreset`、`UI_WORKSPACE_VISUAL_PRESETS` 和对应的服务端持久化都已经接入，但设置页没有实际入口，只能靠临时绑定数组避免 lint 报未使用。

### 9.2 影响判断与处理

- **用户感知层面**:
  主题卡片文案宣称会同步“主界面参数”，但实际行为只改登录页，属于明确的功能感知不一致。
- **维护层面**:
  主界面视觉预设在代码层存在、在界面层缺席，后续很容易被误判成“功能已上线”，增加复查和交接成本。
- **当前处理**:
  - 已恢复主题整套联动的主界面参数同步。
  - 已在设置页补上主界面视觉预设可视化卡片。
  - 已删除仅用于规避 lint 的占位绑定，改为真实模板接入。

### 9.3 目前剩余风险与建议

- **主题联动范围回退已补正**:
  原本在开启“主题锁定背景”后从右侧抽屉切主题，或直接点击抽屉中的“套用主题背景”，会把已保存的 `global` 背景范围默认写回 `login_and_app`。现已改为保留用户当前作用范围，仅在非全局模式下继续套用“登录页 + 主界面”主题联动。
- **主题联动混合态提示已补正**:
  实测联调发现，`workspaceVisualPreset` 会保留上一次手动选择的业务页风格，而“主题锁定背景”会单独注入当前主题的主界面遮罩/模糊参数，导致设置页顶部一度误显示为某个预设。现已改为按真实组合识别，混合态统一显示为“主题联动自定义”，避免把“海报沉浸版 / 控制台弱化版”等名称误当成当前实际参数。
- **整套主题已补齐业务页风格写入**:
  当前已为 5 套主题明确补上业务页风格映射，并把 `workspaceVisualPreset` 一并纳入主题联动保存链路。实测 `Ocean` 整套在开启 `themeBackgroundLinked` 且作用范围为 `global` 时，保存后服务端返回值已同步为 `workspaceVisualPreset: pure_glass`，不再残留旧的手动预设值。
- **地块类道具多选已补正**:
  背包详情里像浇水、除草、除虫、播种这类带 `land_ids` 的使用操作，原先允许选中的地块数超过当前物品库存，前端又会把 `count` 截断成库存上限，形成“文案显示按已选地块数消耗，但请求实际只带较小 count”的不一致。现已改为在 UI 侧按库存数量限制可选地块，并在使用成功后立即刷新土地列表状态。
- **兼容 UseRequest 的 fallback 分支已补写 land_ids**:
  `warehouse.useItem()` 在遇到旧接口编码兼容分支时，原先只写了 `{ item: { id, count } }`，没有继续携带 `land_ids`。这会让土地类道具在特殊兼容路径下失去目标地块参数。现已在 fallback 请求中补齐 repeated `land_ids` 字段。
- **后续建议**:
  建议补一条最小化的 `bag/use` 集成校验，至少覆盖“土地类道具 + `land_ids` + fallback 编码”的请求体构造。当前 `lint` / `build` / `node --check` 能兜住语法和构建，但仍无法替代真实协议层回归。
- **外链字体告警**:
  已处理。`web/uno.config.ts` 已移除 Google Fonts 在线拉取，改为本地字体栈，`pnpm -C web build` 不再出现此前的 Web Fonts 拉取失败告警。
- **背景与图标缓存增长**:
  已处理。服务端已新增未引用背景和过期生成图标缓存的清理逻辑，风险从“无清理机制”降为“后续按实际容量观察阈值是否需要再调优”。
- **背景预设可用性**:
  已处理当前已知外链项。示例背景 `sample-red-horse` 已改为本地 SVG 资源，当前登录背景预设已不再依赖第三方图片站。

### 9.4 补充验证

- `node --check core/src/config/gameConfig.js`
- `node --check core/src/controllers/admin.js`
- `node --check core/src/models/store.js`
- `node --check core/src/services/ui-assets.js`
- `node --check core/src/services/mall.js`
- `pnpm test:ui-assets`
- `pnpm -C web check:ui-appearance`
- `pnpm -C web lint`
- `pnpm -C web build`
- `pnpm -C core build:release`

## 10. 建议执行结果（2026-03-08 23:35）

### 10.1 已执行项

- **自动清理未引用背景文件**:
  已新增 `core/src/services/ui-assets.js`，在服务端启动、保存主题配置、上传新背景时都会清理过期且未引用的 `ui-backgrounds` 文件。
- **自动清理过期生成图标缓存**:
  `gameConfig` 加载时会清理过期或无效的 `data/asset-cache/item-icons` 生成 SVG 缓存，避免长期累积。
- **主题联动最小自动校验**:
  已新增 `web/scripts/check-ui-appearance.mjs`，会校验主题背景配置是否同时包含登录页和主界面参数。
- **本地化字体与示例背景**:
  已移除 UnoCSS 的在线字体拉取，并把示例酒红背景改为仓库内置 `crimson-velvet.svg`。

### 10.2 补充验证

- `pnpm test:ui-assets`
- `pnpm -C web check:ui-appearance`
- `pnpm -C web lint`
- `pnpm -C web build`
- `pnpm -C core build:release`

## 11. 补充复查（2026-03-09）

### 11.1 本轮新增确认

- **SMTP 邮件汇报已全链路接入**:
  `reportConfig.channel` 新增 `email`，设置页已可维护 `smtpHost / smtpPort / smtpSecure / smtpUser / smtpPass / emailFrom / emailTo`，服务端归一化、配置校验、汇报可用性判断和推送下发链路已保持一致。
- **账号保存后立即持久化**:
  管理端保存账号成功后会直接调用 `persistAccountsNow()`，减少扫码登录成功后尚未等到批量落库就异常退出的风险。
- **好友拉取兼容模式改为按账号缓存**:
  `SyncAll / GetAll` 的探测结果现在以账号维度缓存，不再一个账号切到兼容模式后影响整台机器所有账号。
- **好友日志噪声已压低**:
  好友列表调试日志和周期状态日志增加 TTL 去重，长时间挂机时更容易看见真正的新异常。
- **背包使用链路继续补正**:
  `worker` 已补齐 `useBagItem` 调用面，土地类道具在旧编码 fallback 分支也会继续携带 `land_ids`。

### 11.2 本轮验证

- `git diff --check`
- `node --check core/src/services/smtp-mailer.js`
- `node --check core/src/services/push.js`
- `node --check core/src/services/report-service.js`
- `node --check core/src/config-validator.js`
- `pnpm test:ui-assets`
- `pnpm -C web check:ui-appearance`
- `pnpm -C web exec vue-tsc --noEmit`
- `pnpm -C web build`
- `pnpm -C core build:release`

### 11.3 当前仍建议关注

- `smtp-mailer` 采用的是手写 SMTP 协议实现，当前适合纯文本经营汇报；如果后面要支持更复杂的 HTML 模板、附件或更复杂的认证兼容，建议补集成测试并考虑是否引入成熟邮件库。
- 好友拉取模式虽然已按账号缓存，但仍建议在 QQ / 微信混跑环境各做一次实机回归，确认探测结论不会受平台风控瞬时波动误导。
- 背包土地类道具的 `land_ids` fallback 已补齐，但这类问题更偏协议兼容，后续最好补一条最小集成回归，而不是只依赖构建和静态检查。
