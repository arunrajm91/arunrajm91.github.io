---
layout: post
title: "Rolling back a migration the day after cutover"
date: 2026-08-11 13:00:00 +0530
---

This year I cut over a self-hosted Bitbucket Data Center instance — roughly 350 repositories, 28,000 pull requests, 2,500 users — from version 7.x on MySQL to 9.x on PostgreSQL, on new infrastructure. The cutover went smoothly. The day after, a developer comparing a pull request against their local history found commits "missing" on the new server, and within hours I had made the call to roll back to the old one.

Success stories teach less than they claim to. This is the note I wish I had read before starting.

## What actually went wrong

The developer's report looked like data loss. It wasn't — and the distinction is the technical heart of this post-mortem.

Investigation showed that on at least two repositories, a long-lived branch ref on the new server was **stale**: one `dev` branch pointed 315 commits behind the old server's `dev`, another 50 behind. Crucially, every "missing" commit **object** existed in the new server's repository — `git cat-file -t <hash>` returned `commit` for every one. The objects had been transferred; the branch *pointer* had never been fast-forwarded during a late sync. Git's object store and its refs are separate things, and our verification had effectively checked the first while assuming the second.

That distinction decided the response. With object-level loss, you are in disaster recovery. With stale refs, you have a bounded, mechanically fixable problem — but you don't yet know *how many* branches are affected across 350 repositories, and users are actively pushing to a server whose refs you can't yet trust. Unknown scope on a live production system is a rollback trigger, not a debug-in-place situation.

## Why the rollback was cheap

The rollback took under a day, lost no data, and consisted entirely of reversing deliberate decisions made earlier:

- **The old server was frozen, not decommissioned.** At cutover it was stopped intact. Rollback step one was `systemctl start`.
- **Cutover was DNS-only.** Nothing about the old environment was rewired. Rollback step two was pointing the record back.
- **The gap was enumerable.** The freeze gave a precise timestamp; everything created only on the new server since then — a handful of PRs and commits — was listed, per author, in an announcement so people could redo their work. Painful, small, and *known*.
- **The new server stayed up in parallel** on a secondary hostname for reconciliation, rather than being torn down in frustration.

None of this was luck. "How do we undo this?" was designed as a first-class requirement of the cutover, and it is the only reason the story is an anecdote instead of an incident report.

## The root cause was the approach, not a bug

The stale refs were the trigger, but the honest root cause sat further back. This migration crossed four major versions and swapped database engines, and it was executed as a **hand-rolled ETL** — raw table copies and custom scripts — rather than the vendor's export/import path.

The schema had evolved for years across those versions. The raw copy silently missed entire families of tables: hundreds of thousands of comment and activity rows, plugin tables whose ID sequences were never advanced (causing primary-key collisions on the first write), permission exemptions, SSH keys. Each gap was found reactively, root-caused, and backfilled — weeks of genuinely difficult forensic work that I'm proud of and that should never have existed. Every one of those fixes was treating a symptom of the same decision.

A hand-rolled ETL turns one migration project into N independent data-completeness projects, where N is unknown and every miss is silent until a user hits it. Vendor tooling is slower and stricter precisely because it carries the schema knowledge you don't have.

## What I do differently now

1. **Verify refs, not just objects.** For any git migration: `git for-each-ref` on both sides, per repository, with `git merge-base --is-ancestor` to prove every branch on the target is at or ahead of the source. It is read-only, scriptable, and cheap. Sampling a few important repos is not verification; it is hope.
2. **Native tooling first; ETL only with an argument in writing** for why the vendor path cannot work — and treating that argument as a warning sign about the migration itself.
3. **Design the rollback before the cutover.** Freeze rather than destroy, cut over at the routing layer, keep the timestamp. The measure of a migration plan is the cost of being wrong.
4. **Roll back on unknown scope.** The instinct after months of work is to patch forward. But debugging on a live system whose integrity is in question compounds the risk with every user interaction. Rolling back bought unlimited time to sweep all 350 repositories properly — the sweep and re-cutover proceed with the pressure off.

The hardest part was none of the git internals. It was announcing, one day after announcing success, that we were going back — listing exactly whose work needed redoing. That announcement was received far better than a week of "investigating intermittent issues" would have been. Reversing a bad position early is the cheapest it will ever be.

---

*This note also lives in my [engineering-notes](https://github.com/arunrajm91/engineering-notes) repository, alongside the code it references.*
