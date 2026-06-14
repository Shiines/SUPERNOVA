# SOP HIGGFIELD — Photos → Video (Claude + Higgsfield)
## Version 4.0 — Simple operational workflow

> **Codename:** Higgfield
> **Named after:** higgsfield.ai
> **Updated:** June 2026 — v4.0 based on actual workflow observation
> **Pipeline in one line:** Client photos → Claude + Higgsfield → prompt → AI video → delivery

---

## THE REAL WORKFLOW

```
CLIENT sends empty room photos
        ↓
Open CLAUDE + HIGGSFIELD side by side
        ↓
Paste THE PROMPT into Higgsfield (with client photos uploaded)
        ↓
AI generates the cinematic tour
        ↓
QA → deliver via WeTransfer + WhatsApp
```

**No FFmpeg. No manual assembly. No Apply Design separately. No Runway separately.**
Higgsfield AI handles staging + camera movement + video output in one generation pass.

**Target output:** 60–90 second FPV-style cinematic tour of the property — smooth gliding camera through all rooms, staged interiors, professional feel. Reference: `YTDown_YouTube_FPV-DRONE-Tour-REAL-ESTATE-HOME-STAGING` (saved in Downloads).

---

## STEP 1 — RECEIVE CLIENT PHOTOS (5 min)

**What to request from the client:**
- One photo per room, landscape orientation
- Shot from doorway diagonal or room corner — max floor area visible
- Natural light preferred — all blinds open, all lights on
- Minimum quality: sharp, no heavy shadows

**Rooms needed:**
- [ ] Living room
- [ ] Master bedroom
- [ ] Kitchen
- [ ] Bathroom / Ensuite
- [ ] Balcony or terrace (if available)
- [ ] Entrance / corridor (optional but great for opening shot)

**If photos are low quality:** send Annex A (client photo brief) and wait for reshoots.

**Rename files on receipt:**
```
01-living.jpg
02-master.jpg
03-kitchen.jpg
04-ensuite.jpg
05-balcony.jpg
```

---

## STEP 2 — OPEN CLAUDE + HIGGSFIELD (2 min)

1. Open Claude (claude.ai or Claude Code)
2. Open higgsfield.ai in a second window
3. In Higgsfield: create new project → upload all client photos
4. Select model: **Kling 3.0** (default) or **Seedance 2.0** (when audio sync needed)
5. Have the master prompt ready to paste (see Step 3)

**Higgsfield model selection:**

| Situation | Model |
|-----------|-------|
| Standard property tour (default) | Kling 3.0 |
| Penthouse / ultra premium | Veo 3.1 |
| Needs synced ambient audio | Seedance 2.0 |
| Complex multi-room sequence | Sora 2 |

---

## STEP 3 — PASTE THE PROMPT

**MASTER PROMPT — paste into Higgsfield for every Higgfield production:**

```
Create a cinematic property walkthrough video from these room photos.

STYLE: FPV-style continuous camera movement — smooth, slow, deliberate glide through each space. The viewer should feel like they are floating through the property. No abrupt cuts. No static shots.

STAGING: Each room should appear fully furnished and styled. Interior design aesthetic: contemporary warm — natural wood tones, warm cream and beige palette, linen textiles, brass accents, clean lines. Avoid cold greys, avoid cluttered spaces, avoid IKEA-generic furniture. Premium residential feel.

SEQUENCE: Start with the balcony or entrance (exterior first, establishes location and scale). Then move room by room — living room, kitchen, bedroom(s), bathroom. End with a return to the balcony or a window view.

CAMERA: Slow push-forward through doorways. Gentle pan reveals in larger rooms. Slight upward tilt when approaching windows. Never static. Never rushed. Always smooth.

DURATION: 60–90 seconds total.

LIGHTING: Warm afternoon light. Soft shadows. Windows remain bright but not blown out. Interiors feel naturally lit, not artificial.

OUTPUT: 1920x1080, 24fps, cinematic color grade — warm, slightly lifted shadows, not oversaturated.

Do not add any text, watermarks, or voiceover.
```

**Adapt the prompt per property:**
- Studio / compact → add: "Maximize perceived space — wide angles, camera at low height"
- Villa / penthouse → add: "Grand scale — high ceilings visible, sweeping moves, dramatic reveals"
- Off-plan / no balcony → replace exterior with: "Begin at the entrance door, slow push inside"
- Cool/minimal style brief → replace warm palette with: "Monochrome base — white, light grey, concrete — with single warm wood accent"

---

## STEP 4 — GENERATE (5–10 min per run)

1. Upload all client photos in Higgsfield
2. Paste the master prompt (adapted to this property)
3. Click Generate
4. Wait for Higgsfield to produce the video
5. Review the output

**If the first generation is not right:**

