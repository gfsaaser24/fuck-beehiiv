# fuck-beehiiv

Practical specs for running your own newsletter with Listmonk and self-hosted Postgres. Both use Cloudflare for DNS and protected admin access. Deploy the app on a VPS with Docker Compose or Coolify.

| Email delivery | Build spec |
|---|---|
| Amazon SES | [Listmonk + SES](Listmonk_SES_spec.md) |
| Cloudflare Email Service | [Listmonk + Cloudflare Email](Listmonk_Cloudflare_spec.md) |

Each spec has eight implementation tasks with completion checks. They cover imports and scheduled campaigns, including unsubscribe handling, suppression feedback and interrupted-send review.

The Cloudflare Email version adds a Worker that feeds delivery problems back into Listmonk. The SES version uses Listmonk's existing SNS integration with a feedback reliability patch.

These are implementation plans. The proposed deployments have not been built or live-tested. Start with staging and a controlled pilot before moving a full list. A multi-step welcome sequence is separate follow-up work.

Based on [Listmonk v6.2.0](https://github.com/knadh/listmonk/tree/v6.2.0). Preserve its AGPL-3.0 license and attribution.
