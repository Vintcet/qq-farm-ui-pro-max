# GitHub Secrets 配置指南

## 🔐 配置 Docker Hub 和 GitHub Container Registry 凭据

### 步骤 1: 创建 Docker Hub Token（已完成 ✅）

您已创建 Docker Hub Token（在 https://hub.docker.com/settings/security 创建）:
```
YOUR_DOCKERHUB_TOKEN
```

### 步骤 2: 设置 GitHub Workflow 权限（已完成 ✅）

您已设置:
- ✅ Read and write permissions
- ✅ Allow GitHub Actions to create and approve pull requests

### 步骤 3: 添加 GitHub Secrets

请访问：https://github.com/smdk000/qq-farm-ui-pro-max/settings/secrets/actions

添加以下 Secrets:

#### 1. DOCKERHUB_TOKEN

- **Name**: `DOCKERHUB_TOKEN`
- **Secret**: 在 Docker Hub 创建的 Personal Access Token
- **描述**: Docker Hub 访问令牌

#### 2. GHCR_TOKEN (可选)

GitHub Container Registry 使用 `GITHUB_TOKEN` 自动提供，无需手动配置。

---

## 📋 验证配置

### 方法 1: 手动测试登录

```bash
# 测试 Docker Hub 登录
echo "$DOCKERHUB_TOKEN" | docker login -u smdk000 --password-stdin

# 应该显示：Login Succeeded
```

### 方法 2: 触发工作流测试

1. 访问：https://github.com/smdk000/qq-farm-ui-pro-max/actions/workflows/docker-build-push.yml
2. 点击 **Run workflow**
3. 选择分支（main）
4. 输入版本号（v3.6.0）
5. 选择推送目标（Docker Hub 和/或 GHCR）
6. 点击 **Run workflow**

---

## 🚀 使用工作流

### 自动触发

工作流会在以下情况自动触发：

1. **推送标签时**
   ```bash
   git tag v3.6.0
   git push origin v3.6.0
   ```

2. **推送到 main 分支且修改了相关文件**
   - `core/**`
   - `web/**`
   - `core/Dockerfile`
   - `.github/workflows/docker-build-push.yml`

### 手动触发

1. 访问工作流页面
2. 点击 **Run workflow**
3. 填写参数：
   - **Version**: 版本号（如 v3.6.0）
   - **Push to Docker Hub**: true/false
   - **Push to GitHub Container Registry**: true/false
4. 点击运行

---

## 📊 工作流输出

工作流运行完成后会：

1. ✅ 构建多平台镜像（AMD64 + ARM64）
2. ✅ 推送到 Docker Hub（如果启用）
3. ✅ 推送到 GitHub Container Registry（如果启用）
4. ✅ 更新 Docker Hub 镜像描述
5. ✅ 生成部署摘要

### 查看构建结果

访问：https://github.com/smdk000/qq-farm-ui-pro-max/actions

---

## 🔄 下次代码更新后的自动更新

### 方法 1: 使用 Git 标签（推荐）

```bash
# 1. 提交代码更改
git add .
git commit -m "feat: 新功能描述"
git push origin main

# 2. 创建并推送标签
git tag v3.6.1
git push origin v3.6.1

# GitHub Actions 会自动构建并推送 Docker 镜像
```

### 方法 2: 手动触发工作流

1. 访问工作流页面
2. 点击 **Run workflow**
3. 输入新版本号
4. 运行工作流

### 方法 3: 使用自动更新脚本

```bash
# 运行自动更新脚本
./scripts/auto-update-docker.sh v3.6.1 "新功能描述"
```

---

## 📝 版本命名规范

遵循语义化版本控制（Semantic Versioning）：

- **主版本号**.次版本号.修订版本号
- 例如：`v3.6.0`

### 版本类型

- **主版本** (v4.0.0): 重大变更，不向后兼容
- **次版本** (v3.7.0): 新功能，向后兼容
- **修订版** (v3.6.1): Bug 修复，向后兼容

---

## ⚠️ 故障排查

### 问题 1: 工作流失败

检查：
1. Secrets 是否正确配置
2. Docker Hub Token 是否有效
3. Workflow permissions 是否正确设置

### 问题 2: 推送失败

检查：
1. Docker Hub 用户名是否正确
2. Token 权限是否包含 Write
3. 网络连接是否正常

### 问题 3: 构建失败

检查：
1. Dockerfile 路径是否正确
2. 项目结构是否完整
3. 构建资源是否充足

---

## 🔗 相关链接

- **GitHub Actions**: https://github.com/smdk000/qq-farm-ui-pro-max/actions
- **Docker Hub**: https://hub.docker.com/r/smdk000/qq-farm-bot-ui
- **GitHub Packages**: https://github.com/users/smdk000/packages/container/package/qq-farm-bot-ui
- **Docker Hub Token 设置**: https://hub.docker.com/settings/security

---

**最后更新**: 2026-03-01  
**维护者**: smdk000
