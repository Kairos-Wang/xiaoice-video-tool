---
name: xiaoice-video
description: Use the `xiaoice_video_produce` tool when the user wants to create or check XiaoIce video generation tasks, mentions One Click Video or "一键成片", or asks to generate a video from a prompt.
metadata: { "openclaw": { "homepage": "https://github.com/ROUCHER27/xiaoice-video-tool" } }
---

# XiaoIce Video Tool

Use the single tool `xiaoice_video_produce`.

Activate this skill when the user:

- asks to generate a XiaoIce video from text
- asks to check a previously submitted XiaoIce video task
- mentions `xiaoice_video_produce`, `one-click-video`, "One Click Video", or "一键成片"
- asks whether a submitted task is still running, failed, or finished

## Actions

### Create a task

Use `action: "create"` with:

- `prompt`: required text prompt
- `vhBizId`: optional per-request business id
- `sessionId`: optional
- `traceId`: optional
- `options`: optional object for advanced passthrough fields

Prefer sending only `prompt` unless the user provided the extra fields or explicitly asked for advanced options.

Example:

```json
{
  "action": "create",
  "prompt": "Generate a 15 second spring product launch video in a bright style",
  "vhBizId": "demo-biz-id"
}
```

### Get task status

Use `action: "get"` with:

- `taskId`: required task id returned by `create`

If the user asks for status but no `taskId` is available in the conversation, explain that you need the task id before you can check the task.

Example:

```json
{
  "action": "get",
  "taskId": "task-123"
}
```

## Rules

- Use `vhBizId` only. Never use `vhbizmode`.
- For new generation requests, call `create` first.
- For follow-up status checks, call `get` with the known `taskId`.
- If the user asks to "see if it finished", poll with `get`.
- Treat `submitted` and `processing` as non-terminal states.
- Terminal states are `succeeded`, `failed`, and `timeout`.
- On success, return the `videoUrl` if present.
- Always include the current `taskId` and `status` in your reply.
- If create returns `submitted` or `processing`, tell the user the task was accepted and they can check again with the returned `taskId`.
- If status is `failed` or `timeout`, surface any returned error/details instead of claiming success.
- If the tool reports a config error, explain that the OpenClaw plugin needs `plugins.entries.one-click-video.config.serviceBaseUrl` and `internalToken`.

## Configuration Ownership (Portable)

- OpenClaw plugin config only owns:
  - `plugins.entries.one-click-video.config.serviceBaseUrl`
  - `plugins.entries.one-click-video.config.internalToken`
  - `plugins.entries.one-click-video.config.requestTimeoutMs`
- XiaoIce provider credentials belong to `video-task-service`, not the OpenClaw plugin config.
- Never tell the user to edit provider key/model fields under `plugins.entries.one-click-video.config`.

### Canonical Field Meanings

- XiaoIce API key means `VIDEO_PROVIDER_API_KEY`.
- XiaoIce digital-human identity for create requests is `vhBizId`.
- Default service-side `vhBizId` env key is `VIDEO_PROVIDER_VH_BIZ_ID`.
- `VIDEO_PROVIDER_MODEL_ID` is an advanced service-side field. Do not assume every deployment needs it.

## How To Locate Service Config On Any Machine

When users ask to change XiaoIce API key or `vhBizId`, do not guess local paths.

1. Identify where `video-task-service` runs (repo process, systemd, or Docker/Compose).
2. Confirm service address and admin token source.
3. Update runtime first via admin API.
4. Then persist the same values in deployment config to survive restart.

Portable discovery commands (share as needed):

```bash
ps aux | rg "video-task-service|src/service/cli.js|node .*service"
```

```bash
systemctl status video-task-service
```

```bash
docker ps | rg "video|xiaoice|openclaw"
```

## Recommended Update Flow

- Preferred: runtime update via service admin API, effective immediately for new tasks.
- Then persist values in deployment config (`.env`, systemd `EnvironmentFile`, compose env, secret manager, etc.).

Portable command template:

```bash
curl -sS -X PUT "http://<SERVICE_HOST>:<SERVICE_PORT>/v1/admin/config" \
  -H "Content-Type: application/json" \
  -H "X-Admin-Token: <VIDEO_SERVICE_ADMIN_TOKEN>" \
  -d '{
    "apiKey": "<NEW_VIDEO_PROVIDER_API_KEY>",
    "vhBizId": "<NEW_VH_BIZ_ID>"
  }'
```

Verification template:

```bash
curl -sS "http://<SERVICE_HOST>:<SERVICE_PORT>/v1/admin/config" \
  -H "X-Admin-Token: <VIDEO_SERVICE_ADMIN_TOKEN>"
```

Persisted env-style keys:

- `VIDEO_PROVIDER_API_KEY=<NEW_VIDEO_PROVIDER_API_KEY>`
- `VIDEO_PROVIDER_VH_BIZ_ID=<NEW_VH_BIZ_ID>`
- `VIDEO_PROVIDER_API_BASE_URL=<PROVIDER_BASE_URL>`
- `VIDEO_PROVIDER_AUTH_HEADER=<AUTH_HEADER_NAME>`

## Distribution Notes

- This skill is bundled and distributed with the plugin package.
- Do not hardcode machine-specific absolute paths in distributed skill content.
- If referencing skill-local files/scripts, use `{baseDir}` rather than host-specific paths.

## Expected Flow

1. Submit with `create`.
2. Keep the returned `taskId`.
3. Check progress with `get`.
4. When status becomes `succeeded`, use the returned `videoUrl`.
