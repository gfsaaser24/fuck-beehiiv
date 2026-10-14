# Newsletter Build Spec

Build a self-hosted newsletter app for a single publication. Operators should be able to prepare an issue and schedule it without touching code. Confirm the expected subscriber count and sending volume during setup.

**Fork [clawnify/OpenNewsletter](https://github.com/clawnify/OpenNewsletter).** Start at commit `0fa2ba87e30ef941259b36845f2640dd7b3f1f15`. Preserve its MIT license.

The existing editor supports branded templates and personalized previews. Subscribers live in D1. The sending engine has separate provider adapters and tests covering failed sends. These boundaries make it a practical foundation. Its current deployment depends on Clawnify services; replacing those services is part of this project.

Source reviewed on October 14, 2026. This is an implementation spec; the fork has not been deployed or tested with a live subscriber list.

**The first version serves one publication.** Keep the existing editor and audience lists. Include signup forms, CSV imports, scheduled campaigns, reporting and a welcome sequence. AI writing remains optional. Finish a controlled sending pilot before moving the full list.

| Job | Use |
|---|---|
| Website and API | Existing React/Vite UI and Hono API on Cloudflare Workers |
| Subscriber records and send history | Cloudflare D1 |
| Uploaded newsletter images | Cloudflare R2 |
| Background sending | Cloudflare Queues, with Cron checking due work every minute |
| Team login | Cloudflare Access |
| Email delivery | Existing Resend adapter |

Cloudflare runs the application. Resend delivers the messages. Keep the provider interface intact so delivery can be changed later.

**1. Make the fork run independently.**

Replace the `@clawnify/db` wrapper with native D1 access while preserving existing SQL helpers. Replace Clawnify credential lookup with Worker secrets. Remove the platform-specific app integration and hide its CRM connection controls.

Configure Wrangler for static assets and the bindings above. Move request-time schema changes into numbered migrations. Supply an environment example and working setup/deploy commands. Verify that ordinary editing works without an AI key.

Done when a fresh install loads the editor and saves a draft using only the publication operator's accounts.

**2. Protect administration.**

Require Cloudflare Access for administration and verify its signed JWT in the Worker, including audience and expiry. Apply the same checks on alternate hostnames. Protect every administrative API by default.

Keep explicit public routes for signup and confirmation, plus unsubscribe and the signed Resend webhook. Public image access must not expose subscriber data. Remove the old public Clawnify job callbacks when native queue handlers replace them. Keep credentials out of browser responses.

Done when an anonymous request cannot read contacts or trigger a send.

**3. Wire dependable background jobs.**

Use the existing D1 campaign and flow records as the source of truth. Save work before enqueueing it. Cron finds due campaigns and welcome steps; it also recovers unfinished work when enqueueing failed.

Queue messages carry record IDs. A consumer calls the existing sending functions in bounded runs. Start with one consumer at a time and respect Resend's rate limits. Keep the current schedule checks so a cancelled or rescheduled job cannot send at its old time.

Done when a campaign continues after the browser closes or a Worker restarts.

**4. Preserve the sending safeguards.**

Keep the frozen campaign content and recipient records. Retain stable provider idempotency keys, which let a repeated request refer to the original send. Check that a worker still owns its database claim, its reservation to process that batch, immediately before calling Resend; stop if ownership was lost.

Resend remembers these keys for 24 hours. Preserve the existing 23-hour cutoff for uncertain sends. Mark unresolved outcomes “needs review” and stop automatic retries. A missing response must never cause an automatic retry under a fresh key.

Add a persistent pause switch checked before each new batch. An already submitted batch may still arrive. Recheck subscription status before the first submission; uncertain batches need reconciliation without changing their original payload.

Done when duplicated jobs and simulated timeouts produce no duplicate emails in the recovery tests.

**5. Import the list without changing people's choices.**

Add configurable CSV column mapping using the existing importer, including support for beehiiv exports. Preserve consent records and suppression reasons. Normalize addresses and deduplicate. Reimporting must never reactivate an opted-out or suppressed address.

Keep imported subscribers out of welcome automations by default. Use the existing audiences for basic segmentation. Keep double opt-in for new public signups, with rate limits and Cloudflare Turnstile verified on the server.

Done when a mixed sample import preserves every subscriber's intended status and sends no welcome messages.

**6. Finish the everyday newsletter workflow.**

Apply the publication's configurable branding to one reusable template. Keep subject and preview text editable. Verify image uploads, mobile layout, test sends and personalization fallbacks. Reuse the existing HTML sanitization. Include a visible unsubscribe link and the publication's mailing address.

Display schedule times in a configurable publication timezone, storing UTC internally. Show the selected audience and eligible recipient count before the operator commits a campaign. Add plain-text email bodies to the provider adapter if missing.

Done when a team member can prepare an issue and schedule it without developer help.

**7. Connect feedback and reporting.**

Verify signatures on Resend webhooks, the delivery feedback sent to the app, and deduplicate event IDs. Persist hard-bounce and complaint suppressions, and apply them to future sends. Preserve one-click unsubscribe headers. A GET visit to the unsubscribe page must only display the form; POST performs the unsubscribe.

Keep campaign totals and subscriber-level delivery history. Label provider acceptance separately from delivery. Explain that delivery does not prove inbox placement, and that opens or clicks can include automated activity.

Configure SPF/DKIM and aligned DMARC on the sending domain. Check the resulting headers in received test messages.

Done when a complaint, hard bounce or unsubscribe prevents the next eligible send.

**8. Prove recovery and hand it over.**

Retain the upstream test suite. Add focused tests for the new authentication and job adapters. Rehearse the failures below using a fake delivery provider, then make a small live send to engaged subscribers.

Provide a short operator guide covering campaign preparation, the pause control, failed-send review and restoring a backup. Export D1 daily to a private R2 location with defined retention; verify one restore into staging. Assign an owner for failed-job alerts.

Done when the tests pass and the operator can recover an interrupted campaign.

The implementation is ready for a pilot when:

- Unauthorized admin requests fail, including through an alternate hostname.
- Importing the same CSV twice preserves suppressions and does not trigger welcomes.
- Cancelling or rescheduling a campaign prevents its stale job from sending.
- Restarting a send or replaying a queue message does not duplicate delivery.
- An expired uncertain send remains stopped for review.
- A signed bounce/complaint event and a one-click unsubscribe prevent future sends.
- A branded issue renders acceptably in Gmail and Outlook; personalized links reach the correct subscriber action.
- A backup restores successfully into staging.

| Existing code | Agent guidance |
|---|---|
| `src/client/`, `src/shared/` | Keep the editor and templates; make targeted UI changes. |
| `src/server/sending.ts`, `providers/` | Preserve delivery behavior and repair claim ownership checks. |
| `contacts.ts`, `importer.ts` | Extend CSV mapping without weakening consent rules. |
| `flows.ts`, `schedule.ts` | Keep the existing flow state and stale-job guards. |
| `db.ts`, `index.ts`, `wrangler.toml` | Replace platform bindings and extract job handlers into small modules. |
| Existing `*.test.ts` files | Keep coverage; add tests around changed behavior. |

Build in the numbered order. A weekend target is a working staging deployment with a controlled pilot. Move the full list after its sending and recovery checks pass.

Implementation references: [pinned source](https://github.com/clawnify/OpenNewsletter/tree/0fa2ba87e30ef941259b36845f2640dd7b3f1f15), [Cloudflare queue delivery guarantees](https://developers.cloudflare.com/queues/reference/delivery-guarantees/), [Access JWT validation](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/authorization-cookie/validating-json/), [Resend idempotency](https://resend.com/docs/dashboard/emails/idempotency-keys).

