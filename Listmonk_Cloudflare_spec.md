# Newsletter spec: Listmonk + Email SDK + Cloudflare Email

Build a self-hosted newsletter for one publication. Operators prepare and schedule issues in Listmonk. A small Email SDK service delivers each message and records its outcome in Postgres. Configure publication details and expected sending volume during setup.

Use [Listmonk v6.2.0](https://github.com/knadh/listmonk/tree/v6.2.0) and [Email SDK source `2f89fbc34630`](https://github.com/opencoredev/email-sdk/tree/2f89fbc34630e75685a87f2497cff36378827970) as the reviewed baselines. Pin deployed image digests and the SDK artifact in a lockfile. Verify the selected SDK release contains the APIs used; its example documentation has stale version instructions. Preserve Listmonk's AGPL-3.0 and Email SDK's MIT license notices.

This is an implementation plan, not a deployed application. A weekend target is staging with fake-provider checks. A live pilot and full migration depend on the acceptance checks passing. The send service and Listmonk handoff patches are required engineering.

Cloudflare supplies email delivery and protected access. Listmonk and the delivery service run on your VPS. A Worker forwards Cloudflare delivery events into the service.

| Job | Use |
|---|---|
| Newsletter application | Listmonk in Docker on a VPS, managed by Coolify or Compose |
| Database | Self-hosted Postgres 16, separate Listmonk and delivery schemas |
| Public HTTPS and admin access | Cloudflare DNS, Tunnel or secured origin, and Access |
| Campaign preparation and scheduling | Listmonk, with a durable handoff patch |
| Sending jobs | Node.js delivery service and Postgres worker |
| Provider integration | Email SDK Cloudflare REST adapter |
| Delivery feedback | Cloudflare event subscription, Queue and consumer Worker |
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

**4. Configure the Cloudflare adapter.**

Use `@opencoredev/email-sdk/cloudflare` in the delivery service with `accountId` and an API token with Email Sending: Edit permission. Use the REST adapter from Node; no SMTP bridge or Workers runtime is needed for sending.

Enable Email Sending, onboard the domain and verify SPF/DKIM with aligned DMARC. Confirm the account's campaign-day quota and set a conservative send rate. Keep credentials in deployment secrets.

Use bare recipient addresses and keep tags/metadata in Postgres; this adapter rejects those message fields. Preserve required headers without silently dropping duplicate names.

Patch or wrap the reviewed adapter to expose `result.message_id` as the normalized receipt ID. It currently retains that field only in the raw response. Also handle `suppressed_recipients`, which it currently omits from its accepted/rejected result. Validate the outcome for the single intended recipient; empty, contradictory or partial results must not be counted as accepted.

The SDK reference says the REST API has no message ID, while Cloudflare's current API schema includes `message_id`. Follow the current provider contract and verify real response fixtures. An accepted send with no receipt ID must remain unresolved; never invent an ID. A definite suppression or rejection can be recorded without a send receipt.

Distinguish queued acceptance from immediate delivery and permanent rejection. Persist the receipt before reporting acceptance. Do not assume Cloudflare honors the SDK idempotency key. Test against current API response fixtures and one controlled real send.

Done when acceptance, suppression and bounce responses produce the correct stored state and a usable message ID for event correlation.

**5. Connect Cloudflare feedback to both systems.**

Subscribe the sending domain to Email Sending events through a Cloudflare Queue. A small consumer Worker forwards events to an authenticated delivery-service endpoint using service credentials. Validate account and domain. Acknowledge a queue message only after the receiver confirms persistence; configure retries and a dead-letter queue with alerts.

Process `message.delivered`, `message.complained` and `message.bounced`. Inspect `payload.bounce.type`; a soft bounce or `message.deferred` is not an immediate permanent suppression. Correlate `payload.messageId` with the receipt stored by the SDK adapter. The event itself does not contain Listmonk's campaign UUID.

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
- [SDK Cloudflare adapter](https://github.com/opencoredev/email-sdk/blob/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/src/cloudflare.ts)
- [Cloudflare sending API](https://developers.cloudflare.com/api/resources/email_sending/methods/send/)
- [Cloudflare delivery events](https://developers.cloudflare.com/email-service/platform/event-subscriptions/)
- [Cloudflare account limits](https://developers.cloudflare.com/email-service/platform/limits/)
