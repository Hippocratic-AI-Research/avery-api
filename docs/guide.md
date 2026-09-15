# Conversations API — Integration Guide

The `/v1/conversations` endpoints let your application run a turn-based text conversation between a member and a Hippocratic AI agent: create a conversation, exchange messages one turn at a time, read history at any point, end the conversation, and fetch the transcript.

This guide covers environments, authentication, the end-to-end flow with working `curl` examples, and the behaviors your client must handle. The full endpoint/schema reference (every field, every error code, interactive "try it") is the published API reference: **https://hippocratic-ai-research.github.io/avery-api/**

Security note: mint tokens from your server-side application only. Do not put `client_secret` values, bearer tokens, production patient data or PHI into browser/mobile code or hosted documentation pages.

_Verified against the deployed implementation on 2026-09-14._

---

## 1. Environments

**Safety Portal** — sandbox / demo
- API base URL: `https://api.safetyportal.hippocraticai.com`
- Token endpoint: `https://hai-dev-integrations.us.auth0.com/oauth/token`
- `audience`: `https://api.staging.hippocraticdev.com`

**UAT**
- API base URL: `https://uat-api.portal.us.hippocraticai.com`
- Token endpoint: `https://hai-prod-integrations.us.auth0.com/oauth/token`
- `audience`: `https://uat.api.hippocraticai.com`

**Production**
- API base URL: `https://api.portal.us.hippocraticai.com`
- Token endpoint: `https://hai-prod-integrations.us.auth0.com/oauth/token`
- `audience`: `https://api.hippocraticai.com`

Each environment has its own `client_id` / `client_secret` (provided by Hippocratic AI at onboarding — store them in a secret manager) and its own agents, scripts and patients. A token minted for one environment is rejected by the others: the `audience` and issuer must match.

The examples below use shell variables so they work against any environment:

```bash
export API_BASE="https://uat-api.portal.us.hippocraticai.com"
export TOKEN_URL="https://hai-prod-integrations.us.auth0.com/oauth/token"
export AUDIENCE="https://uat.api.hippocraticai.com"
export CLIENT_ID="<provided at onboarding>"
export CLIENT_SECRET="<provided at onboarding — keep in a secret store>"
```

## 2. Prerequisites

Before the first conversation can be created in an environment you need, from Hippocratic AI:

- **Credentials** — `client_id` / `client_secret` for that environment.
- **`agent_id`** and **`script_id`** — the agent persona and the conversation script it follows. These are configured for your use case and are stable per environment.
- **Patients** — a conversation is always about an existing patient. Patients must already be loaded for your partner (this is set up as part of onboarding); the `patient_id` you pass is **your own external identifier** for the patient, exactly as it was supplied when the patient was loaded. Creating a conversation for a patient that hasn't been loaded returns `404 patient_not_found`.

## 3. Authentication

All requests carry an OAuth 2.0 client-credentials bearer token: `Authorization: Bearer <token>`.

Because this flow uses a `client_secret`, token minting belongs in your backend or another trusted server-side environment. Frontend chat clients should call your backend, and your backend should call the Conversations API.

Scopes:

| Scope | Grants |
|---|---|
| `write:chat_messages` | `POST` — create conversation, send message, end conversation |
| `read:chat_messages` | `GET` — read conversation history, transcript, `/v1/whoami` |

### Mint a token

```bash
export TOKEN="$(curl -s -X POST "$TOKEN_URL" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"client_credentials\",
       \"client_id\":\"$CLIENT_ID\",
       \"client_secret\":\"$CLIENT_SECRET\",
       \"audience\":\"$AUDIENCE\",
       \"scope\":\"read:chat_messages write:chat_messages\"}" \
  | jq -r '.access_token')"
```

Tokens are short-lived (hours). Cache one and mint a new one when you receive `401`; do not mint per request.

### Verify the token

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$API_BASE/v1/whoami"
```

```json
{
  "auth_type": "client_credentials",
  "partner_id": "00000000-0000-4000-8000-000000000000",
  "partner_name": "Example Health",
  "credential_id": null,
  "client_id": "<your client_id>",
  "scopes": ["read:chat_messages", "write:chat_messages"]
}
```

Auth failures return the HTTP status **with no JSON body**:

- `401` — missing, expired or invalid token; wrong issuer or audience; not a client-credentials token.
- `403` — token is valid but lacks the scope the endpoint needs.

Everything is scoped to your partner. A conversation, patient, script or agent that belongs to another partner is reported as **not found**, never as forbidden.

## 4. The conversation flow

**create → (send message → show reply) × N → end → transcript**

| Step | Method & path | Scope |
|---|---|---|
| Create a conversation | `POST /v1/conversations` | write |
| Send a message | `POST /v1/conversations/{id}/messages` | write |
| Read status + history | `GET /v1/conversations/{id}` | read |
| End the conversation | `POST /v1/conversations/{id}/end` | write |
| Read the transcript | `GET /v1/conversations/{id}/transcript` | read |

`{id}` is the `conversation_id` (a UUID) returned by create. A malformed id returns `400 {"error": "Invalid uuid ..."}`.

### 4.1 Create a conversation

Creates the conversation and generates the agent's opening greeting before responding — expect a few seconds. The conversation is ready for messages as soon as you get the `201`.

```bash
curl -s -X POST "$API_BASE/v1/conversations" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "agent_id":   "3f0c2a4e-8b1d-4f6a-9c2e-5d7b1a2c3e4f",
    "script_id":  "9a8b7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d",
    "patient_id": "MRN-00012345"
  }'
