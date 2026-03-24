# OpenClaw Skill Portable Config Design (Customer Distribution)

## Context

`one-click-video` plugin ships `skills/xiaoice-video/SKILL.md` inside the extension package.
Customers install the plugin in their own environments, so any machine-specific path in skill text becomes invalid.

## Problem

Previous skill guidance used host-specific absolute paths and fixed localhost endpoints.
That works only on one machine and fails in customer installs where:

- repo location is different
- service runs under systemd or Docker
- service port/base URL differs
- config source is not `.env`

## External Constraints (OpenClaw Docs)

Based on OpenClaw skills documentation:

- plugin skills are loaded from extension package declarations (`packages.extensions[].skills`)
- skill files support variables such as `{baseDir}` for portable references
- skills have source/priority rules and can be overridden by other skill sources

References:

- https://docs.openclaw.ai/tools/skills
- https://docs.openclaw.ai/guides/skills

## Design Goals

1. Keep skill guidance portable across machines.
2. Keep ownership boundaries clear: plugin config vs service-side provider config.
3. Make API key / `vhBizId` updates actionable without assuming deployment style.
4. Avoid absolute paths in distributed skill content.

## Proposed Skill Structure

1. `Configuration Ownership (Portable)`
2. `Canonical Field Meanings`
3. `How To Locate Service Config On Any Machine`
4. `Recommended Update Flow`
5. `Distribution Notes`

## Key Decisions

- Explicitly define:
  - XiaoIce API key = `VIDEO_PROVIDER_API_KEY`
  - digital-human identity in create flow = `vhBizId`
  - default env key for `vhBizId` = `VIDEO_PROVIDER_VH_BIZ_ID`
- Runtime update first via `PUT /v1/admin/config`.
- Persistence second via deployment config (`.env` / systemd / compose / secret manager).
- Use placeholder templates (`<SERVICE_HOST>`, `<SERVICE_PORT>`, `<VIDEO_SERVICE_ADMIN_TOKEN>`) instead of host-fixed examples.
- Keep `VIDEO_PROVIDER_MODEL_ID` as advanced optional field, not a universal must-have.

## Non-Goals

- Not redesigning create/get API contract in this doc.
- Not coupling skill behavior to a single repo path.
- Not requiring one deployment method.

## Implementation Scope

- Update `adapters/openclaw-plugin/skills/xiaoice-video/SKILL.md` with portable discovery + update flow.
- Keep plugin packaging unchanged (`openclaw.plugin.json` + `package.json` already include `skills`).
