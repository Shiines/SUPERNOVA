# SOP EGGSFIELD — Cinematic Property Tour
## Version 2.0 — Ground truth from video analysis

> **Codename:** Eggsfield  
> **Updated:** June 2026 — v2 based on frame-by-frame analysis of the actual `cinematic-tour.mp4`  
> **Purpose:** Exact reproduction protocol for the LIOR cinematic tour. Built for human operators AND for an AI agent that executes it autonomously.

---

## WHAT THIS VIDEO ACTUALLY IS

After analyzing the real `cinematic-tour.mp4` (66s, 1920×1080, 28MB, H.264):

The cinematic tour is a **hybrid video** — not purely AI-generated, not purely filmed. It combines:

| Layer | Source | What it looks like |
|-------|--------|--------------------|
| **Connective tissue** | Real video footage filmed by client (phone walkthrough) | Balcony corridor, entrances, transitions between rooms — raw, naturalistic, moving camera |
| **Room reveals** | Virtual staging stills animated with AI (Runway/Kling) | Each styled room, slow push or pan, warm and polished |
| **Empty contrasts** | Same raw footage — empty rooms visible | Shows the transformation — before each staged reveal |

**Sequence structure observed in the actual video:**
```
00:00 → 00:08  Balcony/corridor walkthrough — real footage, forward dolly, city view
00:08 → 00:14  Living room STAGED — animated virtual staging still, slow push-in
00:14 → 00:20  Living room EMPTY — raw footage of same room, confirms transformation
00:20 → ...    [continues through remaining rooms — same before/after pattern per room]
...  → 01:06   Loop point — last frame matches first for seamless loop
```

**Why this structure works:** The viewer experiences the space as a buyer would — entering from outside, seeing each room staged at its best, with the empty contrast reinforcing the value of the staging. The loop creates an endless preview that works as a website hero background.

---

## I. INPUTS REQUIRED

**All inputs must be confirmed before production begins. No substitutions.**

| # | Input | Format | Who provides | Notes |
|---|-------|--------|--------------|-------|
| 1 | **Raw walkthrough video** | MP4 or MOV, any resolution ≥720p | Client | Phone camera acceptable. Must walk through every room + outdoor areas. Do NOT stabilize — the natural movement is intentional. Min duration: 3–5 min raw. |
| 2 | **Virtual staging photos** | JPG or PNG, ≥2000px wide | LIOR (Phase 2B) | One per room, all staging variants complete before cinematic tour starts. Produced in Phase 2B from client empty-room photos. |
| 3 | **Property brief** | Text | Client | Property type, location, # rooms, target buyer profile |
| 4 | **Room shot list** | Text | Agent derives from raw video review | List of which rooms appear in raw footage, in what order |
| 5 | **Style keyword** | One word | Client or LIOR default | warm / cool / dramatic / minimal. Default: warm |
| 6 | **Project ID** | Text | LIOR | e.g. LIOR-DM001 |

**If raw walkthrough video is missing → client must film it. Send them the filming brief (Annex A).**

---

## II. OUTPUT SPECIFICATIONS

| File | Resolution | Aspect | Duration | Use |
|------|-----------|--------|---------|-----|
| `[ID]-cinematic-v[N].mp4` | 1920×1080 | 16:9 | 55–70s | Website hero, seamless loop |
| `[ID]-cinematic-reels-v[N].mp4` | 1080×1920 | 9:16 | 30–45s | Instagram Reels, TikTok |
| `[ID]-cinematic-sq-v[N].mp4` | 1080×1080 | 1:1 | 30–45s | Instagram feed |

**Technical specs:**
- Codec: H.264, AAC stereo
- Bitrate: 3–4 Mbps (16:9 hero) — matches reference video at 3.4 Mbps
- Frame rate: **24fps** — always. Never 25, never 30.
- File size target: 20–35MB for hero (seamless loop on mobile without buffering)
- First and last frame must be tonally identical (loop continuity)
- Audio: –14 LUFS, –1 dBTP ceiling

---

## III. PRODUCTION PIPELINE — 7 PHASES

**Total time: 4–6 hours for a 5-room property. Always completable within 24h.**

---

### PHASE 0 — FILE SETUP (10 min)

```
Project folder structure:
.tmp/[PROJECT-ID]/
├── 00-brief.txt
├── 01-raw-footage/       ← client walkthrough video(s), untouched
├── 02-raw-cuts/          ← trimmed clips extracted from raw footage
├── 03-staging-stills/    ← LIOR virtual staging photos, renamed by room
├── 04-animated-clips/    ← Runway/Kling output per room
├── 05-assembly/          ← edit timeline files
├── 06-audio/             ← licensed music track
├── 07-exports/           ← final deliverables
└── log.txt               ← every decision, timestamp, issue
```

