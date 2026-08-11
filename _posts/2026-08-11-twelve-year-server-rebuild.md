---
layout: post
title: "Rebuilding a server that ran for twelve years"
date: 2026-08-11 10:00:00 +0530
---

A production API server on Ubuntu 12.04 — an OS that went EOL in 2017 — was still serving real traffic in 2026. It finally had to move: its OpenSSL 1.0.1 was old enough that modern TLS clients were failing to connect, it still accepted TLS 1.0, and the workload needed to relocate to a closer region anyway. This is how you move twelve years of accumulated production without an outage, and what the process teaches you about the server you thought you knew.

## Rebuild, never upgrade

Ubuntu 12.04 to 24.04 is six LTS releases. `do-release-upgrade` chained six times across a decade of packaging changes is a machine no one can reason about — and worse, it faithfully preserves the sediment: every orphaned package, hand-edited config, and forgotten daemon comes along. The only honest path is a fresh 24.04 build where **everything present is present because someone put it there this year**, with the old server as the reference implementation.

The rebuild is also the only moment you can pay down a decade of adjacent debt at zero marginal risk. In the same move: MongoDB went 3.2 → 8.0 (4.7 million documents migrated, zero failures), Apache was replaced with NGINX — all 28 proxy routes converted — TLS floor raised to 1.2/1.3, and an API gateway product that had been abandoned-in-place was decommissioned entirely, with routes going straight to the services. None of those would ever have justified their own risk window on the old machine. Piggybacked on a parallel rebuild, their incremental risk was near zero.

## The archaeology is the actual work

Copying data is the easy 20%. The real work is discovering what the server *is*, because after twelve years, documentation is fiction and the truth lives in process tables and crontabs.

Enumerating the ~33 backend services produced the finding that shaped the whole migration: **several were already dead on the old server** — routes returning 503 in production, and evidently for a long time, because nobody had complained. Without that enumeration, the new server would have been blamed for every one of those failures on cutover day. The parallel build gives you a baseline audit for free, and the rule that falls out of it:

> Migrate what runs. Document what's broken. Never silently "fix" during a migration.

Reviving a long-dead service during cutover is scope creep with the worst possible timing — if it's down and nobody noticed, that revival is a *product* decision. One gateway component turned out to be running but absent from every config and monitoring manifest — the kind of load-bearing stowaway that only turns up when you enumerate reality instead of trusting the docs. It was found, migrated, and this time, monitored.

## Cutover as a checklist, not an event

The pattern that makes this boring (the goal) is parallel-run with a gated DNS flip:

1. Build and data-load the new server while the old one serves everything.
2. Test every route against the new server's IP directly — before DNS is involved.
3. Drop DNS TTL to 60 seconds a day ahead, so the flip (and any retreat) propagates in minutes.
4. Final delta sync of the datastores, flip, and keep the old server intact until confidence is earned.

The pre-cutover checklist had items like "confirm the firewall rule for the two services calling the external database" and "confirm which service owns the one route nobody can explain." Every one was discovered by the audit, written down, and owned. A migration that runs on a checklist rather than adrenaline is the difference between an event and a non-event — and this one needed to be a non-event, because the machine had spent twelve years becoming load-bearing in ways nobody fully remembered.

## Why servers get like this

Nobody chooses to run 12.04 in 2026. It happens one deferred quarter at a time, because the server keeps working and migration is never urgent — until a TLS handshake fails against a modern client, and the years of deferred risk all invoice at once. The rebuild took weeks. The uncomfortable arithmetic is that it would have taken days when it first became due, and the gap between those two numbers is what "we'll do it next quarter" actually costs.

---

*This note also lives in my [engineering-notes](https://github.com/arunrajm91/engineering-notes) repository, alongside the code it references.*
