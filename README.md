# fuck-beehiiv

Practical specs for running a newsletter with Listmonk, self-hosted Postgres and [Email SDK](https://email-sdk.dev/). Deploy the app and a small Node.js delivery service on a VPS through Docker Compose or Coolify. Both paths use Cloudflare for DNS and protected admin access.

| Email provider | Build spec |
|---|---|
| Amazon SES | [Listmonk + Email SDK + SES](Listmonk_SES_spec.md) |
| Cloudflare Email Service | [Listmonk + Email SDK + Cloudflare Email](Listmonk_Cloudflare_spec.md) |

Listmonk provides the newsletter interface and subscriber management. The delivery service uses the selected SDK adapter over HTTP. Postgres stores the send ledger and feedback needed to recover interrupted work.

The specs reference the [full Email SDK documentation](https://email-sdk.dev/llms-full.txt) and pinned source.

Each spec contains eight tasks with completion checks, including the Listmonk handoff patch and one-click unsubscribe preservation. Uncertain sends stop for review. Provider feedback updates suppression in both systems.

These are implementation plans. No deployment or live-list test has been completed. Target staging first, then a controlled pilot. Multi-step welcome sequences are separate follow-up work.

Reviewed baselines: [Listmonk v6.2.0](https://github.com/knadh/listmonk/tree/v6.2.0) and [Email SDK `2f89fbc34630`](https://github.com/opencoredev/email-sdk/tree/2f89fbc34630e75685a87f2497cff36378827970). Preserve Listmonk's AGPL-3.0 and Email SDK's MIT license notices.