Rename staging stills:
```
01-living-staged.jpg
02-master-staged.jpg
03-kitchen-staged.jpg
04-ensuite-staged.jpg
05-balcony-staged.jpg
(etc.)
```

---

### PHASE 1 — RAW FOOTAGE REVIEW (20 min)

Watch the full client walkthrough video once, no editing. Build a **shot inventory** in `log.txt`:

```
RAW FOOTAGE INVENTORY
Total duration: XX:XX
File: [filename]

Timecodes:
00:00–00:12  Entrance / front door approach
00:12–00:28  Corridor to living room
00:28–01:05  Living room (full pan) — KEEP
01:05–01:18  Living room to kitchen transition
01:18–01:52  Kitchen (wide + detail) — KEEP
01:52–02:10  Hallway
02:10–02:45  Master bedroom — KEEP
02:45–03:00  Ensuite door/entry
03:00–03:28  Ensuite — KEEP
03:28–04:02  Balcony walkthrough — KEEP (OPENING SHOT)
04:02–04:15  Return to entrance — LOOP CANDIDATE
```

**Identify:**
- **Opening shot** — must be a MOVING exterior/balcony/entrance shot. Not a static interior. The viewer's first frame should establish location and create the sensation of arrival.
- **Room shots** — one clean pass per room, 8–15s usable per room
- **Loop candidate** — a shot that can return to the opening visually

**Do NOT use shaky, overexposed, or corridor-only footage as room hero shots. Those become B-roll connectors only.**

---

### PHASE 2 — RAW FOOTAGE EXTRACTION (20 min)

Extract clean clips from raw footage using FFmpeg. Each clip = one scene. Trim to the usable window identified in Phase 1.

```bash
# Extract a clip: start at 00:28, duration 37 seconds
ffmpeg -i 01-raw-footage/walkthrough.mp4 \
  -ss 00:00:28 -t 00:00:37 \
  -c:v copy -c:a copy \
  02-raw-cuts/living-empty.mp4

# Extract balcony opening shot
ffmpeg -i 01-raw-footage/walkthrough.mp4 \
  -ss 00:03:28 -t 00:00:34 \
  -c:v copy -c:a copy \
  02-raw-cuts/00-balcony-open.mp4
```

**Speed normalization — raw footage is often filmed at walking pace (slightly fast for cinema):**
```bash
# Slow to 80% speed for cinematic feel (most raw footage needs this)
ffmpeg -i 02-raw-cuts/living-empty.mp4 \
  -vf "setpts=1.25*PTS" \
  -af "atempo=0.8" \
  02-raw-cuts/living-empty-slow.mp4
```

**After extraction, review each clip: does it feel like it's moving at a considered, intentional pace?**
- Too fast (rushed, consumer feel) → apply `setpts=1.4*PTS` (slow to 70%)
- Too slow (boring) → apply `setpts=0.9*PTS` (speed to 110%)
- Target sensation: an elegant, purposeful glide through the space

---

### PHASE 2B — VIRTUAL STAGING PRODUCTION (30–60 min per room)

**Purpose:** Create the furnished room images that become the visual core of the cinematic tour. Each empty room photo → one photorealistic staged interior → fed into Phase 3 for animation.

**Runs in parallel with Phase 2.** While FFmpeg extracts and slows raw cuts, staging can be generated simultaneously. Phase 3 cannot start until Phase 2B is complete.

---

#### STEP 1 — Obtain Empty Room Photos

**Option A — Client provides dedicated property photos (preferred):**
Higher resolution than video frames. Better angles. Request using Annex B photo brief.

**Option B — Extract still frames from raw walkthrough footage:**
Use when client has not provided separate photography. Find the steadiest, widest moment per room in the raw footage.

```bash
# Identify the best frame per room (steady camera, full room visible)
# Play walkthrough.mp4, note timecodes where movement is slowest

# Extract frame — living room example at 00:00:45
ffmpeg -i 01-raw-footage/walkthrough.mp4 \
  -ss 00:00:45 -vframes 1 \
  -q:v 2 \
  03-staging-input/living-room-empty.jpg

# If slightly blurry — try ±0.04s: -ss 00:00:45.04 or -ss 00:00:44.96
```

**Accept or reject: minimum requirements per photo**

| Criteria | Pass | Fail — action |
|----------|------|----------------|
| Resolution | ≥ 1920×1080 | Below 1280×720 → client reshoot |
| Floor plane | Floor clearly visible, unobstructed | Floor cut off or hidden → re-extract |
| Room angle | Corner or doorway diagonal — max floor area visible | Flat against single wall → re-extract |
| Exposure | Balanced — window visible, not blown out | Completely dark or fully blown → re-extract |
| Motion blur | Sharp | Blurred → try adjacent frame ±0.04s |
| Content | Empty or near-empty | Heavy furniture, personal items → flag to client |

