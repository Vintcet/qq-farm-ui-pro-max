# Docker 部署问题诊断与修复报告

**检查时间**: 2026-03-01  
**检查范围**: README.md, 部署脚本，docker-compose 文件

---

## 🔍 发现的问题

### 问题 1: 版本号不一致 ❌

**现象**:
- `scripts/deploy-arm.sh` 使用版本 `3.3.3`
- `scripts/deploy-x86.sh` 使用版本 `3.3.3`
- `docker-compose.prod.yml` 使用版本 `3.3.0`
- `README.md` 使用 `latest`（✅ 正确）

**影响**: 用户部署时可能使用旧版本镜像

**修复方案**: 
- 部署脚本应使用 `latest` 标签或可配置版本
- 统一版本号为最新的 v3.6.0

---

### 问题 2: Docker Hub 镜像名称不完整 ❌

**现象**:
- `docker-compose.prod.yml` 使用 `qq-farm-bot-ui:3.3.0`
- 正确的应该是 `smdk000/qq-farm-bot-ui:latest`

**影响**: Docker 会从 Docker Hub 拉取错误的镜像或失败

**修复方案**: 
- 使用完整的镜像名称 `smdk000/qq-farm-bot-ui`

---

### 问题 3: docker-compose.yml 配置不完整 ❌

**现象**:
- 缺少 `./logs` 和 `./backup` 卷挂载
- 缺少 `LOG_LEVEL` 环境变量

**影响**: 日志文件可能写入容器内，数据丢失风险

**修复方案**: 
- 添加完整的卷挂载和环境变量

---

### 问题 4: GitHub 仓库名称不一致 ⚠️

**现象**:
- 当前仓库：`qq-farm-bot-ui-main`
- GitHub 仓库：`qq-farm-ui-pro-max`
- README 中的链接指向错误的仓库

**影响**: 用户下载的脚本来自错误的仓库

**修复方案**: 
- 统一使用正确的 GitHub 仓库 URL

---

## ✅ 修复内容

### 1. 更新部署脚本

**文件**: `scripts/deploy-arm.sh`, `scripts/deploy-x86.sh`

**修改**:
- 版本号从 `3.3.3` 改为 `latest`
- 添加版本参数支持
- 修复 GitHub 仓库 URL

### 2. 更新 docker-compose.prod.yml

**修改**:
- 镜像名称改为 `smdk000/qq-farm-bot-ui:latest`
- 添加完整的卷挂载
- 添加 LOG_LEVEL 环境变量

### 3. 更新 README.md

**修改**:
- 修正 GitHub 仓库 URL
- 添加版本说明
- 添加故障排查章节

---

## 📝 修复后的部署方法

### 方法 1: 一键部署脚本（推荐）

**ARM64 服务器**:
```bash
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-ui-pro-max/main/scripts/deploy-arm.sh
chmod +x deploy-arm.sh
./deploy-arm.sh
```

**x86_64 服务器**:
```bash
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-ui-pro-max/main/scripts/deploy-x86.sh
chmod +x deploy-x86.sh
./deploy-x86.sh
```

### 方法 2: Docker Compose

**使用生产环境配置**:
```bash
# 下载配置文件
curl -O https://raw.githubusercontent.com/smdk000/qq-farm-ui-pro-max/main/docker-compose.prod.yml

# 启动服务
docker-compose -f docker-compose.prod.yml up -d

# 查看状态
docker-compose -f docker-compose.prod.yml ps

# 查看日志
docker-compose -f docker-compose.prod.yml logs -f
```

### 方法 3: Docker 命令

```bash
docker run -d \
  --name qq-farm-bot-ui \
  --restart unless-stopped \
  -p 3080:3000 \
  -v ./data:/app/core/data \
  -v ./logs:/app/core/logs \
  -v ./backup:/app/core/backup \
  -e ADMIN_PASSWORD=your_password \
  -e TZ=Asia/Shanghai \
  -e LOG_LEVEL=info \
  smdk000/qq-farm-bot-ui:latest
```

---

## 🧪 测试验证

### 部署成功检查清单

- [ ] Docker 容器正常运行
- [ ] 可以访问 `http://localhost:3080`
- [ ] 使用密码可以登录
- [ ] 日志输出正常
- [ ] 数据卷正确挂载

### 验证命令

```bash
# 检查容器状态
docker ps

# 查看实时日志
docker logs -f qq-farm-bot-ui

# 检查数据卷挂载
docker inspect qq-farm-bot-ui | grep -A 10 Mounts

# 测试访问
curl http://localhost:3080/api/ping
```

---

## 🔄 版本升级

### 从旧版本升级

```bash
# 1. 备份数据
tar -czf farm-bot-backup-$(date +%Y%m%d).tar.gz ./data

# 2. 停止旧容器
docker stop qq-farm-bot-ui
docker rm qq-farm-bot-ui

# 3. 拉取新镜像
docker pull smdk000/qq-farm-bot-ui:latest

# 4. 启动新容器
./scripts/deploy-arm.sh  # 或 deploy-x86.sh
```

---

## ⚠️ 常见错误

### 错误 1: 镜像拉取失败

**错误信息**:
```
Error response from daemon: pull access denied
```

**原因**: 
- Docker Hub 镜像名称错误
- 未登录 Docker

**解决方案**:
```bash
# 检查镜像名称
docker pull smdk000/qq-farm-bot-ui:latest

# 如果需要登录
docker login
```

### 错误 2: 端口被占用

**错误信息**:
```
Error starting userland proxy: listen tcp 0.0.0.0:3080: bind: address already in use
```

**解决方案**:
```bash
# 检查端口占用
lsof -i :3080

# 使用不同端口
export PORT=3081
./scripts/deploy-arm.sh
```

### 错误 3: 权限错误

**错误信息**:
```
permission denied while trying to connect to the Docker daemon socket
```

**解决方案**:
```bash
# 使用 sudo
sudo ./scripts/deploy-arm.sh

# 或将用户添加到 docker 组
sudo usermod -aG docker $USER
newgrp docker
```

---

## 📚 相关文档

- [完整部署指南](DEPLOYMENT_GUIDE_v3.6.0.md)
- [Docker Hub 镜像说明](DOCKER_HUB_README.md)
- [GitHub Secrets 配置](GITHUB_SECRETS_SETUP.md)
- [故障排查指南](docs/TROUBLESHOOTING.md)

---

## ✅ 修复总结

### 已修复的问题

- ✅ 统一版本号为 latest（支持自定义）
- ✅ 修正 Docker Hub 镜像名称
- ✅ 完善 docker-compose 配置
- ✅ 统一 GitHub 仓库 URL
- ✅ 添加完整的故障排查指南

### 测试状态

- ✅ ARM 部署脚本语法正确
- ✅ x86 部署脚本语法正确
- ✅ docker-compose 配置验证通过
- ✅ README 链接已更新

---

**维护者**: smdk000  
**最后更新**: 2026-03-01  
**版本**: v3.6.0
