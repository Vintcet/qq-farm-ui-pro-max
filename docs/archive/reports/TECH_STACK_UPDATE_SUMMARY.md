# 📦 技术栈与部署文档更新总结

**更新日期**: 2026-03-02  
**版本**: v3.8.0  
**维护者**: smdk000

---

## 📋 更新概览

本次更新全面梳理和更新了 QQ 农场智能助手项目的技术栈说明和部署文档，确保文档与实际代码保持一致。

---

## ✅ 完成的更新

### 1. README.md 技术栈更新

#### 更新内容
- **版本号更新**: v3.2.5 → v3.8.0
- **新增技术徽章**: 
  - MySQL 8.0
  - Redis 6.0
- **后端技术栈详细化**:
  - 核心框架（Node.js 20+, Express 4.21.0, Socket.io 4.8.3）
  - 数据库与存储（MySQL 8.0, Redis, SQLite）
  - 日志与通知（Winston, pushoo）
  - 打包与部署（pkg 跨平台打包）
- **前端技术栈详细化**:
  - 核心框架（Vue 3.5.28, Vite 7.3.1, TypeScript 5.9.3）
  - 状态管理（Pinia 3.0.4, Vue Router 5.0.3）
  - UI 与样式（UnoCSS 66.5.12, 图标集）
  - 开发工具（ESLint, vue-tsc, rollup-plugin-visualizer）
- **部署技术栈**:
  - 容器化（Docker, Docker Compose, Buildx）
  - CI/CD（GitHub Actions）
  - 多平台支持（linux/amd64, linux/arm64）

#### 环境要求更新
- 细化为三个部署场景：
  - 源码运行（Node.js 20+, pnpm 10+, 可选 MySQL/Redis）
  - 二进制发布版（无需 Node.js）
  - Docker 部署（Docker 20+, Compose v2+）

#### 数据库架构更新
- **新增 MySQL/Redis 架构说明**（生产环境推荐）
  - 架构图
  - 核心表结构（users, cards, accounts, account_configs 等）
  - Redis 缓存策略（键设计、更新策略、TTL）
- **保留 SQLite 架构说明**（单机/离线模式）
  - 配置优化（WAL 模式、忙等待、自动检查点）

#### 数据流架构更新
- 区分两种架构的数据流：
  - MySQL/Redis 架构（生产环境）
  - SQLite 架构（单机/离线模式）

#### 最后更新时间
- 2026-03-01 → 2026-03-02
- 版本：v3.6.0 → v3.8.0

---

### 2. github-sync/README.md 技术栈更新

#### 更新内容
- **版本号更新**: v3.2.5 → v3.8.0
- **新增技术徽章**: MySQL 8.0, Redis 6.0
- **技术栈分类详细化**:
  - 后端技术（核心框架、数据库与存储、日志与通知、打包与部署）
  - 前端技术（核心框架、UI 与样式、开发与质量）
  - 部署与 DevOps（容器化、包管理、CI/CD）
- **环境要求细化**: 与主 README 保持一致

---

### 3. 新增 TECH_STACK.md 完整技术栈文档

#### 文档结构
创建了全新的完整技术栈说明文档，包含以下章节：

1. **整体架构**
   - 架构模式（前后端分离、单体仓库、微内核、多 Worker 并行）
   - 系统架构图（用户层、接入层、应用层、数据层、外部服务）

2. **后端技术栈**
   - 核心运行时（Node.js, pnpm）
   - Web 框架与通信（Express, Socket.io, Axios）
   - 数据库与存储（MySQL, Redis, SQLite, Protobuf）
   - 日志与通知（Winston, pushoo）
   - 打包与部署（pkg）
   - 完整的技术表格（版本、用途、官方文档链接）

3. **前端技术栈**
   - 核心框架（Vue, Vite, TypeScript, Pinia, Vue Router）
   - UI 与样式（UnoCSS, 图标集）
   - 开发与质量工具（ESLint, vue-tsc, 可视化分析）
   - 完整的技术表格

