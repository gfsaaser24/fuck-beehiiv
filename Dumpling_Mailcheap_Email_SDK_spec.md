# Newsletter spec: Dumpling Mailcheap + Email SDK

Build on [Dumpling's Mailcheap fork](https://github.com/dumpling-software/mailcheap/tree/8db6785ad90d05a1372d14706828eb58e9052796). Keep its newsletter interface and Postgres database. Add Email SDK inside the app and fix the sending and recovery gaps below.

In plain English: Mailcheap manages the newsletter. Email SDK connects it to the delivery company. Mailcheap still has to remember what happened to every email.

Use Amazon SES for the shortest first implementation, because this fork already has SES feedback handling. Cloudflare Email Service is an alternative covered below. Choose one provider initially.

This is a build plan. A weekend target is a working staging deployment; moving a live list depends on the completion checks. No deployment or live-send validation has been completed.

## What we keep and what changes

The fork already has real Postgres through Drizzle, a single-column editor, imports, scheduled campaigns, signup and unsubscribe handling, tracking, a public archive and an automation engine. Keep those features.

| Existing issue | Does Email SDK solve it? |
|---|---|
| Sending is coupled to SES | Helps: adapters provide a common sending interface. Provider setup, feedback and the SES-specific dashboard still need work. |
| A message is marked sent before the provider call | No. Change Mailcheap's message states and recovery logic. |
| Unsubscribe and suppression data is cached for a whole batch | No. Recheck eligibility immediately before each attempt. |
| Unknown send outcomes are treated as ordinary failures | Helps expose error information. Mailcheap must stop uncertain sends and reconcile them. |
| Background jobs depend on QStash | No. Keep QStash for this version; replacing it is separate work. |
| Better inbox placement | No automatic improvement. Authentication, reputation and recipient engagement still matter. |

Reviewed baselines: Mailcheap commit `8db6785ad90d` and [Email SDK commit `2f89fbc34630`](https://github.com/opencoredev/email-sdk/tree/2f89fbc34630e75685a87f2497cff36378827970). Pin deployed artifacts and the SDK dependency in the lockfile. Both reviewed snapshots have MIT license notices; preserve them.

## Eight tasks

**1. Deploy the existing app on our stack.**

Run Next.js on Node.js 22+ and self-hosted Postgres through Docker Compose or Coolify. Keep Drizzle and the existing migrations. An existing self-hosted Supabase Postgres instance also works; no database conversion is needed. Add a Dockerfile, persistent volumes, health checks and an environment example. Start with `SENDING_ENABLED=false`.

Use Cloudflare DNS and protected admin access. Preserve working public signup, confirmation, unsubscribe, tracking and archive routes. QStash callbacks and provider events need their own verified authentication. Keep the origin protected and the database private.

Keep image uploads through the existing Supabase Storage integration if available. For plain Postgres deployments, add a small R2 implementation behind `lib/blob.js`; preserve upload validation and permanent public image URLs. Choose one storage backend for the initial deployment.

**Done when:** the app starts from documented configuration, survives a restart and displays an uploaded image without an admin login.

**2. Add Email SDK inside Mailcheap.**

Create a server-only sending module, such as `lib/email-provider.js`, using `@opencoredev/email-sdk`. Mailcheap is already a Node application, so no separate HTTP delivery bridge is required.

Replace the sending call in `lib/sending.js` and route `lib/alerts.js` through the same provider wrapper. Cover previews, confirmations, welcome emails, automation messages and campaigns. Preserve the HTML/text bodies, reply-to and signed unsubscribe headers produced by the existing renderer. Validate supported fields before dispatch.

Set `retry.maxAttempts: 1` and disable automatic provider fallback. The application will own retry decisions. Disable SDK telemetry with `EMAIL_SDK_TELEMETRY=0` and client `telemetry: false`.

| Provider | Required integration |
|---|---|
| SES | Use the SES adapter with explicit server credentials, region and configuration set. Preserve message tags, especially `message_id`, and the returned provider ID. Keep the existing AWS client where still needed for account/domain checks. |
| Cloudflare | Use the REST adapter from Node with account ID and API token. Keep tags in Postgres because this adapter rejects them. Patch or wrap the reviewed adapter to expose `result.message_id` and handle `suppressed_recipients`. Validate the outcome for the single intended recipient. Missing or contradictory acceptance evidence remains unresolved. |

Make the account/domain screens provider-aware. Cloudflare deployments must not require SES credentials to compose or send. Configure the chosen provider's domain authentication and actual account quota.

**Done when:** a fake provider receives the correct message and one-click unsubscribe headers, and no send path silently falls back to the old SES sender.

**3. Repair the message lifecycle.**

Work in `db/schema.js`, `lib/sending.js` and `lib/sending-rules.js`. Reuse the existing message records and uniqueness constraints. Adapt ideas from the SDK's [durable Postgres example](https://github.com/opencoredev/email-sdk/tree/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/examples/durable-postgres); its database logic is application code to implement. Its text-only payload model must be extended for newsletter HTML and headers.

Replace the current `queued → sent` claim with `queued → inflight`. Add an attempt token, lease, attempt history and next-attempt time. Store `accepted` only after verified provider acceptance. Keep `failed`, `suppressed` and `needs_reconciliation` distinct.

Freeze the rendered message and sender settings when creating the send intent. Persist the sending provider/account and receipt ID. The existing `messages.provider` means the recipient's mailbox provider; add a separate `sendingProvider` field.

Commit acceptance, receipt mapping and counter changes together, conditional on the current attempt token. Late workers must not overwrite newer state. An expired lease or lost response goes to review, never straight back to the queue. Retry only when evidence proves the provider did not accept the message, with bounded backoff. Neither the SES nor Cloudflare adapter supplies provider-side deduplication.

Migrate historical rows conservatively. Old `sent` rows without reliable acceptance evidence need review, not automatic resending. Distinguish queued, accepted and delivered counts; unresolved work must remain visible when a campaign settles.

**Done when:** killing the process around a provider call produces an explainable unresolved record instead of a silent skip or automatic duplicate.

**4. Check current eligibility and preserve automations.**

Immediately before the provider request, after any rate-limit wait, reread the contact, suppression and campaign/automation controls. Stop on database failure. Check the actual frozen destination; an address change must not reuse consent from a different address. Confirmations retain their pending-contact rules.

Keep the signed unsubscribe footer and one-click POST route working without login. Back up the signing secret: changing the current `SESSION_SECRET` breaks links in previously sent emails. Pause and the global sending switch must stop new attempts; already submitted mail can still arrive.

Update `sendAutomationEmail()` and `lib/automations/run.js`. The current code treats any existing non-queued message as already sent. Advance a send node only after recorded acceptance. Hold inflight or uncertain sends; record a visible failure or suppression outcome instead of silently advancing as success. Use lease tokens for enrollment updates so an expired runner cannot overwrite a newer runner.

**Done when:** an unsubscribe during a running batch prevents later attempts, and a retry cannot skip an unresolved automation email.

**5. Make scheduling recoverable.**

Retain QStash for batch delivery, scheduled wakeups and the automation timer. This version self-hosts the app and database; QStash remains a hosted dependency. The SDK does not replace it.

Persist message intent before publishing a job. Preserve campaign/contact and enrollment/node uniqueness; use a stable request key for other retryable send requests. Add a periodic database recovery scan for eligible queued messages and missed schedules, including welcome/confirmation messages. Duplicate callbacks must use the same atomic claims. Keep the existing checks that ignore obsolete jobs after rescheduling. Alert when jobs exhaust retries.

Use a shared send-rate budget per sending account across campaigns and signup mail; the current in-process timer is insufficient across instances. A Postgres-backed reservation is adequate. Keep SDK retries disabled so they cannot bypass the budget.

**Done when:** a failed job publication or app restart does not strand known-unsent work, while expired inflight attempts remain held for reconciliation.

**6. Preserve feedback and suppression.**

Email SDK's reviewed webhook utilities do not normalize SES or Cloudflare events. Implement these provider-specific paths.

For SES, retain SNS signature verification and the configured topic check. Pass the SNS event ID into processing. Preserve `message_id` tag correlation for events arriving before the receipt is stored.

For Cloudflare, use an Email Sending event subscription with a Queue and consumer Worker forwarding to an authenticated app endpoint. Store `payload.eventId`, correlate `payload.messageId` with the receipt and acknowledge only after persistence. Handle delivered, deferred, bounced and complained events; temporary failures are not permanent suppressions.

Deduplicate by provider/account/event ID. Commit the event and its state/counter/suppression changes together. Retain unmatched events for later correlation. Apply authenticated hard-bounce suppression immediately; hold unmapped complaint recipients pending reconciliation. A late delivery event must never clear a suppression.

Preserve the fork's global hard-bounce and newsletter-level unsubscribe/complaint rules. Also honor any broader provider suppression. Switching providers must not bypass a block.

**Done when:** replayed and early events produce correct totals, database outages trigger retry, and a complaint or hard bounce stops the next relevant send.

**7. Keep the ordinary newsletter workflow simple.**

Keep the existing editor and templates. Verify text, images and an optional HTML button in a single-column email; a designer rebuild is outside this version.

Import contacts with normalized addresses and preserved opt-outs. Import suppressions before active subscribers. Reimporting must never reactivate someone or trigger welcome sequences. Retain available consent records.

Keep public signup and double opt-in, with rate limits. Verify previews, timezone-aware scheduling and pause/resume. Keep the existing automation engine; prove a basic welcome sequence before enabling it. Show accepted and delivered separately, and treat open tracking as approximate.

**Done when:** an operator can import a list and schedule an issue, and its subscriber links work in received messages.

**8. Verify recovery and prepare the handoff.**

Run the existing unit/database tests and add focused failure tests for the changes: concurrent jobs, crashes around acceptance, lost responses, early/duplicate feedback, an opt-out mid-batch and automation recovery. Also run migrations and recovery checks against the actual Postgres version deployed.

Back up Postgres, assets and signing-secret recovery information. Restore into staging with sending disabled. Reconcile newer opt-outs and uncertain attempts before resuming a restored deployment.

Deliver deployment files and a short operator guide covering provider setup, job recovery and unresolved-send review. Keep logs/monitoring usable when email alerts cannot send.

After those checks, run a small consented pilot. Inspect SPF/DKIM/DMARC and rendering in received messages, then assess actual inbox placement. SDK acceptance is not evidence of inbox delivery.

**Done when:** restore and failure checks pass, and the pilot results are recorded.

## Agent guidance and sources

Keep the editor and schema structure. Put provider-specific sending behind one module and add numbered Drizzle migrations. Start with one provider. Retain existing tests and update all status-dependent queries together. Do not claim exactly-once delivery.

The SDK removes some provider integration work. The lifecycle and suppression fixes above are what make this fork safer to operate.

Reviewed October 15, 2026. Read the [full Email SDK reference](https://email-sdk.dev/llms-full.txt) alongside the pinned implementation. Cloudflare's current API documents response fields missing from the reviewed SDK normalization; verify response fixtures before enabling that provider.

- [Mailcheap sending and claims](https://github.com/dumpling-software/mailcheap/blob/8db6785ad90d05a1372d14706828eb58e9052796/lib/sending.js)
- [Mailcheap database schema](https://github.com/dumpling-software/mailcheap/blob/8db6785ad90d05a1372d14706828eb58e9052796/db/schema.js)
- [Mailcheap automation runner](https://github.com/dumpling-software/mailcheap/blob/8db6785ad90d05a1372d14706828eb58e9052796/lib/automations/run.js)
- [Mailcheap feedback processing](https://github.com/dumpling-software/mailcheap/blob/8db6785ad90d05a1372d14706828eb58e9052796/lib/ses-events.js)
- [SDK Cloudflare adapter](https://github.com/opencoredev/email-sdk/blob/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/src/cloudflare.ts)
- [SDK SES adapter](https://github.com/opencoredev/email-sdk/blob/2f89fbc34630e75685a87f2497cff36378827970/packages/email-sdk/src/ses.ts)
- [Cloudflare sending API](https://developers.cloudflare.com/api/resources/email_sending/methods/send/)
- [Cloudflare delivery events](https://developers.cloudflare.com/email-service/platform/event-subscriptions/)
