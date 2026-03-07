# 🚀 QQ 农场助手 - v3.6.0 Docker 部署完整指南

> 历史说明（2026-03-07）：本文对应旧版本部署说明，当前有效工作流以 `docs/guides/REPO_ROOT_WORKFLOW_GUIDE.md` 为准；日志挂载请使用 `./logs:/app/logs`。

**发布日期**: 2026-03-01  
**版本**: v3.6.0  
**状态**: ✅ 生产就绪

---

## 📋 目录

1. [快速开始](#快速开始)
2. [Docker 多平台构建](#docker-多平台构建)
3. [部署到服务器](#部署到服务器)
4. [数据库保护](#数据库保护)
5. [版本更新内容](#版本更新内容)
6. [故障排查](#故障排查)

---

## 🎯 快速开始

### 方法 1: 一键部署脚本（最简单）

**ARM64 服务器**（树莓派/鲲鹏/飞腾）:
```bash
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-bot-ui/main/scripts/deploy-arm.sh
chmod +x deploy-arm.sh
./deploy-arm.sh
```

**x86_64 服务器**（Intel/AMD）:
```bash
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-bot-ui/main/scripts/deploy-x86.sh
chmod +x deploy-x86.sh
./deploy-x86.sh
```

### 方法 2: Docker Compose

```yaml
version: '3.8'

services:
  qq-farm-bot-ui:
    image: smdk000/qq-farm-bot-ui:latest
    container_name: qq-farm-bot-ui
    restart: unless-stopped
    ports:
      - "3080:3000"
    environment:
      - ADMIN_PASSWORD=your_secure_password
      - TZ=Asia/Shanghai
    volumes:
      - ./data:/app/core/data
      - ./logs:/app/logs
      - ./backup:/app/core/backup
```

启动：
```bash
docker-compose up -d
```

### 方法 3: 手动部署

```bash
docker run -d \
  --name qq-farm-bot-ui \
  --restart unless-stopped \
  -p 3080:3000 \
  -v ./data:/app/core/data \
  -v ./logs:/app/logs \
  -v ./backup:/app/core/backup \
  -e ADMIN_PASSWORD=your_password \
  -e TZ=Asia/Shanghai \
  smdk000/qq-farm-bot-ui:latest
```

访问：`http://localhost:3080`  
默认密码：`admin`（请立即修改）

---

## 🏗️ Docker 多平台构建

### 构建并推送到 Docker Hub 和 GitHub

#### 1. 环境准备

```bash
# 检查 Docker 和 Buildx
docker --version
docker buildx version

# 登录 Docker Hub
docker login

# 登录 GitHub Container Registry
echo $GH_PAT | docker login ghcr.io -u smdk000 --password-stdin
```

#### 2. 构建多平台镜像

```bash
# 使用脚本构建（推荐）
chmod +x scripts/docker-build-multiarch.sh
./scripts/docker-build-multiarch.sh 3.6.0
```

#### 3. 手动构建（可选）

```bash
# 构建并推送到 Docker Hub
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t smdk000/qq-farm-bot-ui:3.6.0 \
  -t smdk000/qq-farm-bot-ui:latest \
  -f core/Dockerfile . \
  --push

# 构建并推送到 GitHub Container Registry
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/smdk000/qq-farm-bot-ui:3.6.0 \
  -t ghcr.io/smdk000/qq-farm-bot-ui:latest \
  -f core/Dockerfile . \
  --push
```

#### 4. 验证构建

```bash
# 查看镜像信息
docker manifest inspect smdk000/qq-farm-bot-ui:3.6.0

# Docker Hub 查看
# https://hub.docker.com/r/smdk000/qq-farm-bot-ui/tags

# GitHub Packages 查看
# https://github.com/users/smdk000/packages/container/package/qq-farm-bot-ui
```

---

## 📱 部署到服务器

### ARM64 服务器部署

适用设备：树莓派 4B、鲲鹏、飞腾等

```bash
# 1. 下载部署脚本
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-bot-ui/main/scripts/deploy-arm.sh

# 2. 赋予执行权限
chmod +x deploy-arm.sh

# 3. 执行部署
./deploy-arm.sh

# 或自定义密码和端口
ADMIN_PASSWORD=YourStrongPassword123! PORT=3080 ./deploy-arm.sh
```

### x86_64 服务器部署

适用设备：Intel、AMD 处理器服务器

```bash
# 1. 下载部署脚本
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-bot-ui/main/scripts/deploy-x86.sh

# 2. 赋予执行权限
chmod +x deploy-x86.sh

# 3. 执行部署
./deploy-x86.sh

# 或自定义密码和端口
ADMIN_PASSWORD=YourStrongPassword123! PORT=3080 ./deploy-x86.sh
```

### 验证部署

```bash
# 查看容器状态
docker ps

# 查看实时日志
docker logs -f qq-farm-bot-ui

# 访问 Web 界面
# http://<服务器 IP>:3080
```

---

## 🛡️ 数据库保护

### 数据卷挂载说明

| 宿主机路径 | 容器内路径 | 说明 |
|-----------|-----------|------|
| `./data` | `/app/core/data` | **核心数据库**（账号配置、用户数据） |
| `./logs` | `/app/logs` | 日志文件（运行日志、操作日志） |
| `./backup` | `/app/core/backup` | 备份文件目录 |

### 备份策略

#### 1. 定期备份（推荐每天执行）

```bash
# 添加到 crontab，每天凌晨 2 点备份
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

- ❌ **不要删除** `./data` 目录，否则所有数据将丢失
- ❌ **不要手动修改** 数据库文件，可能导致数据损坏
- ✅ **定期备份** 数据到安全位置（云存储、外部硬盘等）
- ✅ **升级前备份**，升级失败可快速回滚

---

## 📝 版本更新内容

### v3.6.0 (2026-03-01) - 最新版本

#### 前端更新

**✨ 端云同步时间机器**
- 彻底解决多设备切换配置覆盖问题
- 采用时间戳仲裁防伪墙
- 确保读取最新设定并无感热更新

**⚡ 首屏秒开级切片分离**
- 首屏不再生硬包裹数百个隐形大体积组件
- 利用 Vue 原生 `defineAsyncComponent` 懒加载
- 恶劣网络下白屏率急剧下降

#### 后端更新

**🔐 安全密钥隔离**
- 清除硬编码的第三方 API 密钥
- 实现纯净分离安全脱敏
- 使用 `.env` 环境变量接管

**🛡️ 网络防永久挂死拦截**
- 微信登录轮询 10-15 秒超时保护
- 防止网络堵塞导致进程悬挂
- 杜绝服务器内存侧漏崩溃

### v3.5.2 - 扫码重构与体验优化

- 多平台扫码引擎：支持微信小程序扫码登录
- 深度主题联防：修复深色主题加载条颜色
- UI 级安全审计：统一对比度规范

### v3.4.0 - UI 架构大重构

- 独立换肤抽屉引擎
- 偷菜管理图形化拆分
- 自动化修改强校验拦截
- 体验卡核心支持

### v3.3.4 - 全局沉浸与平滑操作

- 主题色彩一统：帮助中心、账号管理页完美融入毛玻璃架构
- 细节渲染修复：去除硬编码阴影与底色

### v3.3.3 - 丝滑防抖与安全限流

- **核心痛点根除**：GPU 切层硬渲染锁，彻底扑灭 Chrome 闪烁
- **低配福音**：性能模式锁死全部元素运动轨迹
- **便捷操作指导**：悬浮名片提示 + 推荐标签
- **反制防卡拥堵队列**：紧急 VVIP 通道
- **绝对防御限速器**：WebSocket 3 QPS 令牌桶

[查看完整更新日志](https://github.com/smdk000/qq-farm-bot-ui/blob/main/CHANGELOG.DEVELOPMENT.md)

---

## 🔧 故障排查

### 容器无法启动

```bash
# 查看详细日志
docker logs qq-farm-bot-ui

# 检查端口占用
lsof -i :3080

# 检查容器配置
docker inspect qq-farm-bot-ui
```

### 无法访问网页

1. 检查防火墙设置
   ```bash
   # Ubuntu/Debian
   ufw allow 3080/tcp
   
   # CentOS/RHEL
   firewall-cmd --permanent --add-port=3080/tcp
   firewall-cmd --reload
   ```

2. 确认端口映射正确
   ```bash
   docker inspect qq-farm-bot-ui | grep -A 10 Mounts
   ```

3. 查看容器日志
   ```bash
   docker logs --tail 200 qq-farm-bot-ui
   ```

### 数据库丢失

检查数据卷挂载：
```bash
docker inspect qq-farm-bot-ui | grep -A 10 Mounts
```

从备份恢复：
```bash
tar -xzf farm-bot-backup-20260301.tar.gz -C ./data
```

### 内存不足

```bash
# 限制容器内存
docker update --memory="2g" --memory-swap="2g" qq-farm-bot-ui
```

### 架构不匹配

```bash
# 检查服务器架构
uname -m

# x86_64 → 使用 deploy-x86.sh
# aarch64/arm64 → 使用 deploy-arm.sh
```

---

## 📚 完整文档

- **GitHub 仓库**: https://github.com/smdk000/qq-farm-bot-ui
- **Docker Hub**: https://hub.docker.com/r/smdk000/qq-farm-bot-ui
- **GitHub Packages**: https://github.com/users/smdk000/packages/container/package/qq-farm-bot-ui
- **部署指南**: https://github.com/smdk000/qq-farm-bot-ui/blob/main/docs/DEPLOYMENT_GUIDE.md
- **配置模板**: https://github.com/smdk000/qq-farm-bot-ui/blob/main/docs/CONFIG_TEMPLATES.md
- **故障排查**: https://github.com/smdk000/qq-farm-bot-ui/blob/main/docs/TROUBLESHOOTING.md
- **帮助中心**: https://github.com/smdk000/qq-farm-bot-ui/blob/main/docs/HELP_CENTER_MAINTENANCE_GUIDE.md
- **更新日志**: https://github.com/smdk000/qq-farm-bot-ui/blob/main/CHANGELOG.DEVELOPMENT.md

---

## 🆘 获取帮助

### 文档资源

- [README.md](https://github.com/smdk000/qq-farm-bot-ui) - 项目说明
- [DOCKER_HUB_README.md](https://github.com/smdk000/qq-farm-bot-ui/blob/main/DOCKER_HUB_README.md) - Docker Hub 说明
- [GHCR_README.md](https://github.com/smdk000/qq-farm-bot-ui/blob/main/GHCR_README.md) - GitHub Container Registry 说明

### 技术支持

- **GitHub Issues**: https://github.com/smdk000/qq-farm-bot-ui/issues
- **QQ 群**: 227916149
- **Docker Hub**: https://hub.docker.com/r/smdk000/qq-farm-bot-ui
- **GitHub Packages**: https://github.com/users/smdk000/packages/container/package/qq-farm-bot-ui

---

## ⚠️ 免责声明

本项目仅供学习与研究用途。使用本工具可能违反游戏服务条款，由此产生的一切后果由使用者自行承担。

---

## 📄 许可证

ISC License

---

**最后更新**: 2026-03-01  
**维护者**: smdkk000  
**版本**: v3.6.0
