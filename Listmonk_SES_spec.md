# Newsletter spec: Listmonk + Email SDK + SES

Build a self-hosted newsletter for one publication. Operators prepare and schedule issues in Listmonk. A small Email SDK service delivers each message and records its outcome in Postgres. Configure publication details and expected sending volume during setup.

Use [Listmonk v6.2.0](https://github.com/knadh/listmonk/tree/v6.2.0) and [Email SDK source `2f89fbc34630`](https://github.com/opencoredev/email-sdk/tree/2f89fbc34630e75685a87f2497cff36378827970) as the reviewed baselines. Pin deployed image digests and the SDK artifact in a lockfile. Verify the selected SDK release contains the APIs used; its example documentation has stale version instructions. Preserve Listmonk's AGPL-3.0 and Email SDK's MIT license notices.

This is an implementation plan, not a deployed application. A weekend target is staging with fake-provider checks. A live pilot and full migration depend on the acceptance checks passing. The send service and Listmonk handoff patches are required engineering.

Cloudflare provides DNS and protected access. Listmonk and the delivery service run on your VPS. SES delivers email through its API.

| Job | Use |
|---|---|
| Newsletter application | Listmonk in Docker on a VPS, managed by Coolify or Compose |
| Database | Self-hosted Postgres 16, separate Listmonk and delivery schemas |
| Public HTTPS and admin access | Cloudflare DNS, Tunnel or secured origin, and Access |
| Campaign preparation and scheduling | Listmonk, with a durable handoff patch |
| Sending jobs | Node.js delivery service and Postgres worker |
| Provider integration | Email SDK SES adapter over HTTP |
| Delivery feedback | SES/SNS to the delivery service, then Listmonk |
| Media and backups | Persistent media volume; private R2 backups |


**1. Deploy the app and delivery service.**

Run one Listmonk instance, Postgres 16 and a Node.js 22+ delivery-service container on a private Docker network. Deploy through Coolify or Docker Compose. Persist database and media volumes. Keep the database private and configure health checks.

Keep Listmonk's schema and migrations. Give the delivery service a separate schema and numbered migrations on the same Postgres server. An existing self-hosted Supabase Postgres instance can host these schemas; Supabase Auth is unnecessary.

Install `@opencoredev/email-sdk` server-side. Disable SDK telemetry with `EMAIL_SDK_TELEMETRY=0` and `telemetry: false`. Start with its memory test adapter and sending disabled.

Done when a restart preserves a draft and a queued test job without sending real email.

**2. Protect access and connect Listmonk to the service.**

Protect administration with Cloudflare Access and Listmonk authentication. Keep explicit public routes for signup, confirmation, unsubscribe, tracking and images. Use Tunnel or a secured origin to prevent direct-IP bypass. Protect the delivery endpoint with service credentials.

Use Listmonk's HTTP Postback messenger as the integration seam. Patch its payload to include the final rendered HTML and plain text, reply-to, and generated message headers. The reviewed code omits the alternate body and only forwards campaign-level headers. Preserve one-click unsubscribe headers and Listmonk's subscriber/campaign identifiers.

Persist one immutable send intent per campaign UUID and subscriber UUID before acknowledging the handoff. Use a unique key and payload hash: an identical replay returns the existing intent, while changed content under the same key fails. Test sends and confirmation messages need their own explicit keys. Return HTTP 200 only after durable persistence; Listmonk's postback expects 200.

Listmonk advances its recipient checkpoint before dispatch. Patch the producer so batch intent records are committed in Postgres in the same transaction as that checkpoint, before HTTP handoff. The service drains these records after restart. A postback acknowledgement alone cannot close the earlier crash window. Pause rather than advance if preparation fails.

Done when termination before or after HTTP handoff leaves every selected recipient accounted for without duplicate intents.

**3. Add the persistent send ledger.**

Adapt the SDK's [durable Postgres example](https://github.com/opencoredev/email-sdk/tree/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/examples/durable-postgres). Its persistence code is example application code, not functionality installed by the SDK. Extend its text-only message schema and fingerprint to preserve HTML and required headers. Retain the frozen recipient, content version and provider/account on each intent.

Use one recipient per job. Validate normalized messages with `email.validate()` before making them dispatchable; unsupported fields must fail visibly. Keep scheduling in Listmonk and the database because neither selected adapter supports `sendAt`. Record attempts and provider receipts. Support `queued`, `inflight`, `accepted`, `failed`, `suppressed` and `needs_reconciliation`. Keep provider delivery events separate from acceptance.

Let the Postgres worker own retries: set SDK `retry.maxAttempts: 1` and disable fallback. Automatically retry only outcomes proven not sent. Unknown outcomes, partial results and expired inflight leases stay stopped for review. A stable SDK key does not establish provider-side deduplication.

Before every new provider attempt, check current Listmonk subscription eligibility and local suppressions. Fail closed if that check fails. Double-opt-in confirmations use their own pending-subscriber rules. Recheck pause/cancel controls too. Pause must stop delivery-service dispatch, not just Listmonk's producer; already submitted messages may arrive.

Done when duplicate jobs are harmless, an unsubscribe after enqueue prevents sending, and a lost provider response never triggers a blind resend.

**4. Configure the SES adapter.**

Use `@opencoredev/email-sdk/ses` in the delivery service with the selected region and scoped AWS API credentials. Store secrets outside Git. This path calls SES's HTTP API; SES SMTP credentials are not used.

Verify the sending domain and configure DKIM with aligned SPF/DMARC. Confirm production access and enough quota for the expected campaign size. Rate-limit provider attempts and record the returned SES MessageId against the intent.

Preserve Listmonk's campaign/subscriber headers and unsubscribe headers through the adapter. Keep application metadata in Postgres; the SES adapter supports tags but rejects a message-level `metadata` field. Inspect received messages to verify this. The reviewed adapter does not forward the SDK idempotency key to SES; error responses default to an unknown delivery state. Add explicit, tested classification only for responses that prove non-acceptance. Do not treat every server error as safe to retry.

Done when an API test send has a stored receipt, valid authentication and working unsubscribe headers.

**5. Connect SES feedback to both systems.**

Configure SES identity bounce/complaint notifications through SNS, including original message headers and with raw delivery disabled. Receive them at a dedicated delivery-service endpoint. Retain Listmonk's existing signature-verification behavior or use a maintained verifier; require the expected SNS topic and validate subscription confirmations. Expose only this provider endpoint through the Access boundary.

Correlate SES `mail.messageId` with the stored receipt and campaign/subscriber identifiers. If delivered counts are required, add configuration-set delivery events and parse their distinct `eventType` shape explicitly. Otherwise label delivered status unavailable; acceptance is not delivery.

Persist authenticated events with a unique `(provider, account, event_id)` key. Commit event receipt, delivery-state changes and local suppression together; return success only after commit. Retain events that arrive before their send receipt and reconcile them once the mapping exists. Use recipient evidence to apply hard-bounce or complaint suppression immediately.

Use a durable outbox to synchronize suppression and bounce reporting into Listmonk. Make that receiver idempotent and return a retryable error on database failure: the reviewed bounce handler can return 200 after a failed write. Failed synchronization remains queued. Configure first-event blocklisting for hard bounces and complaints, with no immediate permanent block for temporary failures.

The SDK's current webhook normalization covers Resend, Postmark and Mailgun. Implement the provider-specific handler required here. Keep suppression effective if a future provider switch occurs.

Done when duplicate and early events are handled correctly, a database outage causes retry, and complaints or hard bounces prevent the next send in both systems.

**6. Finish imports and the operator workflow.**

Use Listmonk's importer, with a small CSV mapping script if needed. Normalize and deduplicate addresses. Import suppressions first and preserve existing list-level unsubscribes and global blocklisting on every reimport. Retain available consent source/dates. Imported subscribers must not enter welcome automations. Use double opt-in with rate limits for new public signups.

Create a reusable single-column template with optional images and a button. Include a visible unsubscribe link and mailing address. Check personalization fallbacks and plain-text rendering. Ensure subscriber links and one-click unsubscribe work without an administrator login.

Route all required mail types through the service, including campaign previews and signup confirmations. The postback path alone does not establish coverage of Listmonk's system notifications; wire and test those explicitly.

Display the selected audience and eligible count. Verify scheduling against the publication timezone. Keep Listmonk's editor and tracking. Show queued, provider-accepted and delivered counts separately from the ledger. Postback handoff success must not be presented as delivered mail.

Done when an operator can import subscribers and schedule an issue, and the received email renders acceptably in Gmail and Outlook.

**7. Prove recovery and controls.**

Test against the memory adapter and a fake provider HTTP endpoint. Terminate the producer around checkpoint commits and the worker before and after provider acceptance. Test duplicate handoffs, lost responses, stale jobs after rescheduling, and pause/cancel while jobs are queued. Verify current opt-outs before dispatch.

On restart, pause previously active campaigns for review and prevent new attempts for those campaigns until explicitly resumed. Resume known-unsent work from the ledger only. Keep lease-expired and uncertain jobs in `needs_reconciliation`; late events may resolve them without sending again. Require recorded evidence for an operator's retry decision.

Keep control state for the delivery worker after Listmonk finishes handing off a campaign. Never replay a whole campaign to repair totals. Pin each intent to its provider/account; provider switching does not reroute unresolved attempts.

Done when each selected recipient has an explainable outcome and interrupted work cannot silently skip or resend messages.

**8. Back up and run a controlled pilot.**

Back up both Postgres schemas daily and before upgrades to private R2 storage with defined retention. Include media and document secret recovery. Restore into staging with all sending disabled. Reconcile newer opt-outs before using any restored data in production.

Retain send-intent keys and receipt mappings beyond the replay window. Preserve suppressions independently of event cleanup. Restrict stored email content and define its retention.

Deliver Compose/Coolify configuration, environment examples and a short operator guide. Include failed-send review and event replay instructions. Monitor reconciliation jobs, feedback failures and backup failures.

Run a small consented live pilot after the tests pass. Inspect authentication and inbox placement in received messages. Record results before moving the full list.

Done when restore and recovery checks pass and the operator can run the newsletter.

## Scope and acceptance

The first version covers subscriber management and scheduled broadcasts, including public signup, imports, templates, segmentation, suppression and reporting. Multi-step welcome sequences and optional AI writing are separate follow-up work.

A pilot requires:

- Protected administration with working public subscriber links.
- Reimports that preserve every opt-out and trigger no welcome sends.
- A durable intent for every selected recipient before checkpoint advancement.
- Correct HTML, plain text and unsubscribe headers through the SDK.
- No blind retries or automatic provider fallback after uncertain outcomes.
- Pause/cancel and current subscriber eligibility enforced by the delivery worker.
- Replay-safe feedback with tested database-failure recovery.
- Accurate queued/accepted/delivered labels and a restore with sending disabled.

## Agent guidance

Keep `delivery-service/` and deployment configuration separate from the Listmonk patch set. Preserve the existing editor. Use one provider per deployment initially. The SDK supplies adapters; our application owns the durable queue and suppression policy. No exactly-once delivery claim is made.

Reviewed October 15, 2026. Read the [full SDK reference](https://email-sdk.dev/llms-full.txt) alongside the pinned source. The provider's current API contract takes precedence where the SDK documentation is stale.

- [Listmonk HTTP Postback](https://github.com/knadh/listmonk/blob/v6.2.0/internal/messenger/postback/postback.go)
- [Recipient checkpoints](https://github.com/knadh/listmonk/blob/v6.2.0/queries/campaigns.sql)
- [Bounce handler](https://github.com/knadh/listmonk/blob/v6.2.0/cmd/bounce.go)
- [Email SDK durable example](https://github.com/opencoredev/email-sdk/tree/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/examples/durable-postgres)
- [SDK retry implementation](https://github.com/opencoredev/email-sdk/blob/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/src/core/attempts.ts)
- [SDK webhook support](https://github.com/opencoredev/email-sdk/blob/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/src/webhooks.ts)
- [SDK SES adapter](https://github.com/opencoredev/email-sdk/blob/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/src/ses.ts)
- [Listmonk SES feedback configuration](https://github.com/knadh/listmonk/blob/v6.2.0/docs/docs/content/bounces.md)
