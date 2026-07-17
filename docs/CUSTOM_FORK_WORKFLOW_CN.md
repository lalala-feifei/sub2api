> **实操发布手册（推荐先看）**：[CUSTOM_UPGRADE_DEPLOY_CN.md](./CUSTOM_UPGRADE_DEPLOY_CN.md)

# Sub2API 定制 Fork 工作流

> 适用仓库：[`lalala-feifei/sub2api`](https://github.com/lalala-feifei/sub2api)  
> 官方上游：[`Wei-Shaw/sub2api`](https://github.com/Wei-Shaw/sub2api)  
> 本地目录：`/Users/sujing/sub2api`  
> 线上服务器：`opencloudos` / `43.153.79.86`  
> 文档目标：说明如何在保留官方同步能力的同时，做长期定制开发与安全发布

---

## 1. 当前仓库布局

### 1.1 GitHub 仓库

| 仓库 | 作用 |
|---|---|
| [`lalala-feifei/sub2api`](https://github.com/lalala-feifei/sub2api) | 正式开发仓库（官方 fork） |
| [`lalala-feifei/sub2api-legacy-import`](https://github.com/lalala-feifei/sub2api-legacy-import) | 旧私有导入仓备份，仅保留历史，不再作为主开发源 |

### 1.2 远程

本地仓库已配置：

```bash
origin    https://github.com/lalala-feifei/sub2api.git
upstream  https://github.com/Wei-Shaw/sub2api.git
```

### 1.3 分支职责

| 分支 | 职责 | 当前基准 |
|---|---|---|
| `main` | 跟踪官方最新，尽量保持可同步 | 官方最新 `v0.1.159` |
| `prod` | 线上实际运行基线 + 已验证定制 | 线上当前运行版 `0.1.130`（`0cfabaa82`） |
| `feat/*` | 单个定制功能开发 | 从 `prod` 或 `main` 拉出 |

原则：

1. `main` 尽量贴近官方，不长期堆叠重定制
2. 定制功能在 `feat/*` 开发
3. 只有验证通过的改动才进入 `prod`
4. 服务器只部署 `prod` 构建出的镜像

---

## 2. 为什么用这套模型

你的场景同时有三件事：

1. 线上已有稳定运行环境
2. 你要做自己的定制功能
3. 官方会持续发版

因此不能：

- 直接在服务器容器里改代码
- 把所有定制都直接堆在官方 `main`
- 用“服务器导入仓”长期维护，因为缺少官方 Git 历史，后续同步成本极高

正确模型：

```text
官方 upstream
    │ 定期同步
    ▼
你的 fork main          ← 官方最新代码
    │
    ├── feat/xxx        ← 定制开发
    │
    ▼
你的 fork prod          ← 线上可发布版本
    │
    ▼
Docker 镜像 sub2api:<tag>
    │
    ▼
服务器 /www/wwwroot/linshi 运行
```

---

## 3. 日常开发流程

### 3.1 新开定制功能

优先从 `prod` 拉分支，保证改的是“线上真实基线”：

```bash
cd /Users/sujing/sub2api
git checkout prod
git pull origin prod
git checkout -b feat/your-feature

# 开发、本地验证
git add .
git commit -m "feat: your feature"
git push -u origin feat/your-feature
```

如果功能明确要基于官方最新能力开发，也可以从 `main` 拉：

```bash
git checkout main
git pull origin main
git checkout -b feat/your-feature
```

### 3.2 合并到生产分支

```bash
git checkout prod
git pull origin prod
git merge feat/your-feature
# 解决冲突、跑测试
git push origin prod
```

### 3.3 功能开发约定

1. **一个功能一个分支**，不要把多个无关定制揉在一起
2. 定制代码尽量独立：
   - 新文件 / 新模块优先
   - 少直接改官方核心文件
3. 关键定制处加标记，便于以后同步官方时定位：

```go
// CUSTOM: lalala private billing rule
```

4. 提交信息写清楚，例如：
   - `feat: add private admin dashboard card`
   - `fix: adjust quota calculation for internal users`

---

## 4. 官方更新同步流程

当官方发布新版本，例如 `v0.1.170`：

### 4.1 先同步到 main

```bash
cd /Users/sujing/sub2api
git checkout main
git fetch upstream --tags
git merge upstream/main
# 或固定到某个 release：
# git merge v0.1.170
git push origin main
```

### 4.2 再把官方更新合入 prod

```bash
git checkout prod
git pull origin prod
git merge main
# 解决与你定制功能的冲突
git push origin prod
```

### 4.3 冲突处理原则

| 情况 | 处理方式 |
|---|---|
| 官方改了，你没改 | 收官方 |
| 你改了，官方没动 | 保留你的 |
| 双方都改了同一处 | 人工合并，并补测试 |

如果某个定制短期来不及适配新官方版本：

1. 先不要强行合入 `prod`
2. 可暂时保留旧 `prod`
3. 或把该定制拆回独立分支，延后升级

---

## 5. 发布到线上服务器

### 5.1 线上当前事实

- 部署目录：`/www/wwwroot/linshi`
- Compose 项目：`linshi`
- 应用容器：`sub2api`
- 依赖容器：`sub2api-postgres`、`sub2api-redis`
- 当前镜像：`weishaw/sub2api:latest`（实际标签版本 `0.1.130`）
- 数据目录：`/www/wwwroot/linshi/data`
- 配置文件：`/www/wwwroot/linshi/.env`

### 5.2 推荐发布方式

1. 在本地或服务器基于 `prod` 构建自定义镜像
2. 给镜像打不可变 tag
3. 只替换 `sub2api` 应用容器
4. 保留 Postgres / Redis / `data` / `.env`

示例 tag：

```text
sub2api:0.1.130-prod.1
sub2api:0.1.159-prod.1
sub2api:0.1.159-feat-billing.3
```

### 5.3 服务器构建与发布示例

```bash
ssh opencloudos

# 1. 准备代码目录（建议与运行目录分离）
mkdir -p /www/wwwroot/sub2api-src
cd /www/wwwroot/sub2api-src

# 首次：
# git clone https://github.com/lalala-feifei/sub2api.git .

git fetch origin
git checkout prod
git pull origin prod

# 2. 构建镜像
IMAGE_TAG="sub2api:0.1.130-prod.1"
docker build -t "$IMAGE_TAG" .

# 3. 修改运行目录 compose 中的 image
cd /www/wwwroot/linshi
# 将 sub2api.image 改为 sub2api:0.1.130-prod.1

# 4. 发布
docker compose up -d sub2api

# 5. 检查
docker ps --filter name=sub2api
docker logs --tail 100 sub2api
curl -fsS http://127.0.0.1:8080/health
```

### 5.4 回滚

```bash
cd /www/wwwroot/linshi
# 把 image 改回上一个可用 tag
docker compose up -d sub2api
```

原则：

- 永远不要只依赖 `latest`
- 每次发布都保留上一个可用镜像 tag
- 数据库与配置回滚要单独评估，不能默认和代码一起回滚

---

## 6. 本地常用命令

### 6.1 查看远程与分支

```bash
cd /Users/sujing/sub2api
git remote -v
git branch -vv
git log --oneline --decorate -10
```

### 6.2 同步官方

```bash
git checkout main
git fetch upstream --tags
git merge upstream/main
git push origin main
```

### 6.3 从生产基线开发

```bash
git checkout prod
git pull origin prod
git checkout -b feat/xxx
```

### 6.4 对比官方与生产差异

```bash
git fetch upstream
git log --oneline prod..upstream/main | head
git diff --stat prod..upstream/main | tail
```

---

## 7. 推荐目录与职责边界

| 位置 | 职责 |
|---|---|
| 本地 `/Users/sujing/sub2api` | 开发、调试、提交 |
| GitHub fork `lalala-feifei/sub2api` | 代码真相源 |
| 服务器 `/www/wwwroot/sub2api-src` | 可选：构建源码目录 |
| 服务器 `/www/wwwroot/linshi` | 生产运行目录，保留数据与配置 |
| Docker 镜像 `sub2api:<tag>` | 可回滚的发布产物 |

不要混淆：

1. **源码仓库** 负责改功能
2. **运行目录** 负责跑服务和存数据
3. **镜像 tag** 负责发布与回滚

---

## 8. 与旧仓库的关系

旧仓库：

- [`lalala-feifei/sub2api-legacy-import`](https://github.com/lalala-feifei/sub2api-legacy-import)

它来自服务器导入，没有官方完整历史，因此：

1. 不再作为主开发仓
2. 仅作历史备份
3. 如需迁移其中的私有文档或脚本，再逐项 cherry-pick / 手工拷贝到新 fork

当前正式工作仓始终是：

- [`lalala-feifei/sub2api`](https://github.com/lalala-feifei/sub2api)

---

## 9. 最小操作清单

### 场景 A：我要改一个功能

```bash
git checkout prod
git pull
git checkout -b feat/xxx
# 开发
git commit
git push -u origin feat/xxx
# 验证后 merge 到 prod
# 构建镜像并发布
```

### 场景 B：官方更新了

```bash
git checkout main
git fetch upstream --tags
git merge upstream/main
git push origin main

git checkout prod
git merge main
# 解决冲突、回归测试
# 构建新镜像并发布
```

### 场景 C：线上出问题

```bash
# 回滚到上一个镜像 tag
cd /www/wwwroot/linshi
# 修改 compose image
docker compose up -d sub2api
```

---

## 10. 当前初始化结果

本次已完成：

1. 旧私有仓重命名为 `sub2api-legacy-import`
2. 重新 fork 官方仓库为 `lalala-feifei/sub2api`
3. 本地克隆到 `/Users/sujing/sub2api`
4. 配置 `origin` + `upstream`
5. 创建并推送 `prod` 分支，对齐线上 `0.1.130`
6. `main` 保持官方最新 `v0.1.159`

下一步建议：

1. 开始第一个定制功能时，从 `prod` 拉 `feat/*`
2. 在服务器准备独立构建目录 `/www/wwwroot/sub2api-src`
3. 将线上 `docker-compose.yml` 的 `image` 从官方 `latest` 改为你的不可变 tag
