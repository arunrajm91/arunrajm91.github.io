---
layout: post
title: "Your dual-WAN load balancing may be causing the outages it should prevent"
date: 2026-09-09 12:00:00 +0530
---

An office network of 50+ users, a firewall with two WAN uplinks — a leased line and a broadband PPPoE link — and a complaint that arrived several times a week: *"the internet died again, but Wi-Fi shows full bars."* Everyone blamed the Wi-Fi. The access points got rebooted, the mesh units got moved around, and the complaints kept coming.

The firewall's event log told a different story, once I actually exported and counted it: hundreds of PPPoE session drop/recovery pairs on the broadband link, and roughly 250 failover events in the counters. The most recent outage was seventeen minutes, mid-morning, peak office hours. The Wi-Fi was innocent. The question worth writing up is why a *redundant* dual-WAN setup produced *more* user-visible outages than a single link would have.

## The misconfiguration that looks like a feature

The WAN group was in **ratio load-balancing mode, 70/30** across the two links. On paper that reads as "we're using both links we pay for." In practice it means: at any moment, roughly a third of active sessions live on the broadband link. When that link's PPPoE session flaps — and cheap broadband PPPoE flaps a lot — every session pinned to it dies. Users mid-call, mid-upload, mid-anything on that 30% see a dead internet. Then the session re-establishes, the balancer resumes sending new sessions to it, and the stage is set for the next flap.

Load balancing across an unstable link doesn't give you 170% capacity. It gives you a machine for distributing that link's instability across your whole user base, thirty percent at a time.

The second problem compounded the first: **link detection was physical-state only**. The balancer considered the broadband link "up" whenever PPP was up — no probing of whether traffic actually passed. A link can hold its PPP session while being effectively dead, and the balancer will happily keep assigning sessions to it.

## The fix was a downgrade, on purpose

The change I applied was, on the surface, a reduction in sophistication:

- **Basic failover with preempt** instead of ratio balancing — the leased line carries everything; the broadband link is a standby that takes over only on real failure and hands back when the primary recovers.
- **Logical probing on both members** — ping targets through each link (two independent anchors, either-responds semantics, tight intervals), so "up" means "traffic passes," not "PPP negotiated."

The trade-off is honest: the broadband link now sits idle almost all the time, and we pay for it anyway. That is the correct price. The leased line handles the full user load with headroom; the broadband link's actual value is *availability insurance*, and insurance you're actively depending on for capacity isn't insurance.

If the primary ever approaches saturation, the right next step is *policy-based routing* — deliberately pinning bulk, interruption-tolerant traffic (backups, update downloads) to the secondary — not returning to ratio balancing. Choose *which* sessions ride the unstable link, rather than letting the balancer choose randomly.

## You can't fix what you log over

Two side-findings from the diagnosis deserve their own paragraph, because both will be familiar to anyone who has debugged a firewall in anger:

First, the on-box event log was flooded by an application-control feature logging **thousands of filename events every couple of hours**, so log exports covered only hours of history. The evidence of a months-long problem barely survived in counters. Verbose logging of things nobody reads is not observability; it is a denial-of-service against the logs you'll need.

Second, "the internet is down" reports and per-link reality could not be correlated after the fact — so I stopped relying on the firewall's own logging entirely and deployed external monitoring: SNMP interface metrics and blackbox probes *through each WAN link separately* (policy-routed probe targets, so each link's health is measured independently), plus NetFlow into a flow analyzer. The next argument with the ISP will come with graphs attached.

## The general shape

This incident generalizes beyond one firewall vendor:

1. **Redundancy configured as load sharing couples your fate to your worst link.** Failover-with-preempt decouples it.
2. **Link state must be measured logically.** Physical/PPP "up" is a statement about the modem, not the internet.
3. **A flapping link degrades a balanced group more than a dead one.** Detection thresholds tuned for clean failures miss the flap pattern entirely.
4. **Instrument before you escalate.** An ISP ticket that says "it drops sometimes" goes nowhere. One that says "281 PPPoE re-establishments in this window, here's the per-minute graph" gets a different class of response.

The Wi-Fi complaints, incidentally, didn't fully stop — there's a genuine, separate RF problem in that building, which is what happens when five access points from two vendors share a floor without a channel plan. But that's a different post.
