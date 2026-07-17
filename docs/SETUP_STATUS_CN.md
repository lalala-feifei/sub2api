# Sub2API 工作流初始化状态

更新时间：2026-07-17

## 已完成

- 旧私有仓已备份为：[`lalala-feifei/sub2api-legacy-import`](https://github.com/lalala-feifei/sub2api-legacy-import)
- 官方仓库已重新 fork 为：[`lalala-feifei/sub2api`](https://github.com/lalala-feifei/sub2api)
- 本地仓库目录：`/Users/sujing/sub2api`
- 远程：
  - `origin` -> `https://github.com/lalala-feifei/sub2api.git`
  - `upstream` -> `https://github.com/Wei-Shaw/sub2api.git`
- 分支：
  - `main`：官方最新 `v0.1.159`（`c2c19a7cb`）
  - `prod`：线上基线 `0.1.130`（`0cfabaa82`）
- 工作流文档：
  - [`docs/CUSTOM_FORK_WORKFLOW_CN.md`](./CUSTOM_FORK_WORKFLOW_CN.md)

## 线上对照

- SSH Host：`opencloudos`
- 运行目录：`/www/wwwroot/linshi`
- 当前容器镜像：`weishaw/sub2api:latest`
- 当前实际版本：`0.1.130`
- 数据与配置需继续保留在服务器运行目录，不进入 Git 主流程

## 下一步

1. 从 `prod` 创建第一个 `feat/*` 分支开始定制
2. 在服务器建立独立构建目录 `/www/wwwroot/sub2api-src`
3. 发布时使用不可变镜像 tag，而不是 `latest`
