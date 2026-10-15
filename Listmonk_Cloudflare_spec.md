# Newsletter spec: Listmonk + Cloudflare Email

Build a self-hosted newsletter for one publication. An operator should be able to import subscribers and schedule an issue without touching code. Configure the publication name, timezone and expected sending volume during setup.

Use [Listmonk v6.2.0](https://github.com/knadh/listmonk/tree/v6.2.0) as the reviewed baseline. Pin the deployed image digest and record any patches. Preserve Listmonk's AGPL-3.0 license and attribution. Keep changes small and use its existing UI and API.

This is an implementation plan. No deployment or live-list testing has been completed. A weekend target is a working staging installation and a controlled pilot. Full migration follows the checks below.

Cloudflare provides email delivery and protected access. Listmonk and Postgres run on your VPS. A small Worker connects Cloudflare delivery feedback to Listmonk.

| Job | Use |
|---|---|
| Newsletter application | Listmonk in Docker, deployed with Coolify or Docker Compose |
| Database | Self-hosted Postgres 16, using Listmonk's schema |
| Public HTTPS and admin access | Cloudflare DNS, Tunnel or secured origin, and Access |
| Campaign scheduling and sending jobs | Listmonk's built-in manager |
| Email delivery | Cloudflare Email Service over SMTP |
| Bounce and complaint feedback | Cloudflare event subscription, Queue and consumer Worker |
| Uploaded images | Persistent Listmonk media volume |
| Off-server backups | Private Cloudflare R2 bucket |

**1. Deploy the existing application.**

Deploy one Listmonk instance with Postgres on a private Docker network. Keep the database off the public internet. Run the release's installation/migration command as a controlled step. Persist database and media volumes. Configure health checks and logs.

Use Postgres directly. A self-hosted Supabase Postgres instance is an option if already operated, but Supabase Auth and a database conversion are unnecessary. Keep Listmonk's schema and migration tools. The application runs on the VPS; the feedback adapter runs on Workers.

Done when a fresh installation survives a container restart with a saved draft and subscriber intact.

**2. Protect administration and keep subscriber links working.**

Put administrative pages and APIs behind Cloudflare Access, retaining Listmonk's own authentication. Use a separate public hostname or explicit route rules for signup, confirmation, unsubscribe, tracking and public images. Inspect the pinned release's routes before writing those rules. Set the canonical public URL so emails contain working subscriber links.

Use Cloudflare Tunnel or an origin firewall with authenticated ingress to prevent bypassing Access through the server IP. Protect machine endpoints with service credentials. Keep secrets out of Git and browser responses.

Done when anonymous users cannot read contacts or start campaigns, while subscriber links work without an administrator login.

**3. Connect Cloudflare SMTP.**

Enable Email Sending and onboard the sending domain. Verify the resulting SPF/DKIM records and aligned DMARC. Store an API token with Email Sending: Edit permission as a deployment secret.

Configure Listmonk SMTP:

| Setting | Value |
|---|---|
| Host | `smtp.mx.cloudflare.net` |
| Port | `465` |
| Security | Implicit TLS; select Listmonk's TLS mode |
| Authentication | PLAIN |
| Username | `api_token` |
| Password | Cloudflare API token |

Confirm the account's daily quota can accommodate a campaign and set a conservative send rate. Cloudflare's quota is account-specific. Send one personalized message per subscriber. Use Listmonk's scheduler; Cloudflare Queues are for feedback in this spec.

Done when a test campaign arrives with valid authentication and working unsubscribe headers.

**4. Build the feedback adapter.**

Subscribe the sending domain to Cloudflare Email Sending events through a Queue. Deploy a consumer Worker and a dead-letter queue. Process `message.complained` and `message.bounced`; inspect `payload.bounce.type` so a soft bounce is not treated as a hard bounce. A `message.deferred` event is not a permanent suppression.

Map events to Listmonk's authenticated `POST /webhooks/bounce` endpoint using recipient email, source `cloudflare`, type `hard`, `soft` or `complaint`, and metadata containing the Cloudflare event ID. Configure first-event blocklisting for hard bounces and complaints. Authenticate the Worker through Access service credentials and Listmonk API credentials. Keep this endpoint inaccessible to anonymous callers.

Add receiver-side deduplication using the Cloudflare event ID and durable processing state in Postgres. Acknowledge the Queue message only after the receiver confirms persistence. Patch Listmonk's handler to return a retryable error on database failure; the reviewed handler can return HTTP 200 after a failed write. Ensure a repeated or partially processed event cannot inflate counters. Retry transient failures and alert on dead-letter messages.

Cloudflare suppresses eligible addresses itself. Still update Listmonk so its audience selection and reporting reflect those suppressions. Do not invent campaign attribution: the documented events do not provide Listmonk's campaign UUID. Account-level suppression works by recipient email; per-campaign attribution requires a separately verified message-ID mapping.

Done when a hard bounce or complaint updates Listmonk and prevents the next eligible send, duplicate events are harmless, and an outage is recovered by replaying failed events.

**5. Import subscribers without changing their choices.**

Use the existing importer. Supply a small CSV conversion script only where the export needs column mapping. Support email, name, list membership and subscription status; retain available consent dates and source information in subscriber attributes.

Normalize and deduplicate addresses. Import suppressions before active subscribers. Preserve list-level unsubscribes and globally blocklisted addresses on every reimport. Disable any overwrite option that could reactivate them. Test the same mixed CSV twice. Imported contacts must not receive signup or welcome emails.

Use Listmonk's signup form and double opt-in for new public subscribers. Rate-limit the public signup endpoint and verify repeated submissions do not flood an address with confirmation emails.

Done when the imported counts reconcile, prior opt-outs remain excluded and new signups need confirmation.

**6. Finish the everyday newsletter workflow.**

Create one reusable single-column template with optional images and a button. Configure the sender and reply-to address. Include a visible unsubscribe link and mailing address. Enable one-click unsubscribe headers and verify the POST endpoint works without login.

Check subject, preview text, plain-text alternative, personalization fallbacks and test sends in the selected editor. Verify scheduled times against the operator's timezone and store dates consistently. Show the chosen lists and eligible recipient count before sending.

Keep Listmonk's reporting. Label SMTP acceptance accurately: it does not prove delivery or inbox placement. Opens and clicks can include automated activity. Record any unavailable provider detail separately instead of inventing metrics.

Done when an operator can prepare an issue and schedule it, and the received email renders acceptably in Gmail and Outlook.

**7. Make interruptions visible and stop unsafe restarts.**

The reviewed Listmonk recipient query advances its checkpoint when it fetches a batch. A crash can leave unfinished recipients behind that checkpoint. SMTP also has an uncertain outcome when the connection fails after submission. Do not promise exactly-once delivery.

Add a startup guard that places previously running campaigns into a paused/review state before the sending manager starts. Keep scheduled campaigns intact. Pause active campaigns before planned deployments. Disable automatic failover or replay of an uncertain campaign through another provider.

Keep campaign logs and available provider evidence. Provide a review procedure for identifying unresolved recipients. Never replay the whole campaign merely because its totals look wrong. If individual outcomes cannot be established, leave them unresolved for an explicit operator decision. A durable per-recipient send ledger is additional engineering if automatic recovery is required.

Use a fake SMTP server to test process termination during a batch, rejection and an accepted message followed by a lost response. Confirm the campaign stays stopped after restart and the pause control stops new dispatches; already submitted mail may still arrive.

Done when an interruption cannot silently resume sending and the operator can identify which campaign needs review.

**8. Back up, run the pilot and hand over.**

Back up Postgres daily and before upgrades to a private R2 bucket with defined retention. Back up uploaded media and document secret recovery separately. Restore into staging with outbound sending disabled. Never restore an old backup straight into an active sender; reconcile newer opt-outs before allowing sends.

Test the changed code and integrations, then send to a small consented pilot audience. Confirm authentication headers and actual inbox placement. Monitor failures and suppression processing. Record the results before moving the full list.

Deliver a Docker Compose file suitable for Coolify, environment example, setup commands and a short operator guide. Name an owner for failed sends, event failures and backup alerts.

Done when a restore succeeds and the operator can run a campaign and handle an interruption.

## Scope for the first version

Include subscriber management, signup confirmation, imports, scheduled broadcasts, templates, basic segmentation, unsubscribe handling and reporting.

A multi-step welcome sequence is not assumed to exist in the reviewed Listmonk release. Treat it as a separate follow-up integration using the API, with its own suppression and retry checks. Do not advertise it as completed. Keep optional AI writing and advanced automation outside this build.

## Pilot acceptance

- Anonymous administration and direct-origin bypass attempts fail.
- Reimporting a mixed CSV preserves all opt-outs and blocklisted addresses.
- A cancelled or rescheduled campaign does not send at its old time.
- Received email has valid authentication and working one-click unsubscribe.
- Cloudflare hard-bounce and complaint events block future sends in Listmonk.
- Soft deferrals do not permanently block subscribers.
- Duplicate events and a temporary receiver outage are handled correctly.
- An interrupted campaign stays paused after restart.
- A backup restores with sending disabled.

## Agent guidance

Deploy upstream first. Add a small Worker feedback adapter and the required receiver/startup safeguards. Preserve Listmonk's editor and campaign manager. Keep adapter code and deployment configuration in separate directories. Record image digests and patches.

Do not assume SMTP provides provider-side idempotency or claim duplicate-free automatic recovery.

Reviewed October 14, 2026.

- [Listmonk sending manager](https://github.com/knadh/listmonk/blob/v6.2.0/internal/manager/manager.go)
- [Recipient checkpoints](https://github.com/knadh/listmonk/blob/v6.2.0/queries/campaigns.sql)
- [Bounce handler](https://github.com/knadh/listmonk/blob/v6.2.0/cmd/bounce.go)
- [Bounce configuration and webhook format](https://github.com/knadh/listmonk/blob/v6.2.0/docs/docs/content/bounces.md)
- [Cloudflare SMTP](https://developers.cloudflare.com/email-service/api/send-emails/smtp/)
- [Cloudflare delivery events](https://developers.cloudflare.com/email-service/platform/event-subscriptions/)
- [Recipient synchronization](https://developers.cloudflare.com/email-service/examples/email-sending/sync-recipient-records/)
- [Account sending limits](https://developers.cloudflare.com/email-service/platform/limits/)
