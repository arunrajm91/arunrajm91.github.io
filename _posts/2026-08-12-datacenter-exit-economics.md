---
layout: post
title: "The economics of leaving a datacenter"
date: 2026-08-12 11:00:00 +0530
---

I was asked to look at two baremetal servers at a US hosting provider, billing about $675 a month between them, with a simple question attached: migrate or decommission? The audit that answers that question turned out to be far more valuable than the monthly saving — because nobody had looked at these machines, really looked, in years.

This note was written mid-project: the audit and plan are done, the migration is in flight. The findings and the economics stand on their own.

## What the audit found

The bill was the least alarming discovery.

**Server one** — nominally a Jira box:

- Ubuntu 14.04, EOL for years, kernel from 2013, uptime approaching **six years**. An uptime that long is not a stability brag; it means six years of kernel security patches never applied.
- All three Jira instances it existed to run were **down** — no Java process at all. The server's stated purpose had quietly stopped being true, and its $300+/month share of the bill bought Apache serving 80+ legacy vhosts and a MySQL 5.5 instance.
- That MySQL was **exposed to the public internet on 3306**, next to a public phpMyAdmin from 2013 and PHP versions EOL for a decade.
- Every database backup cron job was **commented out**. No backups had run in years.
- Multiple Let's Encrypt certificates: expired.

**Server two** — 1.8GB of RAM, deep into swap, doing two jobs that actually mattered:

- **Authoritative DNS for 50+ domains**, on a machine nobody was watching.
- **NFS server** exporting 1.8TB, of which roughly a third was old backup ZIPs (newest: three years old) and another ~200GB was archives from migrations past — the sediment layer every long-lived server accumulates.

This is the real risk profile of "cheap" legacy hosting: the money is measurable, but the concentration of single points of failure — authoritative DNS, the only file store, and an unbacked-up database on EOL operating systems — is not on any invoice.

## The economics

The replacement architecture prices out at roughly $100–150 a month:

| Component | Replaces | ~Monthly |
|---|---|---|
| Route 53 (50 zones) | Self-hosted authoritative DNS | $10 |
| Small EC2 instance | Apache vhost routing | $20 |
| RDS (small) | MySQL 5.5, *if* the data must live on | $25 |
| S3 (~2TB) | The NFS export | $46 |
| SES | Sendmail/Postfix | $2 |

Call it 75–80% savings — but the honest pitch to the stakeholders was never the $500. It was: *after* this, DNS has an SLA, the data is versioned in S3 instead of one disk at 81% capacity, backups exist, and nothing EOL faces the internet.

## Sequencing: DNS first, compute last

The plan runs in phases, and the ordering is the design:

1. **Backups and a traffic audit before anything moves.** You cannot decommission what you cannot restore, and the vhost audit decides what is worth moving at all — many of 80+ vhosts will turn out to serve nothing.
2. **DNS to Route 53 first.** Moving authoritative DNS is the scariest step, so it goes first, while both environments are fully alive and the change is trivially reversible. Migrating DNS *last*, under time pressure, with the old environment half-dismantled, is how outages happen.
3. **Data to S3 while both ends exist** — bulk transfer with no deadline, then a small delta at the end.
4. **Compute last**, because by then it is the only thing left, and the smallest.

## The general lesson

Long-lived infrastructure decays silently: purposes lapse (three dead Jiras), safety nets rot (commented-out backups), and exposure accumulates (public 3306). None of it announces itself, because decay produces no errors — only risk. The audit, not the migration, is the product. Even if the decision had been "stay on baremetal," we would have left with backups running, DNS documented, and a database off the public internet — which is to say, the audit would still have paid for itself.

---

*This note also lives in my [engineering-notes](https://github.com/arunrajm91/engineering-notes) repository, alongside the code it references.*
