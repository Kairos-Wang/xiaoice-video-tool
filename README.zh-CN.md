# XiaoIce Video Tool

[English README](./README.md)

一个自托管的 XiaoIce 视频生成接入层，面向通用 MCP Agent 产品和 OpenClaw。你只需要跑一个服务，就能在 Agent 侧拿到一个工具：`xiaoice_video_produce`。

创建：

```json
{
  "action": "create",
  "prompt": "生成一个 10 秒的产品展示视频",
  "vhBizId": "demo-biz-id"
}
```

轮询查询：

```json
{
  "action": "get",
  "taskId": "task-123"
}
```

无论是 MCP server 还是 OpenClaw 插件，最终调用的都是同一个本地 HTTP 服务（`video-task-service`）。

## 仓库包含什么

- `video-task-service`：HTTP API、任务状态落盘、处理提供方回调
- `mcp-server`：MCP stdio server，对外暴露 `xiaoice_video_produce`
- `adapters/openclaw-plugin`：可单独分发的 OpenClaw 插件（id: `one-click-video`），带同一个工具和内置 skill

这个仓库是给你自己部署运行的，不提供托管 SaaS。

## 选择接入方式

两种接入方式都要求 `video-task-service` 先跑起来。

| 你要接入的宿主 | 选择 |
| --- | --- |
| 任意支持 MCP 的 Agent 产品 | `mcp-server`（`npm run mcp`） |
| OpenClaw，且想要可单独分发的插件 + skill | `adapters/openclaw-plugin` |

## 快速开始

**前置要求**

- Node.js `>=22`
- 可用的 XiaoIce 提供方凭据
- 一个对外可访问的回调地址（本地调试可用 ngrok）

### 1) 安装

```bash
npm install
cp .env.example .env
```

生成 3 个内部 token，填进 `.env`：

```bash
node -e "const c=require('crypto');const r=()=>c.randomBytes(24).toString('hex');console.log('VIDEO_SERVICE_INTERNAL_TOKEN='+r());console.log('VIDEO_SERVICE_ADMIN_TOKEN='+r());console.log('VIDEO_SERVICE_CALLBACK_TOKEN='+r());"
```

在 `.env` 里至少配置这些提供方字段：

- `VIDEO_PROVIDER_API_BASE_URL`
- `VIDEO_PROVIDER_API_KEY`
- `VIDEO_PROVIDER_VH_BIZ_ID`（也可以在每次创建任务时传 `vhBizId`）
- `VIDEO_PROVIDER_AUTH_HEADER`（部分环境需要 `subscription-key`）

完整列表见 `.env.example`。

### 2) 启动 `video-task-service`

两种模式二选一：

- 你有固定公网回调域名: `VIDEO_USE_NGROK=false`，设置 `VIDEO_CALLBACK_PUBLIC_BASE_URL=...`，然后 `npm run service`
- 本地调试用 ngrok: `VIDEO_USE_NGROK=true`，设置 `NGROK_AUTHTOKEN=...`，然后 `npm run dev:up`

健康检查：

```bash
curl -sS http://127.0.0.1:3105/health
```

## 直接使用 HTTP API

创建任务：

```bash
curl -sS -X POST "http://127.0.0.1:3105/v1/tasks" \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: <VIDEO_SERVICE_INTERNAL_TOKEN>" \
  -d '{
    "prompt": "生成一个 10 秒的产品展示视频",
    "vhBizId": "demo-biz-id"
  }'
```

查询任务：

```bash
curl -sS "http://127.0.0.1:3105/v1/tasks/<taskId>" \
  -H "X-Internal-Token: <VIDEO_SERVICE_INTERNAL_TOKEN>"
```

注意：

- 只使用 `vhBizId`，旧字段 `vhbizmode` 会被拒绝。
- `create` 必须有非空 `prompt`，`get` 必须有 `taskId`。

## 1) 通用 MCP Agent 产品怎么接

服务启动后，再启动 MCP stdio server：

