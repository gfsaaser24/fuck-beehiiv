# fuck-beehiiv

A practical build spec for running your own newsletter on Cloudflare, based on [OpenNewsletter](https://github.com/clawnify/OpenNewsletter).

**[Read the build spec](Newsletter_build_spec.md)**

The spec gives a developer or coding agent 8 tasks with completion checks. It covers subscriber imports and scheduled campaigns, including unsubscribe handling and recovery from interrupted sends.

Cloudflare runs the application and stores the subscriber list. The initial implementation uses Resend for email delivery.

Use the spec to fork the pinned OpenNewsletter version and implement the tasks in order. Start with a controlled pilot and complete the sending checks before moving a full list.

This repository contains an implementation plan. The proposed deployment has not been built or live-tested. The source project retains its own MIT license and attribution.
