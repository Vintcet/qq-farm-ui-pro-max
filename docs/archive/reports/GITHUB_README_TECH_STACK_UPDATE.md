# 📦 GitHub README 技术栈更新完成

**更新时间**: 2026-03-02  
**版本**: v3.8.0  
**维护者**: smdk000

---

## ✅ 更新内容

### 📊 技术栈展示优化

已将 github-sync/README.md 的技术栈部分进行全面升级，采用**表格化展示**，使技术信息更加清晰易读。

#### 主要改进

1. **表格化展示** - 所有技术栈采用表格形式，包含：
   - 类别分类
   - 技术名称
   - 精确版本号
   - 用途说明

2. **三大技术板块**：
   - 💻 **后端技术** - 14 项核心技术
   - 🎨 **前端技术** - 16 项核心技术
   - 🚀 **部署与 DevOps** - 9 项核心技术

3. **技术架构特点** - 新增四大特点说明：
   - 🏗️ 架构模式
   - 🗄️ 数据库架构
   - 🚀 性能优化
   - 🔒 安全加固

---

## 📋 更新详情

### 后端技术栈（14 项）

| 类别 | 技术 | 版本 | 用途 |
|------|------|------|------|
| 运行时 | Node.js | 20+ (推荐 22+) | JavaScript 运行时环境 |
| Web 框架 | Express | 4.21.0 | Web 应用框架 |
| 实时通信 | Socket.io | 4.8.3 | WebSocket 双向通信 |
| HTTP 客户端 | Axios | 1.16.0 | HTTP 请求库 |
| 主数据库 | MySQL | 8.0 | 生产环境主数据库 |
| MySQL 驱动 | mysql2 | 3.18.2 | MySQL 客户端库 |
| 缓存 | Redis | 6.0+ | 缓存/分布式锁/Pub/Sub |
| Redis 客户端 | ioredis | 5.10.0 | Redis 客户端库 |
| 本地存储 | SQLite | 3.x | 离线模式本地数据库 |
| SQLite 驱动 | better-sqlite3 | 12.6.2 | 同步 SQLite 客户端 |
| 协议缓冲 | Protobuf.js | 8.0.0 | QQ 农场协议解析 |
| 日志框架 | Winston | 3.18.3 | 日志记录 |
| 推送服务 | pushoo | 0.1.11 | 多渠道推送通知 |
| 打包工具 | pkg | 5.8.1 | Node.js 打包为二进制 |

### 前端技术栈（16 项）

| 类别 | 技术 | 版本 | 用途 |
|------|------|------|------|
| 框架 | Vue | 3.5.28 | 渐进式前端框架 (Composition API) |
| 构建工具 | Vite | 7.3.1 | 极速开发构建工具 |
| 语言 | TypeScript | 5.9.3 | JavaScript 超集 |
| 状态管理 | Pinia | 3.0.4 | Vue 状态管理库 |
| 路由 | Vue Router | 5.0.3 | Vue 官方路由 |
| HTTP 客户端 | Axios | 1.13.5 | HTTP 请求库 |
| WebSocket | Socket.io-client | 4.8.3 | WebSocket 客户端 |
| 工具库 | @vueuse/core | 14.2.1 | Vue Composition API 工具集 |
| 原子化 CSS | UnoCSS | 66.5.12 | 即时按需原子化 CSS 引擎 |
| 图标 | @iconify-json/carbon | 1.2.18 | Carbon 图标集 |
| 图标 | @iconify-json/fa-solid | 1.2.2 | Font Awesome 图标集 |
| 图标 | @iconify-json/svg-spinners | 1.2.4 | 加载动画图标 |
| 重置样式 | @unocss/reset | 66.5.12 | CSS Reset |
| 类型检查 | vue-tsc | 3.2.5 | TypeScript 类型检查 |
| 代码规范 | ESLint | 9.39.1 | 代码检查工具 |
| 可视化分析 | rollup-plugin-visualizer | 7.0.0 | 打包分析可视化 |