---

#### STEP 2 — Select Staging Style

Style choice is a production decision, not a visual preference. Match to property type and buyer profile. A wrong style creates cognitive dissonance for the right buyer and conversions drop.

**Property → Buyer → Style:**

| Property type | Target buyer | Style to select | Density |
|--------------|-------------|-----------------|---------|
| Studio / 1BR compact | Young expatriate, single professional | Modern Minimal — clean lines, monochrome base + one warm accent | Low–Medium |
| 2BR / 3BR mid-tier | International family, relocation buyer | Contemporary Warm — layered neutrals, wood tones, functional | Medium |
| 3BR+ premium (DIFC / Business Bay / Marina) | Executive expatriate, investor | Contemporary International — curated, art pieces, warm stone palette | Medium |
| Penthouse / Villa | HNWI, Gulf buyer | Curated Luxury — statement scale, bespoke texture layering, no showroom-generic | Medium (never cluttered) |
| Off-plan / new build unit | Global investor, yield-focused | Modern International — neutral palette, maximizes perceived space | Low–Medium |

**Color palette — non-negotiable for Dubai market:**
- Base: warm white / cream / light greige — never cold white, never pure grey
- Accent: warm wood, brushed brass, natural stone, soft terracotta
- Textiles: linen, boucle, natural cotton — not velvet overload
- Art on walls: abstract, landscape, or typographic — not figurative, not culturally specific
- **Banned:** heavy ornate motifs / Nordic grey-blue / suburban beige carpet / IKEA aesthetic

---

#### STEP 3 — Generate the Staging

**Primary tool: Apply Design (applydesign.ai)**

1. Go to applydesign.ai → New Project
2. Upload the empty room photo
3. Room type: select from Living Room / Bedroom / Kitchen / Dining Room / Bathroom
4. Style: use the mapping from Step 2
5. Furniture density: **Medium** (default for most Dubai properties — adjust per table above)
6. Color temperature: **Warm**
7. Generate — 60–90s per room
8. Review all 3–4 variants generated
9. Select best → apply QA checklist (Step 4) → accept or regenerate

**API call (for agent automation):**
```json
POST https://api.applydesign.ai/v1/stage
{
  "image_url": "[URL of empty room photo from 03-staging-input/]",
  "room_type": "living_room",
  "style": "contemporary_warm",
  "density": "medium",
  "color_temperature": "warm"
}
```
Response: `{ "staged_url": "...", "generation_id": "..." }`

Save `generation_id` to `log.txt` for traceability.

**Fallback: Virtual Staging AI (virtualstaging.ai)**
Use when Apply Design is unavailable or for a simpler brief.
1. Upload photo → select room type → select style → Generate
2. Download highest resolution output (2048px+)
3. Apply same QA checklist before accepting

**Agent / advanced option: Stable Diffusion via ComfyUI + ControlNet**
Use when full control required, or when agent has a LIOR-trained LoRA loaded.
```
Workflow: Empty room → ControlNet Depth + Lineart → Inpainting mask (furniture zone) → SDXL generate → 2× upscale
ControlNet strength: 0.75 (preserve geometry, allow furniture generation)
Prompt:  "luxury Dubai apartment interior, [style], warm afternoon light, natural materials,
          editorial architecture photography, 8k, photorealistic"
Negative: "cheap, IKEA, cartoon, oversaturated, cold light, plastic, watermark, floating furniture"
```

---

#### STEP 4 — Quality Review (per staging image)

One failure = regenerate or retouch. Do not accept a staging still that fails any structural item.

**Structural integrity — hard failures:**
- [ ] Furniture rests on the floor — no floating objects
- [ ] No furniture clipping through walls or disappearing at edges
- [ ] Scale correct — sofa is sofa-sized, chairs are chair-sized
- [ ] No duplicate identical objects
- [ ] Room proportions preserved — ceiling height, wall positions match empty photo

**Visual quality:**
- [ ] Light direction matches window position in original empty room
- [ ] Shadows consistent with overall light source
- [ ] Original floor material preserved — not re-textured by AI
- [ ] No AI artifacts on walls (strange patterns, ghost edges, warping)
- [ ] Window view preserved — not replaced or distorted by AI

**Style fit:**
- [ ] Style matches brief from Step 2
- [ ] Color palette warm — no cold grey or blue dominant tones
- [ ] Furniture premium — not suburban or budget-feeling
- [ ] Art on walls neutral — no figurative imagery, no logos
- [ ] No visible brand labels on furniture pieces