```

`201 Created`

```json
{
  "conversation_id": "5c1a0b2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d",
  "created_at": "2026-09-14T15:04:05Z",
  "greeting": { "text": "Hi, I'm your care assistant. How have you been feeling since your last visit?" }
}
```

Show `greeting.text` as the first assistant message. Store `conversation_id`.

Errors: `400 unsupported_script_configuration`, `404 patient_not_found`, `500 prerequisite_data_unavailable | preparation_failed | llm_conversation_creation_failed` (nothing was created; safe to retry).

### 4.2 Send a message

One user turn in, one agent reply out. Send **only the new utterance** — the server holds the history.

```bash
curl -s -X POST "$API_BASE/v1/conversations/5c1a0b2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d/messages" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "text": "I have had a headache since this morning.",
        "idempotency_key": "8f14e45f-ce7e-4b0a-9c2a-1d2e3f4a5b6c" }'
```

`200 OK`

```json
{
  "message_id": "b7e2c1d0-4f3a-4b5c-9d6e-7f8a9b0c1d2e",
  "text": "I'm sorry you're dealing with that.\nHow would you rate the pain from 1 to 10?",
  "events": []
}
```

- `text` — the agent's reply. It may contain `\n` between sentences; each line is a separate agent utterance and appears as a **separate `assistant` message** in history and transcript. Render it as one bubble or several — your choice — but expect the split when you read history back.
- `message_id` — id of the agent's reply (matches the last assistant message for this turn in history).
- `events` — see §5. Empty for most turns.

**Concurrency:** keep at most one in-flight send per conversation. A second concurrent send returns `409 message_processing_busy` — wait for the first to finish, then retry.

**Retries:** always send an `idempotency_key` (a fresh UUID per user message). If you lose the response and retry with the same key and text, you get the **original reply and events back unchanged** — the turn is never re-run. Reusing a key with different text is rejected (`409 idempotency_key_reused`); a key whose original turn failed is rejected (`409 duplicate_message`) — mint a new key and resend.

Errors: `400 message_invalid` (blank text), `404 conversation_not_found`, `409 conversation_not_active | message_processing_busy | duplicate_message | idempotency_key_reused`, `500 message_processing_failed` (check history before resending), `503 conversation_runtime_unavailable | server_shutting_down` (transient — back off and retry).

### 4.3 Read status and history

Works at any time, including after the conversation has ended. Use it to rebuild UI state, recover a lost reply, or read the final `call_disposition`.

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$API_BASE/v1/conversations/5c1a0b2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d"
```

`200 OK`

```json
{
  "conversation_id": "5c1a0b2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d",
  "status": "active",
  "created_at": "2026-09-14T15:04:05Z",
  "agent_id": "3f0c2a4e-8b1d-4f6a-9c2e-5d7b1a2c3e4f",
  "script_id": "9a8b7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d",
  "patient_id": "MRN-00012345",
  "messages": [
    { "message_id": "a6d1…", "role": "assistant", "text": "Hi, I'm your care assistant. How have you been feeling since your last visit?", "created_at": "2026-09-14T15:04:05Z", "events": [] },
    { "message_id": "a7e2…", "role": "user",      "text": "I have had a headache since this morning.", "created_at": "2026-09-14T15:04:32Z", "events": [] },
    { "message_id": "b6e1…", "role": "assistant", "text": "I'm sorry you're dealing with that.", "created_at": "2026-09-14T15:04:34Z", "events": [] },
    { "message_id": "b7e2…", "role": "assistant", "text": "How would you rate the pain from 1 to 10?", "created_at": "2026-09-14T15:04:34Z", "events": [] }
  ]
}
```

- `status` — `active` or `ended`.
- `call_disposition` — present only once `status` is `ended` (may lag a moment after ending).
- `messages[].events` — each turn's events are replayed on the turn's final assistant message, in the same shape as the send-message response, so nothing is lost if a response was dropped.

### 4.4 End the conversation

```bash
curl -s -X POST "$API_BASE/v1/conversations/5c1a0b2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d/end" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "reason": "logged_out" }'
```

`200 OK`

```json
{ "conversation_id": "5c1a0b2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d", "call_disposition": "call_completed" }
```

- `reason` — `expired` (your inactivity timeout fired) and `logged_out` (member signed out) are recorded distinctly; any other string is recorded as a regular user end.
- `call_disposition` — one of `call_completed`, `call_hung_up`, `no_answer`, `failed_verification`, `call_failed`, `immediate_hang_up`. Treat unknown values as opaque.
- Ending an already-ended conversation is idempotent (`200`, same disposition).
- The `disposition` object in the reference is **not populated today** and is omitted from responses. Don't depend on it.

