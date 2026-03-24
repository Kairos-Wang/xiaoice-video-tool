# XiaoIce Video Tool 项目收尾说明

这份文档用于帮助团队同事快速理解并接手 `xiaoice-video-tool`。这里不铺垫概念，重点把项目的模块划分、运行方式、关键文件入口，以及未提交到 GitHub 的配置项说明清楚。

## 1. 项目一句话说明

这个项目是一个本地自托管的 XiaoIce 视频任务工具链。当前实现主要包含三层：

1. `video-task-service`
   负责创建任务、查询任务、接收供应商回调，并把任务状态落到本地 SQLite。
2. `mcp-server`
   负责把外部 Agent / MCP 客户端的工具调用，转换成对本地服务的 HTTP 调用。
3. `OpenClaw plugin`
   负责在 OpenClaw 里注册原生工具 `xiaoice_video_produce`，并附带 skill `xiaoice-video`（中文触发词包含“一键成片”）。

核心目标是把“发起视频生成任务”和“查询任务状态”收敛成一个稳定的本地服务边界，便于 MCP 和 OpenClaw 共同复用。

## 2. 技术栈

### 2.1 运行时与语言

- Node.js `>=22`
  入口定义见 [package.json](/home/yirongbest/xiaoice-video-tool/package.json#L1)
- JavaScript（CommonJS）
- 内置 Node HTTP 服务
  服务主实现见 [server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L1)
- SQLite
  使用 `node:sqlite` 的 `DatabaseSync`，见 [server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L5)

### 2.2 协议与集成

- HTTP API
  本地任务服务暴露 `/health`、`/v1/tasks`、`/v1/callbacks/provider`、`/v1/admin/config`
  说明见 [docs/01-architecture.md](/home/yirongbest/xiaoice-video-tool/docs/01-architecture.md#L23)
- MCP stdio
  MCP CLI 入口见 [cli.js](/home/yirongbest/xiaoice-video-tool/src/mcp/cli.js#L1)
- OpenClaw 原生插件
  插件入口见 [index.js](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/index.js#L1)
  插件 manifest 见 [openclaw.plugin.json](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/openclaw.plugin.json#L1)

### 2.3 开发与测试

- Node 原生测试：`node --test`
- 关键测试文件：
  - HTTP 集成测试 [video-task-service.integration.test.js](/home/yirongbest/xiaoice-video-tool/tests/video-task-service.integration.test.js)
  - MCP 工具测试 [mcp-tool.test.js](/home/yirongbest/xiaoice-video-tool/tests/mcp-tool.test.js)
  - MCP 协议测试 [mcp-server.protocol.test.js](/home/yirongbest/xiaoice-video-tool/tests/mcp-server.protocol.test.js)
  - OpenClaw 插件测试 [openclaw-plugin.test.js](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/__tests__/openclaw-plugin.test.js)

## 3. 整体实现方式

建议先看数据流，再回到文件。

### 3.0 对外字段契约（最新）

本版本对外契约已做字段收敛与规范化处理，create 入参以官方字段名为主：

- `action`: `create | get`（必填）
- `create` 必填：`topic`、`vhBizId`
- `create` 可选：`title`、`content`、`materialList`、`ttsConf`、`aigcWatermark`
- `create` 可选追踪字段：`sessionId`、`traceId`
- `get` 必填：`taskId`

说明：

- `callbackUrl` 不作为对外入参，仍由服务端根据运行时配置生成并注入 provider 请求。
- 历史字段与历史结构在新版本中不再作为对外契约维护。

### 3.1 创建视频任务

1. OpenClaw 插件或 MCP server 接收到工具调用 `xiaoice_video_produce`
2. 根据 `action=create` 组装 HTTP 请求
3. 调用本地 `video-task-service` 的 `POST /v1/tasks`
4. 服务写入 SQLite 初始任务记录，先返回 `submitted`
5. 服务后台再向 XiaoIce provider 提交真正的视频生成请求
6. 后续通过供应商回调把任务推进到 `processing / succeeded / failed / timeout`

这条链路建议直接读这些文件：

- 插件调用边界 [adapters/openclaw-plugin/index.js](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/index.js)
- MCP 调用边界 [src/mcp/tool.js](/home/yirongbest/xiaoice-video-tool/src/mcp/tool.js)
- 共享客户端 [src/shared/video-service-client.js](/home/yirongbest/xiaoice-video-tool/src/shared/video-service-client.js)
- 服务主逻辑 [src/service/server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js)

### 3.2 查询任务状态

1. Agent 调用 `xiaoice_video_produce`，参数 `action=get`
2. 插件或 MCP server 调用 `GET /v1/tasks/:taskId`
3. 服务返回标准化任务对象
4. 调用方把状态返回给 Agent

这部分的设计说明在 [docs/03-mcp-integration.md](/home/yirongbest/xiaoice-video-tool/docs/03-mcp-integration.md#L41)。

### 3.3 OpenClaw skill 的作用

skill 不负责网络请求，也不保存配置。它主要用于告诉 OpenClaw 应该如何“使用”这个工具：

- 什么时候应该触发这个工具
- 工具有哪些参数
- 如何理解 `create` 和 `get`
- 返回 `submitted` / `processing` / `succeeded` / `failed` / `timeout` 时应该怎么回复

skill 文件见 [SKILL.md](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/skills/xiaoice-video/SKILL.md#L1)。

当前 skill 名称是 `xiaoice-video`，已加入中文触发词“一键成片”。

## 4. 关键目录和文件路径

### 4.1 服务端

- 服务 CLI 入口 [src/service/cli.js](/home/yirongbest/xiaoice-video-tool/src/service/cli.js#L1)
- 服务主实现 [src/service/server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L1)
- 服务导出 [src/service/index.js](/home/yirongbest/xiaoice-video-tool/src/service/index.js)

### 4.2 MCP

- MCP CLI 入口 [src/mcp/cli.js](/home/yirongbest/xiaoice-video-tool/src/mcp/cli.js#L1)
- MCP server [src/mcp/server.js](/home/yirongbest/xiaoice-video-tool/src/mcp/server.js)
- MCP tool 映射 [src/mcp/tool.js](/home/yirongbest/xiaoice-video-tool/src/mcp/tool.js)

### 4.3 共享客户端

- 仓库内共享客户端 [src/shared/video-service-client.js](/home/yirongbest/xiaoice-video-tool/src/shared/video-service-client.js#L1)
- OpenClaw 插件打包内客户端 [adapters/openclaw-plugin/lib/video-service-client.js](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/lib/video-service-client.js#L1)

说明：

OpenClaw 插件内保留了一份 vendored 客户端。它不是重复造轮子，而是为了保证插件被 `openclaw plugins install` 之后仍然能独立加载，不依赖仓库根目录结构。

### 4.4 OpenClaw 插件

- 插件入口 [adapters/openclaw-plugin/index.js](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/index.js#L1)
- 插件 manifest [adapters/openclaw-plugin/openclaw.plugin.json](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/openclaw.plugin.json#L1)
- 插件包配置 [adapters/openclaw-plugin/package.json](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/package.json#L1)
- 插件说明 [adapters/openclaw-plugin/README.md](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/README.md#L1)
- 插件 skill [adapters/openclaw-plugin/skills/xiaoice-video/SKILL.md](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/skills/xiaoice-video/SKILL.md#L1)

### 4.5 文档与运行脚本

- 项目总览 [README.md](/home/yirongbest/xiaoice-video-tool/README.md#L1)
- 架构设计 [docs/01-architecture.md](/home/yirongbest/xiaoice-video-tool/docs/01-architecture.md#L1)
- MCP 集成 [docs/03-mcp-integration.md](/home/yirongbest/xiaoice-video-tool/docs/03-mcp-integration.md#L1)
- 部署文档 [docs/04-deployment.md](/home/yirongbest/xiaoice-video-tool/docs/04-deployment.md#L1)
- OpenClaw 插件计划 [docs/07-openclaw-thin-plugin-plan.md](/home/yirongbest/xiaoice-video-tool/docs/07-openclaw-thin-plugin-plan.md#L1)
- ngrok 本地开发 [docs/09-ngrok-local-dev.md](/home/yirongbest/xiaoice-video-tool/docs/09-ngrok-local-dev.md#L1)

脚本入口：

- [scripts/dev-service.js](/home/yirongbest/xiaoice-video-tool/scripts/dev-service.js)
- [scripts/dev-ngrok.js](/home/yirongbest/xiaoice-video-tool/scripts/dev-ngrok.js)
- [scripts/dev-ngrok-status.js](/home/yirongbest/xiaoice-video-tool/scripts/dev-ngrok-status.js)
- [scripts/dev-callback-sync.js](/home/yirongbest/xiaoice-video-tool/scripts/dev-callback-sync.js)
- [scripts/dev-up.js](/home/yirongbest/xiaoice-video-tool/scripts/dev-up.js)
- [scripts/dev-doctor.js](/home/yirongbest/xiaoice-video-tool/scripts/dev-doctor.js)

## 5. 启动方式

### 5.1 最基础本地模式

本地一般按下面这套最小步骤跑起来：

1. 安装依赖

```bash
npm install
```

2. 复制环境变量模板

```bash
cp .env.example .env
```

模板见 [/.env.example](/home/yirongbest/xiaoice-video-tool/.env.example#L1)。

3. 启动服务

```bash
npm run service
```

入口见 [package.json](/home/yirongbest/xiaoice-video-tool/package.json#L6) 和 [src/service/cli.js](/home/yirongbest/xiaoice-video-tool/src/service/cli.js#L1)。

4. 如需 MCP

```bash
npm run mcp
```

MCP CLI 会从环境变量读取：

- `XIAOICE_VIDEO_SERVICE_BASE_URL`
- `VIDEO_SERVICE_INTERNAL_TOKEN`

见 [src/mcp/cli.js](/home/yirongbest/xiaoice-video-tool/src/mcp/cli.js#L16)。

### 5.2 ngrok 辅助模式

常见使用场景：

- provider 需要回调本地服务
- 本机 `127.0.0.1:3105` 不能直接被外部网络访问

入口命令：

```bash
npm run dev:up
```

相关文档见 [docs/09-ngrok-local-dev.md](/home/yirongbest/xiaoice-video-tool/docs/09-ngrok-local-dev.md#L1)。

说明：

`3105` 默认是本地端口，不是公网端口。只有你显式用 ngrok 或其它隧道暴露它时，外部系统才能访问。

## 6. OpenClaw 侧当前情况

### 6.1 已安装内容

- 插件 id：`one-click-video`
- 工具名：`xiaoice_video_produce`
- skill 名：`xiaoice-video`
- 中文触发词：`一键成片`

### 6.2 OpenClaw 配置位置

当前使用中的 OpenClaw 配置文件在：

- [openclaw.json](/home/yirongbest/claw-xiaoice/openclaw.json#L175)

当前插件安装路径在：

- `/home/yirongbest/.openclaw/extensions/one-click-video`

插件运行时配置应写在：

- `plugins.entries.one-click-video.config`

不是写在 skill 文件里，也不是工具入参里。

### 6.3 配置归属（最新）

`one-click-video` 插件配置只承载插件运行所需的最小信息：

- `serviceBaseUrl`
- `internalToken`
- `requestTimeoutMs`

XiaoIce provider 凭据与运行时 provider 配置归属 `video-task-service`，不应放进 `plugins.entries.one-click-video.config`，也不应写进 skill。

## 7. 敏感配置与未上传 GitHub 的值

下面这些字段名在仓库里是公开的，但实际值不应该上传到 GitHub。以下按含义整理，便于接手同学核对。

### 7.1 `serviceBaseUrl`

含义：

- OpenClaw 插件或 MCP 客户端调用本地 `video-task-service` 的地址

默认开发示例：

- `http://127.0.0.1:3105`

说明：

- `127.0.0.1:3105` 是本地回环地址，只能本机访问
- 这不是“自动暴露到公网的端口”
- 如果需要给 provider 回调，通常是另外通过 `VIDEO_CALLBACK_PUBLIC_BASE_URL` 或 ngrok 暴露

读取位置：

- OpenClaw 插件读取配置见 [index.js](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/index.js#L116)
- 共享客户端拼接请求地址见 [video-service-client.js](/home/yirongbest/xiaoice-video-tool/src/shared/video-service-client.js#L102)

### 7.2 `internalToken`

含义：

- 插件调用视频任务服务时带的内部鉴权密钥

行为：

- 插件或客户端请求时会发 `X-Internal-Token` 请求头，见 [video-service-client.js](/home/yirongbest/xiaoice-video-tool/src/shared/video-service-client.js#L111)
- 服务端会校验这个头，不对就直接 `401 Unauthorized`，见 [server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L699) 和 [server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L700)

说明：

- 字段名是公开的
- 具体 token 值不能上传 GitHub
- `.env.example` 里的 `dev-internal-token-change-me` 只是占位示例，不应该用于真实环境，见 [/.env.example](/home/yirongbest/xiaoice-video-tool/.env.example#L3)
- 服务端还会显式拒绝弱默认 token，见 [server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L151)

### 7.3 `VIDEO_SERVICE_INTERNAL_TOKEN`

含义：

- `video-task-service` 的内部调用口令
- MCP server 和 OpenClaw plugin 都需要拿这个值去调用服务

用途：

- 保护 `POST /v1/tasks`
- 保护 `GET /v1/tasks/:taskId`

定义入口：

- 环境变量模板 [/.env.example](/home/yirongbest/xiaoice-video-tool/.env.example#L3)
- 服务端校验见 [server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L559)

说明：

- 字段名在 GitHub 上
- 实际值不能上 GitHub

### 7.4 `VIDEO_SERVICE_ADMIN_TOKEN`

含义：

- 管理接口口令
- 用于更新运行时 provider 配置

主要接口：

- `PUT /v1/admin/config`

定义入口：

- 环境变量模板 [/.env.example](/home/yirongbest/xiaoice-video-tool/.env.example#L4)
- 服务端加载见 [server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L563)

说明：

- 字段名公开
- 实际值不能上 GitHub

### 7.5 `VIDEO_SERVICE_CALLBACK_TOKEN`

含义：

- provider 回调服务时使用的口令

主要接口：

- `POST /v1/callbacks/provider`

支持方式：

- `X-Callback-Token` 请求头
- 或 `?token=` 查询参数

定义入口：

- 环境变量模板 [/.env.example](/home/yirongbest/xiaoice-video-tool/.env.example#L5)
- 服务端加载见 [server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L567)
- 架构说明见 [docs/01-architecture.md](/home/yirongbest/xiaoice-video-tool/docs/01-architecture.md#L29)

说明：

- 字段名公开
- 实际值不能上 GitHub

### 7.6 `VIDEO_CALLBACK_PUBLIC_BASE_URL`

含义：

- 返回给 provider 的公网回调根地址

和 `serviceBaseUrl` 的区别：

- `serviceBaseUrl` 是本地插件访问本地服务用的
- `VIDEO_CALLBACK_PUBLIC_BASE_URL` 是外部 provider 回调你时能访问到的地址

定义入口：

- 模板 [/.env.example](/home/yirongbest/xiaoice-video-tool/.env.example#L6)
- 文档说明 [docs/04-deployment.md](/home/yirongbest/xiaoice-video-tool/docs/04-deployment.md#L16)

说明：

- 字段名公开
- 真实公网域名或 tunnel 地址通常不建议写死进 GitHub

### 7.7 XiaoIce provider 相关字段

以下字段名在仓库里公开，但实际值不应提交：

- `VIDEO_PROVIDER_API_BASE_URL`
- `VIDEO_PROVIDER_API_KEY`
- `VIDEO_PROVIDER_VH_BIZ_ID`
- `VIDEO_PROVIDER_MODEL_ID`
- `VIDEO_PROVIDER_AUTH_HEADER`
- `VIDEO_PROVIDER_AUTH_SCHEME`

模板见 [/.env.example](/home/yirongbest/xiaoice-video-tool/.env.example#L15)。

特别说明：

- `VIDEO_PROVIDER_API_KEY` 是真正的供应商密钥，绝对不能上 GitHub
- `VIDEO_PROVIDER_VH_BIZ_ID` 是服务侧运行配置字段（运维管理用），但 create 请求仍要求显式传入 `vhBizId`
- `VIDEO_PROVIDER_AUTH_HEADER` 取决于环境，有的环境是 `X-API-Key`，有的环境是 `subscription-key`

## 8. 推荐同事上手顺序

建议接手同学按下面顺序看代码：

1. 先看 [README.md](/home/yirongbest/xiaoice-video-tool/README.md#L1)
2. 再看 [docs/01-architecture.md](/home/yirongbest/xiaoice-video-tool/docs/01-architecture.md#L1)
3. 然后看 [src/service/server.js](/home/yirongbest/xiaoice-video-tool/src/service/server.js#L551)
4. 再看 [src/shared/video-service-client.js](/home/yirongbest/xiaoice-video-tool/src/shared/video-service-client.js#L31)
5. 再看 [src/mcp/tool.js](/home/yirongbest/xiaoice-video-tool/src/mcp/tool.js)
6. 最后看 [adapters/openclaw-plugin/index.js](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/index.js#L1) 和 [SKILL.md](/home/yirongbest/xiaoice-video-tool/adapters/openclaw-plugin/skills/xiaoice-video/SKILL.md#L1)

这样能最快理解：

- 服务边界是什么
- 调用链是怎么走的
- OpenClaw 插件和 skill 各自负责什么

## 9. 常见误区

这几条是交接/沟通中最常见的误区，提前写在这里便于避坑。

### 9.1 skill 和插件配置不是一回事

skill 负责“何时用工具、怎么组织参数、怎么回复用户”。

插件配置负责“这个工具实际连到哪台服务、用什么 token 调用”。

所以：

- `action / topic / vhBizId / taskId`（以及 create 可选字段）属于工具参数
- `serviceBaseUrl / internalToken / requestTimeoutMs` 属于插件运行配置

### 9.2 `serviceBaseUrl` 不等于公网回调地址

`serviceBaseUrl` 通常是本地地址，例如：

- `http://127.0.0.1:3105`

它是 OpenClaw 或 MCP 调用本地服务时用的。

真正给 provider 用的公网回调地址通常来自：

- `VIDEO_CALLBACK_PUBLIC_BASE_URL`
- 或 ngrok 同步后的地址

### 9.3 三个 `VIDEO_SERVICE_*_TOKEN` 不要混用

- `VIDEO_SERVICE_INTERNAL_TOKEN`
  给 MCP / OpenClaw 调内部任务接口
- `VIDEO_SERVICE_ADMIN_TOKEN`
  给管理员修改运行时配置
- `VIDEO_SERVICE_CALLBACK_TOKEN`
  给 provider 回调接口

用途不同，最好分别生成独立强随机值。

## 10. 当前交付状态

当前已经完成并验证的部分：

- OpenClaw 原生插件 `one-click-video` 已可安装
- 插件会注册工具 `xiaoice_video_produce`
- 插件已打包并携带 skill `xiaoice-video`
- skill 已支持中文触发词“一键成片”
- 插件已在当前 OpenClaw 环境中完成安装验证

仍需团队在环境侧自行提供的内容：

- `.env` 中的真实 token
- provider API key 和业务 id
- OpenClaw 里 `plugins.entries.one-click-video.config` 对应的真实值

## 11. 最短可执行总结

若只记三件事，建议记这三件：

1. 先把 `video-task-service` 跑起来，再谈 MCP 或 OpenClaw
2. `serviceBaseUrl` 是插件访问本地服务的地址，不是公网 callback 地址
3. `VIDEO_SERVICE_INTERNAL_TOKEN / VIDEO_SERVICE_ADMIN_TOKEN / VIDEO_SERVICE_CALLBACK_TOKEN` 的字段名在仓库里，但真实值都不能提交到 GitHub
