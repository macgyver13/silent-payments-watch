# Silent Payments Watch

[Silent Payments Watch](https://silent-payments-watch.macgyver-dev.workers.dev/) identifies and tracks public activity related to Silent Payments adoption.

It covers [BIP-352](https://github.com/bitcoin/bips/blob/master/bip-0352.mediawiki) and directly related work: scanning, wallets, hardware, indexing, and BIPs 375/376/392.

Inclusion is a match against public sources, not an endorsement, technical review, security assessment, or a canonical roadmap.

## Using the site

Visit [the watch](https://silent-payments-watch.macgyver-dev.workers.dev/) when you want an update. Collection runs daily.

- **Activity** — chronological feed. Filter and sort by date (last activity, updated, merged, pushed, created) and by content type (repo, PR, docs, crate, topic).
- **Weeks** — what showed up each ISO week.
- **Projects** — tracked work grouped by project.
- **Sources** — the catalog of repos, PRs, docs, and other public sources.

Share the data. Other feeds and agents can pull RSS or JSON:

- https://silent-payments-watch.macgyver-dev.workers.dev/feed.xml
- https://silent-payments-watch.macgyver-dev.workers.dev/feed.json

## Contact

Errors or findings: [macgyver.sp@proton.me](mailto:macgyver.sp@proton.me), or join the Silent Payments Discord.

## How it works

This site is a [Source Watch](https://github.com/macgyver13/source-watch) instance. That repo documents the engine.

This watch’s inputs are the two instance configs:

- [`config/watch.yaml`](config/watch.yaml) — identity, relevance terms, and discovery floor.
- [`config/source-seeds.yaml`](config/source-seeds.yaml) — seeded repos, PRs, and docs, plus the live GitHub and Delving collectors that pick up new matches.
