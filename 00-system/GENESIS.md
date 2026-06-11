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

## 2026-06-12 — Luxury Brand Principle (LIOR)
- **"Hermès ne parle pas du coton"** — a luxury brand never exposes its process or materials. Copy stays at the outcome level only. No technical terms (AI, FPV, drone, algorithm) visible to clients. This applies to any premium B2C or B2B product page.
- **Canvas without libraries** — complex 3D room wireframe + particles built in native canvas via manual perspective projection. No Three.js, no p5.js. Faster load, full control, zero dep risk.
- **Cinematic scroll feel** = 3 layers working together: (1) slower reveal transitions (1.1s+ cubic-bezier), (2) gradient bleed between sections (no hard borders), (3) lerp wheel scroller (desktop). None alone is enough.
- **Portfolio card darkness** — `rgba` backgrounds at 0.10 opacity look nearly black on dark sites. Use 0.28–0.38 for visible warmth. Add warm/cool color variance per card to break monotony.

## Learnings Log

| Date | Learning | Impact |
|------|----------|--------|
| 2026-06-05 | System created | Foundation laid |
| 2026-06-07 | Public surface ≠ internal vault; ship-as-published mindset | IP protected; reusable firewall pattern |
| 2026-06-07 | Asset-light deals: frame upside via counterparty's own revenue | Easier yes, lower-risk partnerships |
| 2026-06-07 | Effort proportional to stakes (not max-always) | Respects eco-tokens while keeping rigor where it counts |
| 2026-06-12 | Luxury copy = outcomes only, never process/materials | Applies to any premium product page |
| 2026-06-12 | Cinematic scroll = slow reveals + gradient bleeds + lerp wheel | All 3 layers needed together |
| 2026-06-12 | Dark card backgrounds need 0.28–0.38 glow opacity, not 0.10 | Visual warmth vs invisible gradient |