**Common failures and resolutions:**

| Failure | Resolution |
|---------|-----------|
| Furniture floating above floor | Regenerate — cannot be fixed in post |
| Room over-cluttered | Regenerate with density **Low** |
| Style too suburban or generic | Regenerate with style **Luxury Modern** or **Contemporary International** |
| One piece badly wrong scale | Photoshop Content-Aware Fill over that piece, regenerate region |
| AI warped the floor texture | Photoshop: blend original floor layer over AI (Step 5) |
| Window blown out or replaced | Photoshop: restore original window region from empty photo |
| Color cast too warm/orange | Reduce color temp setting one step; or fix in Lightroom at Step 5 |

---

#### STEP 5 — Retouch (5–10 min per image, when needed)

For minor issues that do not require full regeneration:

**Photoshop retouch workflow:**
```
1. Open staged JPG in Photoshop as base layer
2. Open original empty room photo → Place as separate layer on top
3. Add black layer mask to original layer
4. Use white brush at 100% opacity to selectively restore:
     - Floor: paint over regions where AI re-textured the floor
     - Windows: restore original window view if AI replaced it
     - Walls: remove any ghost edges or AI texture artifacts
5. Flatten all layers
6. Color adjustments:
     Image > Adjustments > Color Balance:
       Midtones: Red +5, Yellow +3
     Curves: light midpoint lift (midpoint up ~8 points)
     Unsharp Mask: 60% / 0.8px / 3
7. Export: File > Export > Export As > JPG, quality 92
```

**Cross-room color matching (multi-room projects):**
After individual retouching, make all rooms feel like they're in the same light:
```
1. Open all staged JPGs in Lightroom
2. Identify the best-looking room as the reference
3. Develop > Copy (White Balance + Tone + HSL) from reference
4. Paste to all other rooms
5. Fine-tune individually — kitchens and bathrooms may need slightly cooler correction
```

---

#### STEP 6 — File and Log

**Save to:**
```
03-staging-stills/
├── 01-living-staged.jpg
├── 02-master-staged.jpg
├── 03-kitchen-staged.jpg
├── 04-dining-staged.jpg          ← only if dining is a separate room
├── 05-ensuite-staged.jpg
├── 06-balcony-staged.jpg         ← only if balcony was staged
└── _inputs/                      ← source empty photos (archive)
    ├── living-empty.jpg
    ├── master-empty.jpg
    └── ...
```

**File specs:**
- Format: JPG, quality 90–95 (never below 85 — compression artifacts are visible when animated)
- Width: minimum 2000px; 2048–3840px preferred (animation needs zoom headroom)
- Color space: sRGB
- Strip metadata before delivery (remove GPS, device info, timestamps)

**Log entry per room (in `log.txt`):**
```
[STAGING] 01-living: style=contemporary_warm | tool=ApplyDesign | gen_id=XXX | retouch=no | approved=YES
[STAGING] 02-master: style=contemporary_warm | tool=ApplyDesign | gen_id=YYY | retouch=minor floor | approved=YES
```

**Phase 2B complete when:** every room in the shot list has one approved staged still filed in `03-staging-stills/`, each passing the full QA checklist. Hand off to Phase 3.

---

### PHASE 3 — STAGING STILLS → ANIMATED VIDEO CLIPS (5–8 min per room)

For each virtual staging still, generate a 6–8 second animated clip using Runway Gen-3.

**Primary tool: Runway Gen-3 Alpha Turbo**

API endpoint: `POST https://api.dev.runwayml.com/v1/image_to_video`

**Standard request:**
```json
{
  "model": "gen3a_turbo",
  "promptImage": "[URL or base64 of staging still]",
  "promptText": "[motion prompt — see library below]",
  "duration": 8,
  "ratio": "1280:768",
  "seed": 42
}
```

**MOTION PROMPT LIBRARY — use exactly as written:**

```
LIVING ROOM (wide, sofa visible):
"Extremely slow cinematic push forward into the living room, natural warm light, shallow depth of field, luxury interior photography, smooth steady camera, no shake, 24fps"

LIVING ROOM (toward window):
"Slow gentle push toward the large windows, warm afternoon light spilling in, elegant furniture in foreground, smooth motion, cinematic real estate"

MASTER BEDROOM:
"Very slow push into bedroom through doorway, soft diffused light on bedding, luxury hotel quality, steady smooth camera, warm tones, cinematic"

KITCHEN (wide):
"Slow pan right across kitchen, warm light on surfaces, island in foreground, smooth tracking motion, architectural photography style"

KITCHEN (island detail):
"Slow tilt down from eye level to kitchen counter detail, then gentle pull back, warm light, luxury residential, smooth"

ENSUITE / BATHROOM:
"Slow reveal push into bathroom, warm soft light on tiles, spa quality, mirror reflection, smooth cinematic movement"

BALCONY / OUTDOOR:
"Slow push forward toward the view, glass railing catching light, city visible beyond, smooth glide, golden hour quality"

DINING ROOM:
"Very slow orbit around dining table, warm pendant light above, elegant place settings, smooth cinematic tracking"
```

