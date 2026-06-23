# Conversations API docs

Interactive reference for the `v1/conversations/*` endpoints, rendered with [Scalar](https://github.com/scalar/scalar) and served from GitHub Pages.

**Live:** https://friendly-adventure-9m971qg.pages.github.io/ (private — org members only)

## Files

- `docs/index.html` — Scalar, loaded from CDN, pointed at `docs/openapi.json`.
- `docs/openapi.json` — a self-contained, conversations-only OpenAPI 3.1 spec (the 5 endpoints + the schemas they use), with descriptions and examples added for these docs.

## Before sharing — two things to set

1. **Real base URL.** `docs/openapi.json` ships with a placeholder. Edit `servers[0].url` (`https://api.example.com`) to the real Conversations API base URL so the "Send Request" client hits it.
2. **Confirm the inferred bits.** The upstream spec had no field docs, so descriptions/examples were written by inference. Anything marked _(inferred)_ — `status`, `triage_tier`, `role`, `error_code`, care-gap and cadence values — is a guess; have a service owner confirm.

## Local preview

`python3 -m http.server -d docs 8000` then open http://localhost:8000

## Updating the spec

Hand-edit `docs/openapi.json`. It does not auto-sync from the upstream `Snow API` spec — if the conversations contract changes upstream, copy the changes over (the field descriptions/examples here are intentionally not in the source spec). The doc page re-renders automatically; no build step.

> Live "Send Request" calls go straight from the browser to the API. If they're blocked by CORS, enable CORS on the API for this origin, or add a `proxyUrl` in `docs/index.html` (note: a proxy routes request bodies through a third party — avoid for sensitive data).
