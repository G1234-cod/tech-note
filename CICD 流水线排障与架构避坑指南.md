# CI/CD 流水线排障与架构避坑指南

> 本文档用于记录双引擎架构下，跨库同步、自动化构建与环境部署中遇到的核心架构级异常。为大管家智能体提供诊断上下文。

---

## 异常记录 001：GitHub Actions 检出私有仓库报 404 错误

### 初始认知
我在配置 `.github/workflows/deploy.yml` 试图通过 `actions/checkout@v4` 拉取私有项目 `garbage-classification-model` 时，CI/CD 容器在运行阶段抛出了致命错误：
`Error: Not Found - https://docs.github.com/rest/repos/repos#get-a-repository`
最初看到这个报错，我的第一反应是仓库名称拼写错误或者路径配置有误，因为日志明确提示“找不到该仓库”，并且给出了 404 状态码。

### 问题解决
查阅 GitHub REST API 的底层鉴权规范后，我发现这是一个典型的安全机制陷阱。为了防止外部人员通过穷举法探测私有仓库的命名，当外部请求在没有携带有效凭证（Token），或者凭证权限不足时试图访问私有仓库，GitHub 的底层逻辑绝不会返回 `403 Forbidden`（权限拒绝），而是统一伪装成 `404 Not Found`。
由于我的流水线运行在 `my-blog` 仓库，默认的 `GITHUB_TOKEN` 无权跨域访问我名下的其他私有资产。
解决方案：我在 GitHub 开发者设置中生成了仅带 `repo` 权限的 PAT (Personal Access Token)，将其作为 `ACTION_PAT` 注入到博客主引擎的 Secrets 中，并在拉取步骤中通过 `token: &#36;&#123;&#123; secrets.ACTION_PAT &#125;&#125;` 强行解锁了私有仓库的检出权限。

### 个人反思
不能完全轻信终端日志表面的 HTTP 状态码。在多模块、带权限隔离的微服务架构下，鉴权拦截经常被系统底层包装成资源丢失或路径不存在的报错。后续遇到类似无厘头的 `Not Found`，必须首要排查跨域通信机制和访问权限配置。