**Quality check per clip:**
- Motion feels intentional, not random
- No AI artifacts (morphing walls, dissolving furniture)
- Warmth and light consistent with brief
- Clip plays smoothly at 24fps

**If Runway fails or produces artifacts after 3 attempts → fallback to FFmpeg Ken Burns:**
```bash
# Ken Burns push-in fallback (zoom from 1.00 to 1.12 over 8s at 24fps = 192 frames)
ffmpeg -loop 1 -i 03-staging-stills/01-living-staged.jpg \
  -vf "scale=3840:2160,zoompan=z='min(zoom+0.0006,1.12)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=192:s=1920x1080,fps=24" \
  -t 8 -pix_fmt yuv420p \
  04-animated-clips/01-living-staged.mp4
```

---

### PHASE 4 — ASSEMBLY EDIT (45 min)

**The assembly structure — reproduce the reference video exactly:**

```
SEGMENT          SOURCE              DURATION    NOTES
──────────────────────────────────────────────────────────────────
Opening shot     raw footage         6–8s        Moving exterior/balcony. Sets location.
Room 1 STAGED    animated still      6–8s        First impression of staging quality
Room 1 EMPTY     raw footage cut     3–4s        Short — the contrast, not a long look
Room 2 STAGED    animated still      6–8s
Room 2 EMPTY     raw footage cut     3–4s
Room 3 STAGED    animated still      6–8s
Room 3 EMPTY     raw footage cut     3–4s
...
Final shot       raw footage         4–6s        Return to exterior/balcony for loop
──────────────────────────────────────────────────────────────────
TARGET TOTAL: 55–70 seconds
```

**Key ratios:**
- Staged clips get 2× more screen time than empty cuts
- Opening shot must create a strong sense of place (Dubai context visible — city, sky, scale)
- Empty cuts are SHORT — just enough to register the transformation, not a tour of emptiness

**Transition types:**
- Between staged and empty: **dissolve 0.4s** — brief, not lingering
- Between rooms: **dissolve 0.6s** — slightly longer, creates scene change
- Opening fade-in: **fade from black 0.8s**
- Closing fade-out: **fade to black 1.2s**

**FFmpeg assembly command:**

First, create a concat list of all clips in order:
```bash
# 05-assembly/sequence.txt
file '../02-raw-cuts/00-balcony-open-slow.mp4'
file '../04-animated-clips/01-living-staged.mp4'
file '../02-raw-cuts/01-living-empty-slow.mp4'
file '../04-animated-clips/02-master-staged.mp4'
file '../02-raw-cuts/02-master-empty-slow.mp4'
file '../04-animated-clips/03-kitchen-staged.mp4'
file '../02-raw-cuts/03-kitchen-empty-slow.mp4'
file '../02-raw-cuts/99-balcony-close-slow.mp4'
```

Then concatenate with xfade transitions:
```bash
ffmpeg \
  -i 02-raw-cuts/00-balcony-open-slow.mp4 \
  -i 04-animated-clips/01-living-staged.mp4 \
  -i 02-raw-cuts/01-living-empty-slow.mp4 \
  -i 04-animated-clips/02-master-staged.mp4 \
  -i 02-raw-cuts/02-master-empty-slow.mp4 \
  -i 04-animated-clips/03-kitchen-staged.mp4 \
  -i 02-raw-cuts/03-kitchen-empty-slow.mp4 \
  -i 02-raw-cuts/99-balcony-close-slow.mp4 \
  -filter_complex \
  "[0][1]xfade=transition=dissolve:duration=0.6:offset=6.4[v01]; \
   [v01][2]xfade=transition=dissolve:duration=0.4:offset=13.8[v02]; \
   [v02][3]xfade=transition=dissolve:duration=0.6:offset=17[v03]; \
   [v03][4]xfade=transition=dissolve:duration=0.4:offset=24.4[v04]; \
   [v04][5]xfade=transition=dissolve:duration=0.6:offset=27.6[v05]; \
   [v05][6]xfade=transition=dissolve:duration=0.4:offset=35[v06]; \
   [v06][7]xfade=transition=dissolve:duration=0.6:offset=38[vfinal]" \
  -map "[vfinal]" \
  -r 24 -pix_fmt yuv420p \
  05-assembly/assembled-raw.mp4
```