4. **数据库架构**
   - MySQL/Redis 架构（生产环境推荐）
     - 架构图
     - 核心表结构（带 SQL 建表语句）
     - Redis 缓存策略（键设计、更新策略）
   - SQLite 架构（单机/离线模式）
     - 配置优化（WAL、忙等待、检查点）

5. **部署技术栈**
   - 容器化（Docker, Docker Compose, Buildx）
   - 多平台支持（amd64, arm64）
   - CI/CD（GitHub Actions 工作流详解）
   - 部署脚本（deploy-arm.sh, deploy-x86.sh）
   - Docker Compose 配置示例

6. **开发工具链**
   - 代码编辑器推荐
   - VS Code 扩展推荐
   - 开发命令大全

7. **技术选型理由**
   - 为什么选择 Node.js
   - 为什么选择 Vue 3
   - 为什么选择 Vite
   - 为什么选择 MySQL/Redis 架构
   - 为什么选择 UnoCSS
   - 为什么选择 pnpm
   - 为什么选择 Docker 部署

8. **性能指标**
   - 构建性能（前端、后端、Docker）
   - 运行性能（启动时间、内存、CPU）
   - 响应性能（API、WebSocket、页面加载）

9. **安全考虑**
   - 密码安全（SHA256、AES-256）
   - 网络安全（HTTPS、CORS、速率限制）
   - 数据安全（SQL 注入、XSS、CSRF 防护）

10. **相关资源**
    - 官方文档链接
    - 技术社区链接

---

### 4. 部署脚本检查

#### 检查内容
- ✅ **deploy-arm.sh** - ARM64 服务器部署脚本
  - 版本标识：latest
  - 默认密码：qq007qq008
  - 默认端口：3080
  - 数据卷挂载：./data, ./logs, ./backup
  - 脚本状态：无需更新

- ✅ **deploy-x86.sh** - x86_64 服务器部署脚本
  - 版本标识：latest
  - 默认密码：qq007qq008
  - 默认端口：3080
  - 数据卷挂载：./data, ./logs, ./backup
  - 脚本状态：无需更新

- ✅ **docker-compose.prod.yml** - 生产环境配置
  - 镜像版本：latest
  - 资源限制：2 CPU, 1GB 内存
  - 健康检查：30 秒间隔
  - 日志轮转：10MB x 3 文件
  - 配置状态：无需更新

- ✅ **Dockerfile** - 多阶段构建
  - Node.js 版本：20-alpine
  - 构建阶段：builder, prod-deps, runner
  - 构建状态：无需更新

---

## 📊 技术栈对比

### 之前（v3.2.5 文档）
```
后端:
- Node.js 20+
- Express 4
- Socket.io 4
- SQLite

前端:
- Vue 3
- Vite 7
- TypeScript 5
- Pinia 3
- UnoCSS

部署:
- Docker
- pnpm 10
- GitHub Actions
```

### 现在（v3.8.0 文档）
```
后端:
- Node.js 20+ (推荐 22+)
- Express 4.21.0
- Socket.io 4.8.3
- Axios 1.16.0
- MySQL 8.0 (mysql2 3.18.2) ⭐ 新增
- Redis (ioredis 5.10.0) ⭐ 新增
- SQLite (better-sqlite3 12.6.2)
- Protobuf.js 8.0.0
- Winston 3.18.3
- pushoo 0.1.11
- pkg 5.8.1

前端:
- Vue 3.5.28 ⭐ 精确版本
- Vite 7.3.1 ⭐ 精确版本
- TypeScript 5.9.3 ⭐ 精确版本
- Pinia 3.0.4 ⭐ 精确版本
- Vue Router 5.0.3 ⭐ 新增
- Axios 1.13.5 ⭐ 新增
- Socket.io-client 4.8.3 ⭐ 新增
- @vueuse/core 14.2.1 ⭐ 新增
- UnoCSS 66.5.12 ⭐ 精确版本
- 图标集 (@iconify-json/*) ⭐ 新增
- ESLint 9.39.1 ⭐ 精确版本
- vue-tsc 3.2.5 ⭐ 新增

部署:
- Docker 20+ ⭐ 版本要求
- Docker Compose v2+ ⭐ 版本要求
- Docker Buildx ⭐ 多平台构建
- pnpm 10.30.2 ⭐ 精确版本
- GitHub Actions ⭐ 工作流详解
```

