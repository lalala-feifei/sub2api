# Sub2API 定制开发、官方升级与服务器部署手册

> 适用仓库：[`lalala-feifei/sub2api`](https://github.com/lalala-feifei/sub2api)  
> 官方上游：[`Wei-Shaw/sub2api`](https://github.com/Wei-Shaw/sub2api)  
> 本地目录：`/Users/sujing/sub2api`  
> 线上服务器：`opencloudos`（`43.153.79.86`）  
> 线上运行目录：`/www/wwwroot/linshi`  
> 文档目标：以后每次「加功能 / 跟官方升级 / 上线」都按同一套流程走，避免定制被覆盖、数据被清掉

---

## 1. 一句话原则

| 原则 | 说明 |
|------|------|
| **代码在 Git，运行靠镜像** | 功能改在 fork 的 `prod` 分支，构建自己的 Docker 镜像再部署 |
| **数据不进 Git** | 服务器上的 `postgres_data` / `redis_data` / `.env` 永不删除、不进仓库 |
| **禁止后台一键官方更新** | 管理台「立即更新」会换成官方镜像，**本地定制功能会丢** |
| **只滚动应用容器** | 升级时只重建 `sub2api`，不动 `postgres` / `redis` |

```text
官方 upstream
    │  同步版本
    ▼
fork main          ← 尽量贴近官方
    │
    ├── feat/xxx   ← 单个功能开发
    │
    ▼
fork prod          ← 线上可发布基线（官方版本 + 已验证定制）
    │
    ▼
docker build → sub2api:batch-user-actions（或其它自建 tag）
    │
    ▼
服务器 /www/wwwroot/linshi
  docker compose up -d --no-deps --force-recreate sub2api
```

---

## 2. 环境与仓库布局

### 2.1 Git 远程

本地仓库应配置：

```bash
cd /Users/sujing/sub2api
git remote -v
# origin    https://github.com/lalala-feifei/sub2api.git
# upstream  https://github.com/Wei-Shaw/sub2api.git
```

若缺少 `upstream`：

```bash
git remote add upstream https://github.com/Wei-Shaw/sub2api.git
```

### 2.2 分支职责

| 分支 | 职责 | 说明 |
|------|------|------|
| `main` | 跟踪官方 | 同步官方 release，尽量少堆长期定制 |
| `feat/*` | 功能开发 | 一个功能一个分支，从 `prod` 或官方 tag 拉出 |
| `prod` | **线上基线** | 只有验证通过的定制才合入；服务器只部署从此分支构建的镜像 |

### 2.3 服务器与 Compose

| 项 | 值 |
|----|-----|
| SSH | `ssh opencloudos` |
| 运行目录 | `/www/wwwroot/linshi` |
| Compose 文件 | `/www/wwwroot/linshi/docker-compose.yml` |
| 应用服务名 | `sub2api` |
| 应用镜像 tag | `sub2api:batch-user-actions`（自建，不是 `weishaw/sub2api:*`） |
| 容器名 | `sub2api` / `sub2api-postgres` / `sub2api-redis` |
| 对外端口 | `127.0.0.1:8080->8080`（前面通常还有 Nginx 反代到 `api.iyiwo.cn`） |
| 数据目录 | `./postgres_data`、`./redis_data`、`.env` |

Compose 关键片段（部署时以服务器文件为准）：

```yaml
services:
  sub2api:
    image: sub2api:batch-user-actions
    container_name: sub2api
    ports:
      - "127.0.0.1:8080:8080"
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:18-alpine
    volumes:
      - ./postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:8-alpine
    volumes:
      - ./redis_data:/data
```

### 2.4 当前已落地的定制（示例）

- **批量禁用 / 启用 / 删除用户**
  - 后端：`POST /api/v1/admin/users/batch-status`、`POST /api/v1/admin/users/batch-delete`
  - 前端：用户管理页多选后的批量操作按钮
  - 标记：`// CUSTOM: lalala batch user actions`（便于以后同步官方时检索）

---

## 3. 日常：新增功能

### 3.1 从线上基线开分支

```bash
cd /Users/sujing/sub2api
git fetch origin
git checkout prod
git pull origin prod
git checkout -b feat/your-feature
```

### 3.2 开发约定

1. **一个功能一个分支**，不要把无关改动揉在一起  
2. 改动尽量可检索：在关键处加注释  

```go
// CUSTOM: lalala batch user actions
```

3. 提交信息清晰，例如：

```text
feat: add batch disable/delete users
fix: complete batch user actions on v0.1.159
```

4. 本地能跑单测就跑（示例）：

```bash
cd backend
go test ./internal/service/ -tags unit -run 'TestAdminService_Batch' -count=1
```

### 3.3 合入 prod 并推送

```bash
git checkout prod
git pull origin prod
git merge feat/your-feature
# 解决冲突 → 再测
git push origin prod
```

功能分支也可单独推送备份：

```bash
git push -u origin feat/your-feature
```

---

## 4. 跟官方升级（保留定制）

目标：拿到官方新版本（例如 `v0.1.170`），**同时保留** `prod` 上已有定制。

### 4.1 同步官方到 main

```bash
cd /Users/sujing/sub2api
git fetch upstream --tags
git checkout main
git merge v0.1.170          # 或：git merge upstream/main
git push origin main
```

### 4.2 把定制迁到新官方版本上

推荐「从官方 tag 重新接定制」，冲突更清晰：

```bash
# 1) 以官方 tag 为底
git checkout -b feat/batch-user-actions-v0.1.170 v0.1.170

# 2) 拣选已有定制提交（示例：批量用户功能相关 commit）
git cherry-pick <batch-feature-commit>

# 3) 解决冲突（常见坑见第 6 节）
# 4) 跑测试
cd backend && go test ./internal/service/ -tags unit -run 'TestAdminService_Batch' -count=1

# 5) 把 prod 切到新基线
git checkout -B prod feat/batch-user-actions-v0.1.170
git push --force-with-lease origin prod   # 若历史相对旧 prod 改写较大，需 force-with-lease
```

若定制只有少量提交，`cherry-pick` 通常比整树 merge 更干净。

### 4.3 绝不要用的升级方式

| 错误做法 | 后果 |
|----------|------|
| 后台左上角「立即更新」 | 换成官方镜像，**定制功能消失** |
| `docker pull weishaw/sub2api:x.y.z` 直接改 compose | 同上 |
| `docker compose down -v` | **数据库清空** |
| 在运行中容器里直接改文件 | 下次重建全丢，无法迭代 |

---

## 5. 部署到服务器（标准发布命令）

以下为**完整、可重复**的一套命令。在本机推送 `prod` 后执行。

### 5.1 本机：确认并推送

```bash
cd /Users/sujing/sub2api
git checkout prod
git log -1 --oneline
git push origin prod
```

### 5.2 服务器：构建自建镜像 + 只滚动应用

```bash
ssh opencloudos

set -euo pipefail

# --- 构建目录（与运行目录分离，避免污染线上 git 工作区）---
BUILD_DIR=/tmp/sub2api-build-prod
rm -rf "$BUILD_DIR"
git clone --branch prod --depth 1 https://github.com/lalala-feifei/sub2api.git "$BUILD_DIR"
cd "$BUILD_DIR"
echo "HEAD=$(git rev-parse --short HEAD)"
git log -1 --oneline

# --- 构建：必须打成 compose 里使用的 tag ---
docker build -t sub2api:batch-user-actions \
  --build-arg GOPROXY=https://goproxy.cn,direct \
  --build-arg GOSUMDB=sum.golang.google.cn \
  -f Dockerfile .

docker images sub2api:batch-user-actions

# --- 只重建应用容器（postgres/redis 不动）---
cd /www/wwwroot/linshi
docker compose -f docker-compose.yml up -d --no-deps --force-recreate sub2api

# --- 健康检查 ---
docker ps --filter name=sub2api
for i in $(seq 1 15); do
  st=$(docker inspect -f '{{.State.Health.Status}}' sub2api 2>/dev/null || echo none)
  echo "health=$st"
  [ "$st" = "healthy" ] && break
  sleep 2
done
docker logs --tail 40 sub2api

# --- 清理构建目录（可选）---
rm -rf "$BUILD_DIR"
```

### 5.3 发布后验收清单

1. `docker ps`：`sub2api` 为 `healthy`，postgres/redis 仍在  
2. 打开 `https://api.iyiwo.cn` 管理台（**强刷** Ctrl/Cmd+Shift+R）  
3. **用户管理**：多选用户后可见「批量禁用 / 批量启用 / 批量删除」  
4. 可选 API 探测（需管理员登录 token）：

```bash
# 路由存在：空 body 应返回 400 参数校验，而不是 404
curl -sS -X POST https://api.iyiwo.cn/api/v1/admin/users/batch-delete \
  -H "Authorization: Bearer <ADMIN_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{}'
```

5. 随便发一条业务请求（如 Grok 文本）确认上游调度正常  

### 5.4 回滚（紧急）

若新镜像有问题，且旧镜像层仍在本机 Docker 中：

```bash
cd /www/wwwroot/linshi
# 查看历史镜像
docker images sub2api

# 临时把 compose 中 image 改回旧 digest/tag，或：
# docker tag <旧镜像ID> sub2api:batch-user-actions
docker compose -f docker-compose.yml up -d --no-deps --force-recreate sub2api
```

更稳妥的长期做法：发布时打**不可变 tag**，例如：

```bash
docker build -t sub2api:0.1.159-batch-users-$(git rev-parse --short HEAD) ...
docker tag sub2api:0.1.159-batch-users-abc8690 sub2api:batch-user-actions
```

compose 继续指向 `batch-user-actions` 作「当前指针」，历史 tag 用于回滚。

---

## 6. 跟官方升级时常见冲突

| 现象 | 处理思路 |
|------|----------|
| `frontend/src/i18n/locales/zh.ts` 被删 | 官方已模块化 i18n，文案改加到 `frontend/src/i18n/locales/zh/admin/overview.ts` 等拆分文件 |
| `admin_service.go` 里找不到 `DeleteUser` 实现 | 用户相关实现已拆到 `admin_user.go`，接口在 `admin_service.go`，实现补到 `admin_user.go` |
| 单测 stub 缺方法 | `UserRepository` 新增方法时，同步补 `*multiUserRepoStub` / `*userRepoStub` |
| 浅克隆构建版本号显示不是 tag | `git clone --depth 1` 无完整 tag 时，`resolve-version.sh` 可能显示 `VERSION` 文件号；**以 commit 与功能为准**。需要准确版本号时可加深 clone 或 `git fetch --tags` |

检索定制标记：

```bash
git grep -n 'CUSTOM: lalala'
```

---

## 7. 禁止事项速查

```text
❌ 管理后台「立即更新」
❌ docker pull weishaw/sub2api 后直接当生产镜像
❌ docker compose down -v
❌ 删除 /www/wwwroot/linshi/postgres_data 或 redis_data
❌ 在 sub2api 容器内手改前端/二进制当正式发布
❌ 把 .env、数据库 dump 明文提交到 GitHub
```

```text
✅ 改代码 → 合 prod → 推 origin
✅ 服务器从 prod 构建 sub2api:batch-user-actions
✅ compose up -d --no-deps --force-recreate sub2api
✅ 强刷后台验收定制功能
```

---

## 8. 推荐的一键脚本（可选）

可将下面内容存为服务器 `/root/bin/deploy-sub2api-prod.sh`，以后一条命令发布：

```bash
#!/usr/bin/env bash
set -euo pipefail

IMAGE_TAG="${IMAGE_TAG:-sub2api:batch-user-actions}"
BUILD_DIR="${BUILD_DIR:-/tmp/sub2api-build-prod}"
RUN_DIR="${RUN_DIR:-/www/wwwroot/linshi}"
REPO_URL="${REPO_URL:-https://github.com/lalala-feifei/sub2api.git}"
BRANCH="${BRANCH:-prod}"

rm -rf "$BUILD_DIR"
git clone --branch "$BRANCH" --depth 1 "$REPO_URL" "$BUILD_DIR"
cd "$BUILD_DIR"
echo "Deploying $(git rev-parse --short HEAD) $(git log -1 --pretty=%s)"

docker build -t "$IMAGE_TAG" \
  --build-arg GOPROXY=https://goproxy.cn,direct \
  --build-arg GOSUMDB=sum.golang.google.cn \
  -f Dockerfile .

cd "$RUN_DIR"
docker compose -f docker-compose.yml up -d --no-deps --force-recreate sub2api

for i in $(seq 1 20); do
  st=$(docker inspect -f '{{.State.Health.Status}}' sub2api 2>/dev/null || echo none)
  echo "health=$st"
  [ "$st" = "healthy" ] && break
  sleep 2
done

docker ps --filter name=sub2api
docker logs --tail 30 sub2api
rm -rf "$BUILD_DIR"
echo "OK"
```

使用：

```bash
# 本机
cd /Users/sujing/sub2api && git push origin prod

# 服务器
ssh opencloudos 'bash /root/bin/deploy-sub2api-prod.sh'
```

---

## 9. 故障对照（部署相关）

| 现象 | 可能原因 | 处理 |
|------|----------|------|
| 后台没有批量按钮 | 浏览器缓存 / 跑的是旧镜像 | 强刷；`docker images` / `docker inspect sub2api` 确认镜像 ID 是新构建 |
| API 404 batch-delete | 未部署含定制的镜像 | 确认 compose 的 image 与刚 build 的 tag 一致后 recreate |
| `No available accounts` | 上游 OAuth 失效等，与本次构建无关 | 账号管理里刷新 Grok OAuth |
| 升级后数据没了 | 误用了 `-v` 或删了数据目录 | 只能靠备份恢复；发布流程严禁 `down -v` |
| 版本号显示偏旧 | 浅克隆无 tag | 看 `git log` / 功能是否在；必要时加深 clone 或改 `VERSION` |

---

## 10. 相关文档

- 更偏分支治理的说明（若仓库中存在）：`docs/CUSTOM_FORK_WORKFLOW_CN.md`
- 初始化状态备忘：`docs/SETUP_STATUS_CN.md`
- 官方部署参考：`deploy/DOCKER.md`、`deploy/README.md`

---

## 11. 变更记录

| 日期 | 说明 |
|------|------|
| 2026-07-17 | 初版：基于 `prod=abc86902a`（官方 v0.1.159 + 批量用户功能）与服务器 `/www/wwwroot/linshi` 实装流程整理 |

---

**记住三步闭环：**

```text
改功能 → 合 prod 并 push → 服务器 build 自建镜像 → 只 recreate sub2api
跟官方 → tag 为底 cherry-pick 定制 → 更新 prod → 同上发布
千万别 → 后台一键官方更新 / down -v
```
