+++
date = '2026-08-15T12:00:00+07:00'
draft = false
title = 'SearXNG: Self-Hosted Search for my Pi Harness'
description = "A self-hosted SearXNG instance on the homelab that gives the Pi Harness assistant reliable web search, aggregating 7 search engines through a single local API with DuckDuckGo as fallback."
image = "/images/helloworld.webp"
imageBig = "/images/helloworld.webp"
categories = ["projects"]
tags = ["self-hosting", "searxng", "docker", "privacy", "homelab", "search"]
authors = ["Experian"]
avatar = "/images/fubar.webp"
+++

# SearXNG: Self-Hosted Search for my Pi Harness

## Overview

My local AI coding assistant, running in the Pi Harness, needs to search the web all the time. For a while it searched through DuckDuckGo, which mostly worked — but sometimes it just fails, because DDG throws CAPTCHAs at automated queries. It's way more forgiving than Google in that regard, but "more forgiving" isn't the same as "reliable."

So I went looking for something better and stumbled upon SearXNG: a metasearch engine that runs on my own network and aggregates multiple search engines through one local API. Now pi's `web_search` tool asks my instance first, and only falls back to DuckDuckGo when something breaks. The privacy is a nice bonus, all those queries stay on my network instead of flowing through someone else's servers. But primarily, the reliability was the reason I built it.

## Architecture

```
[pi AI Agent]
  → web_search tool
    → SearXNG (primary, 5s cap)
      → bing, yandex, arxiv, github, gitlab, arch wiki, annas archive
    → DuckDuckGo Lite (fallback)
```

SearXNG sits on the Tailscale network at `:13900`. No TLS certificates, no reverse proxy — Tailscale already gives me encrypted transport and network isolation. Valkey, a Redis-compatible fork, handles rate limiting. Well, it *can* handle it. I've left the limiter disabled, because this is a single-user homelab and the Tailscale binding already keeps strangers out. Adding rate-limiting complexity for a user count of one seemed like solving a problem I don't have.

## Why SearXNG?

Public search engines are a privacy leak and a single point of failure. Running my own instance fixes both at once:

- **Privacy:** queries don't leave the local network. No tracking, no profiling, no cookies.
- **Aggregation:** one query hits 7 different engines and merges the results.
- **Control:** I can see exactly which engines run, what categories exist, how results are filtered.

## What's Running

### The Engines (7)

| Engine | Shortcut | Categories |
|---|---|---|
| Bing | `!bi` | general, web |
| Yandex | `!yd` | general |
| ArXiv | `!arx` | science |
| GitHub | `!gh` | it, repos |
| GitLab | `!gl` | it, repos |
| Arch Linux Wiki | `!al` | it, software wikis |
| Anna's Archive | `!aa` | files, books |

The API has no `engines` parameter. Want one specific engine? That's what the shortcuts are for — `!gh` in the query hits GitHub, `!arx` hits ArXiv.

### The Engines I Killed

I started with 11 engines. Four of them didn't survive contact with reality:

- **Google** — upstream CAPTCHA-blocks automated instances. Unfixable from config without rotating IPs, proxies, or Tor.
- **Yahoo** — persistent "HTTP protocol error".
- **DuckDuckGo** — persistent "HTTP connection error" on SearXNG's internal API path.
- **Brave** — persistent "too many requests" rate limiting.

There's a joke in here somewhere: I replaced my dependence on DuckDuckGo with a self-hosted instance that can't talk to DuckDuckGo, so I kept DDG as the fallback for my SearXNG instance that can't reach DDG. Whatever. It works, and the fallback path exists precisely because engines are unreliable.

### The pi Integration

- **SearXNG first** — aggregated results from all engines through one local API
- **5-second timeout** — if SearXNG is down or slow, it fails fast
- **DDG fallback** — DuckDuckGo Lite kicks in automatically on any SearXNG failure
- **Category filtering** — the `categories` parameter targets specific engine groups (`general`, `it`, `science`, `files`/`books`)
- **Backend control** — a `backend` parameter lets the agent force DuckDuckGo when SearXNG results are weak, or force SearXNG with no fallback (`auto`/`searxng`/`ddg`)
- **Lossless pagination** — a stride-40 offset mapping handles SearXNG's variable page sizes

## Technical Challenges & Solutions

Setting up the container took an evening. Making it actually *good* took a bit more than that. The whole thing — deployment, integration, debugging, runbook — took two days, most of it spent fighting engines. Here's what fought back.

### Every Tutorial Is Outdated

I started by looking up how everyone else deploys SearXNG. Bad move. The repo every tutorial points to — `searxng-docker` — was archived in March 2026. The official way now is "compose instancing": pulling templates straight from the SearXNG repo's `container/` directory. Morty and Filtron are gone too; Valkey is the new rate-limiting backend. Step one of the internet's instructions was already wrong.

### The Engine Whitelist That Wasn't

I set `keep_only` to whitelist 11 engines — cleaner than disabling ~230 of them one by one. Then SearXNG returned nothing. Turns out `keep_only` only filters which engines are *available*; it doesn't touch their `inactive`/`disabled` state. Google is `inactive: true` by default. Several others — Bing, Yahoo, Yandex, Anna's Archive, GitLab — are `disabled: true`. I was whitelisting engines that were already turned off. The fix was explicitly setting `inactive: false` and `disabled: false` for every engine I actually wanted.

### Google's CAPTCHA Wall

Google serves CAPTCHAs to automated SearXNG instances. SearXNG's handler suspends the engine for 24 hours (`SearxEngineCaptcha: 86400`). There's no config-level fix — it needs rotating IPs, proxies, or Tor, and I wasn't about to build that for one search engine. So I removed it from the list entirely. It's too bad, but so far the other six make up for it

### Anna's Archive Is a Moving Target

Anna's Archive rotates its domains constantly to stay ahead of blocking, and the engine fails silently without a `base_url` configured. The config key has a trap in it: `keep_only` matches the engine's `name:` field — `annas archive`, with a space — not the `engine:` field, `annas_archive`, with an underscore. Use the wrong one and the engine gets pruned silently. It doesn't even appear in `/config`. As of August 2026, `.gl`, `.gd`, and `.li` are reachable; `.org` and `.se` are down. The engine accepts a list of domains and rotates through them per request. I gave it a failover list and moved on.

### Pagination Drift

SearXNG's page sizes vary wildly, page 1 comes back with ~29 results, later pages shrink to ~10 as engines suspend mid-session. A naive offset mapping drifts: the same results come back renumbered, or pages get skipped. I tested this for a while and never saw a single page return more than 40 results, so I simply set the stride to 40. Should be enough.

## Lessons Learned

1. **Self-hosting isn't "install and forget."** Engines degrade over time — CAPTCHAs, rate limits, domain rot. Within a day, four of my eleven engines were dead or dying. That's why the extension now surfaces `unresponsiveEngines` in its results. Though even that has a blind spot: a suspended engine doesn't error, it just returns zero results silently, and `unresponsive_engines` won't list it. The only way to spot a non-contributing engine is the per-result `engines` field.

2. **The `categories` parameter is essential.** Without it, uncategorized searches only hit bing+yandex. Code and science queries got poor results until the agent could opt into `it` or `science`.

3. **Tailscale is a homelab cheat code.** Encrypted transport, DNS, network isolation — no TLS certificates, no reverse proxy, no port forwarding. I planned an entire reverse-proxy step and then deleted it.

4. **Fail fast, fail gracefully.** The 5-second cap means a down SearXNG instance doesn't stall the agent, it falls back to DDG.
