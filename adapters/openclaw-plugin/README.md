# One Click Video OpenClaw Thin Plugin

This plugin uses the OpenClaw plugin id `one-click-video` and registers exactly one tool:

- `xiaoice_video_produce`

The plugin is a thin adapter only:

- validates OpenClaw tool inputs (`create` / `get`)
- reads runtime config from `plugins.entries['one-click-video'].config`
- calls the packaged client in `lib/video-service-client.js`
- returns OpenClaw tool result shape (`content[]` + `isError`)
- bundles skill metadata under `skills/xiaoice-video/SKILL.md`

The packaged client is intentionally vendored into the plugin package so the extension can load after `openclaw plugins install` without depending on this repository layout.

## Customer Distribution Notes

- `skills/xiaoice-video/SKILL.md` is packaged with the plugin and shipped to customer environments.
- Skill content must stay machine-portable: no host-specific absolute paths.
- Prefer placeholder endpoints (`<SERVICE_HOST>:<SERVICE_PORT>`) and deployment discovery steps.
- If skill-local file references are needed, use `{baseDir}` variable style.
- Design details: `docs/11-openclaw-skill-portable-config-design.md`.

## Tool Arguments

Supported fields:

- `action`: `create | get` (required)
- `prompt`: required for `create`
- `taskId`: required for `get`
- `sessionId`: optional
- `traceId`: optional
- `vhBizId`: optional
- `options`: optional object

Rejected field:

- `vhbizmode` (hard-cut, use `vhBizId`)

## OpenClaw Config Example

```json
{
  "plugins": {
    "entries": {
      "one-click-video": {
        "enabled": true,
        "config": {
          "serviceBaseUrl": "http://127.0.0.1:3105",
          "internalToken": "video-internal-token",
          "requestTimeoutMs": 15000
        }
      }
    }
  }
}
```

## Test

Run focused plugin tests:

```bash
node --test adapters/openclaw-plugin/__tests__/openclaw-plugin.test.js
```
