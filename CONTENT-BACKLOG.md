# Content Backlog — Network Authority

Deduplicated against what's already shipped, resequenced to fix one internal contradiction in the source material (see note below), and prioritized to close the two empty category gaps the homepage now visibly admits to ("posts coming soon").

**Already live:** Fixing BGP Flapping Over AWS Direct Connect (cloud-networking) — removed from this list.

## Sequencing note

The source material's own monetization framework says explicitly: don't lead with the "journey to Team Lead" post — good for credibility, not for revenue. Its own 30-day roadmap and "first 5 posts" list then puts that exact post as the very first thing to publish. That's a direct contradiction, not a stylistic difference. Fixed below: the leadership piece is sequenced *after* three technical-authority posts establish expertise, not before.

## Priority order — next 6

1. **Automating Cisco IOS-XE Configuration Backups with Nornir and Git** — `automation`
   Target: Network engineers, NetOps teams. Fast to write — `netmiko_config_backup.py` from the script library already does the work; this post is the narrative wrapper around it.

2. **Palo Alto Policy Cleanup: Finding and Removing Unused Rules Safely** — `security`
   Target: Security engineers. Closes the empty Security category. Natural lead-magnet-v2 tie-in (`firewall_audit.py` — not yet built, candidate for script library v2).

3. **Migrating 500 Sites from MPLS to SD-WAN: Lessons Learned from Real Deployments** — `architecture`
   Target: IT Directors, Network Architects. Closes the empty Architecture category.

4. **My Journey from Network Engineer to Team Lead: Lessons I Wish I Learned Earlier** — `leadership`
   Target: Aspiring leads. Credibility piece — now correctly sequenced after technical authority is established, not before.

5. **Azure Virtual WAN Route Propagation Issues: Root Causes and Fixes** — `cloud-networking`
   Target: Azure architects. Second cloud post, broadens beyond AWS.

6. **How to Push Configuration Changes to 500+ Network Devices Safely** — `automation`
   Target: Enterprise Operations. High search intent, low competition.

## Backlog — unordered, pull from here after the above six

**Automation**
- Building a Production-Ready Network Automation Framework with Python, Nornir, and NetBox
- Ansible vs Nornir for Network Automation: What We Use in Production
- GitOps for Network Engineers: Version Control for Infrastructure Changes

**Cloud Networking**
- AWS Transit Gateway vs Traditional MPLS: Cost, Performance, and Design Tradeoffs
- Designing Multi-Cloud Connectivity Between AWS, Azure, and On-Prem
- Troubleshooting Asymmetric Routing in Hybrid Cloud Environments

**Security**
- Zero Trust Migration Strategy for Enterprises with Legacy VPN Infrastructure
- Zscaler Private Access vs Traditional VPN: A Practical Deployment Review
- How to Audit Firewall Rules Across Multiple Vendors Automatically
- Building a Zero Trust Architecture Without Replacing Your Existing Firewalls

**Architecture**
- Cisco Viptela vs Fortinet SD-WAN: What Actually Matters in Production
- SD-WAN Design Mistakes That Cause Performance Problems at Scale

**Leadership**
- Managing Engineers Smarter Than You: A New Team Lead's Guide

## On "pillar articles" — sequencing correction

The source material suggests writing 3000-5000 word "Complete Guide" pillar pages per category *first*, each linking out to cluster posts. Don't build those yet — a pillar page that links to nothing is a hub with no spokes. Write the cluster posts in each category first (the backlog above); once a category has 3-5 published posts, *then* the pillar/hub page linking to all of them earns its place. Building it now would mean either padding it with generic content to hit a word count, or fabricating depth that isn't backed by published work yet — both work against the actual differentiator (real, specific, working-code content vs. vendor-doc padding).