**Offset formula:** `offset = sum of previous clip durations − sum of previous overlap durations`

After assembly, watch through once and ask: does the **rhythm feel right**? Staged rooms should feel lush and slow. Empty cuts should feel punchy and brief. If a segment drags — trim the source clip. If the transition is jarring — increase dissolve duration.

---

### PHASE 5 — COLOR BRIDGE (20 min)

**The most critical phase.** Raw footage and AI-animated stills look different by default:
- Raw footage: slight blue/grey cast, variable exposure, naturalistic
- AI stills: clean, slightly warm, perfect exposure

The color bridge makes them feel like one continuous world.

**Step 1: Analyze reference color from raw footage (balcony shot)**
```bash
# Extract a frame from the opening shot for reference
ffmpeg -i 02-raw-cuts/00-balcony-open-slow.mp4 -vframes 1 -ss 00:00:02 05-assembly/ref-frame.png
```
Look at this frame. Note: is it warm or cool? Bright or dark? This is the property's natural lighting character.

**Step 2: Apply a unified color grade that bridges both sources**

```bash
# LIOR Signature Grade — warm, cinematic, slightly lifted shadows
ffmpeg -i 05-assembly/assembled-raw.mp4 \
  -vf "
    eq=brightness=0.03:contrast=1.05:saturation=0.88:gamma=0.97,
    colorchannelmixer=rr=1.03:rg=0:rb=0:gr=0:gg=0.99:gb=0:br=0:bg=0:bb=0.91,
    curves=r='0/0 0.25/0.28 0.75/0.80 1/1':g='0/0 0.25/0.26 0.75/0.78 1/0.97':b='0/0 0.25/0.22 0.75/0.72 1/0.91'
  " \
  05-assembly/color-graded.mp4
```

**What this grade does:**
- `eq`: Slight brightness lift, mild contrast, desaturated (editorial, not Instagram), gamma pull toward warm
- `colorchannelmixer`: Boosts red channel, suppresses blue — this is the gold-warmth that bridges raw footage and AI stills
- `curves`: Lifts shadows from pure black (prevents harsh cuts at dark transitions), warm highlight rolloff

**If the grade makes AI clips look oversaturated/orange** → reduce `rr` to 1.01 and `saturation` to 0.90

**If raw footage still looks too cool/flat compared to stills** → increase `rr` to 1.05

The result should make a viewer unable to distinguish "real footage" moments from "animated still" moments by color alone.

---

### PHASE 6 — AUDIO (15 min)

**Music brief for LIOR cinematic tours:**

```
Genre: Neoclassical / Ambient
BPM: 62–76
Instruments: Piano (prominent) + light strings + subtle pad
Mood: Contemplative luxury. Aspirational calm. The feeling of walking into a home you could imagine living in.
Duration: 55–72 seconds (match video)
Reference feel: Nils Frahm "Says" / Ólafur Arnalds "Near Light" — style, not actual tracks
Hard rules:
  - NO drums or percussive elements
  - NO builds or drops
  - NO electronic distortion
  - Must work as a LOOP (end tone compatible with beginning)
Artlist tags: ambient, neoclassical, piano, minimal, luxury, real estate
Epidemic Sound tags: cinematic, ambient, calm, sophisticated
```

**Audio processing:**
```bash
# Trim to video length, fade in 1.5s / fade out 2.5s, normalize
ffmpeg -i 06-audio/music-track.mp3 \
  -af "afade=t=in:st=0:d=1.5, \
       afade=t=out:st=$(echo '[VIDEO_DURATION] - 2.5' | bc):d=2.5, \
       loudnorm=I=-14:TP=-1:LRA=11" \
  -t [VIDEO_DURATION] \
  06-audio/music-ready.aac
```

**Merge with video:**
```bash
ffmpeg -i 05-assembly/color-graded.mp4 -i 06-audio/music-ready.aac \
  -c:v copy -c:a aac -shortest \
  05-assembly/final-with-audio.mp4
```

**Add fade-in from black and fade-out to black:**
```bash
ffmpeg -i 05-assembly/final-with-audio.mp4 \
  -vf "fade=t=in:st=0:d=0.8,fade=t=out:st=[DURATION-1.2]:d=1.2" \
  -c:a copy \
  05-assembly/final-faded.mp4
```

---

### PHASE 7 — EXPORT (15 min)

**Master — website hero (loop-optimized):**
```bash
ffmpeg -i 05-assembly/final-faded.mp4 \
  -c:v libx264 -preset slow -crf 23 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  -pix_fmt yuv420p \
  07-exports/[PROJECT-ID]-cinematic-v1.mp4
```