Errors: `404 conversation_not_found`, `409 message_processing_busy` (a turn is in flight — retry shortly), `500 conversation_end_failed`, `503` (transient).

### 4.5 Read the transcript

Only for **ended** conversations; returns `409 conversation_not_ended` otherwise. Transcript messages carry no `events` — use §4.3 for those.

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "$API_BASE/v1/conversations/5c1a0b2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d/transcript"
```

`200 OK`

```json
{
  "conversation_id": "5c1a0b2d-3e4f-4a5b-8c6d-7e8f9a0b1c2d",
  "status": "ended",
  "started_at": "2026-09-14T15:04:05Z",
  "ended_at": "2026-09-14T15:09:48Z",
  "messages": [ { "message_id": "…", "role": "assistant", "text": "…", "created_at": "…" } ]
}
```

## 5. Events

`events` on a message response (and replayed in history) is the signal channel for things your UI must react to. Each event has a `type`:

| `type` | When | Fields |
|---|---|---|
| `escalation` | A clinical concern is detected and evaluated | `escalation_id`, `status` (`detected` → `in_progress` → `completed`), `tier` (on `completed` only) |
| `transfer_to_human` | The agent hands the member to a human | `context_summary` (may be empty), `escalation_id` (only if an escalation caused it) |
| `end_call` | The agent ends the conversation | `reason` — today `completed` or `error` |

```json
{ "type": "escalation", "escalation_id": "high_temp", "status": "detected" }
{ "type": "escalation", "escalation_id": "high_temp", "status": "completed", "tier": "TIER_2" }
{ "type": "transfer_to_human", "escalation_id": "high_temp", "context_summary": "Member reports a persistent fever and requires a human handoff." }
{ "type": "end_call", "reason": "completed" }
```

**Tiers** are the closed set `TIER_0` … `TIER_4`. `TIER_0` = evaluated, no clinical escalation. `TIER_1` (least serious) → `TIER_4` (most serious). Match the tokens exactly; don't parse or sort them as numbers.

**Rules your client must follow**

1. **Escalations are not a fixed three-step sequence.** Each `status` is emitted at most once per evaluation, only when reached. A concern resolved in a single turn reports `completed` alone; one that never needs a follow-up skips `in_progress`.
2. **Not every escalation gets a `completed`.** A concern displaced by a higher-priority one reports nothing further. Don't block UI state waiting for a terminal event for every `escalation_id` you've seen.
3. **`escalation_id` repeats.** It identifies the *concern*, not the occurrence. If the same concern is re-evaluated later, the id is reused and a new lifecycle runs — treat the **latest** event for an id as its current state. An earlier `TIER_0` is not a permanent all-clear.
4. **`end_call` closes the conversation server-side.** Subsequent sends return `409 conversation_not_active`. Calling `POST …/end` afterwards is still fine.
5. **`transfer_to_human` does *not* end the conversation** and is never followed by an `end_call`. Your UI decides what happens next (typically: stop taking input and show the hand-off).
6. `transfer_to_human` is emitted at most once per conversation. Other event types may repeat.

## 6. Errors

Every error except auth and malformed-id returns:

```json
{ "error_code": "conversation_not_active", "message": "Conversation is not active." }
```

Branch on `error_code` (stable); never parse `message`. Treat unknown codes as a generic failure for that HTTP status.

| HTTP | Codes | Client action |
|---|---|---|
| `400` | `message_invalid`, `unsupported_script_configuration`; or `{"error": "Invalid uuid …"}` | Fix the request |
| `401` | *(no body)* | Mint a new token |
| `403` | *(no body)* scope missing; or `missing_partner` | Fix credentials/scopes |
| `404` | `conversation_not_found`, `patient_not_found`, `conversation_runtime_unavailable` | Check ids / patient loaded / conversation still resumable |
| `409` | `conversation_not_active`, `conversation_not_ended`, `message_processing_busy`, `duplicate_message`, `idempotency_key_reused` | See §4.2 / §4.5 |
| `422` | *(array of field errors)* | Fix the request body |
| `500` | `message_processing_failed`, `conversation_end_failed`, `history_replay_failed`, `prerequisite_data_unavailable`, `preparation_failed`, `llm_conversation_creation_failed` | Retry once; for sends, check history first |
| `503` | `conversation_runtime_unavailable`, `server_shutting_down` | Back off and retry |

## 7. Checklist for a first integration

- [ ] Mint a token in Safety Portal; `GET /v1/whoami` returns your `partner_name` and both scopes.
- [ ] Create a conversation for a loaded test patient; display `greeting.text`.
- [ ] Send messages with a fresh `idempotency_key` each; render `text`; handle `events`.
- [ ] Enforce one in-flight send per conversation; retry `409 message_processing_busy`.
- [ ] Handle `escalation` (incl. rule 3), `transfer_to_human`, `end_call` per §5.
- [ ] On any dropped response, recover from `GET /v1/conversations/{id}`.
- [ ] End with `reason`; read `call_disposition`; fetch the transcript.
- [ ] Repeat in UAT with UAT credentials/ids before production.
