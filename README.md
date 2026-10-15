# fuck-beehiiv

Practical specs for running a newsletter with self-hosted Postgres and [Email SDK](https://email-sdk.dev/). Deploy the application on a VPS through Docker Compose or Coolify, with Cloudflare for DNS and protected admin access.

| Foundation and delivery | Build spec |
|---|---|
| Listmonk + Amazon SES | [Listmonk + Email SDK + SES](Listmonk_SES_spec.md) |
| Listmonk + Cloudflare Email Service | [Listmonk + Email SDK + Cloudflare Email](Listmonk_Cloudflare_spec.md) |
| Dumpling's Mailcheap fork + SES or Cloudflare | [Dumpling Mailcheap + Email SDK](Dumpling_Mailcheap_Email_SDK_spec.md) |

The Listmonk paths add a small Node.js delivery service and a durable handoff patch. The Dumpling path integrates Email SDK directly into its existing Next.js app and repairs its sending state and recovery logic. Its first version keeps QStash for background jobs.

All three use Postgres to retain sending outcomes. Uncertain sends stop for review. Provider feedback updates suppression before another attempt.

Each spec contains eight tasks with completion checks and references the [full Email SDK documentation](https://email-sdk.dev/llms-full.txt) and pinned source.

These are implementation plans. No deployment or live-list test has been completed. Target staging first, then a controlled pilot. Dumpling already includes an automation engine; the Listmonk plans leave multi-step welcome sequences for follow-up work.

Reviewed baselines: [Listmonk v6.2.0](https://github.com/knadh/listmonk/tree/v6.2.0), [Dumpling Mailcheap `8db6785ad90d`](https://github.com/dumpling-software/mailcheap/tree/8db6785ad90d05a1372d14706828eb58e9052796) and [Email SDK `2f89fbc34630`](https://github.com/opencoredev/email-sdk/tree/2f89fbc34630e75685a87f2497cff36378827970). Preserve Listmonk's AGPL-3.0 notice and the reviewed Mailcheap and Email SDK MIT notices.