| Issue | Fix |
|-------|-----|
| Camera moves too fast | Add to prompt: "extremely slow, half the normal camera speed" |
| Rooms look empty (no staging) | Add: "All rooms must be fully furnished — no empty surfaces, no bare floors" |
| Staging style wrong (too suburban) | Add: "Avoid suburban furniture. Premium international residential aesthetic only." |
| Color too cold / grey | Add: "Push color temperature warmer — golden hour quality interior light" |
| Dubai context missing in opening | Add: "Opening shot must show the city skyline or the exterior building — Dubai scale visible" |
| Video too short | Add: "Minimum 75 seconds. Spend 8–12 seconds on each major room." |

**Regenerate up to 3 times.** If still not right after 3 → escalate to human review with a screenshot of the issue.

---

## STEP 5 — QA CHECKLIST

Run before any delivery. All items must pass.

**Visual:**
- [ ] Camera movement is smooth throughout — no jumps, no freezes
- [ ] All rooms from the brief are visible
- [ ] Rooms appear furnished and styled (staging applied)
- [ ] Opening shot establishes location (exterior, balcony, or entrance)
- [ ] Color is warm — no cold blue/grey cast
- [ ] Windows visible — not blown out, not pitch black

**Technical:**
- [ ] Duration: 60–90 seconds
- [ ] Resolution: 1920×1080 minimum
- [ ] No watermarks in the video
- [ ] No text overlaid in the video
- [ ] Plays smoothly (no buffering artifact)

**Brand rules — hard no:**
- [ ] Zero "drone" / "FPV" / "AI" in any client message
- [ ] Zero "luxury" — use: "premium", "curated", "high-end"
- [ ] Zero em dash (—) in client copy
- [ ] Zero prices visible anywhere

---

## STEP 6 — EXPORT + DELIVER

**Download from Higgsfield:**
- 16:9 master → `[PROJECT-ID]-cinematic-v1.mp4`
- 9:16 Reels (crop in Higgsfield or CapCut) → `[PROJECT-ID]-cinematic-reels-v1.mp4`
- 1:1 square (crop in CapCut) → `[PROJECT-ID]-cinematic-sq-v1.mp4`

**Deliver:**
1. Upload all 3 formats to WeTransfer
2. Send WhatsApp to client:

```
Hi [Name], your LIOR cinematic tour is ready.

[WeTransfer URL]

Included:
→ 16:9 hero (website, listing portals, email)
→ 9:16 vertical (Instagram Reels, TikTok)
→ 1:1 square (Instagram feed)

All formats ready to publish.
LIOR Visual Studio
```

3. Update Notion CRM: Status → Delivered, paste WeTransfer link

---

## TIMING

| Step | Time |
|------|------|
| Receive + rename photos | 5 min |
| Open Claude + Higgsfield, upload photos | 5 min |
| Paste prompt, generate | 5–10 min |
| QA | 5 min |
| Export + deliver | 10 min |
| **TOTAL** | **30–35 min per property** |

---

## ROLE OF CLAUDE IN THE WORKFLOW

Claude is used to:
1. **Adapt the master prompt** to the specific property brief (style, rooms, buyer profile)
2. **Diagnose quality issues** when regeneration is needed ("the staging looks suburban — what do I add to the prompt?")
3. **Draft the client WhatsApp message** (zero forbidden words, warm tone)
4. **Update Notion** via API call
5. **Future: call Higgsfield via MCP** (higgsfield.ai/mcp) — direct API trigger without opening the browser

---

## ANNEX A — CLIENT PHOTO BRIEF

*Send to client when they ask how to photograph their property.*

---

**LIOR — How to photograph your property (10 minutes)**

1. **Open everything** — all blinds, curtains, shutters. Turn on every light.
2. **Shoot from the corner** — never flat against one wall. Stand in the doorway diagonal or at the room corner to show maximum floor area.
3. **Landscape only** — hold phone horizontal. Never portrait.
4. **Chest height** — phone at about 110cm. Don't tilt up or down.
5. **Empty the room** — remove boxes, personal items, loose cables.

One photo per room. Send via WhatsApp (full quality, not compressed) or WeTransfer.

Rooms needed: Living room · Master bedroom · Kitchen · Bathroom · Balcony (if available)

---

## VERSION HISTORY

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | June 2026 | Initial SOP — Eggsfield codename |
| 2.0 | June 2026 | Hybrid AI/raw video method |
| 2.1 | June 2026 | Added virtual staging phase |
| 3.0 | June 2026 | FPV drone method (wrong — corrected) |
| 3.1 | June 2026 | Three-tool pipeline: Staging + Runway + Higgsfield |
| **4.0** | **June 2026** | **Correct workflow: photos → Claude + Higgsfield → prompt → video → delivery. 30-35 min per property.** |