```bash
XIAOICE_VIDEO_SERVICE_BASE_URL=http://127.0.0.1:3105 \
VIDEO_SERVICE_INTERNAL_TOKEN=<VIDEO_SERVICE_INTERNAL_TOKEN> \
npm run mcp
```

对外工具信息：

- 工具名：`xiaoice_video_produce`
- 动作：`create | get`

创建：

```json
{
  "action": "create",
  "prompt": "生成一个 15 秒春季新品发布视频",
  "vhBizId": "demo-biz-id"
}
```

查询：

```json
{
  "action": "get",
  "taskId": "task-123"
}
```

## 2) 使用本仓库内的 OpenClaw 插件

这里说的是 [`adapters/openclaw-plugin`](./adapters/openclaw-plugin)，不是 OpenClaw 平台本身的安装说明。

你会拿到：

- 插件 id：`one-click-video`
- 工具：`xiaoice_video_produce`
- 内置 skill：`skills/xiaoice-video/SKILL.md`

这个插件是 thin adapter，只负责把调用转发给 `video-task-service`。

### 2.1 把插件单独打包分发

```bash
cd adapters/openclaw-plugin
npm pack
```

会生成类似 `one-click-video-0.1.0.tgz` 的包，里面包含插件代码和内置 skill。

### 2.2 插件如何配置

OpenClaw 侧只需要配置这些：

```json
{
  "plugins": {
    "entries": {
      "one-click-video": {
        "enabled": true,
        "config": {
          "serviceBaseUrl": "http://127.0.0.1:3105",
          "internalToken": "<VIDEO_SERVICE_INTERNAL_TOKEN>",
          "requestTimeoutMs": 15000
        }
      }
    }
  }
}
```

把敏感信息和提供方配置放在服务端，而不是插件配置里：

- `VIDEO_PROVIDER_API_KEY`
- `VIDEO_PROVIDER_VH_BIZ_ID`
- `VIDEO_PROVIDER_MODEL_ID`

### 2.3 插件怎么用

安装并配置好插件后，调用同一个工具：

```json
{
  "action": "create",
  "prompt": "生成一个 10 秒的产品展示视频"
}
```

再轮询：

```json
{
  "action": "get",
  "taskId": "task-123"
}
```

常见终态是 `succeeded`、`failed`、`timeout`。成功时关注 `videoUrl`。

### 2.4 用户如何使用这个 Skill

插件随包自带：

- `adapters/openclaw-plugin/skills/xiaoice-video/SKILL.md`

它主要帮助 Agent 更稳定地：

- 判断何时调用 `xiaoice_video_produce`
- 优先用最小参数（通常只传 `prompt`）
- 记住查询状态必须拿到 `taskId`
- 配置报错时知道该检查 `serviceBaseUrl` / `internalToken`

一般不需要额外配置，因为 skill 会随插件一起加载。

如果你要定制 skill，建议保持可移植性：

- 不要写机器绝对路径。
- 不要把真实 API Key 或 token 写进 skill 文本里。

## 运行时更新提供方配置

不重启客户端即可更新提供方凭据和默认 `vhBizId`：

```bash
curl -sS -X PUT "http://127.0.0.1:3105/v1/admin/config" \
  -H "Content-Type: application/json" \
  -H "X-Admin-Token: <VIDEO_SERVICE_ADMIN_TOKEN>" \
  -d '{
    "apiKey": "<NEW_VIDEO_PROVIDER_API_KEY>",
    "vhBizId": "<NEW_VH_BIZ_ID>"
  }'
```

## 常见问题

- `prompt is required`: `action=create` 时 `prompt` 不能为空。
- `vhbizmode ... use vhBizId`: 把字段名改成 `vhBizId`。
- OpenClaw 插件 `config_error`: 检查 `serviceBaseUrl` 和 `internalToken`。
- 任务状态一直不更新: 优先确认公网回调地址在提供方网络里可访问。

## 更多文档

- `docs/04-deployment.md`
- `adapters/openclaw-plugin/README.md`
