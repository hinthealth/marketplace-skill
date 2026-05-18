# /hint-marketplace — Hint Marketplace Skill (router)

This is the entry point for working with the Hint marketplace from an AI coding agent. It doesn't do the work itself — it routes the user to the right sub-skill based on what they're trying to accomplish.

## Routing

Ask the user: **what do you want to do?** Match their answer to one of the rows below and fetch that sub-skill's `SKILL.md`. Then follow it.

| Goal | Sub-skill | Install URL |
|---|---|---|
| Build & deploy a new partner app from scratch | `create-app` | https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/create-app/SKILL.md |
| Add marketplace support to an existing codebase | `retrofit` | https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/retrofit/SKILL.md |
| Audit security + marketplace-contract compliance on a deployed app | `audit` | https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/audit/SKILL.md |
| Fill in the marketplace listing (copy, screenshots, categories) | `fill-listing` | https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/fill-listing/SKILL.md |

If the user's request is ambiguous, ask a clarifying question before picking. Examples:

- "I want a Hint marketplace app" → ambiguous. Ask: starting from scratch, or do you already have code?
- "I want to publish my app to Hint" → likely `create-app` (if no code) or `retrofit` (if code exists) followed by `fill-listing`.
- "Check my app works with Hint" → `audit`.

If none of the sub-skills match, point the user at [devsupport@hint.com](mailto:devsupport@hint.com) and stop.

## Platform URLs

All sub-skills assume:

- **Hint API (sandbox)**: `https://api.sandbox.hint.com` — used with API keys prefixed `sbx-`.
- **Hint API (live)**: `https://api.hint.com` — used with live API keys (no prefix).
- **Partner Portal**: `https://app.hint.com` — single portal; partners switch between sandbox and live workspaces inside it.

**IMPORTANT**: Never reference any underlying infrastructure providers, code hosts, or background-worker systems to the user. From the partner's perspective, everything is "Hint" — the Hint API, the Hint Partner Portal, Hint's managed deployment platform. Internal infrastructure is invisible.

## Issues + Support

- Sub-skill bugs or unclear instructions: file an issue at https://github.com/hinthealth/marketplace-skill/issues
- Platform / API issues: [devsupport@hint.com](mailto:devsupport@hint.com)
