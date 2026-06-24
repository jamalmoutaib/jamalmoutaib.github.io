# Project: The War Room — by Jamal Moutaib
**Category:** Content / Voice Arena
**Owner:** JAMO only — FAMO has no access to this domain
**Last updated:** 2026-06-24

---

## What This Project Is

A professional technical blog targeting senior network engineers and infrastructure leaders. Content is rooted in real incidents — anonymized, working code, no vendor fluff.

**Brand name:** The War Room — by Jamal Moutaib
**Previous working title:** "Network Authority" (unresolved until 2026-06-24 — now resolved)

## Facts

| | |
|---|---|
| Live URL | https://jamalmoutaib.github.io/ |
| Repo | https://github.com/jamalmoutaib/jamalmoutaib.github.io |
| Local path | `~/Documents/JMO-OS/projects/content/jamalmoutaib` |
| Stack | Hugo + PaperMod + GitHub Actions → GitHub Pages |
| Deploy trigger | Push to `main` |
| Arena | Voice (unified content engine: YouTube · LinkedIn · Blog) |

## Current Status

🟢 **Live** — 4 posts published, lead magnet live, content backlog exists

| Asset | Status |
|---|---|
| Homepage | ✅ Live with category cards and hero copy |
| Lead magnet | ✅ NetOps Script Library zip at `/downloads/netops-script-library.zip` |
| Resources page | ✅ Live — Kit form placeholder in place |
| About page | ✅ Live |

## Published Posts

| Slug | Category | Date |
|---|---|---|
| my-first-post | automation | 2026-06-23 |
| my-journey-from-network-engineer-to-team-lead | career | 2026-06-23 |
| pmtu-blackhole-cisco-ftd | incident | 2026-06-14 |
| fixing-bgp-flapping-aws-direct-connect | cloud-networking | 2026-06-23 |

## Category Status

| Category | Posts | Homepage link |
|---|---|---|
| automation | 1 | ✅ Live → `/categories/automation/` |
| cloud-networking | 1 | ✅ Live → `/categories/cloud-networking/` |
| security | 0 | ⏳ "posts coming soon" — do NOT add link until a post exists |
| architecture | 0 | ⏳ "posts coming soon" — do NOT add link until a post exists |

## Operations

Standing commands and hard rules live in:
**`JAMO-VOICE-BLOG-PLAYBOOK.md`** (project root, same folder as this file)

Key commands:
- `new post [incident|architecture] [slug]` — scaffold only, never write body
- `publish post [slug]` — 4-phase gate: preflight → build → push → live HTTP verify
- `audit blog` — draft/missing front-matter/category count report
- `verify publish [slug]` — run Phase 4 independently after manual push

Hard rules (never violate):
1. Never edit `/public/` directly
2. Never write post body content — scaffold and stop
3. Never report publish success on `git push` alone — must pass live HTTP check
4. Never add a category link to `hugo.toml` homeInfoParams unless a post in that category actually exists in `content/posts/`
5. Never introduce raw HTML `<div>` blocks into `content/*.md` — verify `grep -r "raw HTML omitted" public/` returns clean before committing

## Content Backlog

19 prioritized ideas in `CONTENT-BACKLOG.md` (project root).

**Next 3 to close empty categories:**
1. Automating Cisco IOS-XE Config Backups with Nornir and Git — `automation`
2. Palo Alto Policy Cleanup: Finding and Removing Unused Rules Safely — `security`
3. Migrating 500 Sites from MPLS to SD-WAN — `architecture`

## Monetisation

- **Lead magnet:** NetOps Script Library (live, direct download)
- **Email list:** Kit.com form — placeholder in resources.md, not yet wired
- **Services:** War Room (60 min) + Architecture Review (30 min) — booking link TBD
- **Next unlock:** Wire Kit form → email sequence → soft CTA in posts

## Voice Arena Cross-Reference

This blog is one of three Voice arena outputs:
- **The War Room blog** (this project) — written long-form, SEO, professional authority
- **JMO Senpai** — faceless Darija YouTube channel (`projects/content/jmo-senpai/`)
- **LinkedIn** — professional brand, 1 post/week cadence (tracked in [[Content Creation]])

See [[Content Creation]] for unified Voice arena strategy.
