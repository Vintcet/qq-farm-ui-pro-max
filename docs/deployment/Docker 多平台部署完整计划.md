# QQ 农场助手 - Docker 多平台构建与部署完整计划

> 历史说明（2026-03-07）：本文保留旧版本执行记录，当前有效工作流以 `docs/guides/REPO_ROOT_WORKFLOW_GUIDE.md` 为准；日志挂载请使用 `./logs:/app/logs`。

**版本**: v3.3.3  
**创建日期**: 2026-03-01  
**作者**: smdk000  
**状态**: ✅ 已完成

---

## 📋 目录

1. [项目概况](#项目概况)
2. [Docker 多平台构建计划](#docker-多平台构建计划)
3. [一键部署脚本](#一键部署脚本)
4. [数据库保护方案](#数据库保护方案)
5. [版本更新内容保护](#版本更新内容保护)
6. [部署命令速查](#部署命令速查)
7. [注意事项](#注意事项)
8. [文件清单](#文件清单)

---

## 🎯 项目概况

### 当前版本信息

- **版本号**: v3.3.3（最新）
- **发布日期**: 2026-03-01
- **Docker 镜像**: `smdk000/qq-farm-bot-ui:3.3.3`
- **支持平台**: 
  - ✅ linux/amd64 (x86_64)
  - ✅ linux/arm64 (ARM64)
- **Docker 仓库**: Docker Hub (@smdk000)

### 核心功能

- ✅ 多账号管理（支持 QQ 和微信）
- ✅ 自动化农场管理（收获、种植、浇水、除草、除虫）
- ✅ 智能防护（60 秒防偷抢收、两季作物识别）
- ✅ Web 控制面板（实时日志、数据分析）
- ✅ 多用户系统（卡密管理、权限控制）
- ✅ 主题切换（5 大主题、深色模式）

---

## 🏗️ Docker 多平台构建计划

### 步骤 1: 环境准备 ✅

```bash
# 1.1 确认 Docker 已安装
docker --version
# 期望输出：Docker version 20.x.x+

# 1.2 确认 Docker Buildx 可用
docker buildx version
# 期望输出：github.com/docker/buildx v0.x.x

# 1.3 登录 Docker Hub
docker login
# 输入用户名：smdk000
# 输入密码
```

### 步骤 2: 创建 Buildx Builder ✅

```bash
# 2.1 创建专用的 builder
docker buildx create --use --name qq-farm-builder

# 2.2 启动 builder
docker buildx inspect --bootstrap qq-farm-builder

# 2.3 验证 builder 状态
docker buildx inspect qq-farm-builder
```

### 步骤 3: 构建并推送多平台镜像

#### 方案 A: 使用现有脚本（推荐）✅

```bash
# 3.1 赋予执行权限
chmod +x scripts/docker-build-multiarch.sh

# 3.2 执行构建（默认版本 3.3.3）
./scripts/docker-build-multiarch.sh 3.3.3

# 或使用最新版本号
./scripts/docker-build-multiarch.sh
```

#### 方案 B: 手动构建（灵活控制）✅

```bash
# 3.1 单独构建 AMD64 版本
docker buildx build \
  --platform linux/amd64 \
  -t smdk000/qq-farm-bot-ui:3.3.3-amd64 \
  -t smdk000/qq-farm-bot-ui:latest-amd64 \
  -f core/Dockerfile . \
  --push

# 3.2 单独构建 ARM64 版本
docker buildx build \
  --platform linux/arm64 \
  -t smdk000/qq-farm-bot-ui:3.3.3-arm64 \
  -t smdk000/qq-farm-bot-ui:latest-arm64 \
  -f core/Dockerfile . \
  --push

# 3.3 创建多平台镜像清单
docker manifest create smdk000/qq-farm-bot-ui:3.3.3 \
  smdk000/qq-farm-bot-ui:3.3.3-amd64 \
  smdk000/qq-farm-bot-ui:3.3.3-arm64

docker manifest annotate smdk000/qq-farm-bot-ui:3.3.3 \
  smdk000/qq-farm-bot-ui:3.3.3-arm64 --arch arm64 --os linux

# 3.4 创建 latest 多平台清单
docker manifest create smdk000/qq-farm-bot-ui:latest \
  smdk000/qq-farm-bot-ui:latest-amd64 \
  smdk000/qq-farm-bot-ui:latest-arm64

docker manifest annotate smdk000/qq-farm-bot-ui:latest \
  smdk000/qq-farm-bot-ui:latest-arm64 --arch arm64 --os linux

# 3.5 推送清单
docker manifest push smdk000/qq-farm-bot-ui:3.3.3
docker manifest push smdk000/qq-farm-bot-ui:latest
```

### 步骤 4: 验证构建 ✅

```bash
# 4.1 查看镜像信息
docker manifest inspect smdk000/qq-farm-bot-ui:3.3.3

# 4.2 在 Docker Hub 查看
# https://hub.docker.com/r/smdk000/qq-farm-bot-ui/tags
```

---

## 🚀 一键部署脚本

### ARM 服务器部署脚本

**文件**: `scripts/deploy-arm.sh`

```bash
#!/bin/bash
# ARM 服务器一键部署脚本
# 适用于树莓派、鲲鹏、飞腾等 ARM64 架构服务器

# 使用方法
chmod +x scripts/deploy-arm.sh
./scripts/deploy-arm.sh

# 自定义密码和端口
ADMIN_PASSWORD=YourStrongPassword123! PORT=3080 ./scripts/deploy-arm.sh
```

### x86 服务器部署脚本

**文件**: `scripts/deploy-x86.sh`

```bash
#!/bin/bash
# x86 服务器一键部署脚本
# 适用于 Intel、AMD 等 x86_64 架构服务器

# 使用方法
chmod +x scripts/deploy-x86.sh
./scripts/deploy-x86.sh

# 自定义密码和端口
ADMIN_PASSWORD=YourStrongPassword123! PORT=3080 ./scripts/deploy-x86.sh
```

### Docker Compose 部署

**文件**: `docker-compose.deploy.yml`

```yaml
version: '3.8'

services:
  qq-farm-bot-ui:
    image: smdk000/qq-farm-bot-ui:3.3.3
    container_name: qq-farm-bot-ui
    hostname: qq-farm-bot
    
    restart: unless-stopped
    
    ports:
      - "3080:3000"
    
    environment:
      - ADMIN_PASSWORD=qq007qq008
      - TZ=Asia/Shanghai
      - NODE_ENV=production
      - LOG_LEVEL=info
    
    volumes:
      - ./data:/app/core/data
      - ./logs:/app/logs
      - ./backup:/app/core/backup
    
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/api/ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        compress: "true"
```

---

## 🛡️ 数据库保护方案

### 数据卷挂载

| 宿主机路径 | 容器内路径 | 说明 |
|-----------|-----------|------|
| `./data` | `/app/core/data` | 核心数据库目录（账号配置、用户数据等） |
| `./logs` | `/app/logs` | 日志文件目录（运行日志、操作日志等） |
| `./backup` | `/app/core/backup` | 备份文件目录（数据库备份、配置备份等） |

### 备份策略

#### 1. 定期备份（推荐每天执行）

```bash
# 每天凌晨 2 点自动备份
0 2 * * * tar -czf /backup/farm-bot-$(date +\%Y\%m\%d).tar.gz ./data
```

#### 2. 升级前备份

```bash
# 升级前手动备份
tar -czf farm-bot-backup-$(date +%Y%m%d).tar.gz ./data

# 停止旧容器
docker stop qq-farm-bot-ui
docker rm qq-farm-bot-ui

# 拉取新镜像并启动
docker pull smdk000/qq-farm-bot-ui:latest
docker run -d --name qq-farm-bot-ui ...
```

#### 3. 从备份恢复

```bash
# 停止容器
docker stop qq-farm-bot-ui
docker rm qq-farm-bot-ui

# 恢复数据
tar -xzf farm-bot-backup-20260301.tar.gz -C ./data

# 重新启动容器
docker run -d --name qq-farm-bot-ui ...
```

### ⚠️ 重要提醒

- ❌ **不要删除** `./data` 目录，否则所有账号配置和用户数据将丢失
- ❌ **不要手动修改** 数据库文件，可能导致数据损坏
- ✅ **定期备份** 数据到安全位置（云存储、外部硬盘等）
- ✅ **升级前备份**，升级失败可快速回滚

---

## 📝 版本更新内容保护

### Update.log 文件保护机制

✅ **Update.log 已包含在 Docker 镜像中**

- **位置**: `/app/Update.log`
- **来源**: 项目根目录的 `Update.log` 文件
- **构建时自动包含**: Dockerfile 中已配置

```dockerfile
# Dockerfile 第 20 行
COPY Update.log ./Update.log
```

### 确保 Update.log 最新

在构建 Docker 镜像前，确保 `Update.log` 文件是最新的：

```bash
# 1. 查看当前 Update.log
cat Update.log

# 2. 更新 Update.log（如有必要）
# 编辑 Update.log 文件，添加最新版本信息

# 3. 构建 Docker 镜像
./scripts/docker-build-multiarch.sh 3.3.3
```

### 版本更新内容

**v3.3.3 更新内容**（已在 Update.log 中）:

- ✅ 回归修复：深色模式兼容性与性能模式覆盖遗漏
- ✅ 修复 `HelpCenter.vue` 独立重定义 `backdrop-filter`
- ✅ 修复 `Friends.vue` Scoped CSS 中 `.dark` 选择器
- ✅ 修复 `NotificationModal.vue` 底部动作条样式

---

## 📋 部署命令速查

### ARM 服务器部署

```bash
# 方法 1: 使用一键脚本
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-bot-ui/main/scripts/deploy-arm.sh
chmod +x deploy-arm.sh
./deploy-arm.sh

# 方法 2: 手动部署
docker pull smdk000/qq-farm-bot-ui:3.3.3
docker run -d \
  --name qq-farm-bot-ui \
  -p 3080:3000 \
  -v ./data:/app/core/data \
  -e ADMIN_PASSWORD=your_password \
  smdk000/qq-farm-bot-ui:3.3.3
```

### x86 服务器部署

```bash
# 方法 1: 使用一键脚本
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-bot-ui/main/scripts/deploy-x86.sh
chmod +x deploy-x86.sh
./deploy-x86.sh

# 方法 2: 手动部署
docker pull smdk000/qq-farm-bot-ui:3.3.3
docker run -d \
  --name qq-farm-bot-ui \
  -p 3080:3000 \
  -v ./data:/app/core/data \
  -e ADMIN_PASSWORD=your_password \
  smdk000/qq-farm-bot-ui:3.3.3
```

### Docker Compose 部署

```bash
# 1. 下载配置文件
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-bot-ui/main/docker-compose.deploy.yml

# 2. 启动服务
docker-compose -f docker-compose.deploy.yml up -d

# 3. 查看状态
docker-compose -f docker-compose.deploy.yml ps

# 4. 查看日志
docker-compose -f docker-compose.deploy.yml logs -f
```

### 常用命令

```bash
# 查看状态
docker ps
docker stats qq-farm-bot-ui
docker inspect qq-farm-bot-ui

# 查看日志
docker logs -f qq-farm-bot-ui
docker logs --tail 100 qq-farm-bot-ui
docker logs --since 1h qq-farm-bot-ui

# 重启容器
docker restart qq-farm-bot-ui
docker stop qq-farm-bot-ui
docker start qq-farm-bot-ui

# 清理旧版本
docker rm qq-farm-bot-ui
docker rmi smdk000/qq-farm-bot-ui:old-version

# 更新镜像
docker pull smdk000/qq-farm-bot-ui:latest
docker-compose pull
docker-compose up -d

# 进入容器
docker exec -it qq-farm-bot-ui /bin/sh
docker exec -it qq-farm-bot-ui ls -la /app/core/data
```

---

## ⚠️ 注意事项

### 1. 平台兼容性检查

```bash
# 检查服务器架构
uname -m

# 输出说明:
# x86_64    → AMD64/x86 架构（使用 deploy-x86.sh）
# aarch64   → ARM64 架构（使用 deploy-arm.sh）
# arm64     → ARM64 架构（使用 deploy-arm.sh）
```

### 2. 环境变量安全

```bash
# ❌ 不要使用默认密码
ADMIN_PASSWORD=qq007qq008  # 生产环境必须修改！

# ✅ 使用强密码
ADMIN_PASSWORD="YourStrongPassword123!"

# ✅ 或使用环境变量
export ADMIN_PASSWORD="YourStrongPassword123!"
docker run ... -e ADMIN_PASSWORD=${ADMIN_PASSWORD}
```

### 3. 端口冲突

```bash
# 检查端口占用
lsof -i :3080

# 如果端口被占用，修改映射端口
docker run -p 3081:3000 ...  # 使用其他端口
```

### 4. 内存限制

```bash
# 如果服务器内存不足，限制容器内存
docker run --memory="2g" --memory-swap="2g" ...
```

### 5. 防火墙设置

```bash
# 开放端口（Ubuntu/Debian）
ufw allow 3080/tcp

# 或（CentOS/RHEL）
firewall-cmd --permanent --add-port=3080/tcp
firewall-cmd --reload
```

---

## 📁 文件清单

### 已创建文件

1. ✅ **HTML 部署指南**: `Docker 多平台部署指南.html`
   - 完整的 HTML 版本部署说明
   - 包含所有部署方法和注意事项
   - 美观的 UI 设计，易于阅读

2. ✅ **ARM 部署脚本**: `scripts/deploy-arm.sh`
   - 一键部署 ARM64 架构服务器
   - 自动检测架构和 Docker 环境
   - 包含错误处理和成功提示

3. ✅ **x86 部署脚本**: `scripts/deploy-x86.sh`
   - 一键部署 x86_64 架构服务器
   - 自动检测架构和 Docker 环境
   - 包含错误处理和成功提示

4. ✅ **构建脚本**: `scripts/docker-build-multiarch.sh`
   - 多平台 Docker 镜像构建和推送
   - 支持 AMD64 和 ARM64 同时构建
   - 自动创建和推送镜像清单

5. ✅ **Docker Compose 配置**: 
   - `docker-compose.yml` - 基础配置
   - `docker-compose.prod.yml` - 生产环境配置
   - `docker-compose.deploy.yml` - 部署配置（新建）

### 相关文档

- ✅ `README.md` - 项目说明
- ✅ `DOCKER-DEPLOYMENT.md` - Docker 部署指南
- ✅ `DATABASE_HANDLING_GUIDE.md` - 数据库处理指南
- ✅ `CHANGELOG.DEVELOPMENT.md` - 开发日志
- ✅ `RELEASE-NOTES.md` - 版本发布说明

---

## 📊 部署检查清单

### 构建前检查

- [ ] Docker 已安装并运行
- [ ] Docker Buildx 可用
- [ ] Docker Hub 已登录
- [ ] 项目代码已更新到最新版本
- [ ] Update.log 文件已更新
- [ ] 本地测试通过

### 部署前检查

- [ ] 服务器架构已确认（ARM64 或 x86_64）
- [ ] Docker 已安装并运行
- [ ] 端口 3080 未被占用
- [ ] 数据目录已创建
- [ ] 管理密码已修改（不使用默认密码）
- [ ] 防火墙已开放端口

### 部署后验证

- [ ] 容器正常运行
- [ ] 访问 `http://localhost:3080` 正常
- [ ] 使用管理密码登录成功
- [ ] Dashboard 页面显示正常
- [ ] 日志输出正常，无错误信息
- [ ] 数据卷挂载正确

### 备份验证

- [ ] 数据目录已备份
- [ ] 备份文件可正常解压
- [ ] 定期备份任务已配置
- [ ] 恢复流程已测试

---

## 🆘 获取帮助

### 文档资源

- **快速开始**: [README.md](README.md)
- **部署指南**: [DOCKER-DEPLOYMENT.md](DOCKER-DEPLOYMENT.md)
- **数据库处理**: [DATABASE_HANDLING_GUIDE.md](DATABASE_HANDLING_GUIDE.md)
- **开发日志**: [CHANGELOG.DEVELOPMENT.md](CHANGELOG.DEVELOPMENT.md)
- **版本说明**: [RELEASE-NOTES.md](RELEASE-NOTES.md)

### 技术支持

- **GitHub Issues**: https://github.com/smdk000/qq-farm-bot-ui/issues
- **Docker Hub**: https://hub.docker.com/r/smdk000/qq-farm-bot-ui
- **QQ 群**: 227916149

---

## 📈 下一步计划

### 短期计划（1-2 周）

- [ ] 添加自动化测试脚本
- [ ] 完善监控告警系统
- [ ] 优化 Docker 镜像大小
- [ ] 添加更多部署示例

### 中期计划（1-2 月）

- [ ] 支持 Kubernetes 部署
- [ ] 添加 Helm Chart
- [ ] 完善 CI/CD 流程
- [ ] 支持多区域镜像同步

### 长期计划（3-6 月）

- [ ] 云端同步功能
- [ ] 插件系统
- [ ] AI 智能推荐
- [ ] 多语言支持

---

**文档创建时间**: 2026-03-01  
**最后更新**: 2026-03-01  
**版本**: v3.3.3  
**维护者**: smdk000
