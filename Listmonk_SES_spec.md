# Newsletter spec: Listmonk + SES

Build a self-hosted newsletter for one publication. An operator should be able to import subscribers and schedule an issue without touching code. Configure the publication name, timezone and expected sending volume during setup.

Use [Listmonk v6.2.0](https://github.com/knadh/listmonk/tree/v6.2.0) as the reviewed baseline. Pin the deployed image digest and record any patches. Preserve Listmonk's AGPL-3.0 license and attribution. Keep changes small and use its existing UI and API.

This is an implementation plan. No deployment or live-list testing has been completed. A weekend target is a working staging installation and a controlled pilot. Full migration follows the checks below.

Cloudflare provides DNS and protected access. Listmonk and Postgres run on your VPS. Amazon SES delivers the email.

| Job | Use |
|---|---|
| Newsletter application | Listmonk in Docker, deployed with Coolify or Docker Compose |
| Database | Self-hosted Postgres 16, using Listmonk's schema |
| Public HTTPS and admin access | Cloudflare DNS, Tunnel or secured origin, and Access |
| Campaign scheduling and sending jobs | Listmonk's built-in manager |
| Email delivery | Amazon SES SMTP |
| Bounce and complaint feedback | SES identity notifications through SNS to Listmonk |
| Uploaded images | Persistent Listmonk media volume |
| Off-server backups | Private Cloudflare R2 bucket |

**1. Deploy the existing application.**

Deploy one Listmonk instance with Postgres on a private Docker network. Keep the database off the public internet. Run the release's installation/migration command as a controlled step. Persist database and media volumes. Configure health checks and logs.

Use Postgres directly. A self-hosted Supabase Postgres instance is an option if already operated, but Supabase Auth and a database conversion are unnecessary. Keep Listmonk's schema and migration tools.

Done when a fresh installation survives a container restart with a saved draft and subscriber intact.

**2. Protect administration and keep subscriber links working.**

Put administrative pages and APIs behind Cloudflare Access, retaining Listmonk's own authentication. Use a separate public hostname or explicit route rules for signup, confirmation, unsubscribe, tracking and public images. Inspect the pinned release's routes before writing those rules. Set the canonical public URL so emails contain working subscriber links.

Use Cloudflare Tunnel or an origin firewall with authenticated ingress to prevent bypassing Access through the server IP. Protect machine endpoints with service credentials. Keep secrets out of Git and browser responses.

Done when anonymous users cannot read contacts or start campaigns, while subscriber links work without an administrator login.

**3. Connect SES SMTP.**

Verify a sending domain in the chosen AWS region. Configure DKIM and an aligned SPF/DMARC setup; use a custom MAIL FROM domain where appropriate. Create scoped SES SMTP credentials and store them as deployment secrets. Configure TLS and a send rate below the account's allowed rate.

Confirm production sending access and enough daily quota for the expected campaign size. Send one personalized email per subscriber. Use Listmonk's scheduler; no additional campaign queue is required.

Done when a test campaign arrives with the expected authentication and working unsubscribe headers.

**4. Connect bounce and complaint feedback reliably.**

Enable Listmonk bounce processing and the SES webhook at `/webhooks/service/ses`. Configure SES identity feedback notifications for bounces and complaints through SNS, with original headers included and raw message delivery disabled. Use this documented notification shape: the reviewed handler does not treat every configuration-set event shape interchangeably.

Allow SNS to reach this specific endpoint without an interactive Access login. Retain SNS signature verification. Set hard bounces and complaints to blocklist after the first event; temporary failures must not immediately blocklist a subscriber.

Patch the reviewed handler so a failed bounce database write returns a retryable error instead of HTTP 200. Deduplicate notifications by SNS MessageId, with durable processing state that marks completion only after the effects succeed. A retry after a partial failure must not inflate bounce counts. Verify receiver-side idempotency with repeated notifications.

Done when simulated hard bounces and complaints block future sends, invalid signatures fail, and a temporary database failure is retried successfully.

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
- SES bounce and complaint notifications prevent the next eligible send.
- Duplicate feedback does not inflate counts; database failures cause retries.
- An interrupted campaign stays paused after restart.
- A backup restores with sending disabled.

## Agent guidance

Deploy upstream first. Add only the startup guard and the necessary feedback reliability patch. Do not rebuild the editor or move campaign processing into Workers. Record patches and image digests so upgrades remain reviewable.

Reviewed October 14, 2026.

- [Listmonk sending manager](https://github.com/knadh/listmonk/blob/v6.2.0/internal/manager/manager.go)
- [Recipient checkpoints](https://github.com/knadh/listmonk/blob/v6.2.0/queries/campaigns.sql)
- [Bounce handler](https://github.com/knadh/listmonk/blob/v6.2.0/cmd/bounce.go)
- [Bounce configuration and webhook format](https://github.com/knadh/listmonk/blob/v6.2.0/docs/docs/content/bounces.md)