### 部署与 DevOps（9 项）

| 类别 | 技术 | 版本/平台 | 用途 |
|------|------|------|------|
| 容器引擎 | Docker | 20+ | 容器化部署 |
| 编排工具 | Docker Compose | v2+ | 多容器编排 |
| 多平台构建 | Docker Buildx | - | 跨平台镜像构建 |
| 镜像仓库 | Docker Hub | - | 公共镜像仓库 |
| 镜像仓库 | GitHub Container Registry | - | GitHub 镜像仓库 |
| 包管理器 | pnpm | 10.30.2 | 快速节省空间的包管理 |
| 工作空间 | pnpm workspace | - | Monorepo 管理 |
| CI/CD | GitHub Actions | - | 持续集成/部署 |
| 自动发布 | GitHub Releases | - | 二进制文件发布 |

---

## 🎯 新增技术架构特点

### 🏗️ 架构模式
- ✅ **前后端分离**: Vue 3 前端 + Node.js 后端
- ✅ **单体仓库 (Monorepo)**: pnpm workspace 管理
- ✅ **微内核架构**: 核心引擎 + 插件化服务
- ✅ **多 Worker 并行**: 每个账号独立 Worker 进程

### 🗄️ 数据库架构
- ✅ **生产环境**: MySQL 8.0 + Redis 6.0+ (主从 + 缓存)
- ✅ **单机模式**: SQLite 3.x (离线运行)
- ✅ **灵活切换**: 支持两种数据库架构平滑切换

### 🚀 性能优化
- ✅ **异步 I/O**: Node.js 非阻塞异步处理
- ✅ **连接池**: MySQL/Redis 连接池管理
- ✅ **缓存策略**: Redis 热点数据缓存 (TTL 智能过期)
- ✅ **令牌桶限流**: WebSocket 3 QPS 匀速请求
- ✅ **紧急通道**: 防偷抢收请求优先处理

### 🔒 安全加固
- ✅ **密码加密**: SHA256 哈希 + AES-256 加密存储
- ✅ **环境变量**: .env 隔离敏感配置
- ✅ **速率限制**: WebSocket 请求限流防封号
- ✅ **SQL 注入防护**: 参数化查询

---

## 📊 展示效果对比

### 之前
- ❌ 简单的列表式展示
- ❌ 缺少用途说明
- ❌ 技术分类不够清晰
- ❌ 缺少架构特点说明

### 现在
- ✅ **表格化展示** - 清晰易读
- ✅ **完整用途说明** - 每项技术都有详细用途
- ✅ **三大技术板块** - 后端、前端、部署分类明确
- ✅ **架构特点** - 新增架构模式、数据库架构、性能优化、安全加固四大特点
- ✅ **技术徽章** - 保留可视化技术图标徽章
- ✅ **平台支持** - 明确标注支持的平台

---

## 🔗 查看更新

更新后的 README 文件位置：
- [github-sync/README.md](../github-sync/README.md)

技术栈详细说明：
- [TECH_STACK.md](../TECH_STACK.md) - 完整技术栈文档

---

## 📝 下一步建议

### 同步更新
1. ✅ 主 README.md 已更新
2. ✅ github-sync/README.md 已更新
3. ✅ TECH_STACK.md 已创建
4. ✅ TECH_STACK_UPDATE_SUMMARY.md 已创建

### 后续维护
- 每次依赖升级时同步更新技术栈表格
- 保持所有文档版本号一致
- 定期检查外部链接有效性

---

## 📞 反馈与支持

如有任何问题或建议，请通过以下方式联系：

- **GitHub Issues**: https://github.com/smdk000/qq-farm-bot-ui/issues
- **QQ 群**: 227916149

---

**更新完成时间**: 2026-03-02  
**维护者**: smdk000  
**版本**: v3.8.0