---

## 🎯 关键改进

### 1. 技术栈精确化
- 所有技术都标注了精确版本号
- 不再使用模糊的"Vue 3"、"Vite 7"等描述
- 便于开发者快速了解项目依赖

### 2. 架构清晰化
- 明确区分生产环境（MySQL/Redis）和单机模式（SQLite）
- 提供两种架构的完整说明
- 包含架构图和数据流图

### 3. 文档结构化
- 新增 TECH_STACK.md 作为完整技术栈参考
- README.md 保持简洁但信息完整
- github-sync/README.md 同步更新

### 4. 部署多样化
- 详细说明 5 种部署方式：
  1. 一键脚本部署（ARM/x86）
  2. Docker Compose 部署
  3. Docker 命令部署
  4. 源码部署
  5. 二进制文件部署
- 每种方式都有完整示例

### 5. 技术选型透明化
- 详细解释为什么选择每项技术
- 包含性能指标和基准测试
- 提供官方文档链接

---

## 📝 文档维护建议

### 定期更新
1. **版本更新时**:
   - 更新 README.md 的版本号
   - 更新 TECH_STACK.md 的版本号
   - 更新徽章版本
   - 更新最后更新日期

2. **依赖升级时**:
   - 同步更新 TECH_STACK.md 中的版本表格
   - 如有重大变更，更新技术选型理由

3. **架构变更时**:
   - 更新架构图
   - 更新数据流图
   - 更新数据库表结构

### 文档一致性
- 确保 README.md、github-sync/README.md、TECH_STACK.md 的版本号一致
- 确保技术栈描述一致
- 确保部署命令一致

### 链接检查
- 定期检查所有外部链接是否有效
- 更新失效的文档链接
- 确保 GitHub Issues、QQ 群等联系方式正确

---

## 🔗 相关文档

### 主要文档
- [README.md](../README.md) - 项目主说明文档
- [TECH_STACK.md](../TECH_STACK.md) - 完整技术栈说明
- [github-sync/README.md](../github-sync/README.md) - GitHub 同步版本说明

### 部署文档
- [docs/deployment/DEPLOYMENT_GUIDE_v3.6.0.md](../docs/deployment/DEPLOYMENT_GUIDE_v3.6.0.md) - v3.6.0 部署指南
- [docker-compose.prod.yml](../docker-compose.prod.yml) - 生产环境配置
- [scripts/deploy-arm.sh](../scripts/deploy-arm.sh) - ARM 服务器部署
- [scripts/deploy-x86.sh](../scripts/deploy-x86.sh) - x86 服务器部署

### 开发文档
- [CHANGELOG.DEVELOPMENT.md](../CHANGELOG.DEVELOPMENT.md) - 开发日志
- [docs/开发交接文档_20260302.md](../docs/开发交接文档_20260302.md) - 开发交接文档

---

## ✅ 检查清单

### 文档更新
- [x] README.md 技术栈更新
- [x] README.md 环境要求更新
- [x] README.md 数据库架构更新
- [x] README.md 版本号更新（v3.8.0）
- [x] github-sync/README.md 技术栈更新
- [x] TECH_STACK.md 创建
- [x] 部署脚本检查

### 内容验证
- [x] 所有版本号准确
- [x] 所有链接有效
- [x] 架构图清晰
- [x] 代码示例正确
- [x] 技术选型理由充分

### 格式检查
- [x] Markdown 格式正确
- [x] 徽章显示正常
- [x] 代码高亮正确
- [x] 表格对齐整齐
- [x] 目录结构清晰

---

## 📞 反馈与支持

如果您在文档中发现任何错误或有改进建议，请：

1. **提交 Issue**: https://github.com/smdk000/qq-farm-bot-ui/issues
2. **加入 QQ 群**: 227916149
3. **邮件联系**: （如有）

---

**更新完成时间**: 2026-03-02  
**维护者**: smdk000  
**版本**: v3.8.0
