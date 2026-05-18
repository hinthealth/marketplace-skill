# /hint-marketplace-retrofit — Add Marketplace Support to an Existing App

**Status:** 📋 not yet implemented. This skill is on the roadmap. Track progress at https://github.com/hinthealth/marketplace-skill/issues or contact [devsupport@hint.com](mailto:devsupport@hint.com).

## What this skill will do

Given a partner's existing application (any stack — Next.js, FastAPI, Rails, Go, etc.), audit it against the Hint marketplace contract and produce a reviewable diff that adds the missing pieces:

- `POST /hint/handshake` route — HMAC-SHA256 signature verification with the partner's webhook secret, session mint + persist
- `POST /hint/connect/:code` route — OAuth code exchange against `$HINT_API_URL/api/oauth/tokens`, persist the practice-scoped access token keyed by practice
- `GET /hint/<anchor_type>?session_key=...` route — render the embedded UI, using the session to determine which practice's data to show
- Env-var hooks for `HINT_API_URL`, `HINT_API_KEY`, `HINT_PARTNER_ID`, `HINT_WEBHOOK_SECRET`, `DATABASE_URL`
- (Optional) `<script src="$HINT_API_URL/hint-sdk.js">` integration on the embedded pages

The skill works whether the partner's app is a fully-wired-up production app (small diff), a partial implementation (medium diff), or an empty folder (effectively the same as `create-app`).

## Until this skill ships

Use [`create-app`](https://raw.githubusercontent.com/hinthealth/marketplace-skill/main/create-app/SKILL.md) as a reference implementation — its server.js template is a complete Node.js example of all three marketplace-contract routes. Port the handshake-verification + OAuth-exchange + anchor-rendering code straight across to whatever stack the partner is using.
