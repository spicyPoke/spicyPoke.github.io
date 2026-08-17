+++
date = '2026-08-15T12:00:00+07:00'
draft = false
title = 'SearXNG: Self-Hosted Search for my Pi Harness'
description = "A self-hosted SearXNG instance on the homelab that gives the Pi Harness assistant reliable web search, aggregating 7 search engines through a single local API with DuckDuckGo as fallback."

categories = ["projects"]
tags = ["self-hosting", "searxng", "docker", "privacy", "homelab", "search"]
authors = ["Experian"]
avatar = "/images/searxng-icon.svg"
+++

# SearXNG: Self-Hosted Search for my Pi Harness

## Overview

My local AI coding assistant, running in the Pi Harness, needs to search the web from time to time. For a while it searched through DuckDuckGo Lite, which worked most of the time, but not everytime, because DDG throws CAPTCHAs at a  certain threshold of automated queries. It's way more forgiving than Google in that regard, but "more forgiving" isn't the same as "reliable".

So I went looking for something better and stumbled upon SearXNG: a metasearch engine that runs on my own network and aggregates multiple search engines through one local API. Now my Pi's `web_search` tool asks my instance first, and only falls back to DuckDuckGo when something breaks. The privacy is a nice bonus, all those queries stay on my network instead of flowing through someone else's servers. But primarily, the reliability was the reason I built it.

## Architecture

```
[pi AI Agent]
  → web_search tool
    → SearXNG (primary, 5s cap)
      → bing, yandex, arxiv, github, gitlab, arch wiki, annas archive
    → DuckDuckGo Lite (fallback)
```

SearXNG sits on the Tailscale network. So I don't need TLS certificates nor reverse proxy, Tailscale already gives me encrypted transport and network isolation. Valkey, a Redis-compatible fork, handles rate limiting. But, I've left the limiter disabled, because this is a single-user homelab and the Tailscale binding already keeps strangers out. Adding rate-limiting complexity for a user count of one seemed like solving a problem I don't have.

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

The API has no `engines` parameter. To use specific engine we need to pass shortcuts, e.g. `!gh` in the query hits GitHub or `!arx` to hit ArXiv.

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

## Configuration

The deployment is two containers: `searxng-core` (the search frontend and API) and `searxng-valkey` (a Redis-compatible fork for rate limiting). The core container mounts `core-config/` into `/etc/searxng/`, which is where `settings.yml` lives. Valkey gets its own volume for persistence.

I started with the official compose template from the SearXNG repo — the old `searxng-docker` repo is archived, and the new recommended approach is "compose instancing" (pulling the template straight from the SearXNG repo's `container/` directory). The template is minimal: just the two services, two volumes, and a `.env` file for port configuration.

### docker-compose.yml

```yaml
name: searxng

services:
  core:
    container_name: searxng-core
    image: docker.io/searxng/searxng:${SEARXNG_VERSION:-latest}
    restart: always
    ports:
      - ${SEARXNG_HOST:+${SEARXNG_HOST}:}${SEARXNG_PORT:-<searxng port>}:${SEARXNG_PORT:-<searxng port>}
    env_file: ./.env
    volumes:
      - ./core-config/:/etc/searxng/:Z
      - core-data:/var/cache/searxng/

  valkey:
    container_name: searxng-valkey
    image: docker.io/valkey/valkey:9-alpine
    command: valkey-server --save 30 1 --loglevel warning
    restart: always
    volumes:
      - valkey-data:/data/

volumes:
  core-data:
  valkey-data:
```

### settings.yml

The config file does three things: select which engines to expose, configure the server, and define per-engine overrides. The `use_default_settings` block with `keep_only` is a whitelist — but it only controls which engines are *available*, not whether they're active. That's why the `engines:` section at the bottom explicitly sets `disabled: false` for each one. Google ships as `inactive: true` by default, and several others — Bing, Yandex, Anna's Archive, GitLab — ship as `disabled: true`. Whitelisting them without flipping those flags was the first gotcha.

Anna's Archive is the special case: it requires a `base_url` or it fails to load entirely (`ValueError: missing required config base_url`). The default template ships it without one because the domains rotate constantly to stay ahead of blocking. I gave it a list of three reachable domains, and the engine picks one randomly per request for failover.

The `secret_key` was generated with `openssl rand -hex 32` and used for signing cookies and encrypting stored queries. The `image_proxy: true` setting routes image results through the SearXNG server so the user's IP isn't exposed to the original image host. `safe_search: 1` is moderate filtering — mostly relevant for Bing on this instance.

```yaml
use_default_settings:
  engines:
    keep_only:
      - bing
      - yandex
      - arxiv
      - annas archive
      - github
      - gitlab
      - arch linux wiki

server:
  secret_key: "<secret key>"
  image_proxy: true

valkey:
  url: "valkey://searxng-valkey:<valkey port>/0"

search:
  formats:
    - html
    - json
  safe_search: 1

engines:
  - name: bing
    disabled: false
  - name: yandex
    disabled: false
  - name: annas archive
    engine: annas_archive
    shortcut: aa
    base_url:
      - https://annas-archive.gl
      - https://annas-archive.gd
      - https://annas-archive.li
    disabled: false
  - name: gitlab
    disabled: false
```

## Technical Challenges & Solutions

Setting up the container took an evening. Making it actually *good* took a bit more than that. The whole thing — deployment, integration, debugging, runbook — took two days, most of it spent fighting engines. Here's what fought back.

### The Engine Whitelist Confusion

I set `keep_only` to whitelist 11 engines — cleaner than disabling ~230 of them one by one. Then SearXNG returned nothing. Turns out `keep_only` only filters which engines are *available*; it doesn't touch their `inactive`/`disabled` state. Google is `inactive: true` by default. Several others — Bing, Yahoo, Yandex, Anna's Archive, GitLab — are `disabled: true`. I was whitelisting engines that were already turned off. The fix was explicitly setting `inactive: false` and `disabled: false` for every engine I actually wanted.

### Google's CAPTCHA Wall

Google serves CAPTCHAs to automated SearXNG instances. SearXNG's handler suspends the engine for 24 hours (`SearxEngineCaptcha: 86400`). There's no config-level fix — it needs rotating IPs, proxies, or Tor, and I wasn't about to build that for one search engine. So I removed it from the list entirely. It's too bad, but so far the other six make up for it

### Anna's Archive Is a Moving Target

Anna's Archive rotates its domains constantly to stay ahead of blocking, and the engine fails silently without a `base_url` configured. I also found that the config key has a trap in it: `keep_only` matches the engine's `name:` field, `annas archive`, with a space, not the `engine:` field, `annas_archive`, with an underscore. Use the wrong one and the engine gets pruned silently. It doesn't even appear in `/config`. As of August 2026, `.gl`, `.gd`, and `.li` are reachable; `.org` and `.se` are down. The engine accepts a list of domains and rotates through them per request. I gave it a failover list and moved on.

### Pagination Drift

SearXNG's page sizes vary wildly, page 1 comes back with ~29 results, later pages shrink to ~10 as engines suspend mid-session. A naive offset mapping drifts: the same results come back renumbered, or pages get skipped. I tested this for a while and never saw a single page return more than 40 results, so I simply set the stride to 40. Should be enough.

