# LIOR — Service Delivery Index
> Version 1.1 — June 2026

## Fichiers de référence (lire EN PREMIER)

| Fichier | Contenu |
|---------|---------|
| `TOOLS.md` | Comment utiliser chaque outil : API calls complets, auth setup, polling, download, web UI fallback. ImgBB · Virtual Staging AI · REimagineHome · Runway · Firefly · Replicate SDXL · RoomSketcher · FFmpeg · Pandoc · ImageMagick · Lightroom · WeTransfer · CallMeBot · ZIP |
| `PROTOCOLS.md` | Livraison A→Z (script complet) · Templates WhatsApp par service · Fallbacks par outil · Checklist de livraison universelle |

**Tout agent qui exécute un SOP LIOR doit avoir lu TOOLS.md et PROTOCOLS.md avant de commencer.**

---

All production SOPs for LIOR services. Each file = one service = one autonomous agent execution path.

---

## Service Map

| # | Service | Category | Timeline | SOP File |
|---|---------|----------|----------|----------|
| 01 | Virtual Staging | Agencies & Developers | 48h | `sop-01-virtual-staging.md` |
| 02 | Full Listing Pack | Agencies & Developers | 72h | `sop-02-full-listing-pack.md` |
| 03 | Space Preview | Agencies & Developers | 5 days | `sop-03-space-preview.md` |
| 04 | Interior Design Brief | Property Owners | 4 days | `sop-04-interior-design-brief.md` |
| 05 | Full Interior Design | Property Owners | 2–4 weeks | `sop-05-full-interior-design.md` |
| 06 | Property Management | Property Owners | Ongoing | `sop-06-property-management.md` |
| 07 | Space Planning | Space & Architecture | 48h | `sop-07-space-planning.md` |
| 08 | Architectural Visualization | Space & Architecture | 5 days | `sop-08-architectural-visualization.md` |
| 09 | Design Concept | Space & Architecture | 4 days | `sop-09-design-concept.md` |
| 10 | Pre-Renovation Brief | Space & Architecture | 4 days | `sop-10-pre-renovation-brief.md` |
| — | Studio Partner | Agency Retainer | Monthly | `sop-studio-partner.md` |

**Cinematic Tour production** → see `../../00-system/SOP/sop-eggsfield.md` (v2.1)

---

## Automation Levels

| Level | Meaning | Services |
|-------|---------|---------|
| **A — Full auto** | Agent executes end-to-end, zero human intervention | 01, 07 |
| **B — Auto + human gate** | Agent does all production, human approves before delivery | 02, 03, 09, 10 |
| **C — Guided** | Agent handles intake, production, coordination; human leads client sessions | 04, 05, 08 |
| **D — Managed** | Agent handles reporting and logistics; human drives relationship | 06, Studio Partner |

---

## Common Protocols (apply to all services)

### Client Intake
Every project starts with an intake form. Agent sends the link, client fills it. No production starts before intake is complete.

Intake form: `questionnaire.html` (site)
Intake data fields: name, company, property address, service requested, rooms/scope, style preferences, delivery deadline, WhatsApp number

### Brief Validation Gate
Before starting any production, agent checks:
- [ ] All required inputs received (photos, video, floor plan — per SOP)
- [ ] Style brief confirmed (or defaults applied per property type)
- [ ] Project ID assigned: `LIOR-[CITY-CODE][YEAR][SEQ]` e.g. `LIOR-DM2601`

If any input is missing → send specific request to client WhatsApp via CallMeBot → log → wait → do not start production.

### Revision Policy
- Revisions included in all services — no extra charge, no extended timeline
- Revisions requested within 48h of delivery → handled within 24h
- Revisions after 48h → scheduled as next available slot
- Scope creep (new rooms, new service elements) → new brief

### Delivery Protocol
1. Package all files in named ZIP: `[PROJECT-ID]-[SERVICE-CODE]-v[N].zip`
2. Upload to WeTransfer (7-day link)
3. Send link via WhatsApp (CallMeBot) with one-line delivery note
4. Log delivery in Notion Leads DB: status → Delivered, delivery date

### File Naming Convention
```
[PROJECT-ID]-[ASSET-TYPE]-[ROOM/VARIANT]-v[N].[ext]
e.g.:
LIOR-DM2601-staged-living-v1.jpg
LIOR-DM2601-cinematic-v1.mp4
LIOR-DM2601-social-reel-v1.mp4
LIOR-DM2601-concept-book-v1.pdf
```

### Communication Rules
- All client comms via WhatsApp (CallMeBot API)
- No email unless client requests it
- Language: English (default), French on request
- No "AI", no "drone", no "FPV" in any client-facing copy
- No prices in any public-facing message
- No "luxury" — use: "premium", "high-end", "exceptional"

### Notion Logging
Update `NOTION_LEADS_DB_ID` at each project milestone:
- `Intake received`
- `Production started`
- `QA passed`
- `Delivered`
- `Revision requested`
- `Revision delivered`
- `Closed`

---

## Tools Reference

| Tool | Use | Access |
|------|-----|--------|
| Apply Design API | Virtual staging (empty → furnished) | `POST https://api.applydesign.ai/v1/stage` |
| Runway Gen-3 API | Image-to-video animation | `POST https://api.dev.runwayml.com/v1/image_to_video` |
| FFmpeg | Video processing, color grade, export | CLI — Bash |
| Adobe Firefly API | Image generation (primary — commercial-safe) | `POST https://firefly-api.adobe.io/v3/images/generate` |
| Replicate SDXL + ControlNet | Architectural renders, moodboards (Firefly fallback) | `POST https://api.replicate.com/v1/predictions` |
| RoomSketcher API | Space planning / floor plan generation | `POST https://api.roomsketcher.com/v2/projects` |
| Photoshop (local) | Retouch, compositing | Desktop / Photoshop Actions |
| Lightroom (local) | Color matching, batch export | Desktop |
| Canva API | Social media format export | `POST https://api.canva.com/rest/v1/designs` |
| CallMeBot | WhatsApp notifications | `GET https://api.callmebot.com/whatsapp.php` |
| Notion API | Project tracking | `NOTION_TOKEN` in `.env` |
| WeTransfer API | File delivery | `POST https://api.wetransfer.com/v2/transfers` |
