---
layout: post
title: "Publishing a website from a server with zero open inbound ports"
date: 2026-08-26 12:00:00 +0530
---

I recently put a product holding page live — static HTML, no JavaScript, served by nginx in a container — from a server whose firewall accepts **no inbound connections at all**. No port 443, no port 80, nothing. The server dials out; the internet never dials in. This is a short field report on the pattern, plus one genuinely nasty nginx bug it surfaced.

## The shape

The mechanism is an outbound tunnel: a lightweight daemon (`cloudflared`, in this case) runs next to the web container, opens a persistent outbound connection to the edge provider, and the provider routes public hostnames down that tunnel. The pieces:

- A static site in a container — nginx serving files, joined to a shared Docker network.
- The tunnel daemon on the same network, with ingress rules mapping `example.com` and `www` to `http://landing:80` by container name.
- DNS records at the provider pointing the hostnames at the tunnel.

What this buys, concretely:

- **No inbound attack surface.** There is no listening socket for the internet to probe. Every scanner that would have been rattling 443's doorknob is now Cloudflare's problem, not nginx's.
- **No certificate handling on the origin** for the public edge — TLS terminates at the provider. (Origin-side TLS between edge and tunnel is available when the threat model wants it.)
- **Cutover without touching DNS.** This deployment is a *holding page* for a platform still being built. Launch day is: repoint the tunnel's ingress rule from `http://landing:80` to the real app's container and port — same tunnel, same DNS, propagation delay zero, rollback equally instant. Cutover and rollback as a config field, not a DNS event, is quietly the best feature here.

The honest caveats: you've made the edge provider a hard dependency and given it TLS visibility — an acceptable trade for a public brochure page, a real conversation for anything sensitive. And because your origin is only reachable through the provider, local testing means `curl` against the container directly; keep that habit or you'll debug the edge when the bug is at home.

## The bug worth the price of admission

The page shipped with the usual security headers — CSP, `X-Content-Type-Options`, referrer policy — declared once at the `server` level of the nginx config. Standard. Then I added per-path caching, which meant a `location` block with its own `add_header Cache-Control ...`.

Verification with `curl -sI` showed the security headers had **vanished from every response the moment that location block matched** — silently, with a valid config and no warning.

This is documented-but-notorious nginx behavior: **`add_header` directives are inherited from the enclosing level *only if* the current level defines none of its own.** One `add_header` in a `location` block doesn't *add* to the server-level set; it *replaces the entire inherited set* with itself. The header you added for caching quietly deleted your security posture.

The fix is structural, not clever: extract the security headers into a `security-headers.conf` snippet and `include` it explicitly in the `server` block *and every `location` that declares any `add_header` of its own*. Inheritance you can't see is inheritance you can't trust; the `include` makes the full header set visible at every level that needs it.

Two habits fall out of this. First, **verify headers on real responses, not in the config** — `curl -sI` against every distinct location type (HTML page, static asset, redirect), because a config review will not catch this. Second, treat any new `add_header` in a `location` as a tripwire: it has just silently detached that location from every header declared above it. This bug also can't be caught by `nginx -t` — the config is *valid*; it's just not what you meant.

## Why a throwaway page got its own infrastructure

One more decision worth recording: the holding page lives in its own folder with its own Dockerfile and compose file, deliberately *outside* the main application's workspace and CI matrix. Putting a placeholder inside the real app's build pipeline couples a thing you'll delete in months to a thing you're actively building — every CI run, dependency bump, and build-matrix change would drag the placeholder along. Disposable things should be built disposably: standalone, boring, and deleted in one `docker compose down` when their day comes.

Total standing infrastructure for a public website: one static-file container, one tunnel daemon, zero open ports, zero certificates to renew on the origin. For anything that fits the "brochure" shape, this is the least infrastructure I know how to run a public site on.
