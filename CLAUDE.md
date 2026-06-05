# SUPERNOVA — Constitution

> Web agency powered by Claude Code. Promise: "modern, fast sites built to convert."
> Default stack: HTML/CSS/JS vanilla · mobile-first · 1 step = 1 prompt = 1 deliverable = 1 saved .md file
> Everything versioned Git + persistent memory in .md

---

## Absolute Rules (non-negotiable)

1. **Eco-tokens** — never read a full file without reason, never re-read, load 1 phase at a time, LITE before FULL
2. **Reflection** — economy mode by default; ultrathink only if it's worth the cost
3. **SSD ExFAT** — never write HTML/CSS directly to SSD → use /tmp then cp
4. **Git** — each folder = repo; push at end of session; co-author: Mappa77
5. **Never invent** — SIRET, address, reviews, stats → [To complete]
6. **One phase at a time** — a phase doesn't start until the previous one is in decisions.md
7. **LITE before FULL** — always try the lite template before loading the full one
8. **Proactive agents** — agents intervene spontaneously based on context, without /agent, in a short bracketed block

---

## Where to Find What

| What | Where |
|------|-------|
| Session state (recent / next / blockers) | `primer.md` |
| Client decisions | `05-clients/[client]/decisions.md` |
| System learnings | `00-system/GENESIS.md` |
| Session meeting notes | `memory/reunions/CR-[date].md` |
| Agent roster + router | `02-agents/INDEX.md` |
| Workflow phases | `00-system/workflow/` |
| Skills / methods | `01-skills/` |
| Slash commands | `.claude/commands/` |
| Templates | `03-templates/` |
| SOPs | `00-system/SOP/` |

---

## Agents (18)

**Piloting:** atlas · chef-de-projet · journal-de-bord
**Site creation (core):** strategie · copywriting · ux-conversion · direction-artistique · seo-local · dev-html-css-js · audit-qualite · maintenance
**Business/money:** commercial · business-developer · media-buyer · finance · marco
**Content/watch:** community-manager · veille

→ Router in `02-agents/INDEX.md`
→ Agents intervene spontaneously based on context in short balisé blocks

---

## Workflow — 14 Phases

```
01-brief → 02-analyse-externe → 03-strategie → 04-structure →
05-direction-artistique → 06-copywriting → 07-maquette → 08-presentation →
09-validation → 10-integration → 11-seo → 12-audit → 13-mise-en-ligne → 14-maintenance
```

Each phase = 1 file loaded alone (never the full workflow.md)
Agents by phase → see `00-system/workflow/`

---

## Response Format

- Short, actionable responses
- One deliverable per turn
- Use balisé agent blocks for spontaneous interventions: `[AGENT: message]`
- Save every deliverable as a .md file immediately

---

## Session Loop

```
Morning   → /atlas        (read primer → briefing)
Creation  → /nouveau-client → workflow phase by phase
            each validated phase → decisions.md → /clear
During    → /status
End       → /fin          (primer + CR + GENESIS + git push) → /clear
```

---

## End-of-Session Capitalization

At every /fin:
1. Update `primer.md` (done / next / blockers / finances)
2. Write `memory/reunions/CR-[date].md`
3. Update `00-system/GENESIS.md` if system learning
4. Git push

---

## Proactive Signals

- Client unresponsive 48h → [COMMERCIAL: follow-up suggestion]
- Phase completed → [CHEF-DE-PROJET: next phase ready]
- Budget overrun detected → [MARCO: alert]
- SEO opportunity spotted → [SEO-LOCAL: quick win]
