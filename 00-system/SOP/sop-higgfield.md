# SOP HIGGFIELD — Cinematic Property Tour (Full AI Pipeline)
## Version 3.0 — Higgsfield AI + Runway + Virtual Staging

> **Codename:** Higgfield  
> **Named after:** [higgsfield.ai](https://higgsfield.ai) — the AI video platform at the core of the pipeline  
> **Updated:** June 2026 — v3.0 based on @indoordronetours reference + Higgsfield AI integration  
> **Purpose:** End-to-end production protocol for LIOR's cinematic property tours. 100% AI-based — no filming, no drones, no on-site equipment. Client sends photos. LIOR delivers the tour.

---

## WHAT HIGGFIELD IS

A fully AI-generated cinematic property tour (55–70 seconds) that combines:

| Layer | Tool | Output |
|-------|------|--------|
| **Virtual staging** | Apply Design / VStagingAI | Furnished room images from empty photos |
| **Primary animation** | Runway Gen-4 Turbo | Room reveal clips — slow cinematic push-in per room |
| **Advanced animation & openings** | Higgsfield AI (Kling 3.0 / Veo 3.1 / Cinema Studio) | Opening sequence, signature reveals, complex camera moves |
| **Assembly + grading** | FFmpeg + ImageMagick | Final 16:9 / 9:16 / 1:1 export |

**Why three tools instead of one:**
- Runway = stable, reliable, well-tested for slow push-in room animations
- Higgsfield = access to Kling 3.0 (photorealism), Veo 3.1 (4K native), Cinema Studio (real optical physics + stacked camera moves) — used for shots that require more sophistication
- Together: every room animated at the quality level it deserves, not the lowest common denominator

**Client brand rules (non-negotiable in all output):**
- Zero "drone" / "FPV" / "AI" / "artificial intelligence" in any client-facing text
- Zero "luxury" — banned sitewide
- Zero em dash (—) in copy
- Zero prices on public-facing content
- Use: "cinematic tour", "property walkthrough", "immersive visit"

---

## I. INPUTS REQUIRED

| # | Input | Format | Who provides | Notes |
|---|-------|--------|-------------|-------|
| 1 | **Empty room photos** | JPG/PNG ≥ 2000px wide | Client | One per room, shot from doorway diagonal. See Annex B for photo brief. |
| 2 | **Property brief** | Text | Client | Type, # rooms, location, target buyer profile |
| 3 | **Style keyword** | One word | Client or LIOR default | warm / cool / dramatic / minimal — default: warm |
| 4 | **Project ID** | Text | LIOR | Format: LIOR-HF001 |
| 5 | **Raw walkthrough video** *(optional)* | MP4/MOV ≥720p | Client | If client provides it: use as opening shot + transitions. If not: Higgsfield Cinema Studio generates the opening. |

**If raw walkthrough video is missing → proceed with 100% AI. Use Higgsfield Cinema Studio to generate opening and connective shots.**

---

## II. OUTPUT SPECIFICATIONS

| File | Resolution | Aspect | Duration | Use |
|------|-----------|--------|---------|-----|
| `[ID]-cinematic-v[N].mp4` | 1920×1080 | 16:9 | 55–70s | Website hero, listing video |
| `[ID]-cinematic-reels-v[N].mp4` | 1080×1920 | 9:16 | 30–45s | Instagram Reels, TikTok |
| `[ID]-cinematic-sq-v[N].mp4` | 1080×1080 | 1:1 | 30–45s | Instagram feed |

**Technical specs:**
- Codec: H.264, AAC stereo
- Frame rate: **24fps** always
- Bitrate: 3–4 Mbps (16:9 hero)
- File size: 20–35MB hero
- First and last frame: tonally identical (loop continuity)
- Audio: –14 LUFS, –1 dBTP ceiling

---

## III. TOOLS & ENVIRONMENT SETUP

```bash
# Required local tools
which ffmpeg  || brew install ffmpeg
which magick  || brew install imagemagick
python3 -c "import json,subprocess" && echo "python3 OK"

# API keys needed in /Users/cashville/.env
IMGBB_API_KEY=               # https://imgbb.com → Account → API
VSTAGING_API_KEY=            # https://virtualstaging.ai → Settings → API Keys
RUNWAY_API_KEY=              # https://app.runwayml.com → Account → API Keys
WETRANSFER_API_KEY=          # https://developers.wetransfer.com
CALLMEBOT_PHONE=             # your WhatsApp number with country code
CALLMEBOT_API_KEY=           # callmebot.com setup
NOTION_TOKEN=                # https://www.notion.so/my-integrations

# Higgsfield — web UI (higgsfield.ai) + MCP integration
# MCP endpoint: https://higgsfield.ai/mcp
# Claude MCP config: add to ~/.claude.json under mcpServers
# "higgsfield": { "type": "http", "url": "https://higgsfield.ai/mcp" }
```

**Verify all keys before any production:**
```bash
for VAR in IMGBB_API_KEY VSTAGING_API_KEY RUNWAY_API_KEY WETRANSFER_API_KEY CALLMEBOT_PHONE CALLMEBOT_API_KEY NOTION_TOKEN; do
  VAL=$(grep "^${VAR}=" /Users/cashville/.env | cut -d= -f2)
  [ -z "$VAL" ] && echo "  ✗ MISSING: ${VAR}" || echo "  ✓ ${VAR}"
done
```

---

## IV. PRODUCTION PIPELINE — 7 PHASES

**Total time: 4–6 hours for a 5-room property.**

---

### PHASE 0 — SETUP (10 min)

```bash
PROJECT="LIOR-HF001"  # replace with actual ID
mkdir -p .tmp/${PROJECT}/{00-brief,01-raw-footage,02-raw-cuts,03-staging-stills,03-staging-stills/_inputs,04-animated-clips,05-assembly,06-audio,07-exports}
touch .tmp/${PROJECT}/log.txt
echo "[$(date '+%Y-%m-%d %H:%M')] PHASE 0 — Setup complete: ${PROJECT}" >> .tmp/${PROJECT}/log.txt
```

**Rename staging input photos to convention:**
```
03-staging-stills/_inputs/
├── 01-living-empty.jpg
├── 02-master-empty.jpg
├── 03-kitchen-empty.jpg
├── 04-ensuite-empty.jpg
└── 05-balcony-empty.jpg    ← if balcony is being staged
```

---

### PHASE 1 — FOOTAGE REVIEW (20 min)

**If raw walkthrough video received from client:**
- Watch full video once. Build shot inventory in log.txt.
- Identify: opening shot (exterior/balcony/entrance), clips per room, loop candidate.
- Extract and slow each usable clip (see Phase 2 below).
- Opening shot from raw footage → goes first in assembly.

**If no raw footage (100% AI mode):**
- Skip to Phase 2B.
- Opening shot will be generated by Higgsfield Cinema Studio in Phase 3.

```
[PHASE 1] Mode: [raw footage / 100% AI]
Rooms confirmed: [list]
Opening source: [raw footage timecode / Higgsfield Cinema Studio]
```

---

### PHASE 2 — RAW CUTS (20 min, only if raw footage available)

```bash
# Extract opening shot
ffmpeg -i 01-raw-footage/walkthrough.mp4 \
  -ss 00:03:28 -t 00:00:34 \
  -c:v copy -c:a copy \
  02-raw-cuts/00-opening.mp4

# Slow to 80% speed for cinematic feel
ffmpeg -i 02-raw-cuts/00-opening.mp4 \
  -vf "setpts=1.25*PTS" \
  -af "atempo=0.8" \
  02-raw-cuts/00-opening-slow.mp4

# Repeat for each room connector clip and loop return shot
```

---

### PHASE 2B — VIRTUAL STAGING (30–60 min per room)

For each empty room photo → generate a furnished, styled version.

**Primary tool: Apply Design (applydesign.ai)**

1. Upload empty room photo
2. Room type: Living Room / Bedroom / Kitchen / Dining / Bathroom
3. Style — use this table:

| Property type | Target buyer | Style | Density |
|--------------|-------------|-------|---------|
| Studio / 1BR | Young expat, single professional | Modern Minimal | Low–Medium |
| 2BR / 3BR mid-tier | International family | Contemporary Warm | Medium |
| 3BR+ premium (DIFC / Marina / Business Bay) | Executive expat, investor | Contemporary International | Medium |
| Penthouse / Villa | HNWI, Gulf buyer | Curated — statement scale, warm stone | Medium (never cluttered) |
| Off-plan unit | Global investor | Modern International | Low–Medium |

4. Color temperature: Warm
5. Generate → select best of 3–4 variants → QA checklist → approve or regenerate

**Dubai color palette — mandatory:**
- Base: warm white / cream / light greige — never cold white, never pure grey
- Accent: warm wood, brushed brass, natural stone
- Textiles: linen, boucle, natural cotton
- Banned: heavy ornate motifs / Nordic grey-blue / IKEA aesthetic / velvet overload

**API option (VStagingAI):**
```bash
VSTAGING_KEY=$(grep VSTAGING_API_KEY /Users/cashville/.env | cut -d= -f2)
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)

# Upload empty photo to get URL
IMG_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" \
  -F "image=@03-staging-stills/_inputs/01-living-empty.jpg" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['url'])")

# Submit staging job
RESPONSE=$(curl -s -X POST "https://api.virtualstaging.ai/v1/stage" \
  -H "Authorization: Bearer ${VSTAGING_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"image_url\":\"${IMG_URL}\",\"room_type\":\"living_room\",\"style\":\"contemporary\",\"furnishing_level\":\"medium\"}")

JOB_ID=$(echo $RESPONSE | python3 -c "import sys,json; print(json.load(sys.stdin).get('job_id','ERROR'))")
echo "VStagingAI job: ${JOB_ID}" >> 00-brief/log.txt
```

**QA checklist per staging image (hard failures = regenerate):**
- [ ] Furniture rests on the floor — no floating objects
- [ ] No furniture clipping through walls
- [ ] Room proportions preserved — same ceiling height, same windows
- [ ] Light direction matches original window position
- [ ] Original floor material preserved — not re-textured by AI
- [ ] Style matches buyer profile from decision table
- [ ] Color palette warm — no cold grey dominant
- [ ] No AI artifacts on walls (ghost edges, warping)

**Save approved stills to:**
```
03-staging-stills/
├── 01-living-staged.jpg       ← JPG, quality 92, ≥2048px wide, sRGB
├── 02-master-staged.jpg
├── 03-kitchen-staged.jpg
├── 04-ensuite-staged.jpg
└── 05-balcony-staged.jpg
```

---

╔══════════════════════════════════════════╗
║  GATE 1 — STAGING VALIDATION             ║
║  Show all staged images to human         ║
║  Await confirmation before Phase 3       ║
╚══════════════════════════════════════════╝

```
STAGING READY — [PROJECT_ID]
[N] rooms staged:
  01-living : style=[X] | tool=[Y] | retouch=[yes/no] | approved=YES
  02-master : ...
Prêt pour animation. Confirmer pour lancer Runway + Higgsfield ?
```

---

### PHASE 3 — ANIMATION (5–10 min per room)

Two tools used in parallel. Choose per room based on camera move needed.

---

#### TOOL A — RUNWAY GEN-4 TURBO (standard rooms)

Best for: slow push-in, moderate camera movement, consistent with previous projects.

```bash
RUNWAY_KEY=$(grep RUNWAY_API_KEY /Users/cashville/.env | cut -d= -f2)
IMGBB_KEY=$(grep IMGBB_API_KEY /Users/cashville/.env | cut -d= -f2)

# Upload still to ImgBB for public URL
IMG_URL=$(curl -s -X POST "https://api.imgbb.com/1/upload" \
  -F "key=${IMGBB_KEY}" \
  -F "image=@03-staging-stills/01-living-staged.jpg" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['url'])")

# Submit to Runway
curl -s -X POST "https://api.runwayml.com/v1/image_to_video" \
  -H "Authorization: Bearer ${RUNWAY_KEY}" \
  -H "X-Runway-Version: 2024-11-06" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"gen4_turbo\",
    \"promptImage\": \"${IMG_URL}\",
    \"promptText\": \"Extremely slow cinematic push forward into the living room, natural warm light, shallow depth of field, smooth steady camera, no shake, 24fps\",
    \"duration\": 8,
    \"ratio\": \"1280:768\",
    \"seed\": 42
  }"
```

**MOTION PROMPT LIBRARY (Runway — use exactly):**

```
LIVING ROOM:
"Extremely slow cinematic push forward into the living room, natural warm light, shallow depth of field, luxury interior photography, smooth steady camera, no shake, 24fps"

MASTER BEDROOM:
"Very slow push into bedroom through doorway, soft diffused light on bedding, luxury hotel quality, steady smooth camera, warm tones, cinematic"

KITCHEN (wide):
"Slow pan right across kitchen, warm light on surfaces, island in foreground, smooth tracking motion, architectural photography style"

ENSUITE / BATHROOM:
"Slow reveal push into bathroom, warm soft light on tiles, spa quality, mirror reflection, smooth cinematic movement"

BALCONY / OUTDOOR:
"Slow push forward toward the view, glass railing catching light, city visible beyond, smooth glide, golden hour quality"

DINING ROOM:
"Very slow orbit around dining table, warm pendant light above, elegant place settings, smooth cinematic tracking"
```

**Poll and download:**
```bash
# Poll GET /v1/tasks/{task_id} until SUCCEEDED → download output URL
# See Phase 3 full bash script in Annex C
```

**Runway fallback (if FAILED × 3):** Ken Burns FFmpeg zoom
```bash
ffmpeg -loop 1 -i 03-staging-stills/01-living-staged.jpg \
  -vf "scale=3840:2160,zoompan=z='min(zoom+0.0006,1.12)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=192:s=1920x1080,fps=24" \
  -t 8 -pix_fmt yuv420p \
  04-animated-clips/01-living-staged.mp4
```

---

#### TOOL B — HIGGSFIELD AI (premium rooms + opening sequences)

Use for:
- **Opening cinematic sequence** (when no raw footage available)
- **Signature reveal** — the hero room of the property
- **Penthouse / villa rooms** requiring more sophisticated camera movement
- Any room where Runway output quality is insufficient

**Higgsfield model selection guide:**

| Room / Shot type | Recommended model | Why |
|-----------------|------------------|-----|
| Opening establishing shot (exterior / entrance) | **Cinema Studio 3.5** | Virtual camera body + stacked moves + optical physics |
| Hero room (living room / penthouse) | **Kling 3.0** | Best photorealism, advanced motion complexity |
| 4K quality required | **Veo 3.1** | Crystal clear, native cinematic flows |
| Room with complex object interaction | **Sora 2** | Deep world simulation, object permanence |
| Standard rooms (fallback) | **Wan 2.7** | Fast, visually rich, balanced |

**Higgsfield Cinema Studio — workflow for opening shot (no raw footage):**

1. Go to higgsfield.ai → Cinema Studio 3.5
2. Choose shot type: "Establishing Shot" or "Dolly Forward"
3. Upload reference image (staging still of best room, OR property exterior photo if available)
4. Camera settings:
   - Camera body: Sony FX3 or Canon Cinema EOS (cinematic feel)
   - Lens: 24mm or 35mm (wide, flattering for interiors)
   - Movement: slow dolly forward + slight upward tilt
   - Depth of field: shallow
5. Prompt: `"Slow cinematic dolly through luxury Dubai apartment entrance, warm afternoon light, high ceilings, smooth glide, editorial architecture photography, 24fps"`
6. Generate → select best → download

**Higgsfield image-to-video (Kling 3.0 / Veo 3.1):**

1. Go to higgsfield.ai → AI Video → select model
2. Upload staged still as reference image
3. Prompt: use same MOTION PROMPT LIBRARY as Runway (above), but Higgsfield prompts can be more detailed — add specific camera move instructions:
   - `"Starting from doorway, slow push toward large window, camera at eye level, then gentle tilt up to reveal ceiling height, warm light, cinematic real estate, 24fps"`
4. Duration: 6–8 seconds
5. Generate → review → download

**MCP integration (agent automation):**
```
# Higgsfield MCP is available at: https://higgsfield.ai/mcp
# Add to ~/.claude.json:
# "higgsfield": { "type": "http", "url": "https://higgsfield.ai/mcp" }
# Once connected, call Higgsfield tools directly from the Claude agent
# without opening the browser
```

---

**Animation decision tree per room:**

```
For each staging still:
  → Is this the opening shot AND no raw footage? → Higgsfield Cinema Studio
  → Is this the hero/signature room? → Higgsfield Kling 3.0
  → Is standard room, slow push-in sufficient? → Runway Gen-4 Turbo
  → Did Runway fail ×3? → Higgsfield Wan 2.7 (or Ken Burns fallback)
```

**Log per clip:**
```
[PHASE 3] 01-living: tool=Runway gen4_turbo | motion=push-forward | dur=8s | approved=YES
[PHASE 3] 00-opening: tool=Higgsfield Cinema Studio | model=Kling3.0 | dur=7s | approved=YES
[PHASE 3] 02-master: tool=Higgsfield Kling3.0 | motion=dolly-tilt | dur=8s | approved=YES
```

---

### PHASE 4 — ASSEMBLY (45 min)

**Sequence structure:**

```
SEGMENT            SOURCE                     DURATION    NOTES
───────────────────────────────────────────────────────────────────────
Opening shot        Higgsfield Cinema Studio   6–8s        Entrance or best room approach
                    OR raw footage cut
Room 1 STAGED       Runway / Higgsfield        6–8s        First staged reveal
Room 1 EMPTY        Raw footage OR short still 3–4s        Contrast. Short — not a tour.
Room 2 STAGED       Runway / Higgsfield        6–8s
Room 2 EMPTY        [as above]                 3–4s
...
Signature reveal    Higgsfield Kling/Veo       6–8s        Best room — goes last before CTA
Return shot         Raw footage OR Higgsfield  4–6s        Loop back to opening
───────────────────────────────────────────────────────────────────────
TARGET TOTAL: 55–70 seconds
```

**If no raw footage for empty cuts** — skip the empty contrast clips. Sequence becomes:
```
Opening → Room 1 STAGED → Room 2 STAGED → ... → Signature reveal → Return
```
This is the 100% AI version. Slightly shorter (40–55s). Acceptable for social media cuts.

**Calculate xfade offsets dynamically:**
```bash
python3 << 'EOF'
import subprocess

clips = [
  "04-animated-clips/00-opening.mp4",
  "04-animated-clips/01-living-staged.mp4",
  "02-raw-cuts/01-living-empty-slow.mp4",
  "04-animated-clips/02-master-staged.mp4",
  "02-raw-cuts/02-master-empty-slow.mp4",
  "04-animated-clips/03-kitchen-staged.mp4",
  "02-raw-cuts/99-return-slow.mp4",
]
trans = [0.6, 0.4, 0.6, 0.4, 0.6, 0.6]

def dur(f):
  r = subprocess.run(["ffprobe","-v","error","-show_entries","format=duration",
      "-of","default=noprint_wrappers=1:nokey=1",f], capture_output=True, text=True)
  return float(r.stdout.strip())

durations = [dur(c) for c in clips]
total = sum(durations) - sum(trans)
print(f"Clip durations: {[round(d,2) for d in durations]}")
print(f"Estimated total: {round(total,2)}s")
print()
cum = 0
for i,(d,t) in enumerate(zip(durations[:-1], trans)):
  cum += d - t
  print(f"  offset {i+1}: {round(cum,3)}")
EOF
```

**FFmpeg xfade assembly:**
```bash
ffmpeg \
  -i 04-animated-clips/00-opening.mp4 \
  -i 04-animated-clips/01-living-staged.mp4 \
  -i 02-raw-cuts/01-living-empty-slow.mp4 \
  -i 04-animated-clips/02-master-staged.mp4 \
  -i 02-raw-cuts/02-master-empty-slow.mp4 \
  -i 04-animated-clips/03-kitchen-staged.mp4 \
  -i 02-raw-cuts/99-return-slow.mp4 \
  -filter_complex \
  "[0][1]xfade=transition=dissolve:duration=0.6:offset=[CALC][v01]; \
   [v01][2]xfade=transition=dissolve:duration=0.4:offset=[CALC][v02]; \
   [v02][3]xfade=transition=dissolve:duration=0.6:offset=[CALC][v03]; \
   [v03][4]xfade=transition=dissolve:duration=0.4:offset=[CALC][v04]; \
   [v04][5]xfade=transition=dissolve:duration=0.6:offset=[CALC][v05]; \
   [v05][6]xfade=transition=dissolve:duration=0.6:offset=[CALC][vfinal]" \
  -map "[vfinal]" -r 24 -pix_fmt yuv420p \
  05-assembly/assembled-raw.mp4
```
*Replace [CALC] with actual offsets from the python script above.*

---

### PHASE 5 — COLOR GRADE (20 min)

LIOR Signature Grade — bridges raw footage clips and AI-animated stills into one visual world:

```bash
ffmpeg -i 05-assembly/assembled-raw.mp4 \
  -vf "
    eq=brightness=0.03:contrast=1.05:saturation=0.88:gamma=0.97,
    colorchannelmixer=rr=1.03:rg=0:rb=0:gr=0:gg=0.99:gb=0:br=0:bg=0:bb=0.91,
    curves=r='0/0 0.25/0.28 0.75/0.80 1/1':g='0/0 0.25/0.26 0.75/0.78 1/0.97':b='0/0 0.25/0.22 0.75/0.72 1/0.91'
  " \
  05-assembly/color-graded.mp4
```

**What this grade does:**
- Slight brightness lift + contrast pull
- Warm channel boost (red up, blue down) — the gold-warmth signature
- Lifted shadows — prevents harsh black transitions
- Unified: viewer cannot distinguish raw footage from AI-animated clips by color alone

---

### PHASE 6 — AUDIO (15 min)

```
Genre:       Neoclassical / Ambient
BPM:         62–76
Instruments: Piano + light strings + subtle pad
Mood:        Contemplative luxury. The feeling of arriving somewhere beautiful.
References:  Nils Frahm "Says" style / Ólafur Arnalds "Near Light" style — feel, not tracks
Rules:       No drums. No builds. No drops. Must loop cleanly.
Sources:     Artlist (commercial license) / Epidemic Sound (commercial license)
```

```bash
VIDEO_DURATION=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 05-assembly/assembled-raw.mp4)
AUDIO_FADE_START=$(python3 -c "print(round(float('${VIDEO_DURATION}') - 2.5, 3))")

ffmpeg -i 06-audio/music-track.mp3 \
  -af "afade=t=in:st=0:d=1.5,afade=t=out:st=${AUDIO_FADE_START}:d=2.5,loudnorm=I=-14:TP=-1:LRA=11" \
  -t "${VIDEO_DURATION}" 06-audio/music-ready.aac

ffmpeg -i 05-assembly/color-graded.mp4 -i 06-audio/music-ready.aac \
  -c:v copy -c:a aac -shortest 05-assembly/final-with-audio.mp4

FINAL_DUR=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 05-assembly/final-with-audio.mp4)
VFADE_OUT=$(python3 -c "print(round(float('${FINAL_DUR}') - 1.2, 3))")

ffmpeg -i 05-assembly/final-with-audio.mp4 \
  -vf "fade=t=in:st=0:d=0.8,fade=t=out:st=${VFADE_OUT}:d=1.2" -c:a copy \
  05-assembly/final-faded.mp4
```

---

### PHASE 7 — EXPORT (15 min)

```bash
# 16:9 master
ffmpeg -i 05-assembly/final-faded.mp4 \
  -c:v libx264 -preset slow -crf 23 -c:a aac -b:a 128k \
  -movflags +faststart -pix_fmt yuv420p \
  07-exports/[PROJECT-ID]-cinematic-v1.mp4

# 9:16 Reels
ffmpeg -i 05-assembly/final-faded.mp4 \
  -vf "crop=ih*9/16:ih:(iw-ih*9/16)/2:0,scale=1080:1920" -t 45 \
  -c:v libx264 -preset slow -crf 24 -c:a aac -b:a 128k \
  -movflags +faststart \
  07-exports/[PROJECT-ID]-cinematic-reels-v1.mp4

# 1:1 square
ffmpeg -i 05-assembly/final-faded.mp4 \
  -vf "crop=ih:ih:(iw-ih)/2:0,scale=1080:1080" -t 45 \
  -c:v libx264 -preset slow -crf 24 -c:a aac -b:a 128k \
  -movflags +faststart \
  07-exports/[PROJECT-ID]-cinematic-sq-v1.mp4
```

---

## V. QA CHECKLIST (run before any delivery)

╔══════════════════════════════════════════╗
║  GATE 2 — FINAL VALIDATION               ║
║  All items below must pass               ║
║  Await human approval before delivery    ║
╚══════════════════════════════════════════╝

**Technical:**
- [ ] Duration 16:9: 55–70s
- [ ] Frame rate: exactly 24fps
- [ ] Resolution: 1920×1080
- [ ] faststart moov atom present
- [ ] File size 16:9: 18–38MB
- [ ] Audio: –12 to –16 LUFS
- [ ] No click at loop point

**Creative:**
- [ ] Opening shot establishes location (Dubai visible — city, sky, or landmark)
- [ ] At least 3 staged room reveals in the tour
- [ ] Each empty cut (if present) shorter than its staged counterpart
- [ ] Color grade: raw and AI clips feel like the same world
- [ ] Music fades in — no hard start
- [ ] Music fades out before black — no cut
- [ ] Loop point: first and last frame continuous
- [ ] No AI artifacts (morphing, dissolving furniture, flickering)
- [ ] Motion slow throughout — no rushed cuts

**Brand:**
- [ ] Zero text overlaid inside the video
- [ ] Zero watermarks in any file
- [ ] Zero voiceover (unless explicitly requested)
- [ ] Warm palette — not blue/grey/cold
- [ ] 24fps feel — not 30fps

---

## VI. DELIVERY

```bash
# 1. WeTransfer (5-step flow)
# authorize → create → upload each file → upload-complete → finalize → get URL

# 2. WhatsApp via CallMeBot
PHONE=$(grep CALLMEBOT_PHONE /Users/cashville/.env | cut -d= -f2)
APIKEY=$(grep CALLMEBOT_API_KEY /Users/cashville/.env | cut -d= -f2)
MESSAGE="Hi [Name], your LIOR cinematic tour is ready.
16:9 hero, Reels 9:16, and square 1:1 — all in the link.
[WeTransfer URL]
All formats licensed for web, listing portals, and social.
LIOR Visual Studio"
curl -s "https://api.callmebot.com/whatsapp.php?phone=${PHONE}&text=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${MESSAGE}'))")&apikey=${APIKEY}"

# 3. Notion update
curl -s -X PATCH "https://api.notion.com/v1/pages/[PAGE_ID]" \
  -H "Authorization: Bearer $(grep NOTION_TOKEN /Users/cashville/.env | cut -d= -f2)" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{"properties":{"Status":{"select":{"name":"Delivered"}},"Delivery Link":{"url":"[URL]"}}}'

# 4. Log
echo "[$(date '+%Y-%m-%d %H:%M')] DELIVERED [PROJECT_ID] | link=[URL] | WA=sent | Notion=updated" \
  >> .tmp/[PROJECT_ID]/log.txt
```

---

## VII. ERROR PROTOCOLS

| Situation | Action |
|-----------|--------|
| API key empty (401) | Stop immediately. Display missing key name. Wait for human to fill it in. |
| VStagingAI job failed ×3 | Switch to Apply Design web UI. Ask human to provide staged image manually. |
| Runway FAILED ×3 | Use Higgsfield AI (Wan 2.7 or Kling 3.0). If still failing, apply Ken Burns FFmpeg. Log all attempts. |
| Higgsfield generation poor quality | Try different model (Kling → Veo → Seedance). Switch back to Runway if Higgsfield ×3. |
| WeTransfer URL empty after finalize | Re-run the 5-step upload flow once. If still empty → upload manually at wetransfer.com, copy link. |
| Assembly duration out of range (<50s or >80s) | Trim or extend source clips. Do not deliver outside spec. |
| File size >45MB after crf=25 | Drop resolution to 1280×720. Note in delivery log. |
| Color grade makes AI clips oversaturated | Reduce `rr` to 1.01 and `saturation` to 0.90 in grade command. |

---

## VIII. ANNEXES

### Annex A — Client Filming Brief (when they send optional walkthrough)

*Send to client when they can provide a raw walkthrough video.*

**How to film your property — 5 minutes, any smartphone**

1. Film in landscape (horizontal). Start outside — walk toward the entrance.
2. Enter slowly and walk through every room.
3. For each room: stand in the doorway first, then walk in slowly.
4. Pause at windows and views.
5. End by returning to the entrance or balcony.

Important:
- Film during the day with natural light. Open all blinds and curtains.
- Walk slowly — slower than you think is necessary.
- Do not talk. Do not stabilize — hold the phone naturally.
- Send us the raw, unedited file via WeTransfer.

---

### Annex B — Client Photography Brief (for Virtual Staging)

**How to photograph your property — 10 minutes, any smartphone**

1. Empty the room completely — remove all personal items, boxes, cables.
2. Open all blinds and curtains. Turn on all interior lights.
3. Shoot from a corner or doorway diagonal — never flat against one wall.
4. Hold the phone at chest height (approx 110cm). Do not tilt up or down.
5. One landscape photo per room. Two photos for large rooms (living, master).

Avoid: night/low-light shots · portrait orientation · filters or edits · tilted angles.

Rooms needed:
- [ ] Living room
- [ ] Master bedroom
- [ ] Kitchen (wide)
- [ ] Dining area (if separate)
- [ ] Ensuite / main bathroom
- [ ] Balcony (optional)

Send via WeTransfer — original uncompressed files.

---

### Annex C — Full Bash Script: Runway Batch Submit + Poll

*(See previous version of agent-higgfield.md for the complete script — Phase 3 full automation)*

---

## IX. VERSION HISTORY

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | June 2026 | Initial SOP — Eggsfield codename |
| 2.0 | June 2026 | Rewrite — hybrid AI/raw method based on cinematic-tour.mp4 |
| 2.1 | June 2026 | Added Phase 2B virtual staging production |
| 3.0 | June 2026 | **Complete rewrite** — correct three-tool pipeline: Virtual Staging + Runway Gen-4 Turbo + Higgsfield AI (Kling 3.0 / Veo 3.1 / Cinema Studio). Higgsfield used for opening sequences, hero rooms, and premium camera moves. Codename origin: higgsfield.ai. Added 100% AI mode (no client footage required). |
