# Provider API Mapping (v1 Draft)

## 1. Purpose

Define how internal task requests map to XiaoIce provider API:

- Provider endpoint: `POST /openapi/aivideo/create`
- Internal source: `POST /v1/tasks`

## 2. Request Field Mapping

| Internal Field | Provider Field | Required | Notes |
| --- | --- | --- | --- |
| `prompt` | `topic` | Yes | Primary text input |
| `vhBizId` | `vhBizId` | Yes | Canonical public business id |
| generated callback URL | `callbackUrl` | Yes | Built from `VIDEO_CALLBACK_PUBLIC_BASE_URL` |
| `options.title` | `title` | No | Pass-through |
| `options.content` | `content` | No | Pass-through |
| `options.materialList` | `materialList` | No | Pass-through |
| `options.ttsConf` | `ttsConf` | No | Pass-through |
| `options.aigcWatermark` | `aigcWatermark` | No | Pass-through |

## 3. Internal Validation Rules

- `prompt` must be non-empty string.
- `vhBizId` must be non-empty string.
- `options` must be JSON object if provided.
- callback base URL must be configured before submission.

## 4. Provider Payload Example

```json
{
  "topic": "A short product launch video in energetic style",
  "vhBizId": "demo-biz-mode",
  "callbackUrl": "https://example.ngrok.app/v1/callbacks/provider?token=<masked>",
  "title": "Launch Clip",
  "content": "30-second narrative",
  "materialList": [],
  "ttsConf": {
    "voice": "female"
  },
  "aigcWatermark": true
}
```

## 5. Internal Create Request Example

```json
{
  "prompt": "A short product launch video in energetic style",
  "vhBizId": "demo-biz-mode",
  "sessionId": "sess-123",
  "traceId": "trace-123",
  "options": {
    "title": "Launch Clip",
    "content": "30-second narrative",
    "materialList": [],
    "ttsConf": {
      "voice": "female"
    },
    "aigcWatermark": true
  }
}
```

## 6. Response/Status Normalization

Provider response and callback statuses must be normalized into:

- `submitted`
- `processing`
- `succeeded`
- `failed`

Normalization constraints:

- never expose raw provider payload directly as final API contract
- keep raw payload in logs (or optional debug field) for diagnostics

## 7. Error Handling and Retry Policy (v1)

Submission path:

- provider `2xx`: mark `submitted`
- provider `4xx`: mark `failed` (no automatic retry)
- provider `5xx` or network timeout: retry with bounded backoff, then mark `failed`

Callback path:

- invalid token: reject with `401/403`, no state mutation
- unknown task binding: reject with `404`
- malformed payload: reject with `400` and log body hash

## 8. Mapping Contract Checklist

- [ ] provider request uses `topic` (not `prompt`)
- [ ] provider request uses `vhBizId` (canonical public field)
- [ ] callback URL uses configured public base URL
- [ ] advanced options pass-through without silent field rename
- [ ] internal API remains stable even if provider contract evolves
