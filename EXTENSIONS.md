# CPA-Manager-Plus 自定义扩展说明

> 本分支 `feat/custom-extension` 用于在 CPA-Manager-Plus 上游基础上的自定义功能扩展。
> 上游原仓库：https://github.com/seakee/CPA-Manager-Plus
> 本仓库 fork：https://github.com/hizzt/CPA-Manager-Plus

## 1. 仓库结构

| Remote | URL | 用途 |
|--------|-----|------|
| `origin`  | `git@github.com:hizzt/CPA-Manager-Plus.git` | 自己的 fork，扩展代码推送到这里 |
| `upstream`| `git@github.com:seakee/CPA-Manager-Plus.git` | 上游原仓库，只拉取不推送 |

## 2. 分支策略

| 分支 | 说明 |
|------|------|
| `main` | 保持与上游 `upstream/main` 同步，冻结在上游最新 release tag |
| `feat/custom-extension` | 自定义扩展开发分支（本分支） |
| `feat/*` | 其他功能扩展分支（按需创建） |

规则：
- **永不推送到 `upstream`**（会收到上游 403）
- 扩展代码只提交在 `feat/*` 分支
- `main` 不做扩展提交，只做上游同步

## 3. 上游 Release 同步流程

上游发布新 release（如 v1.13.0）后，按以下步骤合并到本分支：

```bash
# 1. 拉取上游最新 tag
git fetch upstream --tags

# 2. 在扩展分支上合并新 release
git checkout feat/custom-extension
git merge v1.13.0

# 3. 解决冲突（若有）
git status                      # 查看冲突文件
# 手动解决后
git add <file>
git commit

# 4. 推送
git push origin feat/custom-extension
```

同步 `main`（可选，保持 fork 基线最新）：

```bash
git checkout main
git fetch upstream --tags
git merge upstream/main         # 或 git reset --hard <new-release-tag>
git push origin main
```

## 4. 冲突处理指南

扩展代码集中在独立目录时冲突面最小：

| 位置 | 冲突概率 | 处理 |
|------|---------|------|
| `internal/your-feature/` | 低 | 正常合并 |
| `web/src/features/your-feature/` | 低 | 正常合并 |
| 路由注册处（`http/`、`router/`） | 中 | 保留双方新增路由 |
| `go.mod` / `package.json` | 中 | 保留双方依赖 |

## 5. 扩展代码规范

- 后端：`apps/manager-server/internal/<模块名>/` 下新增独立包，不修改既有包逻辑
- 前端：`apps/web/src/features/<模块名>/` 下新增独立功能模块，在 `router/` 中加路由
- 路由注册：在 `internal/http/` 的现有路由表中追加，避免改动上游已有路由行

## 6. 本地构建与测试

```bash
# 后端（Go 1.24+）
cd apps/manager-server
go build ./cmd/cpa-manager-plus/
go test ./...

# 前端（Node 18+）
cd apps/web
npm install
npm run dev          # 开发调试
npm run build        # 生产构建
```

## 7. 部署（Unraid / Docker）

当前 NAS 上 `cpa-manager-plus` 容器运行方式参考：
`/mnt/user/appdata/cpa-manager-plus/data`（数据卷，含 `usage.sqlite` 和 `data.key`）

自定义版本镜像构建与部署：

```bash
# Mac 本地交叉构建 amd64 镜像
cd apps/manager-server
docker buildx build --platform linux/amd64 \
  -t cpa-manager-plus-custom:latest \
  -f Dockerfile.manager-server --load .

# 传输到 NAS
docker save cpa-manager-plus-custom:latest | gzip > /tmp/cpa-custom.tar.gz
scp /tmp/cpa-custom.tar.gz unraid:/tmp/

# NAS 上加载并启动
ssh unraid
docker load < /tmp/cpa-custom.tar.gz
# 替换原容器（参考原容器环境变量）
```

> 注意：镜像内不含运行数据，数据都在宿主机卷中，容器替换不影响 `usage.sqlite`。

## 8. 注意事项

- 上游安全补丁（如 v1.11.11 修复的权限提升漏洞）必须**及时**同步，不要长期滞后
- 扩展涉及 SQLite schema 变更时，写迁移脚本，确保与上游数据兼容
- 本仓库 fork 公开可见，敏感配置（API Key / OAuth 凭证）不要提交到仓库
- 构建镜像时建议加 `--build-arg VERSION=<版本号>` 便于线上识别