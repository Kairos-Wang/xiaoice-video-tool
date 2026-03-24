# XiaoIce Video Tool

[中文文档](./README.zh-CN.md)

Self-hosted XiaoIce video generation bridge for MCP-based agent products and OpenClaw. You run one service, and your agent gets one tool: `xiaoice_video_produce`.

Create:

```json
{
  "action": "create",
  "prompt": "Generate a 10-second product demo video",
  "vhBizId": "demo-biz-id"
}
```

Poll:

```json
{
  "action": "get",
  "taskId": "task-123"
}
```

Both the MCP server and the OpenClaw plugin call the same local HTTP service (`video-task-service`).

## What's in this repo

- `video-task-service`: HTTP API, task state on disk, provider callback handling
- `mcp-server`: MCP stdio server exposing `xiaoice_video_produce`
- `adapters/openclaw-plugin`: standalone OpenClaw plugin (id: `one-click-video`) shipping the same tool and a bundled skill

This repo is meant to be deployed in your environment. It is not a managed SaaS.

## Pick an integration

Both integration options require `video-task-service` to be running.

| You want to integrate with | Use |
| --- | --- |
| Any agent product that supports MCP | `mcp-server` (`npm run mcp`) |
| OpenClaw, with a separately packaged plugin + skill | `adapters/openclaw-plugin` |

## Quickstart

**Prereqs**

- Node.js `>=22`
- XiaoIce provider credentials
- A public callback URL (or ngrok for local development)

### 1) Install

```bash
npm install
cp .env.example .env
```

Generate internal tokens and paste them into `.env`:

```bash
node -e "const c=require('crypto');const r=()=>c.randomBytes(24).toString('hex');console.log('VIDEO_SERVICE_INTERNAL_TOKEN='+r());console.log('VIDEO_SERVICE_ADMIN_TOKEN='+r());console.log('VIDEO_SERVICE_CALLBACK_TOKEN='+r());"
```

Fill at least these provider fields in `.env`:

- `VIDEO_PROVIDER_API_BASE_URL`
- `VIDEO_PROVIDER_API_KEY`
- `VIDEO_PROVIDER_VH_BIZ_ID` (or pass `vhBizId` per request)
- `VIDEO_PROVIDER_AUTH_HEADER` (some environments require `subscription-key`)

For the full list, see `.env.example`.

### 2) Start `video-task-service`

Pick one mode:

- Stable public callback URL: set `VIDEO_USE_NGROK=false`, set `VIDEO_CALLBACK_PUBLIC_BASE_URL=...`, then run `npm run service`
- Local development with ngrok: set `VIDEO_USE_NGROK=true` and `NGROK_AUTHTOKEN=...`, then run `npm run dev:up`

Smoke check:

```bash
curl -sS http://127.0.0.1:3105/health
```

## Use the HTTP API (direct)

Create a task:

```bash
curl -sS -X POST "http://127.0.0.1:3105/v1/tasks" \
  -H "Content-Type: application/json" \
  -H "X-Internal-Token: <VIDEO_SERVICE_INTERNAL_TOKEN>" \
  -d '{
    "prompt": "Generate a 10-second product demo video",
    "vhBizId": "demo-biz-id"
  }'
```

Query a task:

```bash
curl -sS "http://127.0.0.1:3105/v1/tasks/<taskId>" \
  -H "X-Internal-Token: <VIDEO_SERVICE_INTERNAL_TOKEN>"
```

Notes:

- Use `vhBizId` only. The legacy `vhbizmode` field is rejected.
- `prompt` is required for `create`. `taskId` is required for `get`.

## 1) Integrate with generic MCP-based agent products

Start the MCP stdio server (after the service is running):

```bash
XIAOICE_VIDEO_SERVICE_BASE_URL=http://127.0.0.1:3105 \
VIDEO_SERVICE_INTERNAL_TOKEN=<VIDEO_SERVICE_INTERNAL_TOKEN> \
npm run mcp
```

The MCP tool is:

- name: `xiaoice_video_produce`
- actions: `create | get`

Create:

```json
{
  "action": "create",
  "prompt": "Generate a 15-second spring product launch video",
  "vhBizId": "demo-biz-id"
}
```

Get:

```json
{
  "action": "get",
  "taskId": "task-123"
}
```

## 2) Use the OpenClaw plugin in this repo

This section is about [`adapters/openclaw-plugin`](./adapters/openclaw-plugin). It is not a guide for installing OpenClaw itself.

What you get:

- plugin id: `one-click-video`
- tool: `xiaoice_video_produce`
- bundled skill: `skills/xiaoice-video/SKILL.md`

The plugin is a thin adapter. It only forwards requests to `video-task-service`.

### 2.1 Package the plugin (standalone distribution)

```bash
cd adapters/openclaw-plugin
npm pack
```

This creates a tarball like `one-click-video-0.1.0.tgz`, containing the plugin code plus the bundled skill.

### 2.2 Configure the plugin

In your OpenClaw config, set only:

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

Keep secrets and provider settings on the service side, not in plugin config:

- `VIDEO_PROVIDER_API_KEY`
- `VIDEO_PROVIDER_VH_BIZ_ID`
- `VIDEO_PROVIDER_MODEL_ID`

### 2.3 Use the plugin

Once installed and configured, call the same tool:

```json
{
  "action": "create",
  "prompt": "Generate a 10-second product demo video"
}
```

Then poll:

```json
{
  "action": "get",
  "taskId": "task-123"
}
```

Typical terminal states are `succeeded`, `failed`, and `timeout`. On success, look for `videoUrl`.

### 2.4 How to use the bundled skill

The plugin ships a skill file at:

- `adapters/openclaw-plugin/skills/xiaoice-video/SKILL.md`

It is meant to help an agent:

- decide when to call `xiaoice_video_produce`
- prefer minimal arguments (usually only `prompt`)
- remember that status checks require a `taskId`
- point to the right config location when the plugin reports a config error

You usually do not need extra configuration for this skill: it is packaged and loaded with the plugin.

If you customize the skill, keep it portable:

- Do not add machine-specific absolute paths.
- Do not put real API keys or tokens into skill text.

## Update provider config at runtime

You can update provider credentials and default `vhBizId` without restarting the service clients:

```bash
curl -sS -X PUT "http://127.0.0.1:3105/v1/admin/config" \
  -H "Content-Type: application/json" \
  -H "X-Admin-Token: <VIDEO_SERVICE_ADMIN_TOKEN>" \
  -d '{
    "apiKey": "<NEW_VIDEO_PROVIDER_API_KEY>",
    "vhBizId": "<NEW_VH_BIZ_ID>"
  }'
```

## Troubleshooting

- `prompt is required`: `action=create` needs a non-empty `prompt`.
- `vhbizmode ... use vhBizId`: rename the field to `vhBizId`.
- OpenClaw plugin `config_error`: check `serviceBaseUrl` and `internalToken`.
- Callback never updates task status: verify your public callback base URL is reachable by the provider network.

## More docs

- `docs/04-deployment.md`
- `adapters/openclaw-plugin/README.md`