Target: 20–35MB. If larger → increase crf to 25. If smaller and soft-looking → decrease to 21.

**Reels 9:16 — crop center of frame:**
```bash
ffmpeg -i 05-assembly/final-faded.mp4 \
  -vf "crop=ih*9/16:ih:(iw-ih*9/16)/2:0, scale=1080:1920" \
  -t 45 \
  -c:v libx264 -preset slow -crf 24 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  07-exports/[PROJECT-ID]-cinematic-reels-v1.mp4
```

**Square 1:1:**
```bash
ffmpeg -i 05-assembly/final-faded.mp4 \
  -vf "crop=ih:ih:(iw-ih)/2:0, scale=1080:1080" \
  -t 45 \
  -c:v libx264 -preset slow -crf 24 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  07-exports/[PROJECT-ID]-cinematic-sq-v1.mp4
```

---

## IV. QA CHECKLIST

Run before any delivery. Binary — pass or fail.

**Technical:**
- [ ] Duration 16:9: 55–70s
- [ ] Frame rate: exactly 24fps (`mediainfo` confirms)
- [ ] Resolution: 1920×1080
- [ ] `faststart` moov atom present (streams from first byte on mobile)
- [ ] File size 16:9: 18–38MB
- [ ] Audio integrated loudness: –12 to –16 LUFS
- [ ] No audio click at loop point
- [ ] Plays on iOS Safari (test if possible — most common failure point for autoplay)

**Creative:**
- [ ] Opening shot establishes location clearly (Dubai context visible)
- [ ] At least 3 staged room reveals visible in the tour
- [ ] Each empty cut is shorter than its corresponding staged cut
- [ ] Color grade makes raw + AI content feel unified (same world)
- [ ] Music fades in — no hard start
- [ ] Music fades out before black — no hard cut
- [ ] Loop point: last frame and first frame feel continuous (no visual jump)
- [ ] No AI artifacts (morphing objects, dissolving walls, flickering furniture)
- [ ] Motion is slow throughout — no rushed cuts, no speed variation

**Brand:**
- [ ] No text inside the video (HTML overlay handles all copy)
- [ ] No watermarks in delivery files
- [ ] No voiceover
- [ ] Warm color profile (not blue/grey/cold)
- [ ] 24fps feel — NOT 30fps (30fps looks like a reality TV walkthrough)

**If any item fails → fix and re-run QA. Document fix in log.txt.**

---

## V. AI AGENT EXECUTION GUIDE

For the AI agent that automates this SOP.

**Required tools:**
```
bash / ffmpeg        — video processing, color, export
runway_api           — POST /v1/image_to_video (Runway Gen-3)
file_read / write    — source management
artlist_api          — music search and download (or use pre-licensed library)
mediainfo / ffprobe  — QA verification
```

**Decision tree:**
```
START
│
├─ Raw footage received?
│    No → send Annex A filming brief to client → WAIT → restart
│    Yes → continue
│
├─ Phase 0: create folder structure
│
├─ Phase 1: review raw footage → build shot inventory in log.txt
│
├─ Phase 2 + Phase 2B run in parallel:
│    │
│    ├─ [TRACK A] Phase 2: extract and slow all raw cuts
│    │    For each room: extract, apply setpts=1.25*PTS, verify smooth
│    │
│    └─ [TRACK B] Phase 2B: virtual staging production
│         For each room in shot inventory:
│           → Obtain empty room photo (Option A: client photos | Option B: extract from raw)
│           → Check photo against minimum requirements
│           → Select style from property/buyer decision table
│           → Call Apply Design API with room_type + style + density=medium + temp=warm
│           → Review output against full QA checklist
│           → If fail → regenerate (up to 3 attempts) → if still fail → flag to human
│           → If minor artifact → retouch (Photoshop workflow)
│           → Save to 03-staging-stills/ with standard naming
│           → Log: tool / gen_id / retouch / approved
│
├─ GATE: both Track A and Track B complete before continuing
│
├─ Phase 3: animate each staging still
│    For each still:
│      → call Runway API with matching motion prompt
│      → check output (no artifacts, correct duration, correct motion)
│      → if fail × 3 → use FFmpeg Ken Burns fallback
│      → save to 04-animated-clips/
│
├─ Phase 4: assemble sequence
│    → write sequence.txt
│    → run FFmpeg xfade assembly
│    → verify output plays smoothly, total duration 55–70s
│
├─ Phase 5: color grade
│    → extract reference frame from opening shot
│    → apply LIOR signature grade
│    → verify: AI clips and raw footage feel like same world
│
├─ Phase 6: audio
│    → select music matching brief (from pre-approved library or Artlist)
│    → trim, normalize, fade
│    → merge with video
│    → add fade-in/out
│
├─ Phase 7: export 3 formats
│
├─ QA checklist — all items pass?
│    No → fix failing items → re-run relevant phase → re-QA
│    Yes → deliver
│
END
```

