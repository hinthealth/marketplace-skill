# Hint Marketplace Skill

A bundle of installable AI agent skills for working with the [Hint](https://hint.com) marketplace.

## What this is

Each `SKILL.md` in this repo is a self-contained natural-language playbook that any AI coding agent (Claude Code, Cursor, etc.) can load and execute. The root `SKILL.md` is a **router** — it asks the user what they're trying to do and points them at the matching sub-skill.

## Install

Paste this into your AI coding agent:

> Install this marketplace skill: https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/SKILL.md

The agent will fetch the router, ask you what you're trying to do, and pull the matching sub-skill.

If you already know which sub-skill you want, you can install it directly:

| Sub-skill | What it does | Install URL |
|---|---|---|
| **create-app** | Build & deploy a new partner app from scratch | https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/create-app/SKILL.md |
| **retrofit** | Audit an existing app against the marketplace contract, generate the missing pieces, and wire it up | https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/retrofit/SKILL.md |
| **audit** | Pass/fail security + marketplace-contract audit of a deployed app | https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/audit/SKILL.md |
| **fill-listing** | Draft + apply the marketplace listing via the public API | https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/fill-listing/SKILL.md |

## Repository layout

```
.
├── SKILL.md                     # Router — entry point, picks a sub-skill based on user intent
├── README.md                    # This file
│
├── create-app/SKILL.md          # ✅ Build & deploy a new app
├── retrofit/SKILL.md            # ✅ Add marketplace support to an existing app
├── audit/SKILL.md               # ✅ Pass/fail audit of a deployed app
├── fill-listing/SKILL.md        # ✅ Populate the marketplace listing via the public API
│
└── _common/                     # Shared fragments referenced by multiple sub-skills
    ├── api-conventions.md       # Hosts, auth, response shapes, pagination, reserved env vars
    ├── marketplace-contract.md  # Required routes, signature verification, smoke test
    └── provider-api.md          # Provider API endpoints, practice-scoped auth, JS SDK
```

The `_common/` folder holds cross-cutting conventions so individual sub-skills don't repeat them. When a sub-skill needs to teach the agent about (e.g.) the bare-array response shape on `/api/provider/*`, it links to `_common/api-conventions.md` instead of inlining the same prose.

## Requirements

- A Hint partner account with `product.type = "app"` ([request access](https://hint.com/partners))
- A sandbox partner + sandbox practice (auto-provisioned with the account; create from the Partner Portal under **Sandboxes**)
- A sandbox API key from the Partner Portal

## Versioning

Skill versions live in each `SKILL.md`'s frontmatter. The repo as a whole follows semver via git tags (e.g. `v1.0.0`). Major bumps mean a breaking change to the marketplace contract.

## Issues

File issues against this repository for skill bugs or unclear instructions. For platform / API issues, contact [devsupport@hint.com](mailto:devsupport@hint.com).
