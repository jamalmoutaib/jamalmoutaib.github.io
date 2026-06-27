# Content Backlog — The War Room

A working queue, not an aspirational list. Every entry is tagged by the one test that determines whether it can become a good War Room post: **do you have a real incident behind it?** The blog's credibility and the `blog-article-pipeline` skill both require lived experience — generic explainers compete head-to-head with vendor docs and lose. So the queue is sorted by writeability, not by topic.

## The filter

- **🟢 WRITEABLE NOW** — you have (or clearly have) a real incident, deployment, or decision behind it. These run cleanly through the pipeline. Pull from here.
- **🟡 NEEDS AN ANGLE** — a real topic, but only worth writing once a specific incident gives you a way in. Don't write the generic version; wait for the war story that teaches it. Retitled below toward the incident, not the textbook.
- **⚪ SOMEDAY / OFF-BRAND** — generic explainer with no incident hook, or content that pulls the brand toward commodity territory (most AI-tooling content). Left here as a map of the territory, deliberately not queued.

No word-count targets anywhere in this file, on purpose. Length is whatever the incident needs. The two times a tool pushed "4,000–6,000 word cornerstone" word counts, it produced padding — the PMTU post is strong because it's tight and real, and the leadership draft got pulled *because* it was padded. Specificity is the quality metric, not length.

---

## 🟢 WRITEABLE NOW — the actual queue

These are the next posts. Ordered to close empty homepage category pillars first, then broaden.

1. **Automating Cisco IOS-XE Configuration Backups with Nornir and Git** — `incident` + `automation`
   Fast to write — `netmiko_config_backup.py` from the script library already does the work; this is the narrative wrapper. Closes the empty Automation pillar.
2. **Palo Alto Policy Cleanup: Finding and Removing Unused Rules Safely** — `incident` + `security`
   Adds a second Security post. Natural lead-magnet-v2 tie-in (`firewall_audit.py`).
3. **Migrating Sites from Legacy MPLS to SD-WAN: Lessons from a Real Cutover** — `architecture` + `architecture`
   Closes the empty Architecture pillar. Write from the actual MA684-RABAT SD-WAN migration you completed — real RFC, real phased cutover, real anonymized.
4. **Azure Virtual WAN Hub-and-Spoke at Multi-Country Scale: What the Rollout Taught Me** — `incident` + `cloud-networking`
   Real EMEA multi-country deployment experience. Broadens cloud beyond the AWS/BGP post.
5. **How to Push Config Changes to Hundreds of Devices Without Taking the Network Down** — `incident` + `automation`
   The snapshot-before-changes / re-read-after-write discipline you already run. High search intent.
6. **GitOps for Network Configs: Version Control the Way a Software Team Would** — `incident` + `automation`
   You run Gitea + this exact workflow. Real setup, real before/after.
7. **My First Years Leading a Network Team: The Decisions I Got Wrong First** — `career`
   The leadership post — but only writeable *with real decisions and real mistakes in it*. This is the bar the pulled draft failed. Not free content; the hardest to do well. Write it when you have the specific stories, not before.

---

## 🟡 NEEDS AN ANGLE — real topic, waiting on a real incident

Don't write the generic version. Each of these is retitled toward the incident that would justify it. Promote to the queue above when you have the war story.

**Troubleshooting (highest fit — these are what the blog is *for*)**

- The MTU/MSS deep-dive — but anchored to a *specific* throughput-collapse incident, not "The Ultimate Guide." (You already have the AutoVPN/TCP-window-scaling case half-drafted — that's the angle.)
- "How Senior Engineers Troubleshoot Packet Loss" → *the* packet-loss incident where the obvious cause was wrong.
- "The Network Is Slow" → the specific time it wasn't the network, and what it actually was.
- VPN troubleshooting → a specific tunnel-down or phase-2 mismatch incident (you have FTD/ASA/Checkpoint material).
- DNS in enterprise networks → "The DNS Failure That Looked Like a Firewall Problem" — incident-led, teaches DNS along the way.
- BGP / OSPF design "beyond certifications" → a specific design decision or failure (the OSPF multipath default-route / `default-information originate` case is exactly this).

**Architecture & Ops (writeable when tied to a real build or a real cleanup)**

- HA firewall design → from a real active/standby or cluster deployment you ran.
- Internet breakout / branch security architecture → from a specific branch design.
- Incident response / RCA methodology / reducing MTTR → from a real postmortem (anonymized).
- Change management, runbooks, SOPs → from the FOUNDEVER-OS operational discipline you actually built.
- Monitoring / alert fatigue / capacity planning → from your real Prometheus/Grafana NOC experience.

**Security (writeable from real audits/migrations)**

- Zero Trust migration from legacy VPN → from a real ZTNA/Zscaler/Prisma migration.
- Multi-vendor firewall rule audit → ties to `firewall_audit.py` once a real audit backs it.
- WebRTC/UDP-blocked-by-firewall class of problem → you have the exact Checkpoint/Twilio TURN incident.

---

## ⚪ SOMEDAY / OFF-BRAND — map only, not queued

Kept for completeness, deliberately not in the queue.

**Generic explainers** (compete with vendor docs, lose without an incident): "Understanding TCP Handshakes Beyond the CCNA," "Why ICMP Should Never Be Blocked" (though the PMTU post already covers this in context), "Load Balancing Traffic Properly," "WAN Optimization Reality," standalone "BGP/OSPF for Enterprise Engineers" without an incident, most pure-concept Phase 2 entries.

**Off-brand / commodity** (pull The War Room toward "another AI-for-X blog" — your edge is incidents, AI is *how you write*, not what you're known for): "AI for Network Engineers," "ChatGPT Prompt Engineering for Networking," "AI-Assisted Troubleshooting," "Building an AI-Enabled NOC," "AI vs Traditional Monitoring." The Python/Netmiko/Nornir/GitOps automation entries are NOT in this bucket — those are real practitioner skills with real incidents behind them, and they live in the queue above.

**Executive/strategy (Phases 7)** — "From Engineer to CTO," "Building Technology Strategy," "Measuring Network Success," etc. Only as good as the real experience in them, and most are years ahead of where the blog's credibility currently sits. Revisit once the technical foundation is deep and the leadership posts have landed.

---

## On "pillar / ultimate guide" articles

Don't build 4,000-word "Complete Guide" hub pages yet — a hub that links to nothing is spokes-less. Write the real-incident cluster posts first; once a pillar has 3-5 published posts, *then* a hub page linking them earns its place. Building it now means padding to a word count or fabricating depth — both work against the differentiator.

## The honest bottleneck

The constraint was never ideas — this file has had 19+ for a while, and a 100-title roadmap doesn't change anything. The constraint is that each good post costs a real incident plus the hours to write it as yourself. A longer list makes the planned-vs-written gap *look* bigger, not smaller. Finish the 🟢 queue before mapping more.
