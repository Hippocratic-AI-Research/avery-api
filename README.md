# Conversations API docs

Interactive reference for the `v1/conversations/*` endpoints, rendered with [Scalar](https://github.com/scalar/scalar) and served from GitHub Pages.

**Live:** https://hippocratic-ai-research.github.io/avery-api/

## Files

- `docs/index.html` — Scalar, loaded from CDN, pointed at `docs/openapi.json`. The reference.
- `docs/openapi.json` — a self-contained, conversations-only OpenAPI 3.1 spec: the 5 partner-facing endpoints, the schemas they use, and the event contract, with descriptions and examples written for these docs.
- `docs/guide.md` — the Integration Guide (served at `/guide.html` via GitHub Pages' built-in Jekyll rendering): environments, authentication, end-to-end `curl` walkthrough, event-handling rules, error table, first-integration checklist. This is the single document to hand a new partner or UI team, together with their credentials.

## Provenance

Endpoint paths, methods, field names, types, required/nullable flags, enums and event shapes are taken from the service's generated OpenAPI source. Descriptions of runtime behaviour (which error codes each endpoint emits, status/role/disposition values, idempotency semantics, which events are emitted today) were verified against the implementation on the date noted at the bottom of the spec's `info.description`.

Not included on purpose: non-production test endpoints, and partner-specific onboarding details (token endpoints, credentials). Those are provided directly to each integrating partner.

## Local preview

`python3 -m http.server -d docs 8000` then open http://localhost:8000

## Updating the spec

`docs/openapi.json` is hand-maintained and does not auto-sync from the service. When the conversations contract changes upstream, regenerate the upstream spec, diff the `/v1/conversations*` paths and their schemas against this file, and carry the changes over — keeping the descriptions and examples here. Update the verification date in `info.description`. The page re-renders automatically on merge to `main`; there is no build step.

> Live "Send Request" calls go straight from the browser to the API. If they're blocked by CORS, enable CORS on the API for this origin, or add a `proxyUrl` in `docs/index.html` (note: a proxy routes request bodies through a third party — avoid for sensitive data).
