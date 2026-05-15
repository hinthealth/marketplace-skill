# Hint Marketplace Skill

An installable AI agent skill that builds and deploys a working partner app to the [Hint](https://hint.com) marketplace.

## What this is

`SKILL.md` is a self-contained natural-language playbook that any AI coding agent (Claude Code, Cursor, etc.) can load and execute. Given a description of the app you want to build, the skill drives the full marketplace flow end-to-end:

- Verifies the partner account + sandbox setup
- Generates a Node.js app that implements the Hint marketplace contract (handshake, headless connect, embedded UI surface)
- Deploys the app to Hint-managed infrastructure via the Public API
- Configures the partner + app + anchors for marketplace install

The output is a real, installable marketplace app — usually in under 5 minutes.

## Installation

Paste this into your AI coding agent of choice:

> Install this marketplace skill: https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/SKILL.md

The agent will fetch the skill and place it in whatever location it reads skills from (`.claude/commands/`, `.cursor/rules/`, etc.).

## Usage

After installing, ask your agent to run the skill — e.g. `/hint-marketplace-create-app` (Claude Code) or "use the marketplace skill" (any agent). The skill will prompt for:

1. **What does your app do?** A short description (1-2 sentences).
2. **Which surface?** Either a Core Page (full-screen tab inside the practice portal) or a Clinical Interaction (rendered alongside a patient's chart).
3. **Sandbox API key.** Generated from the Hint Partner Portal.

From there it builds, deploys, and prints the install instructions.

## Requirements

- A Hint partner account with `product.type = "app"` ([request access](https://hint.com/partners))
- A sandbox practice (auto-provisioned with the partner account)
- A sandbox API key from the Partner Portal

## Versioning

Skill version lives in the file's frontmatter. Bumps follow semver — major bumps mean a breaking change to the marketplace contract.

## Issues

File issues against this repository for skill bugs or unclear instructions. For platform / API issues, contact [support@hint.com](mailto:support@hint.com).