**Agent output at end of each phase:**
```
[EGGSFIELD] Phase X complete: [what was done, duration, any issues]
```

**Agent must STOP and ask human if:**
- Raw footage is too dark/blurry to use (>40% of rooms unusable)
- Client provided no outdoor/balcony footage (opening shot missing)
- Runway API fails on >50% of stills after 3 attempts each
- Music doesn't match brief after 5 searches
- Loop point creates visible jump that cannot be resolved by trim adjustment

---

## VI. ANNEX A — CLIENT FILMING BRIEF

*Send this to the client when raw walkthrough footage is missing.*

---

**How to film your property walkthrough — 5 minutes, any smartphone**

We need a simple video walkthrough of your property before we can create your cinematic tour. No professional equipment needed.

**What to do:**
1. Open your phone camera. Film in landscape (horizontal).
2. Start outside — walk toward the entrance from the street or corridor.
3. Enter the property and walk slowly through every room.
4. For each room: stand in the doorway first, then walk in slowly.
5. Pause briefly at windows and views.
6. End by returning to the entrance or the balcony.

**Important:**
- Film during the day with natural light
- Open all blinds and curtains before filming
- Turn on all interior lights
- Walk slowly — slower than you think is necessary
- Do not talk during filming
- Do not stabilize — hold the phone naturally (not on a tripod)

**Send us:** the raw, unedited video. No filters. No cuts. Just the walkthrough.

---

## VII. ANNEX B — CLIENT PHOTOGRAPHY BRIEF (for Virtual Staging)

*Send this to the client when they can provide dedicated property photos (separate from the walkthrough video). Better input = better staging output.*

---

**How to photograph your property for staging — 10 minutes, any smartphone**

We will create furnished room visuals for each room in your property. To do this we need clear photos of each empty room. The quality of these photos directly affects the result — a few extra minutes here makes a significant difference.

**What to do:**

1. Empty the room completely — remove all personal items, boxes, cables
2. Open all blinds and curtains — maximum natural light
3. Turn on all interior lights
4. Film each room from a corner, not from the center of a wall
5. Hold the phone at chest height (approx 110cm from the floor) — not tilted up or down
6. Take one landscape (horizontal) photo per room
7. For large rooms (living room, master bedroom): take a second photo from the opposite corner

**What to avoid:**

- Do not take photos at night or in low light
- Do not stand flat against one wall — always shoot diagonally from a corner
- Do not tilt the phone up (avoid ceiling-heavy photos)
- Do not use portrait (vertical) orientation
- Do not add filters or edits — send raw photos only

**Rooms we need:**

- [ ] Living room (from the main corner, showing max floor area)
- [ ] Master bedroom (from the door or corner)
- [ ] Kitchen (wide — from the entrance if possible)
- [ ] Dining area (if separate from living)
- [ ] Ensuite / main bathroom
- [ ] Balcony or terrace (optional — only if it will be staged)

**Send us:** original, uncompressed files via WhatsApp (full quality) or WeTransfer.

---

## VIII. REFERENCE MEASUREMENTS (from actual `cinematic-tour.mp4`)

For calibration and QA comparison against the production reference:

```
File:          cinematic-tour.mp4
Duration:      66.15s
Resolution:    1920 × 1080
Codec:         H.264 / MPEG-4 AAC
Total bitrate: 3,586 kbps
Video bitrate: 3,458 kbps
Audio bitrate: 127 kbps
Audio:         Stereo, AAC
File size:     28 MB
Frame rate:    [determined from playback — 24fps target]

Color profile observed:
  Opening (balcony):   R≈87 G≈97 B≈107  → cool-blue naturalistic
  Living staged:       Warm, lifted, golden highlights
  Living empty:        Neutral, slightly grey, high-key

Color grade bridges: cool outdoor raw → warm staged AI → consistent feel throughout
```

---

## VIII. VERSION HISTORY

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | June 2026 | Initial SOP — derived from code analysis only |
| 2.0 | June 2026 | Complete rewrite based on frame-by-frame video analysis. Corrected fundamental premise: hybrid video (raw footage + animated stills), not stills-only. Added Annex A, agent decision tree, reference measurements. |
| 2.1 | June 2026 | Added Phase 2B — Virtual Staging Production. Documents the complete process of creating furnished room images from empty room photos (tool selection, style decision table for Dubai market, Apply Design API, quality checklist, retouch workflow, export specs). Updated agent decision tree with parallel Track A/B execution and staging gate. Added Annex B — client photography brief for staging. |
