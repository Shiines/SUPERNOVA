# GENESIS — System Learnings

> What has been built, key discoveries, architectural decisions.
> Updated only at /fin when there's a notable system learning.

---

## 2026-06-05 — System Born
- SUPERNOVA system initialized — agency architecture scaffolded
- 14-phase workflow established
- 18 agents defined
- 13 skills defined
- Full folder structure scaffolded

---

## 2026-06-07 — Public/Internal Firewall
- For any client with proprietary IP, the served folder (`docs/`) is a **public surface**: it must never
  contain the secret method/mechanism, legacy names, or price figures — even inside client-side JS or
  "members-only" pages. Everything sensitive lives in a sibling `_internal/` vault, reachable via a private
  noindex HUB. View-source = anyone can read it; treat shipped code as published.
- A high-value deal can be **asset-light**: trade access/value instead of money. Frame the counterparty's
  upside through *their own* P&L (here: hospitality occupancy), not a share of ours.
- Reasoning effort should be **proportional to stakes** — deep on strategy/legal/money/IP, lean on routine.

---

## Learnings Log

| Date | Learning | Impact |
|------|----------|--------|
| 2026-06-05 | System created | Foundation laid |
| 2026-06-07 | Public surface ≠ internal vault; ship-as-published mindset | IP protected; reusable firewall pattern |
| 2026-06-07 | Asset-light deals: frame upside via counterparty's own revenue | Easier yes, lower-risk partnerships |
| 2026-06-07 | Effort proportional to stakes (not max-always) | Respects eco-tokens while keeping rigor where it counts |

